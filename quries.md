- [KSQL](#ksql-cli-docker)
- [Topic](#topic)
- [Stream](#stream)
- [Table](#table)

## KSQL CLI Docker 
--------------------
docker exec -it ksqldb-cli ksql http://ksqldb-server:8088

## Set auto offset
-------------------
```
set 'auto.offset.reset' = 'earliest'/'latest'/'none'
```

## TOPIC

### Create topic (from within Kafka broker's shell)
----------------------------------------------------
```
- kafka-topics --zookeeper zookeeper:2181 --create --partitions 1 --replication-factor 1 --topic COUNTRY-CSV
- kafka-topics --zookeeper zookeeper:2181 --create --partitions 1 --replication-factor 1 --topic TELEMETRYENRICHMENT-ENRICHED-POSITIONS
- kafka-topics --zookeeper zookeeper:2181 --create --partitions 1 --replication-factor 1 --topic EQUIPMENTTWIN-HISTORICAL-TELEMETRY-READINGS
```

### Delete topic (from within Kafka broker's shell)
----------------------------------------------------
```
- kafka-topics --zookeeper zookeeper:2181 --delete --topic COUNTRY-CSV
- kafka-topics --zookeeper zookeeper:2181 --delete --topic TELEMETRYENRICHMENT-ENRICHED-POSITIONS
- kafka-topics --zookeeper zookeeper:2181 --delete --topic EQUIPMENTTWIN-HISTORICAL-TELEMETRY-READINGS
```

### Change retention time to 1 sec
------------------------------------------
kafka-topics --zookeeper zookeeper:2181 --alter --topic TELEMETRYENRICHMENT-ENRICHED-POSITIONS --config retention.ms=1000;
kafka-topics --zookeeper zookeeper:2181 --alter --topic EQUIPMENTTWINS-ENRICHED-TCI-POSITIONS --config retention.ms=1000;
kafka-topics --zookeeper zookeeper:2181 --alter --topic EQUIPMENTTWINS-HISTORICAL-TELEMETRY-READINGS --config retention.ms=1000; 
kafka-topics --zookeeper zookeeper:2181 --alter --topic EQUIPMENTTWINS-ALERT-READINGS --config retention.ms=1000; 
kafka-topics --zookeeper zookeeper:2181 --alter --topic EQUIPMENTTWINS-ALERTS-GROUPED --config retention.ms=1000; 
kafka-topics --zookeeper zookeeper:2181 --alter --topic EQUIPMENTTWINS-ALERTS-ACTIVE --config retention.ms=1000; 
kafka-topics --zookeeper zookeeper:2181 --alter --topic EQUIPMENTTWINS_ALERTS_VOLTAGE --config retention.ms=1000; 

### Change retention time to 7 days
------------------------------------------
kafka-topics --zookeeper zookeeper:2181 --alter --topic TELEMETRYENRICHMENT-ENRICHED-POSITIONS --config retention.ms=604800000;
kafka-topics --zookeeper zookeeper:2181 --alter --topic EQUIPMENTTWINS-ENRICHED-TCI-POSITIONS --config retention.ms=604800000;
kafka-topics --zookeeper zookeeper:2181 --alter --topic EQUIPMENTTWINS-HISTORICAL-TELEMETRY-READINGS --config retention.ms=604800000 ;
kafka-topics --zookeeper zookeeper:2181 --alter --topic EQUIPMENTTWINS-ALERT-READINGS --config retention.ms=604800000; 
kafka-topics --zookeeper zookeeper:2181 --alter --topic EQUIPMENTTWINS-ALERTS-GROUPED --config retention.ms=604800000; 
kafka-topics --zookeeper zookeeper:2181 --alter --topic EQUIPMENTTWINS-ALERTS-ACTIVE --config retention.ms=604800000; 
kafka-topics --zookeeper zookeeper:2181 --alter --topic EQUIPMENTTWINS_ALERTS_VOLTAGE --config retention.ms=604800000; 

### Topics Partitions:
--------------------
It is good to have 30 partitions per topic if not sure

### Produce messages to topic (from within kafka broker's shell)
--------------------------------
```
kafka-console-producer --broker-list localhost:9092 --topic COUNTRY-CSV --property "parse.key=true" --property "key.separator=:"
```

## STREAM

### Create Stream 
------------------
```
CREATE STREAM MOVEMENTS (LOCATION VARCHAR)
WITH (VALUE_FORMAT='JSON', PARTITIONS=1, KAFKA_TOPIC='movements')
```

### Create Stream with Key
------------------------
```
CREATE STREAM countrystream (countrycode VARCHAR KEY, countryname VARCHAR)
>WITH (KAFKA_TOPIC='COUNTRY-CSV', VALUE_FORMAT='DELIMITED');
```
### Query Stream
----------------
```sql
SELECT TIMESTAMPTOSTRING(ROWTIME, 'yyyy-MM-dd HH:mm:ss','Europe/Amsterdam') AS EVENT_TS,
	ROWKEY as PERSON,
	LOCATION
FROM MOVEMENTS
EMIT CHANGES
```
### Publish messages to stream
--------------------------------------
```sql
INSERT INTO MOVEMENTS VALUES ('robin', 'York')
INSERT INTO MOVEMENTS VALUES ('robin', 'Leeds')
INSERT INTO MOVEMENTS VALUES ('robin', 'Brussels')
INSERT INTO MOVEMENTS VALUES ('robin', 'Leeds')
```

## Table
----------------

### Enable table scan
-------------------------
```sql
SET 'ksql.query.pull.table.scan.enabled'='true';
```

### Create Table (from within Kafka Broker's shell)
-------------------------------
```sql
CREATE TABLE COUNTRYTABLE (countrycode VARCHAR PRIMARY KEY, countryname VARCHAR) WITH (KAFKA_TOPIC='COUNTRY-CSV', VALUE_FORMAT='DELIMITED');
```

### Create Table AS (Materialized view or persistent query)(from within Kafka Broker's shell)
-------------------------------------------------------------------------------------------------
```sql
CREATE TABLE COUNTRYQUERYTABLE
AS
SELECT countrycode, countryname from COUNTRYSTREAM 
WHERE countryname IS NOT NULL
GROUP BY countrycode
EMIT CHANGES;
```

- Insert with valid key and null value deletes entry from the table

## Print logs from control center
--------------------------------
docker-compose logs control-center