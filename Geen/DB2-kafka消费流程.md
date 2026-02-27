# DB2-kafka消费流程

## 1. Kafka安装与配置Debezium-connector组件

### 1.1 解压安装Debezium组件与db2-jdbc驱动一并安装到指定目录下并在集群/单机配置文件中添加plugin_path（此路径的上级路径）

```
# 目录结构
# 1.5是稳定支持java8的最高debezium-connector版本
/home/ftpuser/kafka-connect-1.5/debezium-connector-db2/
 db2jcc4.jar
 debezium-api-1.5.0.Final.jar
 debezium-connector-db2-1.5.0.Final.jar
 debezium-core-1.5.0.Final.jar
 failureaccess-1.0.1.jar
 guava-30.0-jre.jar

/home/ftpuser/kafka_2.12-3.6.1/config
 connect-distributed.properties
 connect-standalone.properties
将plugin.path替换为；
plugin.path=/home/ftpuser/kafka-connect-1.5
```



### 1.2 DB2源的连接配置

#### 1.2.1 properties配置文件启动

```properties
# /home/ftpuser/kafka_2.12-3.6.1/config路径下创建db2-source-standalone.properties

name=db2-connector-src
connector.class=io.debezium.connector.db2.Db2Connector
database.hostname=10.191.52.103
database.port=50000
database.dbname=TESTDB
database.user=db2inst1
database.password=dbserver
table.include.list=test.products
database.server.name=db2-connect
database.history=io.debezium.relational.history.KafkaDatabaseHistory
database.history.kafka.bootstrap.servers=0.0.0.0:9092
database.history.kafka.topic=schemahistory.db2-connector
decimal.handling.mode=double
```



#### 1.2.2 json文件启动



```json
# 创建/home/ftpuser/source-db2-dym.json
{
    "name": "db2-camf-dy",
    "config":{
        "connector.class": "io.debezium.connector.db2.Db2Connector",
        "database.hostname":"10.144.113.241",
        "database.port": "50001",
        "database.user": "camf",
        "database.password":"C_amsf_241",     				      		                                             "database.dbname":"CAMFDB5",
        "table.include.list": "CAMF_TRD(TTRD_TRADE_ORDER_BUFFER|TTRD_SET_INSTRUCTION_BUFFER|TTRD_OTC_TRADE_BUFFER|TTRD_SET_INSTRUCTION_SECU_BUFFER|TTRD_CASHLB_BUFFER|TTRD_TRADE_CONFIRM_BUFFER|TTRD_BANK_SECU_TRANSFER_BUFFER|TTRD_SPECIAL_TRADE_ORDER_BUFFER|TTRD_SPECIAL_INVESTMENT_ORDER_BUFFER)",
        "database.server.name":"db2-camf-dyna",
        "database.history.kafka.bootstrap.servers": "0.0.0.0:9092",
        "database.history.kafka.topic":"schemahistory.db2-camf-connect-dyna",
        "decimal.handling.mode" :"double",
        "snapshot.new.tables": "parallel",
        "key.converter":"org.apache.kafka.connect.json.JsonConverter",
        "value.converter":"org.apache.kafka.connect.json.JsonConverter",
        "key.converter.schemas.enable":"false",
        "value.converter.schemas.enable":"false",
        "transforms": "unwrap",
        "transforms.unwrap.type":"io.debezium.transforms.ExtractNewRecordState",
        "transforms.unwrap.drop.tombstones": "false",
        "transforms.unwrap.delete.handling.mode": "drop"
    }
}
```



### 1.3 kafka认证配置

由于下游kafka是SHA-512，建议在client.properties、producer.properties、consumer.properties中增加以下配置：

```properties
security.protocol=SASL_PLAINTEXT
sasl.mechanism=SCRAM-SHA-512
sasl.jass.config=org.apache.kafka.common.security.scram.ScramLoginModule required username="ipms" password="Kafka@9632"
```



## 2. Kafka启动debezium-connector组件

### 2.1 单机模式（需指定配置文件启动）

```shell
$ cd /home/ftpuser/kafka_2.12-3.6.1
$ bin/connect-standalone.sh -daemon config/connect-standalone.properties config/db2-source-standalone.properties

# 启动成功后验证db2-connectors组件是否正常运行
$ curl http://0.0.0.0:8083/connectors/
["db2-connector-src"]

$ curl http://0.0.0.0:8083/connectors/"db2-connector-src"/status
{"name":"db2-connector-src","connector":{"state":"RUNNING","woker_id":"172.16.86.4:8083"},"tasks":[{"id":0,"state":"RUNNING","woker_id":"172.16.86.4:8083"}],"type":"source"}

```



### 2.2 集群模式

```shell
# 集群模式可不附加数据库连接配置，单独指定json配置文件启动
$ cd /home/ftpuser/kafka_2.12-3.6.1
$ bin/connect-standalone.sh -daemon config/connect-distributed.properties

$ curl -H "Content-Type: application/json" -X POST -d @"/home/ftpuser/source-db2-dym.json" http://0.0.0.0:8083/connectors/
```



## 3. db2-kafka数据捕获和消费

```shell
# 元数据topic
$ cd /home/ftpuser/kafka_2.12-3.6.1
$ bin/kafka-console-consumer.sh --bootstrap-server 0.0.0.0:9092 --topic schemahistory.db2-connector --from-beginning

# 捕获表数据topic
$ bin/kafka-console-consumer.sh --bootstrap-server 0.0.0.0:9092 --topic db2-connect.TEST.PRODUCTS --from-beginning
```



## 4. kafka常用命令

```shell
# zookeeper服务开启和关闭
$ bin/zookeeper-server-start.sh -daemon config/zookeeper.properties
$ bin/zookeeper-server-stop.sh -daemon

# kafka服务开启和关闭
$ bin/kafka-server-start.sh -daemon config/server.properties
$ bin/kafka-server-stop.sh -daemon

# 创建topic
$ bin/kafka-topics.sh --create \
	--topic <topic名称>
	--bootstrap-server 0.0.0.0:9092
	--partitions <分区数>
	--replication-factpr <副本数> # 单机模式为1， 集群模式可大于1

$ bin/kafka-topics.sh --create --topic WMDP_ORD --bootstrap-server 0.0.0.0:9092 --partitions 1 -- replication-factor 1 --command-confg /home/ftpuser/kafka_2.12-3.6.1/config/client.properties

# 删除topic
$ bin/kafka-topics.sh --delete --topic db2-src-connector.test.products --bootstrap-server 0.0.0.0:9092

# 删除connector
$ curl -X DELETE http://0.0.0.0:8083/connectors/db2-cource-connector

# 查看topic列表
$ bin/kafka-topics.sh --bootstrap-server 0.0.0.0:9092 --list --command-config /home/ftpuser/kafka_2.12-3.6.1/config/client.properties

# 查看 Kafka Broker 支持的 API 版本信息
$ bin/kafka-broker-api-versions.sh --bootstrap-server 0.0.0.0:9092 --describe

# 查看指定 Topic的详细信息
$ bin/kafka-topics.sh --bootstrap-server 0.0.0.0:9092 --describe --topic db2-src-connector.test.products
```



注释
