#### bgsave工作过程
![bgsave工作过程](https://images2018.cnblogs.com/blog/1075473/201807/1075473-20180725172641202-1573986143.png)

1. 客户端执行bgsave命令，redis主进程收到指令并判断此时是否在执行bgrewriteaof(AOF文件重新过程，后续会讲解)，如果此时正好在执行则bgsave直接返回，不fork子进程，如果没有执行bgrewriteaof重写AOF文件，则进入下一个阶段；
2. 主进程调用fork方法创建子进程，在创建过程中redis主进程阻塞，所以不能响应客户端请求；
3. 子进程创建完成以后，bgsave命令返回“Background saving started”，此时标志着redis可以响应客户端请求了；
4. 子经常根据主进程的内存副本创建临时快照文件，当快照文件完成以后对原快照文件进行替换；
5. 子进程发送信号给redis主进程完成快照操作，主进程更新统计信息（info Persistence可查看）,子进程退出

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
