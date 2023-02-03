# Kafka Connect

[Kafka Connect](https://docs.confluent.io/platform/current/connect/index.html) is a tool to stream data between Kafka and other data systems.

## Prerequisites

+ Internet connection (to download connectors on the fly)
+ Allocate at least 5GB for Docker Desktop. Otherwise, Out of Memory will fail some service

## Setup

### 1) Install Kafka clients

Install Kafka clients to use kafka commands on host machine.
```
brew install kafka
```

This way you don't have to log into containers and call bash scripts there.

### 2) Spin up the stack

```
docker compose up -d
```

What Docker Compose does:

+ Start necessary services
+ Load plugins at `/data/plugins` into `kafka-connect`
+ Install Kafka connectors at runtime using [Confluent Hub Client](https://docs.confluent.io/kafka-connectors/self-managed/confluent-hub/client.html#install-while-online)

## Connectors

There two kinds of connectors: `source` and `sink`.

+ `Source` is fetching data from an external datasource and producing records into Kafka
+ `Sink` is fetching data from Kafka and producing records into an external datasource

### 1) FileStreamSource and FileStreamSink

#### 1.1) Check if the connector plugins are loaded

```
curl http://localhost:8083/connector-plugins
```

Response
```json
[
  {
    "class": "org.apache.kafka.connect.file.FileStreamSinkConnector",
    "type": "sink",
    "version": "6.1.9-ccs"
  },
  {
    "class": "org.apache.kafka.connect.file.FileStreamSourceConnector",
    "type": "source",
    "version": "6.1.9-ccs"
  }
]
```

#### 1.2) Create connector FileStreamSource

Create a connector that writes the file `/etc/kafka/connect-distributed.properties` into topic `kafka-config-topic`.
```
echo '{"name":"load-kafka-config", "config":{"connector.class":"FileStreamSource","file":"/etc/kafka/connect-distributed.properties","topic":"kafka-config-topic"}}' | curl -X POST -d @- http://localhost:8083/connectors --header "Content-Type:application/json"
```

Response
```
{
    "name": "load-kafka-config",
    "config":
    {
        "connector.class": "FileStreamSource",
        "file": "/etc/kafka/connect-distributed.properties",
        "topic": "kafka-config-topic",
        "name": "load-kafka-config"
    },
    "tasks":[],
    "type": "source"
}
```

Subscribe the topic to see how it looks like.
```
kafka-console-consumer --bootstrap-server=localhost:9092 --topic kafka-config-topic --from-beginning
```

If the console shows the config file's content, it's working.

#### 1.3) Create connector FileStreamSink

Create a connector that dumps the content into the file `copy-of-connect-distributed.properties`.
```
echo '{"name":"dump-kafka-config", "config":{"connector.class":"FileStreamSink","file":"copy-of-connect-distributed.properties","topics":"kafka-config-topic"}}' | curl -X POST -d @- http://localhost:8083/connectors --header "content-Type:application/json"
```

Response
```
{
    "name": "dump-kafka-config",
    "config":
    {
        "connector.class": "FileStreamSink",
        "file": "copy-of-connect-distributed.properties",
        "topics": "kafka-config-topic",
        "name": "dump-kafka-config"
    },
    "tasks":[],
    "type": "sink"
}
```

Check the newly create file `/home/appuser/copy-of-connect-distributed.properties` inside docker container.

#### 1.4) Delete a connector

Try deleting `FileStreamSink`
```
curl -X DELETE http://localhost:8083/connectors/dump-kafka-config
```

### 2) JdbcSource and ElasticSearchSink

#### 2.1) Check if the connector plugins are loaded

```
curl http://localhost:8083/connector-plugins
```

Response
```json
[
  {
    "class": "io.confluent.connect.elasticsearch.ElasticsearchSinkConnector",
    "type": "sink",
    "version": "14.0.3"
  },
  {
    "class": "io.confluent.connect.jdbc.JdbcSourceConnector",
    "type": "source",
    "version": "10.6.0"
  }
]
```

#### 2.2) Create dummy database table

```
mysql -h 127.0.0.1 -uadmin -padmin
```

In MySQL console, run these commands
```
use test;
create table login (id INT NOT NULL AUTO_INCREMENT PRIMARY KEY, username varchar(30), login_time datetime);
insert into login(username, login_time) values ('john', now());
insert into login(username, login_time) values ('jane', now());
```

#### 2.3) Create connector JdbcSource

Create a connector that reads data from MySQL
```
echo '{"name":"mysql-login-connector", "config":{"connector.class":"JdbcSourceConnector","connection.url":"jdbc:mysql://mysql:3306/test?user=admin","connection.password":"admin", "mode":"timestamp","table.whitelist":"login","validate.non.null":false,"mode": "timestamp+incrementing","incrementing.column.name": "id","timestamp.column.name":"login_time","topic.prefix":"mysql."}}' | curl -X POST -d @- http://localhost:8083/connectors --header "Content-Type:application/json"
```

Response
```
{
    "name": "mysql-login-connector",
    "config":
    {
        "connector.class": "JdbcSourceConnector",
        "connection.url": "jdbc:mysql://mysql:3306/test?user=admin",
        "connection.password": "admin",
        "mode": "timestamp",
        "table.whitelist": "login",
        "validate.non.null": "false",
        "timestamp.column.name": "login_time",
        "topic.prefix": "mysql.",
        "name": "mysql-login-connector"
    },
    "tasks":
    [],
    "type": "source"
}
```

Subscribe the topic to see how it looks like.
```
kafka-console-consumer --bootstrap-server=localhost:9092 --topic mysql.login --from-beginning
```

One of the messages after JSON-formatted looks like:
```
{
    "schema":
    {
        "type": "struct",
        "fields":
        [
            {
                "type": "int32",
                "optional": false,
                "field": "id"
            },
            {
                "type": "string",
                "optional": true,
                "field": "username"
            },
            {
                "type": "int64",
                "optional": true,
                "name": "org.apache.kafka.connect.data.Timestamp",
                "version": 1,
                "field": "login_time"
            }
        ],
        "optional": false,
        "name": "login"
    },
    "payload":
    {
        "id": 1,
        "username": "john",
        "login_time": 1675422733000
    }
}
```

If the console shows the messages that contain MySQL data, it's working.

#### 2.4) Create connector ElasticSearchSink

Create a connector that reads data from Kafka and write them into ElasticSearch
```
echo '{"name":"elastic-login-connector", "config":{"connector.class":"ElasticsearchSinkConnector","connection.url":"http://elasticsearch:9200","type.name":"mysql-data","topics":"mysql.login","key.ignore":true}}' | curl -X POST -d @- http://localhost:8083/connectors --header "Content-Type:application/json"
```

Response
```
{
    "name": "elastic-login-connector",
    "config":
    {
        "connector.class": "ElasticsearchSinkConnector",
        "connection.url": "http://elasticsearch:9200",
        "type.name": "mysql-data",
        "topics": "mysql.login",
        "key.ignore": "true",
        "name": "elastic-login-connector"
    },
    "tasks":
    [],
    "type": "sink"
}
```

Search for the records in the index

```
curl -s -X "GET" "http://localhost:9200/mysql.login/_search?pretty=true"
```

Response
```
{
    "took": 175,
    "timed_out": false,
    "_shards":
    {
        "total": 1,
        "successful": 1,
        "skipped": 0,
        "failed": 0
    },
    "hits":
    {
        "total":
        {
            "value": 2,
            "relation": "eq"
        },
        "max_score": 1.0,
        "hits":
        [
            {
                "_index": "mysql.login",
                "_id": "mysql.login+0+1",
                "_score": 1.0,
                "_source":
                {
                    "id": 2,
                    "username": "jane",
                    "login_time": 1675422733000
                }
            },
            {
                "_index": "mysql.login",
                "_id": "mysql.login+0+0",
                "_score": 1.0,
                "_source":
                {
                    "id": 1,
                    "username": "john",
                    "login_time": 1675422733000
                }
            }
        ]
    }
}
```

## References

+ [Connector REST API](https://docs.confluent.io/platform/current/connect/references/restapi.html#connectors)
+ [JDBC Connector Configuration Properties](https://docs.confluent.io/kafka-connectors/jdbc/current/source-connector/source_config_options.html#jdbc-source-connector-configuration-properties)
+ [ElasticSearch Connector Configuration Properties](https://docs.confluent.io/kafka-connectors/elasticsearch/current/configuration_options.html#connector)

## FAQ

### 1) How can I clean up environment?

```
docker compose down -v
```

`-v` delete all relevant volumes to start over.

### 2) Where is connect-file connector?

Since `FileStreamSource` and `FileStreamSink` [have been moved out of Kafka Connect](https://docs.confluent.io/platform/current/connect/filestream_connector.html#kconnect-long-filestream-connectors), we have to build them from sources.

Check out https://github.com/a0x8o/kafka

Build jar files
```
./gradlew jar
```
