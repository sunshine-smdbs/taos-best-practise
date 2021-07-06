# TDengine最佳实践



阅读本文档需要具备的基础知识：

- Linux系统的基础知识，及基本命令
- 网络基础知识：TCP/UDP、http、RESTful、域名解析、hostname、fqdn、hosts、防火墙、四层负载均衡

本文档的阅读对象有：架构师、研发工程师，及DBA、运维工程师



### 写在前面

TDengine是一时序大数据平台，核心是时序数据库(TSDB: Time-Seried DataBase)。虽然其CLI/API支持类SQL的语法来进行数据写入查询、数据库维护，但其本质上不是RDBMS(如MySQL、Oracle)。采用SQL语法的目的仅仅是为了减少读者的学习成本，切记！



与大多数数据库产品不同的是，TDengine客户端与服务端的连接方式。对于一般数据库来说，客户端发起的连接是至集群某一节点的TCP长连接，需要访问数据库服务时建立，使用完了即关闭。

TDengine设计思想是全分布式的：**以taosc连接方式为例**，客户端除首次获取集群的meta-data时需要连接到指定节点(firseEP/secondEP) 外，一旦获取到了meta-data，客户端驱动会将其缓存下来，后续客户端与服务端的交互由驱动根据需要建立一(客户端)对多(节点)的TCP/UDP连接。获得meta-data后，与指定节点firstEP的连接即刻释放。

应用通过taosc创建的taos_connect连接只是**虚拟连接**，作为应用访问集群的接口。最佳实践是：在应用生命周期内，该连接永远有效，无需频繁建立、关闭。仅当应用关闭前，释放连接即可。

这样设计的优点是：

- 不存在单点故障：firstEP/secondEP提供了客户端获取meta-data的备份功能；集群有多副本支持
- 不存在单点瓶颈：获取meta-data的时间短，后续客户端不用频繁访问firstEP
- 水平扩展简单便捷：增加新节点后，meta-data发生改变，客户端会按需动态刷新



#### **TDengine的重要特性**

- 超级表：带上静态属性标签(tags)的表结构描述，本身不存有数据。超级表用来作为模板创建子表/普通表，创建子表时指定对应的标签值即可。

- 时间戳列必须是第一列，一个表最少要有两列：第一列时间戳，另一列数据列。

- 索引：仅首列时间戳带索引(建表时自动建立)，其他列无论是普通列还是标签列，均无索引，也无法建索引。

- group by：仅对标签列、表名(tbname)有效，普通列仅当唯一值小于10万可用于group by。

- 写入数据：**按时间戳顺序批量写入**可以达到最佳写入速度，常用的方案有：单表单条拼多表，单表多条拼多表。三种接口的写入速度排序为：taosc-stmt > taosc-sql > RESTful。
  时序数据库仅保留指定时间的数据，写入时如报时间戳越界错误，意味着写入数据的时间戳太旧(超过保留期限)，或太新(写入的是未来的时间)。

  乱序写入需要认真评估。TDengine允许少量(5~10%的乱序写入，但如果写入的数据存在大量乱序，将严重影响写入速度。如数据是从消息队列(如kafka)消费入库，需确保每个采集量进入消息队列是按顺序进入指定的partition。详见  https://www.taosdata.com/blog/2019/10/08/922.html

- 三种接口：客户端可通过taosc-sql、taosc-stmt、RESTful三种接口访问服务端。其中taosc-sql和RESTful均通过sql语句交互，taosc-stmt通过原生接口交互，性能较优。

  目前taosc-stmt仅支持C/C++、Java连接器，其他语言暂不支持。

  RESTful接口写入性能与taosc-sq相比，性能约下降30%，可以满足绝大部分应用场景，推荐使用。

- 排序：默认支持的排序结果集为10万条记录，最大支持500万，可通过maxNumOfOrderedRes选项配置。

- 返回结果集：RESTful默认返回结果集上限为10240条，最大支持1000万，可通过restfulRowLimit选项配置。

- Update：数据库创建时，可以设置update开关，默认为关。打开时，如insert数据之时间戳已经存在，后面的数据将覆盖已存在的记录。如关闭，则丢弃后面写入的数据。

- 不支持手动删除指定记录，数据在超过保留天数后自动删除。但支持删除库、表的操作。

- 时钟：客户端与服务端需保持时钟同步。



### 一、安装部署篇

- #### 安装基本流程

  目前TDengine服务端仅支持Linux X64系统，推荐CentOS 7.9 和 Ubuntu 18.04。硬件平台支持X64和arm64。

  客户端支持Windows X64、Linux X64。

  mac版在开发中，将提供有限功能，用于开发环境。

  

  安装服务端之前，需做好以下准备：

  a) 准备好至少一台Linux X64服务器，可用磁盘空间足够

  b) 配置好hostname

  c) 网络环境正常，网络内TCP/UDP可正常通信

  d) 防火墙已关闭

  

  **服务端安装步骤**

  1) 从官网上下载tar.gz的安装包 https://www.taosdata.com/cn/getting-started/

  2) 上传至服务器指定目录，解压缩  ```sudo tar xzvf TDengine-server-2.1.3.2-Linux-x64.tar.gz```

  3) 进入解压后的子目录，安装  ```cd TDengine-server-2.1.3.2 && sudo ./install```

  4) 编辑 /etc/taos/taos.cfg，将firstEP和fqdn修改为本机的hostname【后续节点如需加入集群，需修改fqdn为该节点的hostname】

  5) 编辑/etc/hosts，将**集群所有节点**的域名解析添加进去(如已部署DNS server，则无需编辑)

  6) 启动taosd服务  ```sudo systemctl start taosd```

  7) 进入taos CLI查看数据节点  ```taos```   ```> show dnodes;```

  

  **配置文件 taos.cfg**

  典型的服务端配置文件

  ```
   firstEp                   node1:6030   //集群第一个节点
   fqdn                      node1				//本机hostname
  # arbitrator                arbi:6042		//双副本时需配置arbitrator，独立机器
   logDir										 /var/log/taos	//日志文件夹
   dataDir									 /var/lib/taos	//数据文件夹
   numOfThreadsPerCore       2.0					//每CPU核线程数
   ratioOfQueryCores         2.0					//每CPU核查询线程比例
   numOfCommitThreads        4.0					//落盘线程数
   minTablesPerVnode         1000					//每vnode创建表数初始值
   tableIncStepPerVnode      1000					//每vnode新增表数步长值
   keepColumnName            1						//保留原列名
   balance                   0						//关闭自动负载均衡
   blocks                    60						//每vnode写入缓存块数
   maxSQLLength              1048576			//最大SQL长度
   maxNumOfOrderedRes				 100000				//超级表排序结果集最大记录数，上限500万
   update                    1						//打开update，创建数据库时生效
   cachelast                 1						//打开cachelast，创建数据库时生效
   timezone                  Asia/Shanghai (CST, +0800)
   locale                    en_US.UTF-8
   charset                   UTF-8
   maxShellConns             50000
   maxConnections            50000
   monitor                   1						//打开内部log监控数据库
   restfulRowLimit					 10240				//RESTful返回结果集最大记录数，上限1000万
   logKeepDays               -1						//压缩保存一天日志
   debugflag                 131					//设置131日志开关，仅打印最小日志信息
  ```

  

  

- #### 客户端安装与配置

  如已安装服务端，已包含客户端驱动，无需安装。【如果在某台服务器上安装了TDengine服务端，将其作为客户端连接另外的taosd集群，请检查客户端与服务端版本是否一致】

  客户端安装步骤与服务端基本相同，解压缩后运行安装脚本即可完成。

  编辑/etc/hosts，将集群所有节点的域名解析添加进去(如已部署DNS server，则无需编辑)。确保客户端通过网络可以访问到服务端，如不能ping通服务端节点的fqdn，请联系贵司网络管理员。

  客户端安装成功后，需配置/etc/taos/taos.cfg才能访问TDengine服务：firstEP/secondEP。

  典型的客户端配置文件

  ```
   firstEp                   node1:6030		//集群第一个节点
   secondEp                  node2:6030		//集群第二个节点
   logDir										 /var/log/taos  //日志文件夹
   numOfThreadsPerCore       2.0
   maxSQLLength              1048576
   timezone                  Asia/Shanghai (CST, +0800)
   locale                    en_US.UTF-8
   charset                   UTF-8
   maxConnections            50000
   logKeepDays               -1
   debugflag                 131
  ```

  时区一般默认注释掉，将获取操作系统的当前时区作为taosd的时区。但某些情况下，需要手工指定时区时，可尝试指定为特定时区。【不是所有的操作系统均支持指定时区，建议事先确认后再实施】

  客户端建议配置logDir并**确保运行应用的账号拥有该文件夹访问权限**，否则日志将无法生成。

  

  一般情况下，建议安装TDengine客户端驱动。客户也可以不安装客户端，仅导入动态链接库也可实现完整的客户端驱动功能，只是缺少相关工具如taos CLI、taosdemo。默认taos.cfg路径为/etc/taos。

  Linux系统的方法如下：

  1) 拷贝动态链接库至/usr/local/taos/drivers   ```sudo cp libtaos.so.2.1.3.2 /usr/local/taos/drivers```

  2) 建立符号链接  

  ```
  sudo ln -s /usr/local/taos/drivers/libtaos.so.2.1.3.2 /lib/libtaos.so.1
  sudo ln -s /lib/libtaos.so.1 /lib/libtaos.so
  ```

  Windows系统同样支持，可将dll拷贝到c:\Windows\System32，默认taos.cfg路径为c:\TDengine\cfg





- #### 集群部署：扩容、缩容、迁移vnode

  #### 扩容 - 增加节点

  1) 第一个节点部署完成后，在后续节点依次安装TDengine服务端，编辑/etc/hosts，将**集群所有节点**的域名解析添加进去(如已部署DNS server，则无需编辑)

  2) 将第一节点taos.cfg拷贝至当前节点，配置taos.cfg，修改fqdn为本机hostname

  3) 启动本机taosd  ```sudo systemctl start taosd```

  4) 运行taos CLI，将本节点添加至集群  ```> create dnode 'node2:6030';```

  5) 查看数据节点列表  ```> show dnodes;```

  节点正常加入集群后，数据节点列表中会显示该节点处于ready状态。

  如该节点为offline状态，请检查：

  a) 该节点taosd是否已启动、防火墙是否关闭、数据文件夹是否清空；

  b) 所有节点/etc/hosts域名解析是否完整、正确；

  c) 该节点firstEP、fqdn是否正确配置。

  

  #### 缩容 - 减少节点

  缩容只能通过drop dnode来实现，直接停止taosd进程只能将该节点下线，不能完成缩容。切记！

  1) 运行taos CLI登入集群，查看集群节点信息  ```> show dnodes;```

  2) 从当前集群中删除指定节点  ```> drop dnode 'node2:6030';```

  3) 查看集群节点信息，确认待删除节点已从列表中剔除  ```> show dnodes;```

  集群在完成drop dnode操作之前，须将该dnode的数据迁移走。

  数据迁走需要一定时间，时间长短取决于该节点上的须迁出的数据量大小，以及网络带宽、磁盘IO。**在节点从列表中剔除之前，万不能停止该节点taosd服务**。

  一个节点被drop之后，不能立即重新加入集群。加入任何集群之前，需要将节点重新部署(清空数据文件夹、配置taos.cfg、重启taosd、将节点添加至集群)。

  

  #### 迁移vnode

  当一个数据库的某个vnode数据过热时，可手动迁移该vnode至指定的数据节点。

  该操作仅在自动负载均衡选项关闭(balance=0)时方可执行。

  1) 运行taos CLI登入集群，查看集群节点信息  ```> show dnodes;```

  2) 查看待数据库(mydb)虚拟节点组信息  ```> show mydb.vgroups;```

  3) 将当前节点(source-dnodeId)的vnode(vgId)，迁移至指定节点(dest-dnofeId)  ```> ALTER DNODE <source-dnodeId> BALANCE "VNODE:<vgId>-DNODE:<dest-dnodeId>";```

  

  

- #### Arbitrator

  当集群中数据库配置为双副本(replica=2)时，需部署arbitrator以防止集群某节点下线时，相关联的虚拟节点组重新选主遇到“脑裂”的问题。

  arbitrator须部署在一台独立的机器上，因占用资源很少，可以与其他服务公用一台机器。如需单独部署，1C2G的配置即可。

  

  

- #### 时间同步：客户端/服务器

  所有集群服务器，以及客户端机器均需保持时钟同步。

  生产环境中，无论是否能连接互联网，建议部署本地ntp服务器，采用ntpdate与ntpd结合的方式实现各机器之间的时钟同步。

  ```
  ntpdate cn.pool.ntp.org
  hwclock --systohc
  ntpd start
  ```

  

  

- #### 中文字符

  TDengine默认字符集为UTF-8。

  在Windows系统中，大多采用GBK/GB18030存储中文字符，TDengine的客户端驱动会将其统一转为UTF-8编码，发送至服务端存储。应用开发时，在调用接口时配置对当前的编码方式即可。

  在系统中，中文字符或其他大字符集字符需用nchar类型存储，不能用binary进行存储，否则在用taosdump工具导出导入时将出现乱码，导致不可修复的异常。

  

  

- #### mac版本环境搭建与使用

  mac版本目前尚未正式发布，仅用于开发研究使用。

  mac os X的版本要求：Catalina 或 Big Sur

  #### 安装步骤：

  1) 建议先安装XCode command line tool、git、brew，再通过brew安装cmake、tmux

  2) pull TDengine最新develop分支代码  ```git pull```

  3) 编译步骤：

  ```
  mkdir debug && cd debug
  cmake .. && cmake --build .
  ```

  4) 快速启动taosd并运行taos CLI(建议启动tmux，开两个窗口)

  ```
  build/bin/taosd -c test/cfg    //第一个tmux窗口
  build/bin/taos -c test/cfg		 //第二个tmux窗口
  ```

  

  mac作为客户端开发环境使用taosc连接服务端时，需用到动态链接库。该库位于：

  ```
  ./build/lib/libtaos.2.1.3.2.dylib
  ```

  同时该目录下有两个符号链接，最终均指向该文件

  ```
  ./build/lib/libtaos.1.dylib
  ./build/lib/libtaos.dylib
  ```






- #### 多级存储

  TDengine 2.0 **仅企业版**支持多级存储。

  多级存储支持三级：0级、1级、2级。每级支持16个挂载点。

  最热的数据放在0级，最冷的数据放在2级。

  如需在某一级存储需部署多块硬盘，部署方案建议采用：LVM 或 LVM+RAID5





- #### taos.cfg参数说明

  taos.cfg里面配置的参数在启动taosd时，作为默认参数载入运行实例。主要的参数有：

  ##### ratioOfQueryCores         每CPU核查询线程比例

  【全局参数】设置每个CPU核启动的查询线程与核数的比例

  ##### numOfCommitThreads        落盘线程数

  【全局参数】设置每个vnode落盘线程数，建议与挂载的硬盘数量一致

  ##### minTablesPerVnode         每vnode创建表数初始值

  【全局参数】每个vnode首次创建普通表的上限，默认为1000。如系统表数小于1000，须修改本参数。假设共500张表，集群CPU有24核，则建议设置round(500/24)=20

  ##### tableIncStepPerVnode     每vnode新增表数步长值

  【全局参数】每个vnode后续新增表的步长值，默认为1000。建议与minTablesPerVnode保持一致

  ##### keepColumnName            保留原列名

  【全局参数】当select语句带有函数时，默认返回结果列名为函数+列名。如需保留原列名，需打开本选项

  ##### balance         自动负载均衡

  【全局参数】默认关闭自动负载均衡

  ##### blocks             每vnode写入缓存块数

  【数据库参数】每个vnode配置的写入缓存计算公式为cache*blocks。cache默认为16MB，blocks=6，建议cache保持不变，通过修改blocks来调整写入缓存大小。

  该参数大小将影响每张表落盘的平均记录数，如希望达成更好的写入、查询性能，需设置尽可能大的blocks，但这也需要服务器配置更大的内存。

  该参数与数据库的超级表数量、超级表的平均行长度、子表数、副本数、虚拟节点组数相关。

  ##### maxSQLLength        最大SQL长度

  【全局参数】设置一条SQL语句最大的长度。默认65480，最大1048576。设置为较大值，将允许一条SQL语句拼接尽可能多组记录，或多个表的记录，以实现更快的写入速度。

  ##### update            更新

  【数据库参数】每个数据库可设置是否允许覆写已存在的记录，创建数据库时生效。打开本选项，写入相同时间戳的记录时，将覆写已存在的记录，反之，则丢弃后面写入的记录。

  ##### cachelast         内存中缓存最后一条数据

  【数据库参数】每个数据库可设置是否允许在内存中缓存每张表的最后一条数据，创建数据库时生效。

  ##### monitor            内部监控数据库

  【全局参数】打开本选项，会创建log库，用于内部监控。建议打开。

  ##### logKeepDays          压缩保存一天日志

  【全局参数】设置日志文件保存时长。0 - 仅保留两个日志文件；正数 - 保留天数，不压缩；负数 - 保留天数，压缩。

  ##### debugflag                日志级别开关

  【全局参数】设置日志级别。131 - 仅打印最小日志信息；135 - 打印详细日志信息；143 - 打印全量日志。

  ##### rpcForceTcp      强制使用TCP通信

  【全局参数】该选项打开后，所有从本节点发出的信息均通过TCP发出。
  
  关闭时，小于15KB走UDP，超过的走TCP。
  
  
  
  
  
- #### 常见问题

  ##### 部署了单机版TDengine，可以升级为集群吗？

  A  单机版就是一个节点的集群，请参看集群扩容章节进行扩容

  ##### 部署了两个单机版TDengine，可以合并为一个集群吗？

  A  不能。两个单机版是两个独立的集群，无法合并。只能将其中一台taosd停下后，清空数据文件夹，重新加入另一台的集群

  ##### TDengine已成功安装，也启动了，但taos CLI登录报错Unable to establish connection是怎么回事？

  A taosd服务没有正常启动，可能的原因有：

  a) taosd是否已启动、防火墙是否关闭、数据文件夹是否清空

  b) 所有节点/etc/hosts域名解析是否完整、正确

  c) 各节点firstEP、fqdn是否正确配置

  另外，可以通过以下步骤来查看客户端与服务端通信是否正常：

  a) 停止taosd，在服务端运行 ```taos -n server```

  b) 在客户端运行 ```taos -n client -h <server>```

  c) 查看哪些通信端口工作异常，再找网络管理员逐一排查

  ##### 如何读日志文件？

  A 服务端日志有taosinfo和taosdlog，客户端日志taoslog。

  用户无需读懂TDengine系统日志，在发生错误时，grep日志里面的ERROR信息，发送给涛思技术支持团队一般可判定大部分问题。

  某些情况，需要将ERROR发生的前20万行后5万行发给涛思进行分析定位。

  ##### 为何客户端连接服务端失败，报taos connect failed, reason: Invalid app version？

  A 客户端与服务端版本不一致所致。

  TDengine版本号分为四段，客户端与服务端需保证前三段相同，才能正常工作。例如2.1.3.0的服务端可以和2.1.3.2的客户端通信，但不能与2.1.2.0通信。

  

