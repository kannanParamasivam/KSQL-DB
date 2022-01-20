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

## Streams and tables
------------------------
https://www.confluent.io/blog/kafka-streams-tables-part-1-event-streaming/


```sql
CREATE OR REPLACE STREAM STREAM_EQUIPMENTTWIN_TCI_MEASUREMENTS
WITH (KEY_FORMAT='KAFKA', VALUE_FORMAT='PROTOBUF', KAFKA_TOPIC='EQUIPMENTTWIN-TCI-MEASUREMENTS')
AS
select 
	"equipment_number_with_prefix",
	FILTER(equipments, (v) => v -> equipment_number = SPLIT("equipment_number_with_prefix",' ')[1])[1] AS equipment,
	source AS source,	
	FILTER(measurement_attributes, v => v -> category = 'TEMPERATURE_CONTROL' AND v -> type = 'FUEL')[1] as fuel_measurement,
    FILTER(measurement_attributes, v => v -> category = 'TEMPERATURE_CONTROL' AND v -> type = 'SET_TEMP')[1] as set_temp_measurement,
    FILTER(measurement_attributes, v => v -> category = 'TEMPERATURE_CONTROL' AND v -> type = 'VOLTAGE')[1] as voltage_measurement,
    FILTER(measurement_attributes, v => v -> category = 'TEMPERATURE_CONTROL' AND v -> type = 'OPERATING_MODE')[1] as operating_mode_measurement
FROM STREAM_EQUIPMENTTWIN_ENRICHED_POSITIONS 
WHERE CASE WHEN FILTER(equipments, (v) => v -> equipment_number = SPLIT("equipment_number_with_prefix",' ')[1])[1] IS NULL THEN NULL ELSE FILTER(equipments, (v) => v -> equipment_number = SPLIT("equipment_number_with_prefix",' ')[1])[1] -> equipment_type END = 'EQUIPMENT_TYPE_CONTAINER'
AND ARRAY_LENGTH(FILTER(measurement_attributes, v => v -> category = 'TEMPERATURE_CONTROL')) > 0
PARTITION BY "equipment_number_with_prefix"
EMIT CHANGES;



CREATE OR REPLACE STREAM STREAM_EQUIPMENTTWIN_ALERTS_FUEL
WITH (KEY_FORMAT='KAFKA', VALUE_FORMAT='PROTOBUF', KAFKA_TOPIC='EQUIPMENTTWIN-ALERT-READINGS')
AS
SELECT 
	"equipment_number_with_prefix", 
    SOURCE -> device_id,
    'LOW_FUEL' AS alert_type,
    CASE
    	WHEN CAST(FUEL_MEASUREMENT -> value AS DOUBLE) < 26 THEN 'Error'
        WHEN CAST(FUEL_MEASUREMENT -> value AS DOUBLE) < 35 THEN 'Warning'
        ELSE 'InValid'
    END AS threshold
    	
FROM STREAM_EQUIPMENTTWIN_TCI_MEASUREMENTS 
WHERE 
	FUEL_MEASUREMENT IS NOT NULL
PARTITION BY "equipment_number_with_prefix"
EMIT CHANGES;

-- CREATE OR REPLACE STREAM STREAM_EQUIPMENTTWIN_ALERT_READINGS("equipment_number_with_prefix" varchar KEY)
--     WITH (KEY_FORMAT='KAFKA', VALUE_FORMAT='PROTOBUF', PARTITIONS=1, KAFKA_TOPIC='EQUIPMENTTWIN-ALERT-READINGS');

 CREATE OR REPLACE TABLE TABLE_EQUIPMENTTWIN_ALERTS_CTE("equipment_number_with_prefix" varchar, device_id VARCHAR, alert_type VARCHAR, threshold VARCHAR)
    WITH (KEY_FORMAT='KAFKA', VALUE_FORMAT='PROTOBUF', PARTITIONS=1, KAFKA_TOPIC='EQUIPMENTTWIN-ALERT-READINGS');   


-- Alerts table V1 (Auto deletion is not happening)

CREATE OR REPLACE TABLE TABLE_EQUIPMENTTWIN_ALERTS WITH (kafka_topic='EQUIPMENTTWINS-ALERTS', KEY_FORMAT='PROTOBUF', VALUE_FORMAT='PROTOBUF') AS
    SELECT "equipment_number_with_prefix", device_id, alert_type, latest_by_offset(threshold) as threshold
    FROM STREAM_EQUIPMENTTWIN_ALERT_READINGS
    GROUP BY "equipment_number_with_prefix", device_id, alert_type
    HAVING COUNT(*) > 0;

-- CREATE OR REPLACE TABLE TABLE_EQUIPMENTTWIN_ALERTS WITH (kafka_topic='EQUIPMENTTWINS-ALERTS', KEY_FORMAT='PROTOBUF', VALUE_FORMAT='PROTOBUF') AS
--     SELECT "equipment_number_with_prefix", device_id, alert_type, latest_by_offset(threshold) as threshold
--     FROM TABLE_EQUIPMENTTWIN_ALERTS_CTE
--     WHERE threshold <> 'InValid'
--     GROUP BY "equipment_number_with_prefix", device_id, alert_type
--     HAVING COUNT(*) > 0;
    



-- Query the Table
select * from TABLE_EQUIPMENTTWIN_ALERTS 
WHERE
	"equipment_number_with_prefix" = '000001 JBHU-FUEL-WARNING'
    AND device_id = 'device-123'
    AND alert_type = 'LOW_FUEL';


```