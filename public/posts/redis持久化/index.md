# Redis持久化

每隔一段时间Redis会把数据持久化到硬盘进行数据的保存。其中持久化方案包括RDB和AOF模式  ，其中RDB全称为Redis DataBase，AOF全称为Append Only File。Redis默认使用的是RDB方式进行数据的持久化。
<!--more-->



### RDB

从英文的全称可以看出区别，RDB是把所有的数据进行以快照的方式进行持久化。快照文件的生成可以使用两个命令来完成，即`save`和`bgsave`。

```
127.0.0.1:6379> help save
  SAVE -
  summary: Synchronously save the dataset to disk
  since: 1.0.0
  group: server
127.0.0.1:6379> help bgsave
  BGSAVE -
  summary: Asynchronously save the dataset to disk
  since: 1.0.0
  group: server
```

从summary可以看出，`save`是以同步的方式把数据以快照方式写入磁盘，而`bgsave`是异步的写入。所谓同步即redis在执行rdb备份时，期间redis不能响应外部的请求直到完成本次备份。而异步是redis调用Linux的内核函数fork()生成一个子进程，并且由该子进程来完成数据的持久化，所以备份时redis仍然可以响应外部的请求。

#### 持久化配置

配置文件`redis.conf`中的save即为`bgsave`模式，官方配置文件中有save三种默认的选项配置：

```
218 save 900 1                     #每900秒(15分钟)有1个键更新就执行一次bgsave
219 save 300 10                    #每300秒(5分钟)有10个键更新就执行一次bgsave
220 save 60 10000                  #每60秒(1分钟)有10000个键更新就执行一次bgsave
```

三种配置不是互斥的，只要有一个满足了条件就会触发备份操作。同时设置多个save是为了匹配不同的场景需要。若按照上面配置`save 60 10000`进行，并且redis在一分钟内持续更新了10000个键，但是服务器却在第59.99秒发生宕机，那后果可想而知:这一分钟内更新的所有键将全部丢失并且无法恢复。生产环境需要根据请求的调整合适的配置。

#### 备份文件配置

```
253 dbfilename dump.rdb
263 dir ./
```

默认情况下备份保存在./dump.rdb中,默认在配置文件`redis.conf`所在的目录。可以手动更改配置文件备份的地址和文件名重启redis，也可以在redis命令行工具中热变更，比如将备份调整为/var/redis/redis-6379.rdb:

```
127.0.0.1:6379> CONFIG SET dir /var/redis
OK
127.0.0.1:6379> CONFIG SET dbfilename resid-6379.rdb
OK
```

完成后可以使用`CONFIG REWRITE`将变更合并到配置文件中。

#### 其它选项

```
241 rdbcompression yes
```

将备份出的rdb文件按照LZF算法进行压缩，可以大大减少备份文件的体积，此选项是默认开启的。当然为了节省CPU时间，可以改为no,但是快照文件的大小会大大增加。

```
250 rdbchecksum yes
```

默认情况下，一串CRC64d的检验和会追加到快照文件的尾部，这个可以使得快照文件更耐破坏。但是在保存和加载快照文件会损失约10%的性能。



### AOF

与RDB模式不同的是，AOF仅仅保存修改命令，类似**mysql**的bin-log日志。也就是说RDB保存的是数据快照，AOF保存的是修改命令。默认情况下AOF是关闭的，Redis仅使用RDB方式持久化数据。

```
699 appendonly no
```

在命令行中使用 `CONFIG SET appendonly yes`来开启AOF持久化模式，同样的可以使用`CONFIG REWRITE`将变更合并到配置文件中。

相应的持久化数据会默认保存到appendonly.aof中，路径即是`dir`中定义的路径。如果变更在写入过程中服务器发生了宕机，变更的命令位于来不及写入到**.aof**文件,也会造成数据丢失。Redis通过fsync()系统调用将执行的命令写入磁盘，这里Redis也提供了三种模式用于控制fsync的速度。

#### 持久化配置

```
728 appendfsync always
729 appendfsync everysec
730 appendfsync no
```

**always**:Redis每次操作都写入**.aof**文件中，最慢但也是最安全的选项，几乎不存在数据的丢失

**no**:Redis从不执行fsync()系统调用，而是让操作系统决定什么时候将数据写入磁盘，最快但数据存在数据丢失风险

**everysecond**:每秒调用一次fsync()，折中方案也是默认配置

#### 日志重写

如果生产环境开启了AOF模式，那么生成的.aof文件会变得越来越大。aof重写的主要包括**1.删除抵消的命令** **2.合并重复的命令**，比如说一个Python Django项目使用Redis来作为计数器，该计数器在5分钟发生了1000次的值更新，但是数据集中包含最终值的键只有一个，而在AOF中却包含1000个条目，不需要其中的999个条目来重建当前状态。所以这999个条目可以在重建时可以全部删除。

Redis支持一个有趣的功能：它可以在后台fork()出一个子进程重建AOF，而不会中断对客户端的服务。每当您发出**BGREWRITEAOF**，Redis都会编写最短的命令序列来重建内存中的当前数据集。除了手动的在Redis命令行中手动执行外，还可以在配置文件中声明自动重建的配置。

```
770 auto-aof-rewrite-percentage 100
771 auto-aof-rewrite-min-size 64mb
```

`auto-aof-rewrite-min-size 64mb`指的是数据量达到64mb就触发执行一次重写操作，如果redis启动时数据量大于64mb，则会自动进行一次重写操作。此后这条配置项将不再有效。

`auto-aof-rewrite-percentage 100`此第一次重写后后当数据再次达到100%，即128mb时，会再次执行一次重写操作，以此类推，不断执行自动重写。

如果`auto-aof-rewrite-percentage 0`,则会禁用自动重写

`auto-aof-rewrite-min-size`的值不能太小，否则会频繁重写造成性能降低



### 总结

- RDB持久性按指定的时间间隔执行数据集的时间点快照
- AOF持久性会记录服务器接收的每个写入操作，这些操作将在服务器启动时再次播放，以重建原始数据集。使用与Redis协议本身相同的格式记录命令，并且仅采用追加方式。当日志太大时，Redis可以在后台重写日志
- 如果希望，只要您的数据在服务器运行时就一直存在，则可以完全禁用持久性。当做memcache使用
- 可以在同一实例中同时合并AOF和RDB。请注意，在这种情况下，当Redis重新启动时，AOF文件将用于重建原始数据集，因为它可以保证是最完整的

