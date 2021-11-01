## binlog写入机制

​		binlog的写入逻辑比较简单：事务执行时，创建日志写入binlog cache中，事务提交时，再将binlog cache写到binlog文件中。

​		每个线程都会有自己一个独有的binlog cache，并且binlog cache会被分配一篇内存，参数**binlog_cache_size**用于控制线程内binlog cache内存大小。

​		每个binlog cache

![9724A480-3AE6-4AB0-8FEA-80930DB48931](https://gitee.com/zuoyii/picture/raw/master/img/9724A480-3AE6-4AB0-8FEA-80930DB48931.png)

- write并未将日志写入page cache，并没有将数据持久化到磁盘，所以速度比较快。
- fsync才是将数据持久化到硬盘的操作。

write和fsync的时机，是有参数sync_binlog控制的：

> 1. Sync_binlog=0 的时候，每次提交事务都会直接write，不fsync
> 2. sync_binlog=1 的时候，每次提交事务都会write与fsync
> 3. sync_binlog=N(N>1)时，每次提交都会write，且提交N次事务，才会fsync

​		因此，在出现瓶颈的场景里，将sync_binlog设置成一个比较大的值，可以提升性能，因为write只是保存在内存中，没有IO操作。但是实际的业务场景中，考虑到丢失日志的可控性，一般不建议将值设成0或者太高，一般情况设置成100~1000即可。

