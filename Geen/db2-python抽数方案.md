# db2-python抽数方案



## 包和脚本部署位置

```shell
# clidriver包（重命名为db2_client，与oracle和mysql客户端保持一致）
/wmdpapp/db2_client
# python包
/data/zdy_shell/site-packages/ibm_db-3.1.4-py2.7-linux-x86_64.egg
# python脚本
/data/zdy_shell/exportDb2Db.py	# 主脚本
/data/zdy_shell/exportDb2Oper.py	# db2抽数脚本
```



## 关键配置

### 环境变量

```shell
# clidriver根目录
export DB2_HOME=/wmdpapp/db2_client
# 依赖目录
export LD_LIBRARY_PATH=$DB2_HOME/lib64:$DB2_HOME/lib:$LD_LIBRARY_PATH
```



### python包引入

```python
import sys
sys.path.insert(0, '/data/zdy_shell/site-packages/ibm_db-3.1.4-py2.7-linux-x86_64.egg')
```



## 主要思路

因为原有的oracle和mysql脚本框架基本可以挪用，所以主要是修改连接信息和替换变量名

```python
# ibm_db_dbi的连接串不是关键字，而是拼接的ODBC字符串，所以连接串是以下形式的
# 因为抽数采用游标分批抽数的方式，只有dbi支持游标，所以用该方法
# DB-API 2.0标准的调用方式，后两个空字符串是满足 DB-API 签名的合法参数值,
ibm_db_dbi.connect("DATABASE=%s;" % _db +
                   "HOSTNAME=%s;" % _host +
                   "PORT=%s;" % _port +
                   "PROTOCOL=TCPIP;" +
                   "UID=%s;" % _user +
                   "PWD=%s;" % _passwd
                   , "", ""
                  )
```





## 主要问题和解决方案

1. 元数据校验出现字段数对不上的情况

   排查思路

   ```sql
   -- 查字段名的SQL
   SELECT GROUP_CONCAT(col) AS cols
   FROM (
   ...
   FROM table_column
       WHERE table_id = ...
       ORDER BY column_num ASC) t
   -- 当与里面子查询进行对比时，发现是外部查询GROUP_CONCT的字段查出来只有39个字段，而完整字段数是50个
   -- 更重要的是拼出来的字段名，末尾是被截断的，并不完整
   -- 最终锁定是group_concat默认字节是1024，字段超长被截断
   ```

   解决方案：

   ```python
   # exportDb2Db.py
   # 这里是只抽取出原始字段名命名为入库字段名的元素插入到列表中，之后改为join拼接元素成为一个整体字符串
   _sql="""
   SELECT concat(`COLUMN_NAME` " AS ", `DCL_COLUMN_NAME`) AS cols
   FROM table_column
   WHERE table_id={TABLE_ID}
   ORDER BY column_num ASC
   """
   ...
   for _row in _res:
       _cols.append(_row["cols"])
   _cols = ", ".join(_cols)
   ```

2. 空文件出现文件0b，但多一个换行符且元数据文件中文件大小为1b

- 发现导出后有对行数判断小于1时写入一个换行符的逻辑，该行代码注销或删掉即可

  

3. 数据文件由于字段值有换行符出现字段数对不上的情况

   排查思路：

- 当时是觉得re.sub替换的原始逻辑不对，导致换行符替换没生效，后来验证却不是；

- 之后猜测是按行抽取时，换行符将一行数据隔开为两行，但后续验证实际上是按每行的每个字段值读取的；

- 最后通过验证字段值的类型，锁定字符串值为unicode格式，未进入正则替换的分支，原判断是str

   解决方案：
   ```python
   # 修改后代码
   if _obj and isinstance(_obj, unicode):
       _r_obj = re.sub(r'[\r\n]', '', _obj)