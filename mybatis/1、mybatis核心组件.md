### Mybatis功能结构

![Ut3BaB](https://gitee.com/zuoyii/picture/raw/master/img/Ut3BaB.png)

**API接口层**：提供给外部使用的接口API，开发人员通过这些本地API来操纵数据库。接口层接收到调用请求就会调用数据处理层来完成具体的数据处理。

**数据处理层**：负责具体的SQL查找、SQL解析、SQL拼接、SQL执行和返回结果映射处理等。它主要的目的是根据调用的请求完成一次数据库操作。

**基础支持层**：负责最基础的功能支持，包括连接管理，事务管理，配置加载和缓存处理，这些都是公用的东西，将他们抽取出来作为最基础的组件。为上层的数据处理层提供最基础的支撑。

## mybatis的核心组件

如果我们想要了解mybatis源码，那么需要先知道mybatis几个核心组件在项目中所充当的作用

### 1、Configuration

Mybatis所有的配置信息都保存在Configuration对象之中，配置文件中的大部分配置都会储存到该类中，项目启动时最先初始化该类。

### 2、SqlSession

作为Mybatis工作的主要顶层API，表示和数据库交互时的回话，完成必要的数据库增删改查功能。

### 3、RowBounds

Mybatis分页对象

### 4、Executor

MyBatis执行器，是MyBatis调度的核心，负责SQL语句的生成和查询缓存维护。

### 5、SteatementHandler

封住哪个了JDBCStatement操作，负责对JDBC statement的操作，如设置参数等。

### 6、ParameterHandler

负责对用户传递的参数转黄成JDBC Statement所对应的数据类型。

### 7、REsultSetHandler

负责将JDBC返回的ResultSet结果集对象转换成List类型集合对象。

### 8、TypeHandler

负责java数据类型和jdbc数据类型之间的映射和转换，如将String转换成varchar

### 9、MappedStatement

维护SQL映射层与XML之间的连接与映射关系，并识别我们的SQL语句属于\[select、insert、delete、update]那种类型

### 10、MappedStatement

负责根据用户传递的parameterObject，动态地生成SQL语句，将信息封装到BoundSql对象中，并返回

### 11、BoundSql

表示动态生成的SQL语句及对应的参数信息

![mP19fD](https://gitee.com/zuoyii/picture/raw/master/img/mP19fD.jpg)
