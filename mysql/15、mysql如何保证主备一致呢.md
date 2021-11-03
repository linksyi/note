### binlog的三种格式

#### statement

```sql
mysql> CREATE TABLE `t` ( 
  `id` int(11) NOT NULL, 
  `a` int(11) DEFAULT NULL,
  `t_modified` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),KEY `a` (`a`),
  KEY `t_modified`(`t_modified`)
) ENGINE=InnoDB;
  
  insert into t values(1,1,'2018-11-13');
  insert into t values(2,2,'2018-11-12');
  insert into t values(3,3,'2018-11-11');
  insert into t values(4,4,'2018-11-10');
  insert into t values(5,5,'2018-11-09');
```

​		当我们执行以下语句时

```sql
delete from t /* comment*/ where a>4 and t_modufued<='2018-11-10' limit 1;
```

​		当binlog_format=statement时，binlog里记录的就是SQL语句的原文。

```sql
-- 查看binlog
show binlog events in 'master.00001';
```

![9F06AA56-BAC2-4BE7-B522-138EFDF2B4AD](https://gitee.com/zuoyii/picture/raw/master/img/9F06AA56-BAC2-4BE7-B522-138EFDF2B4AD.png)

- 第一行SET@@SESSION.GTID_NEXT='ANONYMOUS'再主备切换时用到
- 第二行开启事务
- 第三行完整的sql记录
- 第四行提交事务，xid为MVCC中事务id



​		当我们使用 **show warnings**查看mysql提供的警告时

![D024F342-939A-4C32-914D-35DD69B1D61D](https://gitee.com/zuoyii/picture/raw/master/img/D024F342-939A-4C32-914D-35DD69B1D61D.png)

​		mysql会提示我们，这个binlog并不安全，可能会导致主备不一致，因为当主备库索引不同时，list所返回的数据是不同的，所以在备库同步该binlog时，就有可能照成主备不一致的情况

#### row

​		当我们将binlog_format='row'，再查看binlog中的内容

![F7887AA3-8B20-4CD0-8712-ADC06F8773B7](https://gitee.com/zuoyii/picture/raw/master/img/F7887AA3-8B20-4CD0-8712-ADC06F8773B7.png)

​		通过该图，我们并不能看到详细信息，需要接祖mysqlbinlog的工具，用下面这样的命令查看binlog中的内容,这个事务的 binlog 是从 8900 这个位置开始的，所以可以用 start-position 参数来指定从这个位置的日志开始解析。

```mysql
mysqlbinlog -vv data/master.000001 --start-position=8900;
```

![4D82390B-E2A3-47BC-A726-FA7F83F6CF38](https://gitee.com/zuoyii/picture/raw/master/img/4D82390B-E2A3-47BC-A726-FA7F83F6CF38.png)

- Server_id=1,表示在失误在server_id为1的库上执行。
- 在日志记录中，where条件是指定表中id的
- 因为在sql语句中使用了-vv这样的参数，所以可以看到具体的值

