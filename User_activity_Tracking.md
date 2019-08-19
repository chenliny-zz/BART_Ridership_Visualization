# Tracking User Activity

## Assessment Goal:
You work at an ed tech firm. You've created a service that delivers assessments, and now lots of different customers (e.g., Pearson) want to publish their assessments on it. You need to get ready for data scientists who work for these customers to run queries on the data.

Through 3 different activites, you will spin up existing containers and prepare the infrastructure to land the data in the form and structure it needs to be to be queried.
  1) Publish and consume messages with kafka.
  2) Use spark to transform the messages.
  3) Use spark to transform the messages so that you can land them in hdfs.

---------

The Docker cluster contains containers for zookeeper, kafka, and mids with additional installs. I:
- specified container startup dependencies
- exposed and mapped TCP ports for communications between containers and with the virtual machine
- mapped the storage layer to the virtual machine


Create a kafka directory
```
mkdir ~w205/kafka
cd ~/w205/kafka
```

The Docker file
```yml
---
version: '2'
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:latest
    environment:
      ZOOKEEPER_CLIENT_PORT: 32181
      ZOOKEEPER_TICK_TIME: 2000
    expose:
      - "2181"
      - "2888"
      - "32181"
      - "3888"

  kafka:
    image: confluentinc/cp-kafka:latest
    depends_on:
      - zookeeper
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:32181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:29092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
    expose:
      - "9092"
      - "29092"

  cloudera:
    image: midsw205/cdh-minimal:latest
    expose:
      - "8020" # nn
      - "50070" # nn http
      - "8888" # hue
    #ports:
    #- "8888:8888"

  spark:
    image: midsw205/spark-python:0.0.5
    stdin_open: true
    tty: true
    volumes:
      - ~/w205:/w205
    command: bash
    depends_on:
      - cloudera
    environment:
      HADOOP_NAMENODE: cloudera

  mids:
    image: midsw205/base:latest
    stdin_open: true
    tty: true
    volumes:
      - ~/w205:/w205
```

Obtain test data:
```
curl -L -o assessment-attempts-20180128-121051-nested.json https://goo.gl/ME6hjp
```

Spin up containers
```
docker-compose up -d
```

Expected output
```
    Name                        Command            State   Ports
    -----------------------------------------------------------------------
    kafkasinglenode_kafka_1       /etc/confluent/docker/run   Up
    kafkasinglenode_zookeeper_1   /etc/confluent/docker/run   Up
```

Check zookeeper logs
```
docker-compose logs zookeeper | grep -i binding
```

Expected logs output:
```
zookeeper_1  | [2016-07-25 03:26:04,018] INFO binding to port 0.0.0.0/0.0.0.0:32181
(org.apache.zookeeper.server.NIOServerCnxnFactory)
```

Check kafka broker:
```
docker-compose logs kafka | grep -i started
```

Expected kafka logs output:
```

    kafka_1      | [2017-08-31 00:31:40,244] INFO [Socket Server on Broker 1], Started 1 acceptor threads (kafka.network.SocketServer)
    kafka_1      | [2017-08-31 00:31:40,426] INFO [Replica state machine on controller 1]: Started replica state machine with initial state -> Map() (kafka.controller.ReplicaStateMachine)
    kafka_1      | [2017-08-31 00:31:40,436] INFO [Partition state machine on Controller 1]: Started partition state machine with initial state -> Map() (kafka.controller.PartitionStateMachine)
    kafka_1      | [2017-08-31 00:31:40,540] INFO [Kafka Server 1], started (kafka.server.KafkaServer)
```

Create a topic called assessment:
```
docker-compose exec kafka-topics --create --topic assessment --partitions 1 --replication-factor 1 --if-not-exists --zookeeper:32181
```

Check the topic
```
docker-compose exec kafka kafka-topics --describe --topic assessment --zookeeper zookeeper:32181
```

Exec into the mids container and exam test data
```
   16  docker-compose exec mids bash -c "cat /w205/kafka/assessment-attempts-20180128-121051-nested.json | jq '.[]' -c"
```

Publish messages to kafka:
```
docker-compose exec mids bash -c "cat /w205/kafka/assessment-attempts-20180128-121051-nested.json | jq '.[]' -c | kafkacat -P -b kafka:29092 -t assessment && echo 'Produced 100 messages.'"
```

Test consume the messages using kafkacat:
```
docker-compose exec mids bash -c "kafkacat -C -b kafka:29092 -t assessment -o beginning -e"
```

Run pyspark using the spark container
```
docker-compose exec spark pyspark
```

Load messages from kafka to Spark
```
raw_assessment = spark.read.format("kafka").option("kafka.bootstrap.servers", "kafka:29092").option("subscribe","assessment").option("startingOffsets", "earliest").option("endingOffsets", "latest").load()
```

Cache it
```
raw_assessment.cache()
```

Cast key and value into strings
```
assessment = raw_assessment.select(raw_assessment.value.cast('string'))
```

Write to hdfs
```
assessment.write.parquet("/tmp/assessment")
```


Unrolling json
```
import json

extracted_assessment = assessment.rdd.map(lambda x: Row(**json.loads(x.value))).toDF()
```

Create view
```
extracted_assessment.registerTempTable('assessment')
```

DataFrames
```
spark.sql("select keen_id from assessment limit 10").show()
```

DataFrames 2
```
spark.sql("select keen_timestamp, sequences.questions[0].user_incomplete from assessment limit 10").show()
```

Grab data
```
assessment_info = spark.sql("select q.keen_id, a.keen_timestamp, q.id from assessments a join questions q on a.keen_id = q.keen_id limit 10").show()
```

Write to hdfs
```
assessment_info.write.parquet("/tmp/assessment_info")
```

Exit pyspark
```
exit()
```

Take down containers
```
docker-compose down
```
