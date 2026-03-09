# DB2表捕获步骤（简略）



## 1. 将by_asncdc.tar包上传至/home/db2inst1/目录下

```shell
cd /home/db2inst1
tart -xvf by_asncdc.tar
```



## 2. db2inst1用户登录并执行以下命令：

```shell
cp $HOME/asncdc $HOME/sqllib/function
chmod 755 $HOME/sqllib/function

cd $HOME/sqllib/bnd
db2 bind db2schema.bnd blocking all grant public sqlerror continue

db2 connect to CAMFDB5

db2 -tvmf $HOME/by_asncdc.sql
```



## 3. DBeaver执行以下SQL：

```sql
VALUES ASNCDC.ASNCDCSERVICES('start', 'asncdc');

CALL ASNCDC.ADDTABLE('MYSCHEMA', 'MYTABLE', 'TASK');

VALUES ASNCDC.ASNCDCSERVIECES('reinit', 'asncdc');

INSERT INTO MYSCHEMA.MYTABLE VALUES ('1', '测试S', 1.5);

-- 观察ASNCDC模式下的cdc表是否有新数据被捕获
```





# DB2表捕获步骤（详细）

## 1. 准备工作

### 1.1 登录 Db2 数据库
以具有必要权限的用户（如 `db2inst1`）登录到 Db2 数据库主机。

```bash
su - db2inst1
```

### 1.2 编译 Debezium 管理 UDF
在 Db2 服务器主机上，使用 Db2 提供的命令编译 Debezium 管理 UDF。

```bash
cd $HOME/asncdctools/src
./bldrtn asncdc
```

### 1.3 启动数据库
如果数据库尚未运行，启动数据库。将 `DB_NAME` 替换为您的数据库名称。

```bash
db2 start db DB_NAME
```

### 1.4 确保 JDBC 可读取元数据目录
运行以下命令确保 JDBC 可以读取 Db2 元数据目录。

```bash
cd $HOME/sqllib/bnd
db2 bind db2schema.bnd blocking all grant public sqlerror continue
```

### 1.5 备份数据库
确保数据库最近已备份。如果需要执行备份，请运行以下命令。将 `DB_NAME` 和 `BACK_UP_LOCATION` 替换为适当的值。

```bash
db2 backup db DB_NAME to BACK_UP_LOCATION
```

### 1.6 重新启动数据库
备份后重新启动数据库。

```bash
db2 restart db DB_NAME
```

## 2. 安装和配置 Debezium 管理 UDF

### 2.1 连接到数据库
使用以下命令连接到目标数据库。将 `DB_NAME` 替换为您的数据库名称。

```bash
db2 connect to DB_NAME
```

### 2.2 复制并设置权限
将 Debezium 管理 UDF 复制到指定目录，并为其设置适当的权限。

```bash
cp $HOME/asncdctools/src/asncdc $HOME/sqllib/function
chmod 777 $HOME/sqllib/function
```

### 2.3 启用 ASN 捕获代理的 UDF
运行以下 SQL 脚本启用启动和停止 ASN 捕获代理的 UDF。

```bash
db2 -tvmf $HOME/asncdctools/src/asncdc_UDF.sql
```

### 2.4 创建 ASN 控制表
运行以下 SQL 脚本创建 ASN 控制表。

```bash
db2 -tvmf $HOME/asncdctools/src/asncdctables.sql
```

### 2.5 启用表添加和删除的 UDF
运行以下 SQL 脚本启用将表添加到捕获模式并从捕获模式中删除表的 UDF。

```bash
db2 -tvmf $HOME/asncdctools/src/asncdcaddremove.sql
```

## 3. 将表置于捕获模式

### 3.1 启动 ASN 代理
使用以下命令启动 ASN 代理。

```sql
VALUES ASNCDC.ASNCDCSERVICES('start','asncdc');
```

根据返回的结果，您可能需要在终端窗口中输入指定的命令，例如：

```bash
/database/config/db2inst1/sqllib/bin/asncap capture_schema=asncdc capture_server=SAMPLE &
```

### 3.2 将表置于捕获模式
对每个要置于捕获模式的表，调用以下命令。将 `MYSCHEMA` 和 `MYTABLE` 替换为实际的架构名称和表名称。

```sql
CALL ASNCDC.ADDTABLE('MYSCHEMA', 'MYTABLE');
```

### 3.3 重新初始化 ASN 服务
使用以下命令重新初始化 ASN 服务。

```sql
VALUES ASNCDC.ASNCDCSERVICES('reinit','asncdc');
```

## 4. 验证和后续操作

### 4.1 验证表是否处于捕获模式
通过查询相关系统表或使用 Db2 命令行工具验证表是否已成功置于捕获模式。例如，可以查询 `SYSCAT.CAPTURES` 表：

```sql
SELECT * FROM SYSCAT.CAPTURES WHERE CAPTURENAME = 'MYTABLE';
```

### 4.2 监控和调整
监控捕获代理的性能，根据需要调整捕获代理的配置参数，如 `COMMIT_INTERVAL` 和 `SLEEP_INTERVAL`，以平衡 CPU 负载和延迟。这些参数位于 `IBMSNAP_CAPPARMS` 表中。

```sql
-- 查看当前配置
SELECT * FROM IBMSNAP_CAPPARMS;

-- 更新 COMMIT_INTERVAL 和 SLEEP_INTERVAL
UPDATE IBMSNAP_CAPPARMS SET COMMIT_INTERVAL = 60, SLEEP_INTERVAL = 10;
```

---

通过上述步骤，您可以成功将表置于捕获模式，并确保 Debezium 能够捕获这些表的变更事件。
