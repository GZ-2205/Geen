# Flink加工与推送实时流

## 1. 单机部署Flink

安装包解包后直接进入安装路径内

```shell
$ cd /home/ftpuser/flink-1.15.0/
# 这里直接用集群启动脚本，单机启动脚本报错
$ ./bin/start-cluster.sh
# 进入flink_sql终端
$ ./bin/sql-client.sh
```



## 2. Flink_SQL加工逻辑

### 2.1 动态表和静态表结构

```sql
-- 38个接口，主要分为动态表、静态表和目标表

-- 动态表结构
CREATE TABLE TTRD_DYN
(
-- 映射Debezium消息体中的 'after' 字段
OPERATION CHAR(1),
...
主要字段
...
-- 映射Debezium元数据字段，消息队列自带的技术字段
`source_table` STRING METADATA FROM 'value.source.table' VIRTUAL,
`event_ts` TIMESTAMP(3) METADATA FROM 'value.source.timestamp' VIRTUAL,
`database` STRING METADATA FROM 'value.source.database' VIRTUAL,
-- WATERMARK是Flink用于处理事件时间的机制，这里使用源库中变更发生的时间
WATERMARK FOR  event_ts AS event_ts - INTERVAL '5' SECOND
) WITH (
	'connector' = 'kafka',
    'topic' = 'db2-camf-dyn.CAMF_TRD.TTRD_DYN',
    'properties.bootstrap.servers' = '0.0.0.0:9092',
    'properties.group.id' = 'db2-camf-dyn',
    
    -- 核心配置，使用专用debezium-json格式
    'format' = 'debezium-json',
    -- 从value中提取kafka消息
    'value.fields-include' = 'ALL',
    'debezium-json.schema-include' = 'true',
    -- 从最早的偏移量开始消费，保证原始数据完整性
    'scan.startup.mode' = 'earliest-offset'
);

-- 静态表表结构同动态表

```

### 2.2 目标表结构

```sql
-- 目标表结构
CREATE TABLE BASE_DL(
...
主要字段
...
) WITH (
-- 核心连接器配置
	'connector' = 'kafka',
    'topic' = 'WMDP_BAS_DL',
    'properties.bootstrap.servers' = '21.16.228.185:9092',
    'properties.group.id' = 'consumer-group'
    
    -- 格式必须和写入端一致，纯json格式无法执行带表关联的加工逻辑
    'format' = 'debezium-json',
    -- 从最近的偏移量开始消费，保证数据在集群重启情况下数据不重复
    'scan.startup.mode' = 'latest-offset',
    -- 下游kafka连接配置
    'properties.security.protocol' = 'SASL_PLAINTEXT',
	'properties.sasl.mechanism' = 'SCRAM-SHA-512',
	'properties.sasl.jass.config' = 'org.apache.kafka.common.security.scram.ScramLoginModule required username="ipms" password="Kafka@9632";'
);
```



### 2.3 加工逻辑

```sql
INSERT INTO BAS_DL
SELECT
...
,CAST(to_date(SUBSTR(TTP.UPDATE, 1, 8), 'yyyyMMdd') as VARCHAR(10)) as INS_UPD_DT -- UPDATE_TIME(截取前8位转10位日期)
,CAST(date_format(to_timestamp(TTP.UPDATE_TIME, 'yyyyMMdd HH:mm:ss.SSS'), 'HH:mm:ss') as VARCHAR(8)) as INS_UPD_TM -- UPDATE_TIME(截取中间8位时间)
...
FROM
...
WHERE
...;
```

