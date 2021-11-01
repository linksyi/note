### 全字段排序

```mysql
create table 't' (
	id int(32) not null,
  city varchar(32) not null,
  name varchar(32) not null,
  age int(11) not null,
  addr varchar(128) default null,
  primary key (id),
  key city(city)
) ENGINE=InnoDB;

select city,name,age from t where city = '长沙' order by name limit 1000;
```



当我们在执行上述sql排序语句时，通常情况如下：

>1、初始化sort_buffer，确定放入name，city，age；
>
>2、从索引city找到第一个满足city='杭州'的主键id，并取出name、city、age三个字段的值
>
>3、持续找到city != '长沙'条件的数据，退出查询
>
>4、sort_buffer对数据中name值进行排序；
>
>5、返回前1000行数据

![D143F711-6CB2-4B31-8D15-715306B5B283](https://gitee.com/zuoyii/picture/raw/master/img/D143F711-6CB2-4B31-8D15-715306B5B283.png)

查询一个排序语句是否使用了临时文件。

```mysql
-- 打开optimizer_trace，只对本线程有效 
SET optimizer_trace='enabled=on'; 
-- @a保存Innodb_rows_read的初始值
select VARIABLE_VALUE into @a from performance_schema.session_status where variable_name = 'Innodb_rows_read';
-- 执行语句
select city, name,age from t where city='杭州' order by name limit 1000;
-- 查看 OPTIMIZER_TRACE 输出
SELECT * FROM `information_schema`.`OPTIMIZER_TRACE`
-- @b保存Innodb_rows_read的当前值
select VARIABLE_VALUE into @b from performance_schema.session_status where variable_name = 'Innodb_rows_read';
-- 计算Innodb_rows_read差值
select @b-@a;
```

### rowid 排序

如果mysql认为排序的单行长度太大，就会采用rowid进行排序。

> 1、初始化sort_buffer，确定放入两个字段,id,name
>
> 2、从索引city找到第一个满足city='杭州'的主键id，并取出id、name两个字段的值
>
> 3、持续找到city != '长沙'条件的数据，退出查询
>
> 4、sort_buffer对数据中name值进行排序；
>
> 5、放回前1000条数据的id
>
> 6、回表查询city，age字段的值并返回

![6DAF943A-13B2-418A-BD95-934986E6EDD4](https://gitee.com/zuoyii/picture/raw/master/img/6DAF943A-13B2-418A-BD95-934986E6EDD4.png)