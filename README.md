# kafka Mirror Maker 2.0

surppose  you are familiar with the use of Kafka Connect, for connecting Kafka cluster with other systems. Here  to show how 
Connector already existed for that type of task. Its name is Mirror Maker 2.0. there was a tool just called Mirror Maker, but it was not part of the Kafka Connect ecosystem and, due to a lot of architectural drawbacks, it was completely rewritten as a plugin for Kafka Connect.

## Ingredients

The best choice for locally deploying all services from this tutorial is to use Docker. So we need Docker installed on our local system. Step by step we will create the docker-compose configuration file. After that by executing only one single command we will get all the services up and running.

Our Docker Compose will deploy these services:

  - Apache Kafka — 2 units
  - Apache Zookeeper — 2 units 
  - Kafka Connect — 1 unit.
Confluent provides very handy conflueninc docker images for the these services, allowing you to configure them only by setting environment variables:
  - confluentinc/cp-zookeeper:6.2.0
  - confluentinc/cp-kafka:6.2.0
  - confluentinc/cp-kafka-connect:6.2.0


```
---
version: '2'
networks:
  kafka_network:
    name: kafka-network
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:6.2.0
    hostname: zookeeper
    container_name: zookeeper
    networks:
      - kafka_network
    ports:
      - "2181:2181"
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
  zookeeper2:
    image: confluentinc/cp-zookeeper:6.2.0
    hostname: zookeeper2
    container_name: zookeeper2
    networks:
      - kafka_network
    ports:
      - "2182:2182"
    environment:
      ZOOKEEPER_CLIENT_PORT: 2182
      ZOOKEEPER_TICK_TIME: 2000
  broker:
    image: confluentinc/cp-kafka:6.2.0
    hostname: broker
    container_name: broker
    networks:
      - kafka_network
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
    environment:
      KAFKA_ZOOKEEPER_CONNECT: "zookeeper:2181"
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://broker:29092
      KAFKA_BROKER_ID: 1
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_AUTHORIZER_CLASS_NAME: kafka.security.auth.SimpleAclAuthorizer
      KAFKA_ALLOW_EVERYONE_IF_NO_ACL_FOUND: "true"
  broker2:
    image: confluentinc/cp-kafka:6.2.0
    hostname: broker2
    container_name: broker2
    networks:
      - kafka_network
    depends_on:
      - zookeeper2
    ports:
      - "9093:9092"
    environment:
      KAFKA_ZOOKEEPER_CONNECT: "zookeeper2:2182"
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://broker2:29093
      KAFKA_BROKER_ID: 2
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_AUTHORIZER_CLASS_NAME: kafka.security.auth.SimpleAclAuthorizer
      KAFKA_ALLOW_EVERYONE_IF_NO_ACL_FOUND: "true"
```

Notice how configuration properties were provided at the “environment” section! Corresponding “KAFKA_” and “ZOOKEEPER_” prefixes were added according to the convention for confluentinc images. Other sections are the standard Docker boilerplate and those for setting the images, ports, networks, dependencies, etc.

Let’s pay attention on the next broker’s environment variables:

KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1 — this is because our cluster consists of only 1 standalone broker, so we can’t replicate any topic.
KAFKA_AUTHORIZER_CLASS_NAME: kafka.security.auth.SimpleAclAuthorizer with
KAFKA_ALLOW_EVERYONE_IF_NO_ACL_FOUND: "true" — for the sake of simplicity in this tutorial I turned off all the security settings. Do not do that in your production environment!
Configuration of the Kafka Connect and deploying services
Let’s take a look at Kafka Connect’s configuration parameters and then use them for our setup:

bootstrap.servers —a list of Kafka bootstrap servers. Here we will set the first broker: broker:29092
rest.port — port exposed by Kafka Connect daemon for the REST requests
group.id — name of Kafka Connect’s cluster. It doesn’t mean anything in this case, because we only have a standalone instance
config.storage.topic, offset_storage_topic, status_storage_topic — names of service topics where Kafka Connect will store configs, offsets, statuses of the connectors and the tasks. Kafka Connect will automatically create them in Kafka Cluster which is specified at bootstrap.servers property
config.storage.replication.factor, offset.storage.replication.factor, status.storage.replication.factor — replication factors of the these topics. Again, we have only standalone brokers, so we will set all of this values to 1
key.converter, value.converter — classes, that will serve for serializing and deserializing the messages. We will turn both of them into org.apache.kafka.connect.converters.ByteArrayConverter
connector.client.config.override.policy — policy for overriding configuration of Kafka Connect by the Connectors. By setting it to All, we can override anything
Kafka Connect confluentinc Docker image should be also configured by using environment variables. According to the convention we should append the following snippet to the docker-compose.yml file:


```  
connect:
    image: confluentinc/cp-kafka-connect:6.2.0
    hostname: connect
    container_name: connect
    networks:
      - kafka_network
    depends_on:
      - zookeeper
      - broker
    ports:
      - "28082:28082"
      - "9998:9998"
      - "9999:9999"
      - "18889:18889"
    environment:
      - CONNECT_BOOTSTRAP_SERVERS=broker:29092
      - CONNECT_REST_PORT=28082
      - CONNECT_GROUP_ID="quickstart"
      - CONNECT_CONFIG_STORAGE_TOPIC=quickstart_config
      - CONNECT_CONFIG_STORAGE_REPLICATION_FACTOR=1
      - CONNECT_OFFSET_STORAGE_TOPIC=quickstart_offsets
      - CONNECT_OFFSET_STORAGE_REPLICATION_FACTOR=1
      - CONNECT_STATUS_STORAGE_TOPIC=quickstart_status
      - CONNECT_STATUS_STORAGE_REPLICATION_FACTOR=1
      - CONNECT_KEY_CONVERTER=org.apache.kafka.connect.converters.ByteArrayConverter
      - CONNECT_VALUE_CONVERTER=org.apache.kafka.connect.converters.ByteArrayConverter
      - CONNECT_REST_ADVERTISED_HOST_NAME="localhost"
      - CONNECT_CONNECTOR_CLIENT_CONFIG_OVERRIDE_POLICY=All
```

(Be careful with an indentations! Configuration snippets from this article are available at my Github Gists)

Now, our services are ready to be deployed! Change the directory withcd, to the directory containing the docker-compose.yml file and execute the following:

```docker-compose up -d```
In a short while, check out docker ps command. It should look like this:


The result of docker ps command
Lets enter Kafka Connect Docker container and take a look at the running processes:

```
docker exec -it connect /bin/bash
ps -efww
```
Here we can see process with PID=1. A huge line starts with java -Xms and end with .../etc/kafka-connect/kafka-connect.properties.
```
cat /etc/kafka-connect/kafka-connect.properties
group.id="quickstart"
status.storage.replication.factor=1
key.converter=org.apache.kafka.connect.converters.ByteArrayConverter
config.storage.topic=quickstart_config
offset.storage.replication.factor=1
plugin.path=/usr/share/java/,/usr/share/confluent-hub-components/
offset.storage.topic=quickstart_offsets
bootstrap.servers=broker:29092
value.converter=org.apache.kafka.connect.converters.ByteArrayConverter
rest.advertised.host.name="localhost"
rest.port=28082
status.storage.topic=quickstart_status
connector.client.config.override.policy=All
config.storage.replication.factor=1
Voila! Our configuration has been propagated correctly. We are going in the right direction!
```
Creating the test topic
We need some topics in the Kafka instance and stream of some data for it. To create it, enter into broker’s container from the new terminal and execute kafka-topics command:

```
docker exec -it broker /bin/bash
kafka-topics --create --topic to_replicate --bootstrap-server broker:29092
```
Using kafka-console-producer start pushing messages into it:

```while [[ true ]]; do echo "$RANDOM" |  kafka-console-producer --topic to_replicate --bootstrap-server broker:29092; sleep 1; done &
Starting the topic’s mirroring```
Let’s return back to our ‘connect’ container

For creating/deleting/modifying/monitoring connectors Kafka Connect provides REST API, nicely described in the docs. For now we should use only /connectorsendpoint.

To add a new connector we have to POST configuration in JSON format. I tried to provide the most minimal configuration for the connector. Create connectors.json file (e.g. via cat command):
```
{
  "name":"test_mirror",
  "config": {
    "connector.class": "org.apache.kafka.connect.mirror.MirrorSourceConnector",
    "name":"test_mirror",
    "source.cluster.alias":"source",
    "topics":"to_replicate",
    "source.cluster.bootstrap.servers":"broker:29092",
    "target.cluster.bootstrap.servers":"broker2:29093",
    "producer.override.bootstrap.servers":"broker2:29093",
    "offset-syncs.topic.replication.factor":"1"
  }
}
```
name — the name of the connector, which should be unique
connector.class — org.apache.kafka.connect.mirror.MirrorSourceConnector for Mirror Maker 2.0 Connector
source.cluster.alias — name of a Kafka source cluster. By default this will become a prefix at the mirrored topic
topics — comma-separated list of topics to be mirrored
source.cluster.bootstrap.servers , source.cluster.bootstrap.servers, producer.override.bootstrap.servers — clusters bootstrap servers. The last one is also required, otherwise mirrored data will be written back into the source cluster
offset-syncs.topic.replication.factor — as with previous replication factors should be set to 1, otherwise offset’s topics will fail during the process of creation
Finally, we are ready to start the the process of mirroring! Preform the POST request for creating a new connector:

```cat connectors.json | curl -X POST -H 'Content-Type: application/json' localhost:28082/connectors --data-binary @-
To check that connector has been created:```

```curl localhost:28082/connectors```
Response should contain an array with our connector’s name.

Validating
Lets visit our second broker from the new terminal:

```docker exec -it broker2 /bin/bash```
and take a look at its topics:

```kafka-topics --list --bootstrap-server broker2:29093```
Here we go: there is a topic with name source.to-replicate. As expected, the name consists of prefix from the property source.cluster.alias plus dot plus source topic name (dot is a default separator, it also can be configured).

Finally we should check that our messages are replicated also. We will use kafka-console-consumerfor this:

```kafka-console-consumer --topic source.to_replicate --bootstrap-server broker2:29093```
After a while you should see random numbers popping up in the console. Congratulation, now you know how to set-up Kafka Connect Mirror Maker 2.0.

Conclusion
Hope this tutorial helped you to start using Mirror Maker. My first time it really was quite a challenge. Sometimes It’s not as straightforward as we may expect.
