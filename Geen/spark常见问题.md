spark3常见问题

1.子查询中出现多个数据类型不同的列相加并命名成新列，在外部查询调用时会发生寻找旧列ID丢失的问题

原理：
如果数据类型不同的列相加时会先命名成新列然后被隐式转换成固定类型，比如decimal(38, 8)，生成一个同名新列ID并抛弃旧列ID，而执行计划中仍旧查询旧列ID且不会动态修改，所以在外部查询调用时执行计划仍旧寻找旧列ID，发生列丢失的情况

解决方案：
对列加和结果进行显式数据类型转换，从而在执行计划层面指定固定列ID，避免隐式转换丢失列ID的情况


spark-submit所需参数：

spark.dynamicAllocation.enabled=true
False(默认)，任务启动会占满所有指定的Executor资源
True开启动态资源分配，按照任务数量动态分配Executor的数量

spark.sql.parquet.writeLegacyFormat=true
传统格式写入Parquet文件，False(默认)新标准格式，True使用旧版Spark/Hive兼容方式

spark.sql.legacy.charVarcharAsString=false
True(默认)把CHAR和VARCHAR类型当做STRING处理
False则会按CHAR或VARCHAR类型按照元数据的长度截断，内部还是先按STRING处理，插入或者查询表时还是按照元数据格式写入/展示

spark.yarn.maxAppAttempts=1
最大重试次数