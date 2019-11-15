# Change Data Capture with Kafka and Debezium

## Demo Overview

This demo aims to capture modification/changes done in Mysql and transform it into continuous stream of data using Apache Kafka and Debezium connector. 

## Demo Steps

Use docker images to start following:
1. Start Zookeeper 
2. Start Kafka
3. Start Mysql (preloaded data)
4. Mysql terminal
5. Kafka Connect Service
6. Register and start Debezium-mysql connector
7. Watch Kafka topic
8. Modify records in mysql and view the captured data change in Kafka topic


## Detailed steps with docker commands

### Zookeeper 
$ docker run -it --rm --name zookeeper -p 2181:2181 -p 2888:2888 -p 3888:3888 debezium/zookeeper:1.0

### Kafka
$ docker run -it --rm --name kafka -p 9092:9092 --link zookeeper:zookeeper debezium/kafka:1.0

### MySql
$ docker run -it --rm --name mysql -p 3306:3306 -e MYSQL_ROOT_PASSWORD=debezium -e MYSQL_USER=mysqluser -e MYSQL_PASSWORD=mysqlpw debezium/example-mysql:1.0


### MySql Command line
$ docker run -it --rm --name mysqlterm --link mysql --rm mysql:5.7 sh -c 'exec mysql -h"$MYSQL_PORT_3306_TCP_ADDR" -P"$MYSQL_PORT_3306_TCP_PORT" -uroot -p"$MYSQL_ENV_MYSQL_ROOT_PASSWORD"'

### Kafka Connect
$ docker run -it --rm --name connect -p 8083:8083 -e GROUP_ID=1 -e CONFIG_STORAGE_TOPIC=my_connect_configs -e OFFSET_STORAGE_TOPIC=my_connect_offsets -e STATUS_STORAGE_TOPIC=my_connect_statuses --link zookeeper:zookeeper --link kafka:kafka --link mysql:mysql debezium/connect:1.0


### Debezium Connector
$ curl -i -X POST -H "Accept:application/json" -H "Content-Type:application/json" localhost:8083/connectors/ -d '{ "name": "inventory-connector", "config": { "connector.class": "io.debezium.connector.mysql.MySqlConnector", "tasks.max": "1", "database.hostname": "mysql", "database.port": "3306", "database.user": "debezium", "database.password": "dbz", "database.server.id": "184054", "database.server.name": "dbserver1", "database.whitelist": "inventory", "database.history.kafka.bootstrap.servers": "kafka:9092", "database.history.kafka.topic": "dbhistory.inventory" } }'

Configuration 
```bash
{
  "name": "inventory-connector",
  "config": {
    "connector.class": "io.debezium.connector.mysql.MySqlConnector",
    "tasks.max": "1",
    "database.hostname": "mysql",
    "database.port": "3306",
    "database.user": "debezium",
    "database.password": "dbz",
    "database.server.id": "184054",
    "database.server.name": "dbserver1",
    "database.whitelist": "inventory",
    "database.history.kafka.bootstrap.servers": "kafka:9092",
    "database.history.kafka.topic": "schema-changes.inventory"
  }
}
```

### Topic Watcher
$ docker run -it --name watcher --rm --link zookeeper:zookeeper --link kafka:kafka debezium/kafka:1.0 watch-topic -a -k dbserver1.inventory.customers

### Clean up
$ docker stop mysqlterm watcher connect mysql kafka zookeeper

## Result
Modifying any record in mysql would trigger a change event that would hold previous and current state of the record. The change events can be monitoried and consumed from Kafka topic for further processing. KStream application can be used to consume and transform change events.

