#### bgsave工作过程
![bgsave工作过程](https://images2018.cnblogs.com/blog/1075473/201807/1075473-20180725172641202-1573986143.png)

1. 客户端执行bgsave命令，redis主进程收到指令并判断此时是否在执行bgrewriteaof(AOF文件重新过程，后续会讲解)，如果此时正好在执行则bgsave直接返回，不fork子进程，如果没有执行bgrewriteaof重写AOF文件，则进入下一个阶段；
2. 主进程调用fork方法创建子进程，在创建过程中redis主进程阻塞，所以不能响应客户端请求；
3. 子进程创建完成以后，bgsave命令返回“Background saving started”，此时标志着redis可以响应客户端请求了；
4. 子经常根据主进程的内存副本创建临时快照文件，当快照文件完成以后对原快照文件进行替换；
5. 子进程发送信号给redis主进程完成快照操作，主进程更新统计信息（info Persistence可查看）,子进程退出

#### RDB配置
```
save m n
#配置快照(rdb)促发规则，格式：save <seconds> <changes>
#save 900 1  900秒内至少有1个key被改变则做一次快照
#save 300 10  300秒内至少有300个key被改变则做一次快照
#save 60 10000  60秒内至少有10000个key被改变则做一次快照
#关闭该规则使用svae “” 

dbfilename  dump.rdb
#rdb持久化存储数据库文件名，默认为dump.rdb

stop-write-on-bgsave-error yes 
#yes代表当使用bgsave命令持久化出错时候停止写RDB快照文件,no表明忽略错误继续写文件。

rdbchecksum yes
#在写入文件和读取文件时是否开启rdb文件检查，检查是否有无损坏，如果在启动是检查发现损坏，则停止启动。

dir "/etc/redis"
#数据文件存放目录，rdb快照文件和aof文件都会存放至该目录，请确保有写权限

rdbcompression yes
#是否开启RDB文件压缩，该功能可以节约磁盘空间
```

#### AOF重写过程
![重写过程图](https://images2018.cnblogs.com/blog/1075473/201807/1075473-20180726171841786-525684493.png)

**aof_rewrite_buf**代表重写缓冲区，**aof_buf**代表写写命令存放的缓冲区

1. 开始bgrewriteaof，判断当前有没有bgsave命令(RDB持久化)/bgrewriteaof在执行，倘若有，则这些命令执行完成以后在执行。
2. 主进程fork出子进程，在这一个短暂的时间内，redis是阻塞的。
3. 主进程fork完子进程继续接受客户端请求，所有写命令依然写入AOF文件缓冲区并根据appendfsync策略同步到磁盘，保证原有AOF文件完整和正确。由于fork的子进程仅仅只共享主进程fork时的内存，因此Redis使用采用重写缓冲区(aof_rewrite_buf)机制保存fork之后的客户端的写请求，防止新AOF文件生成期间丢失这部分数据。此时，客户端的写请求不仅仅写入原来aof_buf缓冲，还写入重写缓冲区(aof_rewrite_buf)。
4. 子进程通过内存快照，按照命令重写策略写入到新的AOF文件。
	- 子进程写完新的AOF文件后，向主进程发信号，父进程更新统计信息。
	- 主进程把AOFaof_rewrite_buf中的数据写入到新的AOF文件(避免写文件是数据丢失)。
5. 使用新的AOF文件覆盖旧的AOF文件，标志AOF重写完成。

#### AOF配置参数
```
auto-aof-rewrite-min-size 64mb
#AOF文件最小重写大小，只有当AOF文件大小大于该值时候才可能重写,4.0默认配置64mb。

auto-aof-rewrite-percentage  100
#当前AOF文件大小和最后一次重写后的大小之间的比率等于或者等于指定的增长百分比，如100代表当前AOF文件是上次重写的两倍时候才重写。

appendfsync everysec
#no：不使用fsync方法同步，而是交给操作系统write函数去执行同步操作，在linux操作系统中大约每30秒刷一次缓冲。这种情况下，缓冲区数据同步不可控，并且在大量的写操作下，aof_buf缓冲区会堆积会越来越严重，一旦redis出现故障，数据
#always：表示每次有写操作都调用fsync方法强制内核将数据写入到aof文件。这种情况下由于每次写命令都写到了文件中, 虽然数据比较安全，但是因为每次写操作都会同步到AOF文件中，所以在性能上会有影响，同时由于频繁的IO操作，硬盘的使用寿命会降低。
#everysec：数据将使用调用操作系统write写入文件，并使用fsync每秒一次从内核刷新到磁盘。 这是折中的方案，兼顾性能和数据安全，所以redis默认推荐使用该配置。

aof-load-truncated yes
#当redis突然运行崩溃时，会出现aof文件被截断的情况，Redis可以在发生这种情况时退出并加载错误，以下选项控制此行为。
#如果aof-load-truncated设置为yes，则加载截断的AOF文件，Redis服务器启动发出日志以通知用户该事件。
#如果该选项设置为no，则服务将中止并显示错误并停止启动。当该选项设置为no时，用户需要在重启之前使用“redis-check-aof”实用程序修复AOF文件在进行启动。

appendonly no 
#yes开启AOF，no关闭AOF

appendfilename appendonly.aof
#指定AOF文件名，4.0无法通过config set 设置，只能通过修改配置文件设置。

dir /etc/redis
#RDB文件和AOF文件存放目录
```

#### 持久性方法优缺点分析

##### RDB
优点：
+ RDB 是一个非常紧凑（compact）的文件，体积小，因此在**传输速度上比较快，因此适合灾难恢复**。 
+ RDB 可以**最大化 Redis 的性能**：父进程在保存 RDB 文件时唯一要做的就是 fork 出一个子进程，然后这个子进程就会处理接下来的所有保存工作，父进程无须执行任何磁盘 I/O 操作。
+ RDB 在**恢复大数据集时的速度比 AOF 的恢复速度要快**。

缺点：
+ RDB是一个快照过程，**无法完整的保存所以数据**，尤其在数据量比较大时候，一旦出现故障丢失的数据将更多。
+ 当redis中数据集比较大时候，RDB由于RDB方式需要对数据进行完成拷贝并生成快照文件，fork的子进程会耗CPU，并且**数据越大，RDB快照生成会越耗时**。
+ RDB文件是特定的格式，**阅读性差**，由于格式固定，可能存在**不兼容**情况。

##### AOF　
优点：
+ **数据更完整**，秒级数据丢失(取决于设置fsync策略)。
+ **兼容性较高**，由于是基于redis通讯协议而形成的命令追加方式，无论何种版本的redis都兼容，再者aof文件是明文的，**可阅读性较好**。

缺点：
+ **数据文件体积较大**,即使有重写机制，但是在相同的数据集情况下，AOF文件通常比RDB文件大。
+ 相对RDB方式，AOF**速度慢**于RDB，并且在数据量大时候，恢复速度AOF速度也是慢于RDB。
+ 由于频繁地将命令同步到文件中，AOF持久化对性能的影响相对RDB较大，但是对于我们来说是可以接受的。
##### 混合持久化
优点：
+ 混合持久化**结合了RDB持久化和AOF持久化的优点**, 由于绝大部分都是RDB格式，加载速度快，同时结合AOF，增量的数据以AOF方式保存了，数据更少的丢失。

缺点：
+ **兼容性差**，一旦开启了混合持久化，在4.0之前版本都不识别该aof文件，同时由于前部分是RDB格式，阅读性较差

### redis的集群方式

#### 主从复制

##### 主从复制原理
1. 从服务器连接主服务器，发送SYNC命令； 
2. 主服务器接收到SYNC命名后，开始执行BGSAVE命令生成RDB文件并使用缓冲区记录此后执行的所有写命令； 
3. 主服务器BGSAVE执行完后，向所有从服务器发送快照文件，并在发送期间继续记录被执行的写命令； 
4. 从服务器收到快照文件后丢弃所有旧数据，载入收到的快照； 
5. 主服务器快照发送完毕后开始向从服务器发送缓冲区中的写命令； 
6. 从服务器完成对快照的载入，开始接收命令请求，并执行来自主服务器缓冲区的写命令；（从服务器初始化完成）
7. 主服务器每执行一个写命令就会向从服务器发送相同的写命令，从服务器接收并执行收到的写命令（从服务器初始化完成后的操作）

##### 主从复制优缺点
**优点：**
+ 支持主从复制，主机会自动将数据同步到从机，可以进行读写分离
+ 为了分载Master的读操作压力，Slave服务器可以为客户端提供只读操作的服务，写服务仍然必须由Master来完成
+ Slave同样可以接受其它Slaves的连接和同步请求，这样可以有效的分载Master的同步压力。
+ Master Server是以非阻塞的方式为Slaves提供服务。所以在Master-Slave同步期间，客户端仍然可以提交查询或修改请求。
+ Slave Server同样是以非阻塞的方式完成数据同步。在同步期间，如果有客户端提交查询请求，Redis则返回同步之前的数据

**缺点：**
+ Redis不具备自动容错和恢复功能，主机从机的宕机都会导致前端部分读写请求失败，需要等待机器重启或者手动切换前端的IP才能恢复。
+ 主机宕机，宕机前有部分数据未能及时同步到从机，切换IP后还会引入数据不一致的问题，降低了系统的可用性。
+ Redis较难支持在线扩容，在集群容量达到上限时在线扩容会变得很复杂

#### 哨兵方案：
![正常状态](https://img-blog.csdn.net/20180827220230340?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1JpY2hhcmRseWdv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
![主机下线](https://img-blog.csdn.net/20180827220235769?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1JpY2hhcmRseWdv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
![将从服务器升级为新的主服务器](https://img-blog.csdn.net/20180827220241156?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1JpY2hhcmRseWdv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

##### 哨兵工作方案
+ 每个Sentinel（哨兵）进程以每秒钟一次的频率向整个集群中的Master主服务器，Slave从服务器以及其他Sentinel（哨兵）进程发送一个 PING 命令。
+ 如果一个实例（instance）距离最后一次有效回复 PING 命令的时间超过 down-after-milliseconds 选项所指定的值， 则这个实例会被 Sentinel（哨兵）进程标记为**主观下线（SDOWN）**
+ 如果一个Master主服务器被标记为主观下线（SDOWN），则正在监视这个Master主服务器的所有 Sentinel（哨兵）进程要以每秒一次的频率确认Master主服务器的确进入了主观下线状态
+ 当有足够数量的 Sentinel（哨兵）进程（大于等于配置文件指定的值）在指定的时间范围内确认Master主服务器进入了主观下线状态（SDOWN）， 则Master主服务器会被标记为**客观下线（ODOWN）**
+ 在一般情况下， 每个 Sentinel（哨兵）进程会以每 10 秒一次的频率向集群中的所有Master主服务器、Slave从服务器发送 INFO 命令。
+ 当Master主服务器被 Sentinel（哨兵）进程标记为客观下线（ODOWN）时，Sentinel（哨兵）进程向下线的 Master主服务器的所有 Slave从服务器发送 INFO 命令的频率会从 10 秒一次改为每秒一次。
+ 若没有足够数量的 Sentinel（哨兵）进程同意 Master主服务器下线， Master主服务器的**客观下线状态就会被移除**。若 Master主服务器重新向 Sentinel（哨兵）进程发送 PING 命令返回有效回复，Master主服务器的**主观下线状态就会被移除**。

##### 哨兵模式的优缺点
**优点：**
+ 哨兵模式是基于主从模式的，所有主从的优点，哨兵模式都具有。
+ 主从可以自动切换，系统更健壮，可用性更高。

**缺点：**
+ Redis较难支持在线扩容，在集群容量达到上限时在线扩容会变得很复杂。
