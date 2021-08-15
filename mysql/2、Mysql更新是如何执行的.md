## mysql是如果执行更新语句的

### 重要的日志模块：redo log

​		在进行mysql操作是，如果每一次的更新操作都需要写进磁盘，然后磁盘也要找到对应的那条记录，然后再更新，整个过程 IO 成本、查找成本都很高。为了解决这个问题，MySQL 的设计者就用了redo log 与binlog记录日志。

​		其实就是 MySQL 里经常说到的 WAL 技术，WAL 的全称是 Write-Ahead Logging，它的关键点就是先写日志，再写磁盘。

​		InnoDB 引擎就会先把记录写到 redo log里面，并更新内存，这个时候更新就算完成了。同时，InnoDB 引擎会在适当的时候，将这个操作记录更新到磁盘里面、

​		如果redo log写满了，InnoDb会将redo log中一部分数据写入磁盘，腾出空间来记录更新记录。

​		InnoDB的redo log是固定大小的，可以自由配置，数据结构为"环"，从头开始写，写到末尾就又回到开头循环写，如下图所示：

![5625ECB6-9063-4AAF-B67B-59344ECCA8EE](https://gitee.com/zuoyii/picture/raw/master/img/5625ECB6-9063-4AAF-B67B-59344ECCA8EE.png)

​		write pos 是当前记录的位置，一边写一边后移，写到第 3 号文件末尾后就回到 0 号文件开头。checkpoint 是当前要擦除的位置，也是往后推移并且循环的，擦除记录前要把记录更新到数据文件。

​		write pos 和 checkpoint 之间的是“粉板”上还空着的部分，可以用来记录新的操作。如果 write pos 追上 checkpoint，表示“粉板”满了，这时候不能再执行新的更新，得停下来先擦掉一些记录，把 checkpoint 推进一下。

​		有了 redo log，InnoDB 就可以保证即使数据库发生异常重启，之前提交的记录都不会丢失，这个能力称为 crash-safe。

​		要理解 crash-safe 这个概念，可以想想我们前面赊账记录的例子。只要赊账记录记在了粉板上或写在了账本上，之后即使掌柜忘记了，比如突然停业几天，恢复生意后依然可以通过账本和粉板上的数据明确赊账账目。



### 重要的日志模块：binlog

​		前面我们讲过，Mysql整体来看，其实就有两块：一块是Server层，它主要做的是Mysql功能层面的事情；还有一块是引擎层，负责储存相关具体内容。

​		**redo log是InnoDB引擎特有的日志，而Server层也有自己的日志，称为binlog。**

​		因为最开始 MySQL 里并没有 InnoDB 引擎。MySQL 自带的引擎是 MyISAM，但是 MyISAM 没有 crash-safe 的能力，binlog 日志只能用于归档。而 InnoDB 是另一个公司以插件形式引入 MySQL 的，既然只依靠 binlog 是没有 crash-safe 能力的，所以 InnoDB 使用另外一套日志系统——也就是 redo log 来实现 crash-safe 能力。



这两种日志有三个不同点：

1. redo log 是 InnoDB 引擎特有的；binlog 是 MySQL 的 Server 层实现的，所有引擎都可以使用。
2. redo log 是物理日志，记录的是“在某个数据页上做了什么修改”；binlog 是逻辑日志，记录的是这个语句的原始逻辑，比如“给 ID=2 这一行的 c 字段加 1 ”。
3. redo log 是循环写的，空间固定会用完；binlog 是可以追加写入的。“追加写”是指 binlog 文件写到一定大小后会切换到下一个，并不会覆盖以前的日志。

执行器与InnoDB引擎在执行update语句时的内部流程：

1. 执行器先找到引擎ID=2这一行。ID是主键，引擎直接通过B+树搜索找到这一行。如果ID=2这一行所在的数据页本来就在内存中，就直接返回给执行器；否则，需要先从磁盘读入内存，然后在返回
2. 执行器拿到引擎给的行数据，把值加上1，比如原来是N，现在就是N+1，得到新的一行数据，在调用引擎接口写入这行新数据
3. 引擎将这行新数据更新到内存，同时记录redo log中，此时redo log处于prepare状态。并告知执行器随时可以提交事务
4. 执行器收到引擎返回结果，将执行结果写入binlog
5. 执行器提交引擎事务接口，引擎将刚刚写入redo log日志改成commit状态，更新完成

![8E265942-C07C-4F51-A4CF-CB74D0B8A5AB](https://gitee.com/zuoyii/picture/raw/master/img/8E265942-C07C-4F51-A4CF-CB74D0B8A5AB.png)

最后的写入过程拆成两个步骤：prepare和commit，这就是"这就是两阶段提交"。



### 两阶段提交

​		由于redo log和binlog是两个独立的逻辑，如果不用两阶段提交，要么就是先写完redo log再写binlog，或者采用反过来的顺序。

1. ​		先写 redo log 后写 binlog。假设在 redo log 写完，binlog 还没有写完的时候，MySQL 进程异常重启。由于我们前面说过的，redo log 写完之后，系统即使崩溃，仍然能够把数据恢复回来，所以恢复后这一行 c 的值是 1。但是由于 binlog 没写完就 crash 了，这时候 binlog 里面就没有记录这个语句。因此，之后备份日志的时候，存起来的 binlog 里面就没有这条语句。然后你会发现，如果需要用这个 binlog 来恢复临时库的话，由于这个语句的 binlog 丢失，这个临时库就会少了这一次更新，恢复出来的这一行 c 的值就是 0，与原库的值不同。
2. ​		先写 binlog 后写 redo log。如果在 binlog 写完之后 crash，由于 redo log 还没写，崩溃恢复以后这个事务无效，所以这一行 c 的值是 0。但是 binlog 里面已经记录了“把 c 从 0 改成 1”这个日志。所以，在之后用 binlog 来恢复的时候就多了一个事务出来，恢复出来的这一行 c 的值就是 1，与原库的值不同。



​	如果不使用两阶段提交方式，那么数据库的状态就有可能和用日志恢复出来的库状态不一致。

​	redo log和binlog都可以用于表示事务的提交状态，而两阶段提交就是让这两个状态保持逻辑上的一致。