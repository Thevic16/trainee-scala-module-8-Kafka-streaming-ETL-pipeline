# Presentation Streaming ETL pipeline

[References Problems](https://www.notion.so/References-Problems-ac80fddf7e084978bba307ac97752288)

## **Streaming ETL pipeline.**

## What is it?

A streaming ETL pipeline, sometimes called a “streaming data pipeline”, is a set of software services that ingests events, transforms them, and loads them into destination storage systems. It’s often the case that you have data in one place and want to move it to another as soon as you receive it, but you need to make some changes to the data as you transfer it.

Maybe you need to do something simple, like transform the events to strip out any personally identifiable information. Sometimes, you may need to do something more complex, like enrich the events by joining them with data from another system. Or perhaps you want to pre-aggregate the events to reduce how much data you send to the downstream systems.

A streaming ETL pipeline enables streaming events between arbitrary sources and sinks, and it helps you make changes to the data while it’s in-flight.

![Untitled](Presentation%20Streaming%20ETL%20pipeline%20c0992b008d704921ab4acfbb9079c158/Untitled.png)

One way you might do this is to capture the changelogs of upstream Postgres and MongoDB databases using the [Debezium](https://debezium.io/) Kafka connectors. The changelog can be stored in Kafka, where a series of deployed programs transforms, aggregates, and joins the data together. The processed data can be streamed out to ElasticSearch for indexing. Many people build this sort of architecture, but could it be made simpler?

[About Debezium](https://www.notion.so/About-Debezium-138ddb48c87347018835eaab04809bc3)

## Why ksqlDB?

Gluing all of the above services together is certainly a challenge. Along with your original databases and target analytical data store, you end up managing clusters for Kafka, connectors, and your stream processors. It's challenging to operate the entire stack as one.

ksqlDB helps streamline how you write and deploy streaming data pipelines by boiling it down to just two things: storage (Kafka) and compute (ksqlDB).

![Untitled](Presentation%20Streaming%20ETL%20pipeline%20c0992b008d704921ab4acfbb9079c158/Untitled%201.png)

## Implement it

Suppose you work at a retail company that sells and ships orders to online customers. You want to analyze the shipment activity of orders as they happen in real-time. Because the company is somewhat large, the data for customers, orders, and shipments are spread across different databases and tables.

It demonstrates capturing changes from Postgres and MongoDB databases, forwarding them into Kafka, joining them together with ksqlDB, and sinking them out to ElasticSearch for analytics.

![Untitled](Presentation%20Streaming%20ETL%20pipeline%20c0992b008d704921ab4acfbb9079c158/Untitled%202.png)

### Get the connectors

download the connectors for Postgres, MongoDB, and Elasticsearch to a fresh directory. The easiest way to do this is by using [confluent-hub](https://docs.confluent.io/current/connect/managing/confluent-hub/client.html).

Create a directory for your components:

```
mkdir confluent-hub-components
```

 Acquire the Postgres Debezium connector:

```
confluent-hub install --component-dir confluent-hub-components --no-prompt debezium/debezium-connector-postgresql:1.1.0
```

Likewise for the MongoDB Debezium connector:

```
confluent-hub install --component-dir confluent-hub-components --no-prompt debezium/debezium-connector-mongodb:1.1.0
```

And finally, the Elasticsearch connector:

```
confluent-hub install --component-dir confluent-hub-components --no-prompt confluentinc/kafka-connect-elasticsearch:10.0.1
```

### Start the stack

 Set up and launch the services in the stack. But before bring it up, It is required to make a few changes to the way that Postgres launches so that it works well with Debezium.  To simplify some of this, you launch a Postgres Docker container [extended by Debezium](https://hub.docker.com/r/debezium/postgres) to handle some of the customization. Also, you must create an additional configuration file at `postgres/custom-config.conf` with the following content:

```
listen_addresses = '*'
wal_level = 'logical'
max_wal_senders = 1
max_replication_slots = 1
```

This sets up Postgres so that Debezium can watch for changes as they occur.

With the Postgres configuration file in place, create a `docker-compose.yml`
 file that defines the services to launch.

```docker
---
version: '2'

services:
  mongo:
    image: mongo:4.2.5
    hostname: mongo
    container_name: mongo
    ports:
      - "27017:27017"
    environment:
      MONGO_INITDB_ROOT_USERNAME: mongo-user
      MONGO_INITDB_ROOT_PASSWORD: mongo-pw
      MONGO_REPLICA_SET_NAME: my-replica-set
    command: --replSet my-replica-set --bind_ip_all

  postgres:
    image: debezium/postgres:12
    hostname: postgres
    container_name: postgres
    ports:
      - "5432:5432"
    environment:
      POSTGRES_USER: postgres-user
      POSTGRES_PASSWORD: postgres-pw
      POSTGRES_DB: customers
    volumes:
      - ./postgres/custom-config.conf:/etc/postgresql/postgresql.conf
    command: postgres -c config_file=/etc/postgresql/postgresql.conf

  elastic:
    image: elasticsearch:7.6.2
    hostname: elastic
    container_name: elastic
    ports:
      - "9200:9200"
      - "9300:9300"
    environment:
      discovery.type: single-node

  zookeeper:
    image: confluentinc/cp-zookeeper:7.3.0
    hostname: zookeeper
    container_name: zookeeper
    ports:
      - "2181:2181"
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000

  broker:
    image: confluentinc/cp-kafka:7.3.0
    hostname: broker
    container_name: broker
    depends_on:
      - zookeeper
    ports:
      - "29092:29092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: 'zookeeper:2181'
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://broker:9092,PLAINTEXT_HOST://localhost:29092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1

  schema-registry:
    image: confluentinc/cp-schema-registry:7.3.0
    hostname: schema-registry
    container_name: schema-registry
    depends_on:
      - zookeeper
      - broker
    ports:
      - "8081:8081"
    environment:
      SCHEMA_REGISTRY_HOST_NAME: schema-registry
      SCHEMA_REGISTRY_KAFKASTORE_BOOTSTRAP_SERVERS: "PLAINTEXT://broker:9092"

  ksqldb-server:
    image: confluentinc/ksqldb-server:0.28.3
    hostname: ksqldb-server
    container_name: ksqldb-server
    depends_on:
      - broker
      - schema-registry
    ports:
      - "8088:8088"
    volumes:
      - "./confluent-hub-components/:/usr/share/kafka/plugins/"
    environment:
      KSQL_LISTENERS: "http://0.0.0.0:8088"
      KSQL_BOOTSTRAP_SERVERS: "broker:9092"
      KSQL_KSQL_SCHEMA_REGISTRY_URL: "http://schema-registry:8081"
      KSQL_KSQL_LOGGING_PROCESSING_STREAM_AUTO_CREATE: "true"
      KSQL_KSQL_LOGGING_PROCESSING_TOPIC_AUTO_CREATE: "true"
      KSQL_CONNECT_GROUP_ID: "ksql-connect-cluster"
      KSQL_CONNECT_BOOTSTRAP_SERVERS: "broker:9092"
      KSQL_CONNECT_KEY_CONVERTER: "org.apache.kafka.connect.storage.StringConverter"
      KSQL_CONNECT_VALUE_CONVERTER: "io.confluent.connect.avro.AvroConverter"
      KSQL_CONNECT_VALUE_CONVERTER_SCHEMA_REGISTRY_URL: "http://schema-registry:8081"
      KSQL_CONNECT_CONFIG_STORAGE_TOPIC: "_ksql-connect-configs"
      KSQL_CONNECT_OFFSET_STORAGE_TOPIC: "_ksql-connect-offsets"
      KSQL_CONNECT_STATUS_STORAGE_TOPIC: "_ksql-connect-statuses"
      KSQL_CONNECT_CONFIG_STORAGE_REPLICATION_FACTOR: 1
      KSQL_CONNECT_OFFSET_STORAGE_REPLICATION_FACTOR: 1
      KSQL_CONNECT_STATUS_STORAGE_REPLICATION_FACTOR: 1
      KSQL_CONNECT_PLUGIN_PATH: "/usr/share/kafka/plugins"

  ksqldb-cli:
    image: confluentinc/ksqldb-cli:0.28.3
    container_name: ksqldb-cli
    depends_on:
      - broker
      - ksqldb-server
    entrypoint: /bin/sh
    tty: true
```

[Docker compose images options](https://www.notion.so/Docker-compose-images-options-3786777d7ecb4eef9ac5b82a4077e18e)

There are a couple things to notice here. The Postgres image mounts the custom configuration file. Postgres adds these configuration settings into its system-wide configuration. 

The compose file also sets up MongoDB as a replica set named `my-replica-set`. 

Bring up the entire stack by running:

`docker-compose up`

### Create the customers table in Postgres

It's pretty common for companies to keep their customer data in a relational database. You can model this information in a Postgres table. Start by logging into the container:

```docker
docker exec -it postgres /bin/bash
```

Log into Postgres as the user created by default:

```docker
psql -U postgres-user customers
```

[Psql](https://www.postgresguide.com/utilities/psql/)

Create a table that represents the customers. For simplicity, model a customer with three columns: an id, a name, and the age of the person:

```sql
CREATE TABLE customers (id TEXT PRIMARY KEY, name TEXT, age INT);
```

[5.1. Table Basics](https://www.postgresql.org/docs/15/ddl-basics.html)

[Chapter 8. Data Types](https://www.postgresql.org/docs/15/datatype.html)

### **Configure MongoDB for Debezium.**

Now that Postgres is set up, you can configure MongoDB. Start by logging into the container:

```docker
docker exec -it mongo /bin/bash
```

Log into the Mongo console using the username specified in the Docker Compose file:

```docker
mongo -u $MONGO_INITDB_ROOT_USERNAME -p mongo-pw admin
```

[About MongoDB](https://www.notion.so/About-MongoDB-2d64b71e738f4f3fb53241e756dc21ff)

Because MongoDB has been started as a replica set, it needs to be initiated. Run the following command to kick it off:

```docker
rs.initiate()
```

Now that this node has become the primary in the replica set, you need to configure access so that Debezium can replicate changes remotely. Switch into the `config` database:

```docker
use config
```

Create a new role for Debezium. This role enables the user that you will create to access system-level collections, which are normally restricted:

```sql
db.createRole({
    role: "dbz-role",
    privileges: [
        {
            resource: { db: "config", collection: "system.sessions" },
            actions: [ "find", "update", "insert", "remove" ]
        }
    ],
    roles: [
       { role: "dbOwner", db: "config" },
       { role: "dbAdmin", db: "config" },
       { role: "readWrite", db: "config" }
    ]
})
```

Switch into the `admin`database and create the user here so that it can be authenticated:

```sql
use admin
```

Create the user for Debezium. This user has `root`on the `admin`database, and can also access other databases needed for replication:

```sql
db.createUser({
  "user" : "dbz-user",
  "pwd": "dbz-pw",
  "roles" : [
    {
      "role" : "root",
      "db" : "admin"
    },
    {
      "role" : "readWrite",
      "db" : "logistics"
    },
    {
      "role" : "dbz-role",
      "db" : "config"
    }
  ]
})
```

### **Create the logistics collections in MongoDB.**

With the user created, you can create the database for orders and shipments, which are stored as collections in a database called `logistics`:

```sql
use logistics
```

First create the `orders`:

```sql
db.createCollection("orders")
```

And likewise the `shipments`:

```sql
db.createCollection("shipments")
```

### **Start the Postgres and MongoDB Debezium source connectors.**

With all of the seed data in place, you can process it with ksqlDB. Connect to ksqlDB's server by using its interactive CLI. Run the following command from your host:

```sql
docker exec -it ksqldb-cli ksql http://ksqldb-server:8088
```

Before you issue more commands, tell ksqlDB to start all queries from earliest point in each topic:

```sql
SET 'auto.offset.reset' = 'earliest';
```

Now you can ask Debezium to stream the Postgres changelog into Kafka. Invoke the following command in ksqlDB, which creates a Debezium source connector and writes all of its changes to Kafka topics:

```sql
CREATE SOURCE CONNECTOR customers_reader WITH (
    'connector.class' = 'io.debezium.connector.postgresql.PostgresConnector',
    'database.hostname' = 'postgres',
    'database.port' = '5432',
    'database.user' = 'postgres-user',
    'database.password' = 'postgres-pw',
    'database.dbname' = 'customers',
    'database.server.name' = 'customers',
    'table.whitelist' = 'public.customers',
    'transforms' = 'unwrap',
    'transforms.unwrap.type' = 'io.debezium.transforms.ExtractNewRecordState',
    'transforms.unwrap.drop.tombstones' = 'false',
    'transforms.unwrap.delete.handling.mode' = 'rewrite'
);
```

Run another source connector to ingest the changes from MongoDB. Specify the same behavior for discarding the Debezium envelope:

```sql
CREATE SOURCE CONNECTOR logistics_reader WITH (
    'connector.class' = 'io.debezium.connector.mongodb.MongoDbConnector',
    'mongodb.hosts' = 'mongo:27017',
    'mongodb.name' = 'my-replica-set',
    'mongodb.authsource' = 'admin',
    'mongodb.user' = 'dbz-user',
    'mongodb.password' = 'dbz-pw',
    'collection.whitelist' = 'logistics.*',
    'transforms' = 'unwrap',
    'transforms.unwrap.type' = 'io.debezium.connector.mongodb.transforms.ExtractNewDocumentState',
    'transforms.unwrap.drop.tombstones' = 'false',
    'transforms.unwrap.delete.handling.mode' = 'drop',
    'transforms.unwrap.operation.header' = 'true'
);
```

## **Create the ksqlDB source streams.**

For ksqlDB to be able to use the topics that Debezium created, you must declare streams over it. Because you configured Kafka Connect with Schema Registry, you don't need to declare the schema of the data for the streams, because it's inferred from the schema that Debezium writes with.

[About Schema Registery](https://www.notion.so/About-Schema-Registery-a61f651f91df4ca3869619fda60606ea)

Run the following statement to create a stream over the `customers` table:

```sql
CREATE STREAM customers WITH (
    kafka_topic = 'customers.public.customers',
    value_format = 'avro'
);
```

[About streams](https://www.notion.so/About-streams-a65fb9b38f8f4346857702aca6c2a181)

[About Avro format](https://www.notion.so/About-Avro-format-807e597435a2479eb5b1727869290b9f)

Do the same for `orders`. For this stream, specify that the timestamp of the event is derived from the data itself. Specifically, it's extracted and parsed from the `ts`
field.

```sql
CREATE STREAM orders WITH (
    kafka_topic = 'my-replica-set.logistics.orders',
    value_format = 'avro',
    timestamp = 'ts',
    timestamp_format = 'yyyy-MM-dd''T''HH:mm:ss'
);
```

Finally, repeat the same for `shipments`:

```sql
CREATE STREAM shipments WITH (
    kafka_topic = 'my-replica-set.logistics.shipments',
    value_format = 'avro',
    timestamp = 'ts',
    timestamp_format = 'yyyy-MM-dd''T''HH:mm:ss'
);
```

### Join the streams together

The goal is to create a unified view of the activity of shipped orders. To do this, we want to include as much customer information on each shipment as possible. Recall that the `orders` collection that we created in MongoDB only had an identifier for each customer, but not their name. Use this identifier to look up the rest of the information by using a stream/table join. To do this, you must re-key the stream into a table by the `id` field:

```sql
CREATE TABLE customers_by_key AS
    SELECT id,
           latest_by_offset(name) AS name,
           latest_by_offset(age) AS age
    FROM customers
    GROUP BY id
    EMIT CHANGES;
```

[About Table](https://www.notion.so/About-Table-49aba0ab14d14c17b72080d157a9098e)

Now you can enrich the orders with more customer information. The following stream/table join creates a new stream that lifts the customer information into the order event:

```sql
CREATE STREAM enriched_orders AS
    SELECT o.order_id,
           o.price,
           o.currency,
           c.id AS customer_id,
           c.name AS customer_name,
           c.age AS customer_age
    FROM orders AS o
    LEFT JOIN customers_by_key c
    ON o.customer_id = c.id
    EMIT CHANGES;
```

You can take this further by enriching all shipments with more information about the order and customer. Use a stream/stream join to find orders in the relevant window of time. This creates a new stream called `shipped_orders`that unifies the shipment, order, and customer information:

```sql
CREATE STREAM shipped_orders WITH (
    kafka_topic = 'shipped_orders'
)   AS
    SELECT o.order_id,
           s.shipment_id,
           o.customer_id,
           o.customer_name,
           o.customer_age,
           s.origin,
           o.price,
           o.currency
    FROM enriched_orders AS o
    INNER JOIN shipments s
    WITHIN 7 DAYS
    ON s.order_id = o.order_id
    EMIT CHANGES;
```

### Start the Elasticsearch sink connector

The application must perform searches and analytics over this unified stream of information. To make this easy, spill the information out to Elasticsearch and run the following connector to sink the topic:

```sql
CREATE SINK CONNECTOR enriched_writer WITH (
    'connector.class' = 'io.confluent.connect.elasticsearch.ElasticsearchSinkConnector',
    'connection.url' = 'http://elastic:9200',
    'type.name' = 'kafka-connect',
    'topics' = 'shipped_orders'
);
```

[About ElasticSearch](https://www.notion.so/About-ElasticSearch-c1ea4141f29445048d0a2cf0f2180000)

### **Insert data to Postgres:**

Seed the table with some initial data:

```sql
INSERT INTO customers (id, name, age) VALUES ('5', 'fred', 34);
INSERT INTO customers (id, name, age) VALUES ('7', 'sue', 25);
INSERT INTO customers (id, name, age) VALUES ('2', 'bill', 51);
```

### Insert data to MongoDB:

Populate the `orders`collection with some initial data. Notice that the `customer_id`references identifiers that you created in your Postgres customers table:

```sql
db.orders.insert({"customer_id": "2", "order_id": "13", "price": 50.50, "currency": "usd", "ts": "2020-04-03T11:20:00"})
db.orders.insert({"customer_id": "7", "order_id": "29", "price": 15.00, "currency": "aud", "ts": "2020-04-02T12:36:00"})
db.orders.insert({"customer_id": "5", "order_id": "17", "price": 25.25, "currency": "eur", "ts": "2020-04-02T17:22:00"})
db.orders.insert({"customer_id": "5", "order_id": "15", "price": 13.75, "currency": "usd", "ts": "2020-04-03T02:55:00"})
db.orders.insert({"customer_id": "7", "order_id": "22", "price": 29.71, "currency": "aud", "ts": "2020-04-04T00:12:00"})
```

Do the same for shipments. Notice that the `order_id` references order ids you created in the previous collection.

```sql
db.shipments.insert({"order_id": "17", "shipment_id": "75", "origin": "texas", "ts": "2020-04-04T19:20:00"})
db.shipments.insert({"order_id": "22", "shipment_id": "71", "origin": "iowa", "ts": "2020-04-04T12:25:00"})
db.shipments.insert({"order_id": "29", "shipment_id": "89", "origin": "california", "ts": "2020-04-05T13:21:00"})
db.shipments.insert({"order_id": "13", "shipment_id": "92", "origin": "maine", "ts": "2020-04-04T06:13:00"})
db.shipments.insert({"order_id": "15", "shipment_id": "95", "origin": "florida", "ts": "2020-04-04T01:13:00"})
```

Check that the data arrived in the index by running the following command from your host:

```
curl http://localhost:9200/shipped_orders/_search?pretty
```

Your output should resemble:

```json
{
  "took" : 75,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 5,
      "relation" : "eq"
    },
    "max_score" : 1.0,
    "hits" : [
      {
        "_index" : "shipped_orders",
        "_type" : "kafka-connect",
        "_id" : "22",
        "_score" : 1.0,
        "_source" : {
          "O_ORDER_ID" : "22",
          "SHIPMENT_ID" : "71",
          "CUSTOMER_ID" : "7",
          "CUSTOMER_NAME" : "sue",
          "CUSTOMER_AGE" : 25,
          "ORIGIN" : "iowa",
          "PRICE" : 29.71,
          "CURRENCY" : "aud"
        }
      },
      {
        "_index" : "shipped_orders",
        "_type" : "kafka-connect",
        "_id" : "17",
        "_score" : 1.0,
        "_source" : {
          "O_ORDER_ID" : "17",
          "SHIPMENT_ID" : "75",
          "CUSTOMER_ID" : "5",
          "CUSTOMER_NAME" : "fred",
          "CUSTOMER_AGE" : 34,
          "ORIGIN" : "texas",
          "PRICE" : 25.25,
          "CURRENCY" : "eur"
        }
      },
      {
        "_index" : "shipped_orders",
        "_type" : "kafka-connect",
        "_id" : "29",
        "_score" : 1.0,
        "_source" : {
          "O_ORDER_ID" : "29",
          "SHIPMENT_ID" : "89",
          "CUSTOMER_ID" : "7",
          "CUSTOMER_NAME" : "sue",
          "CUSTOMER_AGE" : 25,
          "ORIGIN" : "california",
          "PRICE" : 15.0,
          "CURRENCY" : "aud"
        }
      },
      {
        "_index" : "shipped_orders",
        "_type" : "kafka-connect",
        "_id" : "13",
        "_score" : 1.0,
        "_source" : {
          "O_ORDER_ID" : "13",
          "SHIPMENT_ID" : "92",
          "CUSTOMER_ID" : "2",
          "CUSTOMER_NAME" : "bill",
          "CUSTOMER_AGE" : 51,
          "ORIGIN" : "maine",
          "PRICE" : 50.5,
          "CURRENCY" : "usd"
        }
      },
      {
        "_index" : "shipped_orders",
        "_type" : "kafka-connect",
        "_id" : "15",
        "_score" : 1.0,
        "_source" : {
          "O_ORDER_ID" : "15",
          "SHIPMENT_ID" : "95",
          "CUSTOMER_ID" : "5",
          "CUSTOMER_NAME" : "fred",
          "CUSTOMER_AGE" : 34,
          "ORIGIN" : "florida",
          "PRICE" : 13.75,
          "CURRENCY" : "usd"
        }
      }
    ]
  }
}
```

# Demonstration:

Run on Kafka:

```sql
kafka-topics.sh --zookeeper 127.0.0.1:2181 --list
```

In posgress:

```sql
SELECT * FROM customers;
```

```sql
INSERT INTO customers (id, name, age) VALUES ('9', 'victor', 23);
```

In mongoDB:

```sql
db.orders.find()
```

```sql
db.orders.insert({"customer_id": "9", "order_id": "31", "price": 525.50, "currency": "usd", "ts": "2020-04-03T11:20:00"})
```

```sql
db.shipments.find()
```

```sql
db.shipments.insert({"order_id": "31", "shipment_id": "97", "origin": "new jersey", "ts": "2020-04-04T19:20:00"})
```

Watch the update on elastic:

```
curl http://localhost:9200/shipped_orders/_search?pretty
```