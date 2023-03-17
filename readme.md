# README

short demo for serializing data with kafa-avro-console-producer
and writing data to postgres with kafka connect

## Prerequisits

* A recent version of Confluent Platform installed (7.3.x)
* Linux or MacOS operating system 
* bash
* Docker and Docker Compose

## How to run

1. start environment

```bash
docker-compose up -d
```

2. create topic

```bash
kafka-topics --create --topic orders-avro --bootstrap-server localhost:9092
```

3. produce some data
```bash
kafka-avro-console-producer \
  --topic orders-avro \
  --bootstrap-server localhost:9092 \
  --property schema.registry.url=http://localhost:8081 \
  --property value.schema="$(< orders-avro-schema.json)" \
  --property key.serializer=org.apache.kafka.common.serialization.StringSerializer \
  --property parse.key=true \
  --property key.separator=":"
```

copy and paste the following to terminal and hit enter

```bash
6:{"number":6,"shipping_address":"9182 Shipyard Drive, Raleigh, NC. 27609","subtotal":72.00,"tax":3.00,"grand_total":75.00,"shipping_cost":0.00}
7:{"number":7,"shipping_address":"644 Lagon Street, Chicago, IL. 07712","subtotal":11.00,"tax":1.00,"grand_total":14.00,"shipping_cost":2.00}
```

4. set postgres db password

connect to db  
```bash
docker exec -it postgres psql -U postgres
```

set password
```bash
postgres=# alter user postgres password 'start123';
```

5. configure kafka connect
```bash

curl -X PUT \
     -H "Content-Type: application/json" \
     --data '{
    "value.converter.schema.registry.url": "http://schema-registry:8081",
    "key.converter.schema.registry.url": "http://schema-registry:8081",
    "connector.class": "io.confluent.connect.jdbc.JdbcSinkConnector",
    "tasks.max": "1",
    "key.converter": "org.apache.kafka.connect.storage.StringConverter",
    "value.converter": "io.confluent.connect.avro.AvroConverter",
    "topics": "orders-avro",
    "connection.url": "jdbc:postgresql://postgres/postgres",
    "connection.user": "postgres",
    "connection.password": "start123",
    "dialect.name": "PostgreSqlDatabaseDialect",
    "pk.mode": "record_key",
    "pk.fields": "KEY",
    "delete.enabled": "true",
    "auto.create": "true",
    "value.converter.enhanced.avro.schema.support": "true"
  }' \
     http://localhost:8083/connectors/jdbc-pg-02/config | jq . 

```

6. check the data sinked to postgres db (output similiar)

```bash

postgres=# select * from "orders-avro";
 number |            shipping_address             | subtotal | shipping_cost | tax | grand_total | KEY
--------+-----------------------------------------+----------+---------------+-----+-------------+-----
      6 | 9182 Shipyard Drive, Raleigh, NC. 27609 |       72 |             0 |   3 |          75 | 6
      7 | 644 Lagon Street, Chicago, IL. 07712    |       11 |             2 |   1 |          14 | 7
(2 rows)     

```