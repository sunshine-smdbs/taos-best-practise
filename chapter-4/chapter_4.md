# TDengine最佳实践-第四章



## 四、性能调优

<!--为方便区别，文中CACHE为操作系统内存缓存区，cache为TDengine内存缓冲区-->

在性能调优前，需要理解TDengine与操作系统的运行原理。数据进入到TDengine后首先会使用write()函数写入wal日志，然后将数据放入TSDB中的cache中。当cache存储的数据超过1/3时，会触发fsync()进行落盘，同时切换wal日志。

- write()函数会将数据写入操作系统的Cache中，不会马上落盘，只有当满足操作系统的条件[1] 后才会写入磁盘。
- TSDB中的cache通过系统参数cahce和blocks共同决定，cacheSize=cache*blocks，注意这部分内存仅用于数据写入。
- fsync()函数会确保数据写入磁盘。
- 当cache中的数据小于minRows时，TSDB会将数据放入last文件，与下次的数据一起合并写入。
- 当cache中的数据大于maxRows时，TSDB会将数据进行分割存储。

与传统关系型数据库不同TDengine采用列式存储来保证压缩率和存储速度。因此数据在磁盘的存储结构为col1{row1,row2...rown},col2{row1,row2...rown}.....coln{row1,row2...rown}。

TDengine进行查询数据时，会先访问mnode获取表的基础信息，然后将分布在不同dnode中的数据在当前节点汇总后发送给客户端。当前节点会在内存中缓存所有查询数据。

### 4.1.磁盘最慢写入速度 vs 模型校核

##### 写入速度

众所周知，磁盘在大数据块连续写入时速度最快，小数据块随机写入时速度最慢。对于TDengine而言，写入磁盘的数据中wal日志占绝大部分（TSDB数据具有很高的压缩率），而wal日志是通过write()进行写入的。因此wal日志的写入速度与磁盘IO、内存大小、操作系统参数具有直接关系。可以使用如下语句对磁盘进行测试，获得随机写的速度。在进行性能估算时，可以以此速度为基准。

```shell
fio -ioengine=sync -direct=0 -thread -rw=randwrite -filename=/taos/test -runtime=600 -numjobs=1 -filesize=20g -bsrange=4k-500k -loop 1000 -name="Test"
```

```shell
-ioengine=sync fio存储引擎，sync为系统默认落盘模式。TDengine直接调用write()和fsync()函数进行落盘，不会使用其他引擎。
-direct=0 是否直接写入磁盘。1 直接写入，0 不直接写，使用OS_buffer。TDengine使用的落盘函数均使用OS buffer。
-rw=randwrite 使用随机写模式。TDengine落盘是会写wal日志、data、last等多个文件，因此采用随机写更符合TDengine的实际写入。
-numjobs=1 并行进程数。在不使用异步io时，多进程不会对写入速度产生影响。
-filesize=20g 测试文件大小，如果设置了runtime，则在到期前反复写入。该值应接近可用存储空间大小，至少要大于内存大小。
-bsrange=4k-500k 单次IO块大小，IO块大小随机在4k-500k内选择。
-runtime=600 测试时间单位秒（s）。测试时间越长越接近于真实环境，要求不低于10分钟。
-loop 1000 单次写入完成后循环1000次，受runtime控制。
-name="Test" 测试名称
```

##### 模型校核

TDengine落盘的文件包括wal日志和TSDB数据，由于TDengine进行压缩，TSDB数据会远小于wal日志。wal日志由操作系统定时写入磁盘，写入的数据是连续的。TSDB落盘时会对数据进行拼接，对于数据文件进行追加，对于单个数据文件属于连续写入，当数据文件较多时，写入行为呈现随机写入。在上一小节我们获得了随机写的速度。那么可由此估算TDengine的数据落盘速度：

```shell
rowSpeed=DiskIO/[rowSize*（1+comPercent）]
```

```shell
rowSpeed #每秒写入记录数
DiskIO #上一小节获得的磁盘写入速度
rowSize #单条记录大小
comPercent #数据压缩率
```

<!--在数据落盘阶段，数据模型对写入速度影响不大。不论是单列还是多列模型，在落盘时都会进行压缩和拼接。列越长消耗的CPU越高。-->

<!--由于wal日志落盘会使用操作系统CACHE，因此CACHE大小会影响wal日志的写入速度。当操作系统保有足够的CACHE时，数据会直接写入内存，此时的写入速度接近内存的写入速度。而当CACHE不足时，数据的写入速度则会与磁盘写入速度持平。-->

### 4.2.平衡cache大小与碎片化与oom

#### cahce

TDengine中的cache主要用于缓存数据，单个cache大小为16MB。在本章开头中提到如果落盘时cache中不足minRows会放入last文件中，在下次写入时进行合并。如此来说，每次TSDB落盘时都会读取last文件，如果last文件中内容过多，必然会影响系统的性能。因此需要合理设置cache，尽量减少写入到last文件的数据。

cache大小的基本估算原则为：tableNums x minRows x 3，这只是一个最基础的大小，cache中还会存储表的元数据，因此在计算cache时需要再上浮一些。

那么cache是否越大越好呢？也不尽然，当cache中存储了大量数据，而系统突然掉电，这部分数据就会损失掉。因此设置cache时还要考虑到数据安全。

cache设置需要遵循以下两个基本原则：

- cache的最小值需要能包含所有表的minRows；
- cahce的最大值不能超过操作系统内存的50%。

#### 碎片化

碎片化是指数据存储在磁盘上时不连续、不规律、块过小等。造成碎片化的原因有多种，包括但不限于以下情况：

- 同一个vnode中表的结构相差悬殊；
- 同一个database中采集间隔不一致。

碎片化的危害：

- 落盘占用空间变大
- 查询效率低下

碎片化后如何除处理:

- 将数据导出后重新导入；
- 使用拼接工具(工具开发暂未完成)。

#### OOM

OOM是操作系统的保护机制，但操作系统内存(包括swap)不足时，会杀掉某些进程，从而保证操作系统的稳定运行。以上语句涉及多个概念：

##### 内存不足如何判定？

通常内存不足包含两方面，一是剩余内存小于vm.min_free_kbytes；二是程序请求的内存大于剩余内存。还有一种情况是程序占用了特殊的内存地址。

##### 内存不足时杀谁？

先杀占用内存暴增的，然后杀oom分数高的。影响评分的因素包括占用内存大小和程序优先级[3]。

##### 如何防止OOM？

造成OOM的原因不只是某个程序内存占用过多，还有部分责任是操作系统的过度分配(over-commit memory)，由以下参数控制：

```shell
vm.overcommit_kbytes = 0
vm.overcommit_memory = 0
vm.overcommit_ratio = 50
```

但是不建议修改以上参数。

要防止OOM，需要在项目建设之初合理规划内存，并合理设置swap。

### 4.3.如何查看碎片化程度 _block_dist

语法：

```sql
SELECT _BLOCK_DIST FROM stable_name \G;
```

输出结果：

```shell
block_dist: summary: 
//数据块中存储数据分布
5th=[216], 10th=[216], 20th=[216], 30th=[216], 40th=[232], 50th=[232] 60th=[232], 70th=[232], 80th=[248], 90th=[248], 95th=[248], 99th=[248]
//数据块中存储数据条数汇总
Min=[210(Rows)] Max=[250(Rows)] Avg=[231(Rows)] Stddev=[11.58]  
//总数据条数
Rows=[16211], 
//总数据块数
Blocks=[70], 
//总数据大小
Size=[1441.520(Kb)] 
//压缩率
Comp=[0.83] 
//驻留内存中的数据条数
RowsInMem=[0] 
```

数据块中条数分布的越均匀，查询效率越高。



[1].操作系统CACHE中数据写入磁盘受以及几个参数控制

```shell
vm.dirty_background_ratio = 10  ##CACHE数据超过该值会启动异步写磁盘，此时CACHE仍可写入数据。
vm.dirty_ratio = 40   ##CACHE数据超过该值会触发同步写磁盘，写入CACHE被阻塞
vm.dirty_expire_centisecs = 3000   ##CAHCE中数据过期时间 30s
vm.dirty_writeback_centisecs = 500  ##每隔5s将CACHE中过期数据异步写入磁盘
```

[2].OOM启动后会首先杀掉内存暴增的程序，然后杀掉omm分数高的。oom分数查看

```shell
/proc/PID/oom_score
```

为防止程序被oom杀掉，可以手动调整oom评分，omm_adj取值范围为【-17，15】，默认0，值越小优先级越高。当设置为-17时，在该程序上oom失效。

```shell
/proc/PID/oom_adj
```



