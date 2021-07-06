
# 数据读写

### 1. 拼 sql：单表单条拼多表，多条拼多表（示例 SQL）

##### 单条拼多表

- INSERT INTO tablename1 [(tb1_fieldname1,...)] [USING stable_name TAGS(tag_value1,...)] VALUES(ts1, v1, v2,...)
- - - - - tablename2 [(tb2_fieldname1,...)] [USING stable_name TAGS(tag_value1,...)] VALUES(ts1, v1, v2,...)
- - - - - ...
- - - - - tablenamen [(tbn_fieldname1,...)] [USING stable_name TAGS(tag_value1,...)] VALUES(ts1, v1, v2,...);

##### 多条拼多表

- INSERT INTO tablename1 [(tb1_fieldname1,...)] [USING stable_name TAGS(tag_value1,...)] VALUES(ts1, v1, v2,...)(ts2, v1, v2,...)...(tsn, v1, v2,...)
- - - - - tablename2 [(tb2_fieldname1,...)] [USING stable_name TAGS(tag_value1,...)] VALUES(ts1, v1, v2,...)(ts2, v1, v2,...)...(tsn, v1, v2,...)
- - - - - ...
- - - - - tablenamen [(tbn_fieldname1,...)] [USING stable_name TAGS(tag_value1,...)] VALUES(ts1, v1, v2,...)(ts2, v1, v2,...)...(tsn, v1, v2,...);

### 2. taosc vs restful vs stmt

- taosc-stmt > taosc-sql > jni > restful

- taosc-stmt仅支持C/C++、Java连接器，其他语言暂不支持。taosc-stmt通过原生接口交互，以二进制方式写入性能最优，taosc-sql优于jni优于restful，restful要多走一层http协议，写入性能约为taosc的70%，可以满足绝大部分场景。

### 3. mybatis vs jdbc template vs jdbc

- JDBC > JDBC template > mybatis

### 4. jdbc 版本适配一览表、兼容性说明

- 
| TDegine | jdbc-Driver | JDK |
|:-------:|:-----------:|:-------:|
| v2.1.3.X| v2.0.31     |1.8.X    |
| v2.0.18.0 - v2.1.2.x | v2.0.22 - v2.0.30 | 1.8.X |
| v2.0.8.0 - v2.0.17.x | v2.0.12 - v2.0.21 | 1.8.X |
| v2.0.0.0 - v2.0.7.x | v2.0.4 - v2.0.11 | 1.8.X |
| v1.6.1.x 及以上 | v1.0.1-v1.0.3 | 1.8.X |

- 客户端版本需要与服务端版本保持一致
- TDengine版本号分为四段，客户端与服务端需保证前三段相同，才能正常工作。例如2.1.3.0的服务端可以和2.1.3.2的客户端通信，但不能与2.1.2.0通信。







