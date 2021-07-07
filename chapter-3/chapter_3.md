### 三、数据读写

#### 3.1 写在前面

本章主要分析采取什么方式读写TDengine更佳。

TDengine支持多种接口写入数据，包括SQL，Prometheus，Telegraf，EMQ MQTT Broker，HiveMQ Broker，CSV文件等。

数据可以单条插入，也可以批量插入，可以插入一个数据采集点的数据，也可以同时插入多个数据采集点的数据。支持多线程插入，支持时间乱序数据插入，也支持历史数据插入。

要提高写入效率，需要**批量写入**。一批写入的记录条数越多，插入效率就越高。但一条记录不能超过16K。

**全列写入速度会远快于指定列**，因此建议尽可能采用全列写入方式，此时空列可以填入NULL。

写入的数据的时间戳必须大于当前时间减去配置参数keep的时间。如果keep配置为3650天，那么无法写入比3650天还老的数据。写入数据的时间戳也不能大于当前时间加配置参数days。如果days配置为2，那么无法写入比当前时间还晚2天的数据。

对同一张表，如果新插入记录的时间戳已经存在，默认（没有使用 UPDATE 1  创建数据库）新记录将被直接抛弃。如果在创建数据库时使用 UPDATE 1 选项，插入相同时间戳的新记录将**覆盖**原有记录。

目前，TDengine**不支持删除表中数据**，可以删除整个表。

乱序写入是需要严谨评估的一个点，TDengine允许少量乱序数据写入比如(5%~10%)，这样写入速率受到的影响很小，但如果写入数据存在大量的乱序，将严重影响写入速度。

说明：

TDengine不支持delete、truncate、update（只能设置数据库参数update）语句。



应用通过C/C++，JDBC。GO，或者Python Connector执行SQL插入语句来写入数据，用户还可以通过TAOS Shell，手动输入SQL Insert语句来写入。

#### 3.2 基本Insert语法

- **插入一条记录**

  ```mysql
  INSERT INTO d1001 VALUES ('2018-10-03 14:38:04.500', 10.3, 219, 0.31);
  ```

  往表d1001插入一条数据

  

- **插入一条记录，数据对应到指定的列**

  ```mysql
  INSERT INTO d1001 (ts,current) VALUES ('2018-10-03 14:38:04.600', 7.9);
  ```
  
  往表d1001指定的ts，current列插入一条数据

#### 3.3 拼SQL

- **一张表插入多条记录**

  ```mysql
  INSERT INTO d1001 VALUES 
  ('2018-10-03 14:38:05.500', 10.2, 220, 0.23) 
  ('2018-10-03 14:38:06.500', 10.3, 218, 0.25);
  ```

  **注意**：在使用“插入多条记录”方式写入数据时，不能把第一列的时间戳取值都设为now，否则会导致语句中的多条记录使用相同的时间戳，于是就可能出现相互覆盖以致这些数据行无法全部被正确保存。

  

- **按指定的列插入多条记录**

  ```mysql
  INSERT INTO d1001 (ts, voltage) VALUES ('2018-10-03 14:40:06.500',220 ) ('2018-10-03 14:41:06.500',219 ) ;
  ```




- **向多个表插入单条记录**

  典型场景

  ```sql
  INSERT INTO 
  d1001 VALUES ('2018-10-03 14:38:09.500', 10.3, 219, 0.31) 
  d1002 VALUES ('2018-10-03 14:39:02.500', 12.3, 221, 0.31);
  ```

  

- **向多个表插入多条记录**

  典型场景

  ```mysql
  INSERT INTO 
  d1001 VALUES ('2018-10-03 14:38:09.500', 10.3, 219, 0.31) ('2018-10-03 14:39:01.500', 12.6, 218, 0.33) 
  d1002 VALUES ('2018-10-03 14:39:02.500', 12.3, 221, 0.31) ('2018-10-03 14:39:03.500', 12.3, 221, 0.31) ;
  ```

  同时向表d1001和d1002中分别插入多条记录。

  

- **同时向多个表按列插入多条记录**

  ```mysql
  INSERT INTO 
  d1001 (ts, current) VALUES ('2018-10-04 14:39:02.500', 13.3) ('2018-10-04 14:39:03.500', 11.3) 
  d1002 (ts, current) VALUES ('2018-10-04 14:40:02.500', 12.3) ('2018-10-04 14:41:02.500', 13.3);
  ```

  同时向表d1001和d1002中按列分别插入多条记录。  

  

  #### 自动建表

  ```mysql
  INSERT INTO tb_name USING stb_name TAGS (tag_value1, ...) VALUES (field_value1, ...);
  ```

  如果用户在写数据时并不确定某个表是否存在，此时可以在写入数据时使用自动建表语法来创建不存在的表，若该表已存在则不会建立新表。自动建表时，要求必须以超级表为模板，并写明数据表的 TAGS 取值。

- **插入记录时自动建表，并指定具体的 TAGS 列**

  ```mysql
  INSERT INTO tb_name USING stb_name (tag_name1, ...) TAGS (tag_value1, ...) VALUES (field_value1, ...);
  ```

  在自动建表时，可以只是指定部分 TAGS 列的取值，未被指定的 TAGS 列将取为空值。

- **同时向多个表按列插入多条记录，自动建表**

  ```mysql
  INSERT INTO 
  tb1_name (tb1_field1_name, ...) [USING stb1_name TAGS (tag_value1, ...)] 
  VALUES (field1_value1, ...) (field1_value2, ...) ...            
  tb2_name (tb2_field1_name, ...) [USING stb2_name TAGS (tag_value2, ...)] 
  VALUES (field1_value1, ...) (field1_value2, ...) ...;
  ```

  以自动建表的方式，同时向表tb1_name和tb2_name中按列分别插入多条记录。

  

  从 2.0.20.5 版本开始，子表的列名可以不跟在子表名称后面，而是可以放在 TAGS 和 VALUES 之间，例如像下面这样写：

  ```mysql
  INSERT INTO tb1_name [USING stb1_name TAGS (tag_value1, ...)] (tb1_field1_name, ...) VALUES (field1_value1, ...) (field1_value2, ...) ...;
  ```

  注意：虽然两种写法都可以，但并不能在一条 SQL 语句中混用，否则会报语法错误。

- **历史数据写入**

  可使用IMPORT或者INSERT命令，IMPORT的语法，功能与INSERT完全一样。

  ```SQL
  INSERT INTO table_name USING stable_name TAGS(tag_value1, ...) VALUES(values1,...); 
  ```

  



#### 3.4 数据查询

##### 查询语法

```mysql
SELECT select_expr [, select_expr ...]
        FROM {tb_name_list}
        [WHERE where_condition]
        [SESSION(ts_col, tol_val)]
        [STATE_WINDOW(col)]
        [INTERVAL(interval_val [, interval_offset]) [SLIDING sliding_val]]
        [FILL(fill_mod_and_val)]
        [GROUP BY col_list]
        [ORDER BY col_list { DESC | ASC }]
        [SLIMIT limit_val [SOFFSET offset_val]]
        [LIMIT limit_val [OFFSET offset_val]]
        [>> export_file];
```



##### 查询示例

```mysql
taos> SELECT d1001.* FROM d1001,d1003 WHERE d1001.ts = d1003.ts;
               ts            |       current        |   voltage   |        phase         |
======================================================================================
     2018-10-03 14:38:05.000 |             10.30000 |         219 |              0.31000 |
Query OK, 1 row(s) in set (0.020443s)
```

更多查询语法使用、查询技巧、查询函数，详见https://www.taosdata.com/cn/documentation/taos-sql#insert



##### 关联查询

| 超级表关联查询                                               | 子表关联查询                                                 | 普通表关联查询                                               |
| :----------------------------------------------------------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| 连接条件必须包含时间戳列                           (timestamp join condition missing) | 可以在同一个超级表下关联，也可以跨超级表进行关联查询         | 只能使用timestamp进行关联，不能使用普通列关联(not support ordinary column join) |
| 连接条件不能使用普通列                                          (not support ordinary column join) | 只能使用timestamp作为关联条件    (not support ordinary column join) | 可以最多进行10个表关联                                       |
| 如果超级表下面有多张子表的tag完全一样，不能使用关联查询(Duplicated join key) | 可以最多进行10个表关联                                       |                                                              |
| 每个超级表只支持一个tag用于连接(only support one join tag for each table) |                                                              |                                                              |
| 可以最多进行10个表关联                                       |                                                              |                                                              |



##### 边界限制

- 数据库名最大长度为 32
- 表名最大长度为 192，每行数据最大长度 16k 个字符（注意：数据行内每个 BINARY/NCHAR 类型的列还会额外占用 2 个字节的存储位置）
- 列名最大长度为 64，最多允许 1024 列，最少需要 2 列，第一列必须是时间戳
- 标签名最大长度为 64，最多允许 128 个，可以 1 个，一个表中标签值的总长度不超过 16k 个字符
- SQL 语句最大长度 65480 个字符，但可通过系统配置参数 maxSQLLength 修改，最长可配置为 1M
- SELECT 语句的查询结果，最多允许返回 1024 列（语句中的函数调用可能也会占用一些列空间），超限时需要显式指定较少的返回数据列，以避免语句执行报错。
- 库的数目，超级表的数目、表的数目，系统不做限制，仅受系统资源限制
- 支持对标签、TBNAME进行GROUP BY操作，也支持普通列进行GROUP BY，前提是：仅限一列且该列的唯一值小于10万个。
- IS NOT NULL支持所有类型的列。不为空的表达式为 <>""，仅对非数值类型的列适用。



#### 3.4 写入方式对比

taos-jdbcdriver 的实现包括 2 种形式： JDBC-JNI 和 JDBC-RESTful（taos-jdbcdriver-2.0.18 开始支持 JDBC-RESTful）

taosc-stmt，参数绑定API，以二进制方式写入。通过免除SQL语法解析消耗大幅提升了写入性能。taosc-stmt仅支持C/C++、Java连接器，其他语言暂不支持。

JDBC-JNI 通过调用客户端 libtaos.so（或 taos.dll ）的本地方法实现。

JDBC-RESTful 则在内部封装了 RESTful 接口实现。

<table >
<tr align="center"><th>对比项</th><th>JDBC-JNI</th><th>JDBC-restful</th></tr>
<tr align="center">
	<td>支持的操作系统</td>
	<td>linux、windows</td>
	<td>全平台</td>
</tr>
<tr align="center">
	<td>是否需要安装client</td>
	<td>需要</td>
	<td>不需要</td>
</tr>
<tr align="center">
	<td>server升级后是否需要升级client</td>
	<td>需要</td>
	<td>不需要</td>
</tr>
<tr align="center">
	<td>写入性能</td>
	<td colspan="2">JDBC-restful是JDBC-JNI的50%～90%</td>
</tr>
<tr align="center">
	<td>查询性能</td>
	<td colspan="2">JDBC-restful与JDBC-JNI没有差别</td>
</tr>
</table>



#### 3.5 应用程序框架对比

JDBC已经能满足大部分用户最基本的需求，但是在使用JDBC时，必须自己来管理数据库资源如：获取PreparedStatement，设置SQL语句参数，关闭连接等步骤。

JdbcTemplate是Spring对JDBC的封装，目的是使JDBC更加易于使用。JdbcTemplate是Spring的一部分。

JdbcTemplate处理了资源的建立和释放。它帮助我们避免一些常见的错误。比如忘了总要关闭连接。它运行核心的JDBC工作流，如Statement的建立和执行，而我们只需要提供SQL语句和提取结果。

- ORM普遍会拖慢Java写入TDengine性能
- 高性能写入时，建议放弃ORM，直接调用JDBC实现TDengine的写入
- ORM写入效率：JDBC Template 优于 Mybatis



#### 3.6 连接池配置

- 建议maxLifeTime设为0(TDengine连接器连接时长无限制，永久有效)

- connectionTestQuery建议采用：select server_status()

- 可设置固定大小的连接池，即最小、最大连接数相等

- HikariCP配置示例:

  ```shell
  maximumPoolSize=100 (可调整)
  minimumIdle=100 (可调整)
  connectionTimeOut=30000
  maxLifetime=0
  idleTimeout=0
  ```



#### 3.7 TDengine与jdbcdriver版本对应表

TDengine企业版与社区版均遵循以下对应关系：

| taos-jdbcdriver 版本 | TDengine 版本      | JDK 版本 |
| -------------------- | ------------------ | -------- |
| 2.0.31               | 2.1.3.0 及以上     | 1.8.x    |
| 2.0.22 - 2.0.30      | 2.0.18.0 - 2.1.2.x | 1.8.x    |
| 2.0.12 - 2.0.21      | 2.0.8.0 - 2.0.17.x | 1.8.x    |
| 2.0.4 - 2.0.11       | 2.0.0.0 - 2.0.7.x  | 1.8.x    |
| 1.0.3                | 1.6.1.x 及以上     | 1.8.x    |
| 1.0.2                | 1.6.1.x 及以上     | 1.8.x    |
| 1.0.1                | 1.6.1.x 及以上     | 1.8.x    |

客户端版本需要与服务端版本保持一致。

TDengine版本号分为四段，客户端与服务端需保证前三段相同，才能正常工作。例如2.1.3.0的服务端可以和2.1.3.2的客户端通信，但不能与2.1.2.0通信



