# mybatis是如何执行的

## mybatis 简单SQL执行流程
简单sql就是使用sqlsession所提供的方法进行查询，我们所提供statement(xml中sqlId，如id多个则会报错)

```java
    String resource = "org/apache/ibatis/mytest/mybatis-config.xml";
    InputStream resourceAsStream = Resources.getResourceAsStream(resource);
    SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(resourceAsStream);
    SqlSession sqlSession = sqlSessionFactory.openSession();
    List<Object> testMapper = sqlSession.selectList("queryIdById");

    System.out.println(testMapper);

  }

  private SqlSessionFactory getSqlSessionFactoryXmlConfig(String resource) throws Exception {
    try (Reader configReader = Resources.getResourceAsReader(resource)) {
      SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(configReader);
      BaseDataTest.runScript(sqlSessionFactory.getConfiguration().getEnvironment().getDataSource(),
        "org/apache/ibatis/submitted/result_handler_type/CreateDB.sql");

      return sqlSessionFactory;
    }
  }


// xml
  <select id="queryIdById" parameterType="int" resultType="int">
    select num from t_test
  </select>
```



### 第一步:读取配置文件

读取MyBatis的核心配置文件。mybatis-config.xml为MyBatis的全局配置文件，用于配置数据库连接、属性、类型别名、类型处理器、插件、环境配置、映射器（mapper.xml）等信息，这个过程中有一个比较重要的部分就是映射文件其实是配在这里的；

我们需要输入文件地址，将该配置文件地址提供给mybatis，mybatis会更具我们提供的文件地址，将该文件给转换成对应的文件输入流。

### 第二步:获取SqlSession

通过第一步我们获取到mybatis相应的配置文件输入流，这时我们需要将该配置流给传入到**SqlSessionFactoryBuilder**对象来构建**SqlSessionFactory**对象。并将对应的配置流给解析成configuration对象作为上下文进行使用。

创建完工厂后，后续我们每次请求都需要从SqlSessionFactory获取一个SqlSession对象，SqlSessionFactory提供了一个openSession方法，让我们从工厂中获取SqlSession。

### 第三步：执行sql
在执行SQL前，通过第一步我们所创建的configuration对象，调用配置对象所提供的`getMappedStatement(String id)`方法来获取我们对应的MappedStatement对象,而后开始调用执行器，并传入MappedStatement。
```java
      MappedStatement ms = configuration.getMappedStatement(statement);
      //转而用执行器来查询结果,注意这里传入的ResultHandler是null
      return executor.query(ms, wrapCollection(parameter), rowBounds, Executor.NO_RESULT_HANDLER);
```

通过mappedStatement对象获取我们最终需要需要执行的sql,并封装成对象`boundSql`，之后创建二级缓存key(需要在配置文件中开启二级缓存，否则为null)
```java
  public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
    //得到绑定sql
    BoundSql boundSql = ms.getBoundSql(parameter);
    //创建缓存Key
    CacheKey key = createCacheKey(ms, parameter, rowBounds, boundSql);
    //查询
    return query(ms, parameter, rowBounds, resultHandler, key, boundSql);
 }
 ```

在执行最终SQL前mybatis还会查一遍一级缓存(默认开启)，判断当前SQL是否满足命中缓存,从而减少对数据库的查询压力.如果一级缓存也没命中则执行通过jdbc连接数据查询
```java
      list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
      if (list != null) {
        //若查到localCache缓存，处理localOutputParameterCache
        handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
      } else {
        //从数据库查
        list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
      }
```
> 一级缓存有些地方又叫局部缓存，生命周期为当个sqlsession范围内

## mybatis Mapper动态SQL执行流程
```java
    String resource = "org/apache/ibatis/mytest/mybatis-config.xml";
    InputStream resourceAsStream = Resources.getResourceAsStream(resource);
    SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(resourceAsStream);
    SqlSession sqlSession = sqlSessionFactory.openSession();
    TestMapper mapper = sqlSession.getMapper(TestMapper.class);
    Blog blog = mapper.selectBlog(1);
    int i = mapper.queryIdById(5);

    System.out.println(i);
```
### 第一步:读取配置文件

读取MyBatis的核心配置文件。mybatis-config.xml为MyBatis的全局配置文件，用于配置数据库连接、属性、类型别名、类型处理器、插件、环境配置、映射器（mapper.xml）等信息，这个过程中有一个比较重要的部分就是映射文件其实是配在这里的；

我们需要输入文件地址，将该配置文件地址提供给mybatis，mybatis会更具我们提供的文件地址，将该文件给转换成对应的文件输入流。



### 第二步:获取SqlSession

通过第一步我们获取到mybatis相应的配置文件输入流，这时我们需要将该配置流给传入到**SqlSessionFactoryBuilder**对象来构建**SqlSessionFactory**对象。并将对应的配置流给解析成configuration对象作为上下文进行使用。

创建完工厂后，后续我们每次请求都需要从SqlSessionFactory获取一个SqlSession对象，SqlSessionFactory提供了一个openSession方法，让我们从工厂中获取SqlSession。

### 第三步:获取Mapper

SqlSession提供一个方法`getMapper(Class<?> mapper)` 来获取mapper接口对象，只需要传入对应的class即可，在这之中，SqlSession会通过上下文去获取mapper对象，因为我么再扫描配置文件的时候已经，已经将xml写在配置文件里面。

## 第四步 执行Sql

当我们获取到mapper后，直接执行mapper对应的方法并传入参数

> 因为我们的mapper其实是一个interface，我们都知道java的interface是无法直接运行的，一定得有对应的实现类，但是我们并没有提供对应的实现类，只是提供的xml。

这时，mybatis的代理对象MapperProxy，会将我们的执行进行拦截，并判断我们的对象是否为**Object**,如果对象为Object对象，则直接执行方法，不做任何处理；否则将会构建一个MapperMethodInvoker执行器，通过执行器来执行我的方法。

```java
  @Override
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    try {
      if (Object.class.equals(method.getDeclaringClass())) {
        return method.invoke(this, args);
      } else {
        // 构建适配器对象并执行
        return cachedInvoker(method).invoke(proxy, method, args, sqlSession);
      }
    } catch (Throwable t) {
      throw ExceptionUtil.unwrapThrowable(t);
    }
  }


  private MapperMethodInvoker cachedInvoker(Method method) throws Throwable {
    try {
      return MapUtil.computeIfAbsent(methodCache, method, m -> {
        // 判断当前函数对象是否为默认方法
        if (m.isDefault()) {
          try {
            if (privateLookupInMethod == null) {
              return new DefaultMethodInvoker(getMethodHandleJava8(method));
            } else {
              return new DefaultMethodInvoker(getMethodHandleJava9(method));
            }
          } catch (IllegalAccessException | InstantiationException | InvocationTargetException
              | NoSuchMethodException e) {
            throw new RuntimeException(e);
          }
        } else {
          return new PlainMethodInvoker(new MapperMethod(mapperInterface, method, sqlSession.getConfiguration()));
        }
      });
    } catch (RuntimeException re) {
      Throwable cause = re.getCause();
      throw cause == null ? re : cause;
    }
  }
```

之后mybatis在这个时候会开始进行判断我们的SQL执行的语法，是CRUD中哪种类型执行对应的逻辑。

然后通过configuration获取到**MappedStatement**对象，该对象存储这我们映射xml对应的sql，开始拼接SQL。