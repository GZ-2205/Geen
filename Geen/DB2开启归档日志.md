## DB2开启归档日志

1. 检查当前日志配置

```shell
db2 get db cfg for DB_NAME | grep -i log
```



2. 修改数据库配置参数

```shell
# 指定归档日志存储位置
db2 update db cfg for DB_NAME using LOGARCHMETH1 DISK:/data1/logs_archive/
# 主日志文件数量
db2 update db cfg for DB_NAME using LOGPRIMARY 10
# 辅助日志文件数量
db2 update db cfg for DB_NAME using LOGSECOND 20
# 每个日志文件的大小（4KB页）
db2 update db cfg for DB_NAME using LOGFILSIZ 1024
```



3. 启用归档日志模式

```shell
# 指定归档日志存储位置
db2 update db cfg for DB_NAME using LOGARCHMETH1 DISK:/data1/logs_archive/

# 注：11.5版本废弃
db2 update db cfg for DB_NAME using LOGRETAIN ON

db2 update db cfg for DB_NAME using TRACKMOD ON

```





4. 重启数据库使配置生效

```shell
db2stop force
db2start
```



5. 验证归档日志配置

```shell
db2 get db cfg for DB_NAME | grep -i "LOGARCHMENTH1\|LOGRETAIN\|TRACKMOD"
```



6. 执行全量备份（重要）

```shell
# 注意备份前需关闭所有DB2连接
## 查看当前已连接信息
db2 list applications for database DB_NAME

## 强制关闭所有连接
db2 force applications all

# 启用归档日志必须执行一次全量备份
db2 backup db DB_NAME to /path/to/backup

# 涉及DB2所属目录创建必须按实例用户分组设置，确保数据库用户拥有读写执行权限
# 示例：db2inst1:db2iadm1
```

