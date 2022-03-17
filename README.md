# Apache Flink meets Red Panda
 
I've created this repository in the context of testing the interoperability of Apache Flink and Red Panda. 
In some sense, it is also summarizes the capabilities of Apache Flink's Apache Kafka Connector. 

## Summary

- :heavy_check_mark: Reading & Writing Value *and* Key with different formats  
- :heavy_check_mark: Reading & Writing Metadata (Headers, Offset)
- :heavy_check_mark: `upsert-kafka` & `kafka` Connector
- :x: Writing Exactly-Once Transactions
- :question: Integration with Schema Registry

## Environment Setup

```
docker-compse up -d
```
This starts a local environment running consisting of Red Panda, a Flink Cluster and a SQL Client to submit Flink SQL queries to the Flink Cluster.

``
docker-compose run sql-client
``

Opens the SQL Client.

The Flink UI is available under http://localhost:8081, where you should be able to see the running Jobs and be able to cancel them.

## Example Queries

### Reading & Writing Value and Key

```sql
/* Test Data Generator */
CREATE TABLE animal_sightings (
  `timestamp` TIMESTAMP(3),
   `name` STRING,
   `country` STRING,
   `number` INT
) WITH (
  'connector' = 'faker', 
  'fields.name.expression' = '#{animal.name}',
  'fields.country.expression' = '#{country.name}',
  'fields.number.expression' = '#{number.numberBetween ''0'',''1000''}',
  'fields.timestamp.expression' = '#{date.past ''15'',''SECONDS''}'
);

CREATE TABLE animal_sightings_panda (
  `timestamp` TIMESTAMP(3),
   `name` STRING,
   `country` STRING,
   `number` INT
)WITH (
   'connector' = 'kafka',
   'topic' = 'animal_sightings',
   'properties.bootstrap.servers' = 'redpanda:29092',
   'scan.startup.mode' = 'earliest-offset',

   'key.format' = 'json',
   'key.json.ignore-parse-errors' = 'true',
   'key.fields' = 'name',
   
   'value.format' = 'json',
   'value.json.fail-on-missing-field' = 'false',
   'value.fields-include' = 'ALL'
);

/* Write into Red Panda */
INSERT INTO animal_sightings_panda SELECT * FROM animal_sightings;

/* See if it works by reading from the Red Panda topic */
SELECT * FROM animal_sightings_panda;
```

### Reading Metadata

This assumes that `animal_sightings` is already being written to e.g. by the query from the previous example.

```sql
CREATE TABLE animal_sightings_with_metadata (
  `name` STRING,
  `country` STRING,
  `number` INT,
  `append_time` TIMESTAMP(3) METADATA FROM 'timestamp',
  `partition` BIGINT METADATA VIRTUAL,
  `offset` BIGINT METADATA VIRTUAL,
  `headers` MAP<STRING, BYTES> METADATA,
  `timestamp-type` STRING METADATA,
  `leader-epoch` INT METADATA,
  `topic` STRING METADATA
) WITH (
   'connector' = 'kafka',
   'topic' = 'animal_sightings',
   'properties.bootstrap.servers' = 'redpanda:29092',
   'format' = 'json', 
   'scan.startup.mode' = 'earliest-offset'
);

SELECT * FROM animal_sightings_with_metadata;

```

### Writing & Reading Compacted Topics (with Upsert Semantics)

```sql
/* Test Data Generator */ 
CREATE TABLE animal_sightings (
  `timestamp` TIMESTAMP(3),
   `name` STRING,
   `country` STRING,
   `number` INT
) WITH (
  'connector' = 'faker', 
  'fields.name.expression' = '#{animal.name}',
  'fields.country.expression' = '#{country.name}',
  'fields.number.expression' = '#{number.numberBetween ''0'',''1000''}',
  'fields.timestamp.expression' = '#{date.past ''15'',''SECONDS''}'
);

/* Define Red Panda-backed table for total sightings */ 
 CREATE TABLE animal_sightings_total (
   `name` STRING,
   `total_number` BIGINT,
    PRIMARY KEY (name) NOT ENFORCED
)WITH (
   'connector' = 'upsert-kafka',
   'topic' = 'animal_sightings_total',
   'properties.bootstrap.servers' = 'redpanda:29092',
   'key.format' = 'json',
   'value.format' = 'json'
);

/* Submit query to continuously derive totals */
INSERT INTO animal_sightings_total 
SELECT 
  name, 
  SUM(number) AS total_number
 FROM 
   animal_sightings
 GROUP BY 
  name;

/* Read totals from Red Panda */
SELECT * FROM animal_sightings_total;
```

### Writing Exactly-Once to Red Panda with Transactions (fails)

```sql
/* enable checkpointing on the Flink side */
SET 'execution.checkpointing.interval' = '100ms';

/* Test Data Generator */ 
CREATE TABLE animal_sightings (
  `timestamp` TIMESTAMP(3),
   `name` STRING,
   `country` STRING,
   `number` INT
) WITH (
  'connector' = 'faker', 
  'fields.name.expression' = '#{animal.name}',
  'fields.country.expression' = '#{country.name}',
  'fields.number.expression' = '#{number.numberBetween ''0'',''1000''}',
  'fields.timestamp.expression' = '#{date.past ''15'',''SECONDS''}'
);

CREATE TABLE animal_sightings_panda_txn (
   `timestamp` TIMESTAMP(3) METADATA,
   `name` STRING,
   `country` STRING,
   `number` INT
) WITH (
   'connector' = 'kafka',
   'topic' = 'animal_sightings_txn',
   'properties.bootstrap.servers' = 'redpanda:29092',
   'format' = 'json', 
   'scan.startup.mode' = 'earliest-offset',
   'sink.delivery-guarantee' = 'exactly-once',
   'sink.transactional-id-prefix' =  'flink',
   'properties.max.in.flight.requests.per.connection'='1'
);

INSERT INTO animal_sightings_panda_txn SELECT * FROM animal_sightings;
```
Checking the Flink UI under https://localhost:8081 will show that the Job is restarting frequently with 

```
org.apache.flink.util.FlinkRuntimeException: Failed to send data to Kafka animal_sightings_txn-0@-1 with FlinkKafkaInternalProducer{transactionalId='flink-0-3', inTransaction=true, closed=false} 
	at org.apache.flink.connector.kafka.sink.KafkaWriter$WriterCallback.throwException(KafkaWriter.java:405)
	at org.apache.flink.connector.kafka.sink.KafkaWriter$WriterCallback.lambda$onCompletion$0(KafkaWriter.java:391)
	at org.apache.flink.streaming.runtime.tasks.StreamTaskActionExecutor$1.runThrowing(StreamTaskActionExecutor.java:50)
	at org.apache.flink.streaming.runtime.tasks.mailbox.Mail.run(Mail.java:90)
	at org.apache.flink.streaming.runtime.tasks.mailbox.MailboxProcessor.processMailsNonBlocking(MailboxProcessor.java:353)
	at org.apache.flink.streaming.runtime.tasks.mailbox.MailboxProcessor.processMail(MailboxProcessor.java:317)
	at org.apache.flink.streaming.runtime.tasks.mailbox.MailboxProcessor.runMailboxLoop(MailboxProcessor.java:201)
	at org.apache.flink.streaming.runtime.tasks.StreamTask.runMailboxLoop(StreamTask.java:809)
	at org.apache.flink.streaming.runtime.tasks.StreamTask.invoke(StreamTask.java:761)
	at org.apache.flink.runtime.taskmanager.Task.runWithSystemExitMonitoring(Task.java:958)
	at org.apache.flink.runtime.taskmanager.Task.restoreAndInvoke(Task.java:937)
	at org.apache.flink.runtime.taskmanager.Task.doRun(Task.java:766)
	at org.apache.flink.runtime.taskmanager.Task.run(Task.java:575)
	at java.lang.Thread.run(Thread.java:750)
Caused by: org.apache.flink.kafka.shaded.org.apache.kafka.common.errors.OutOfOrderSequenceException: The broker received an out of order sequence number.
```

This is tracked in https://github.com/redpanda-data/redpanda/issues/4029. 