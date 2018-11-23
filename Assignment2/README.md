# Study on kafka cluster

## How to set up

### Download docker image

> docker pull wurstmeister/zookeeper

> docker pull wurstmeister/kafka

Use command docker images to check whether the two images have been installed

> docker images

### Start

1. start zookeeper

> docker run -d --name zookeeper -p 2181 -t wurstmeister/zookeeper

2. start kafka

```docker
docker run  -d --name kafka \
-p 9092:9092 \
-e KAFKA_BROKER_ID=0 \ 
-e KAFKA_ZOOKEEPER_CONNECT=192.168.204.128:2181 \ 
-e KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://192.168.204.128:9092 \
-e KAFKA_LISTENERS=PLAINTEXT://0.0.0.0:9092 -t wurstmeister/kafka
```

four parameters
```docker
KAFKA_BROKER_ID=0               
KAFKA_ZOOKEEPER_CONNECT=192.168.204.128:2181
KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://192.168.204.128:9092
KAFKA_LISTENERS=PLAINTEXT://0.0.0.0:9092
```
 **NOTICE: 192.168.204.128 must be replaced by your own host ip**

 3. test kafka
 
 enter docker container

 > docker exec -ti kafka /bin/bash

 enter directory of kafka

 > cd opt/kafka_2.12-1.1.0/

 **NOTICE:version may be different**

 4. kafka cluster

 use docker command to set up two or more kafka, only  need to change the **brokerId** and **port**

 ```
 docker run -d --name kafka1 \
-p 9093:9093 \
-e KAFKA_BROKER_ID=1 \
-e KAFKA_ZOOKEEPER_CONNECT=192.168.204.128:2181 \
-e KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://192.168.204.128:9093 \
-e KAFKA_LISTENERS=PLAINTEXT://0.0.0.0:9093 -t wurstmeister/kafka
```

## Producer & Consumer

After testing kafka and set up kafka cluster (more than 2)

 enter docker container

 > docker exec -ti kafka /bin/bash

 enter directory of kafka

 > cd opt/kafka_2.12-1.1.0/

Create topic named mykafka
> ./bin/kafka-topics.sh --create --zookeeper 192.168.204.128:2181 --replication-factor 2 --partitions 2 --topic mykafka

Start a producer

> ./bin/kafka-console-producer.sh --broker-list localhost:9092 --topic mykafka

Start a consumer

> ./bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic mykafka --from-beginning

## Stream Processing

### Overview

Kafka's Streams API is a new feature in Apache Kafka v1.0. The Streams API, available as a Java library that is part of the official Kafka project, is the easiest way to write mission-critical real-time applications and microservices with all the benefits of Kafka’s server-side cluster technology.

Kafka Streams is a library for building streaming applications, specifically applications that transform input Kafka topics into output Kafka topics (or calls to external services, or updates to databases, or whatever). It lets you do this with concise code in a way that is distributed and fault-tolerant. Stream processing is a computer programming paradigm, equivalent to data-flow programming, event stream processing, and reactive programming, that allows some applications to more easily exploit a limited form of parallel processing.

### Demo

A stream processing application built with Kafka Streams looks like this:
```java
import org.apache.kafka.common.serialization.Serdes;
import org.apache.kafka.streams.KafkaStreams;
import org.apache.kafka.streams.StreamsConfig;
import org.apache.kafka.streams.kstream.KStream;
import org.apache.kafka.streams.kstream.KStreamBuilder;
import org.apache.kafka.streams.kstream.KTable;

import java.util.Arrays;
import java.util.Properties;

public class WordCountApplication {

  public static void main(final String[] args) throws Exception {
    Properties config = new Properties();
    config.put(StreamsConfig.APPLICATION_ID_CONFIG, "wordcount-application");
    config.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "kafka-broker1:9092");
    config.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass());
    config.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, Serdes.String().getClass());

    KStreamBuilder builder = new KStreamBuilder();
    KStream<String, String> textLines = builder.stream("TextLinesTopic");
    KTable<String, Long> wordCounts = textLines
        .flatMapValues(textLine -> Arrays.asList(textLine.toLowerCase().split("\\W+")))
        .groupBy((key, word) -> word)
        .count("Counts");
    wordCounts.to(Serdes.String(), Serdes.Long(), "WordsWithCountsTopic");

    KafkaStreams streams = new KafkaStreams(builder, config);
    streams.start();

    Runtime.getRuntime().addShutdownHook(new Thread(streams::close));
  }

}
```

Despite being a humble library, Kafka Streams directly addresses a lot of the hard problems in stream processing:
* Event-at-a-time processing (not microbatch) with millisecond latency
* Stateful processing including distributed joins and aggregations
A convenient DSL
* Windowing with out-of-order data using a DataFlow-like model
* Distributed processing and fault-tolerance with fast failover
* Reprocessing capabilities so you can recalculate output when your code changes
* No-downtime rolling deployments

For those who want to skip the preamble and just dive into the docs, you can just go to the Kafka Streams documentation. The purpose of this blog post will be less the “what”, which the documentation covers in depth, and more the “why”.

