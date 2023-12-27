---
layout: post
title: 分库分表路由组件分析介绍
subtitle: Maxlec
author: Maxlec
categories: DBRouter
banner:
  image: https://github.com/Andrewmeo/images/blob/master/tales/0022.jpg?raw=true
  opacity: 0.618
  background: "#000"
  height: "100vh"
  min_height: "38vh"
  heading_style: "font-size: 4.25em; font-weight: bold; text-decoration: underline"
  subheading_style: "color: gold"
tags: aop hash datasource spi
top: 2
---

## 1 分库分表路由组件开发

自己开发分库分表路由组件有以下好处：

1. 更容易维护。市场上已经有一些成熟的路由组件，比如ShardingSphere，但这个组件非常庞大，维护起来不方便。
2. 更容易扩展。可以结合自身的业务需求，对组件进行相应的扩展。比如路由策略，扫描指定库表数据等等。
3. 更安全。

## 2 整体开发流程和执行流程分析

开发流程分析：

1. 因为是基于SpringBoot的starter开发，所以在项目启动时需要进行数据库的初始化配置。
2. 然后要使用AOP面向切面编程，自定义一个路由注解，用于修饰需要进行分库分表的接口方法处。
3. 当执行该方法时就会通过AOP定义库表索引。连接到数据库后就会根据库表索引切换数据源。
4. 最后通过拦截Mybatis，然后使用反射修改SQL语句的表名，完成分库分表流程。

简单的执行流程：

1. 在调用dao层接口时换数据源 -> 更换库
2. 在执行SQL之前替换原有SQL->更换表
3. 执行SQL

微观流程：

1. 使用SPI机制扫描注入 DataSourceAutoConfig
2. 获取并存储配置文件中配置的数据源信息
3. 创建 DynamicDataSource 代替 mybatis 中数据源 bean，重写 determineCurrentLookupKey() 方法，设置数据源策略
4. 创建 TransactionTemplate，供后续声明式事务
5. 创建 DBRouterConfig 用于存储DB的信息(库表数量，路由字段)
6. 创建 IDBRouterStrategy 路由策略，供后续可以手动设置路由(一个事务中需要切换多个数据源会导致数据源失效，因此需要先设置路由)
7. 创建 DynamicMybatisPlugin 拦截器，用于动态修改 SQL 操作哪张表
8. 创建 DBRouterJoinPoint(AOP)，用于在不手动设置路由情况下，AOP 设置路由策略

![分库分表路由组件描述图](https://github.com/Andrewmeo/images/blob/master/dbrouter/diagram.jpg?raw=true)

当调用dao层(mapper)对应的接口执行数据库操作时，激发切面拦截，在环绕通知中获取注解中的路由字段，以及目标方法入参对象，然后通过BeanUtils.getProperty获取对象中的路由字段属性，调用路由策略利用路由字段属性计算库表路由，放入线程工作空间中。然后就是获取数据源连接，其就是从线程工作空间中获取的，具体执行哪一个数据库。

在进行Mybatis的拦截方法intercept中，通过Invocation获取StatementHandler对象，从对象中提取出SQL语句的元信息。然后从元信息中获取自定义注解判断是否进行分表操作，因为有可能只进行分库不分表的操作。然后就是替换原有的SQL，将原来的表名替换为路由的库表。

## 3 注解

```java
/**
 * 分库分表注解
 */
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE, ElementType.METHOD})
public @interface DBRouter {
    /** 分库分表字段 */
    String key() default "";
}
```

```java
/**
 * 分表注解
 */
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE, ElementType.METHOD})
public @interface DBRouterStrategy {
    boolean splitTable() default false;
}
```

## 4 核心类

1. AbstractRountingDataSource类：用于动态数据源切换。我们通过重写其determineCurrentLookupKey抽象方法实现数据源获取。
2. DBRouterStrategyHashCode类：实现路由计算，由HashMap扰动函数实现散列。
3. DynamicMybatisPlugin类：实现Interceptor接口用于拦截SQL方法。
4. DataSourceAutoConfig类：核心配置类，Bean的配置管理类，在使用该组件时需要加载该配置类进行Bean的装配。其中还包含setEnvironment方法通过环境对象读取数据库信息，如db01、db02，以db01与db02为键，以其配置项信息为值存入map集合对象。方便其他定义Bean的方法中获取数据库信息。

### DataSourceAutoConfig

首先就来看Bean的管理配置类。其中声明了与分库分表策略以及SQL拦截等相关的六个Bean。以及数据源配置项的读取方法。

| Bean                     | 说明                                                         |
| ------------------------ | ------------------------------------------------------------ |
| DBRouterJoinPoint        | 拦截自定义注解标识的分库分表操作，获取注解中的路由字段，以及目标方法入参对象，然后通过BeanUtils.getProperty获取对象中的路由字段属性。 |
| DBRouterStrategyHashCode | 接收路由字段属性，通过HashMap的扰动函数生散列数据库路由：tbId，dbId，放入线程工作空间中。 |
| DynamicDataSource        | 设置数据库路由：db+dbId。通过DataSourceAutoConfig读取的数据源信息生成多个目标数据源。 |
| DynamicMybatisPlugin     | Mybatis拦截器类，拦截SQL语句，获取线程工作空间中的数据库路由：tbIdx变量。 |
| DBRouterConfig           | 数据路由配置信息，该对象中包含dbCount、tbCount、routerKey三个属性信息。方便其他类获取。 |
| TransactionTemplate      | 配置事务。                                                   |

讲解这些Bean具体声明之前需要先讲解配置类中setEnvironment方法，该方法通过环境对象读取数据库信息。

1. 首先是获取dbCount、tbCount、routerKey等。其中routerkey可以不在SpringBoot配置文件中指明，我们也可以通过定义注解属性的方式传入。这些数据会封装到dBRouterConfig对象中。

2. 以及获取数据库列表db01、db02，以db01与db02为键，以其配置项信息为值存入map集合对象作为数据源。方便其他定义Bean的方法中获取数据库信息。

```java
/**
 * 数据源配置解析以及注册一些Bean
 */
@Configuration
public class DataSourceAutoConfig implements EnvironmentAware {

    /**
     * 数据源配置组
     * value：数据源详细信息
     */
    private Map<String, Map<String, Object>> dataSourceMap = new HashMap<>();

	// 默认数据源配置
    private Map<String, Object> defaultDataSourceConfig;

    // 分库数量
    private int dbCount;

    // 分表数量
    private int tbCount;

	// 路由字段
    private String routerKey;

    /**
     * AOP，用于分库
     * @param dbRouterConfig
     * @param dbRouterStrategy
     * @return
     */
    @Bean(name = "db-router-point")
    @ConditionalOnMissingBean
    public DBRouterJoinPoint point(DBRouterConfig dbRouterConfig, IDBRouterStrategy dbRouterStrategy) {
        return new DBRouterJoinPoint(dbRouterConfig, dbRouterStrategy);
    }

    /**
     * 将DB的信息注入到spring中，供后续获取
     * @return
     */
    @Bean
    public DBRouterConfig dbRouterConfig() {
        return new DBRouterConfig(dbCount, tbCount, routerKey);
    }

    /**
     * 配置插件bean,用于动态的决定表信息
     * @return
     */
    @Bean
    public Interceptor plugin() {
        return new DynamicMybatisPlugin();
    }

    /**
     * 用于配置 TargetDataSources 以及 DefaultTargetDataSource
     * TargetDataSources: 额外的数据源
     * 可以用指定的key获取其他的数据源来达到动态切换数据源
     * DefaultTargetDataSource: 默认的数据源
     * 如果没有要用的数据源就会使用默认的数据源
     * @return
     */
    @Bean
    public DataSource dataSource() {
        Map<Object, Object> targetDataSources = new HashMap<>();
        for (String dbInfo : dataSourceMap.keySet()) {
            Map<String, Object> objMap = dataSourceMap.get(dbInfo);
            targetDataSources.put(dbInfo, new DriverManagerDataSource(objMap.get("url").toString(), objMap.get("username").toString(), objMap.get("password").toString()));
        }

        // 设置数据源
        DynamicDataSource dynamicDataSource = new DynamicDataSource();
        dynamicDataSource.setTargetDataSources(targetDataSources);
        dynamicDataSource.setDefaultTargetDataSource(new DriverManagerDataSource(defaultDataSourceConfig.get("url").toString(), defaultDataSourceConfig.get("username").toString(), defaultDataSourceConfig.get("password").toString()));

        return dynamicDataSource;
    }

    /**
     * 依赖注入
     * @param dbRouterConfig
     * @return
     */
    @Bean
    public IDBRouterStrategy dbRouterStrategy(DBRouterConfig dbRouterConfig) {
        return new DBRouterStrategyHashCode(dbRouterConfig);
    }

    /**
     * 配置事务
     * @param dataSource
     * @return
     */
    @Bean
    public TransactionTemplate transactionTemplate(DataSource dataSource) {
        DataSourceTransactionManager dataSourceTransactionManager = new DataSourceTransactionManager();
        dataSourceTransactionManager.setDataSource(dataSource);

        TransactionTemplate transactionTemplate = new TransactionTemplate();
        transactionTemplate.setTransactionManager(dataSourceTransactionManager);
        transactionTemplate.setPropagationBehaviorName("PROPAGATION_REQUIRED");
        return transactionTemplate;
    }

    /**
     * 读取yml中的数据源信息
     * @param environment
     */
    @Override
    public void setEnvironment(Environment environment) {
        String prefix = "xhy-db-router.jdbc.datasource.";

        dbCount = Integer.valueOf(environment.getProperty(prefix + "dbCount"));
        tbCount = Integer.valueOf(environment.getProperty(prefix + "tbCount"));
        routerKey = environment.getProperty(prefix + "routerKey");

        // 分库分表数据源
        String dataSources = environment.getProperty(prefix + "list");
        assert dataSources != null;
        for (String dbInfo : dataSources.split(",")) {
            Map<String, Object> dataSourceProps = PropertyUtil.handle(environment, prefix + dbInfo, Map.class);
            dataSourceMap.put(dbInfo, dataSourceProps);
        }

        // 默认数据源
        String defaultData = environment.getProperty(prefix + "default");
        defaultDataSourceConfig = PropertyUtil.handle(environment, prefix + defaultData, Map.class);

    }

}
```

### DBRouterJoinPoint

@Around环绕通知，它集成了@Before、@AfterReturing、@AfterThrowing、@After四大通知。需要手动进行接口内方法的反射后才能执行接口中的方法，换言之，@Around 其实就是一个动态代理。@Around 注解的主要作用是在目标方法的执行前后进行一些额外的操作，例如日志记录、性能监控、多数据源动态切换、事务管理等。

> 执行目标方法：Object result = joinPoint.proceed();

以下定义的切面类中，定义切入点为DBRouter注解，以及环绕通知，该环绕通知相当于前置通知@Before。在在执行目标方法前获取用于分库分表计算的key，一般为id字段作为key。

然后就是通过BeanUtils.getProperty获取对象中的具体的key属性值，如id值。最后传递给路由策略类进行路由计算。

```java
/**
 * 数据路由切面，通过自定义注解的方式，拦截被切面的方法，进行数据库路由
 */
@Aspect
public class DBRouterJoinPoint {

    private Logger logger = LoggerFactory.getLogger(DBRouterJoinPoint.class);

    private DBRouterConfig dbRouterConfig;

    private IDBRouterStrategy dbRouterStrategy;

    public DBRouterJoinPoint(DBRouterConfig dbRouterConfig, IDBRouterStrategy dbRouterStrategy) {
        this.dbRouterConfig = dbRouterConfig;
        this.dbRouterStrategy = dbRouterStrategy;
    }

    @Pointcut("@annotation(com.bree.dbrouter.annotation.DBRouter)")
    public void aopPoint() {
    }

    /**
     * 所有需要分库分表的操作，都需要使用自定义注解进行拦截，拦截后读取方法中的入参字段，根据字段进行路由操作。
     * 1. dbRouter.key() 确定根据哪个字段进行路由
     * 2. getAttrValue 根据数据库路由字段，从入参中读取出对应的值。比如路由 key 是 uId，那么就从入参对象 Obj 中获取到 uId 的值。
     * 3. dbRouterStrategy.doRouter(dbKeyAttr) 路由策略根据具体的路由值进行处理
     * 4. 路由处理完成后放行。 jp.proceed();
     * 5. 最后 dbRouterStrategy 需要执行 clear 因为这里用到了 ThreadLocal 需要手动清空。关于 ThreadLocal 内存泄漏介绍 https://t.zsxq.com/027QF2fae
     */
    @Around("aopPoint() && @annotation(dbRouter)") // 环绕执行，就是在调用目标方法之前和调用之后，都会执行一定的逻辑
    public Object doRouter(ProceedingJoinPoint joinPoint, DBRouter dbRouter) throws Throwable {
        String dbKey = dbRouter.key(); // 一般获取ID进行分库分表算法实现

        // 如果dbKey和路由字段为空则抛出异常
        if (StringUtils.isBlank(dbKey) && StringUtils.isBlank(dbRouterConfig.getRouterKey())) {
            throw new RuntimeException("annotation DBRouter key is null！");
        }

        dbKey = StringUtils.isNotBlank(dbKey) ? dbKey : dbRouterConfig.getRouterKey();
        // 传入路由key，以及方法入参，生成路由属性
        String dbKeyAttr = getAttrValue(dbKey, joinPoint.getArgs());
        // 路由策略
        dbRouterStrategy.doRouter(dbKeyAttr);
        // 返回结果
        try {
            return joinPoint.proceed();
        } finally {
            dbRouterStrategy.clear();
        }
    }

    private Method getMethod(JoinPoint jp) throws NoSuchMethodException {
        Signature sig = jp.getSignature();
        MethodSignature methodSignature = (MethodSignature) sig;
        return jp.getTarget().getClass().getMethod(methodSignature.getName(), methodSignature.getParameterTypes());
    }

    public String getAttrValue(String attr, Object[] args) {
        if (1 == args.length) {
            Object arg = args[0];
            if (arg instanceof String) {
                return arg.toString();
            }
        }

        String filedValue = null;
        for (Object arg : args) {
            try {
                if (StringUtils.isNotBlank(filedValue)) {
                    break;
                }
                filedValue = BeanUtils.getProperty(arg, attr);
            } catch (Exception e) {
                logger.error("获取路由属性值失败 attr：{}", attr, e);
            }
        }
        return filedValue;
    }

}
```

### DBRouterStrategyHashCode

在类中我们接收的路由字段属性dbKeyAttr，然后进行数据库表和数据库的路由计算。首先获取所有数据库表，然后要进行库表路由的取余操作获取具体的库表索引。这里的取余操作是通过与运算实现。

```java
/**
 * 哈希路由
 */
public class DBRouterStrategyHashCode implements IDBRouterStrategy {

    private Logger logger = LoggerFactory.getLogger(DBRouterStrategyHashCode.class);

    private DBRouterConfig dbRouterConfig;

    public DBRouterStrategyHashCode(DBRouterConfig dbRouterConfig) {
        this.dbRouterConfig = dbRouterConfig;
    }

    /**
     * 计算方式:
     * size = 库*表的数量
     * idx : 散列到的哪张表
     * dbIdx = idx / dbRouterConfig.getTbCount() + 1;
     * dbIdx : 用于计算哪个库,idx为0-size的值，除以表的数量 = 当前是几号库，又因库是从一号库开始算的，因此需要+1
     * tbIdx : idx - dbRouterConfig.getTbCount() * (dbIdx - 1);用于计算哪个表，
     * idx 可以理解为是第X张表，但是需要落地到是第几个库的第几个表
     * 例子：假设2库8表，idx为14，因此是第二个库的第6个表才是第14张表
     * (dbIdx - 1) 因为库是从1开始算的，因此这里需要-1
     * dbRouterConfig.getTbCount() * (dbIdx - 1) 是为了算出当前库前面的多少张表，也就是要跳过前面的这些表，
     * 然后来计算当前库中的表
     * @param dbKeyAttr 路由字段
     */
    @Override
    public void doRouter(String dbKeyAttr) {

        // 获取所有表
        int size = dbRouterConfig.getDbCount() * dbRouterConfig.getTbCount();

        // 扰动函数；在 JDK 的 HashMap 中，对于一个元素的存放，需要进行哈希散列。而为了让散列更加均匀，所以添加了扰动函数。
        // 因此在这里借鉴 HashMap 源码
        // 得到具体的数据库表序号，比如第16张表
        int idx = (size - 1) & (dbKeyAttr.hashCode() ^ (dbKeyAttr.hashCode() >>> 16)); // 按位"与"和"异或"。

        // 库表索引；相当于是把一个长条的桶，切割成段，对应分库分表中的库编号和表编号
        // 获取对应的库，库是从1开始算的，因此要在此基础上+1
        // 得到具体的数据库序号，比如2号数据库
        int dbIdx = idx / dbRouterConfig.getTbCount() + 1;
        // 获取当前库的第几张表
        int tbIdx = idx - dbRouterConfig.getTbCount() * (dbIdx - 1);

        // 设置库表信息到上下文,String.format("%02d", dbIdx),数据不为两位的话则在前面补0,这里的策略主要和设置的库表名称有关
        // 例如: 库名称为test_01 那就写%02d。表名称user_001 对应%03d
        DBContextHolder.setDBKey(String.format("%02d", dbIdx));
        DBContextHolder.setTBKey(String.format("%03d", tbIdx));
        logger.debug("数据库路由 dbIdx：{} tbIdx：{}",  dbIdx, tbIdx);
    }

    @Override
    public void setDBKey(int dbIdx) {
        DBContextHolder.setDBKey(String.format("%02d", dbIdx));
    }

    @Override
    public void setTBKey(int tbIdx) {
        DBContextHolder.setTBKey(String.format("%03d", tbIdx));
    }

    @Override
    public int dbCount() {
        return dbRouterConfig.getDbCount();
    }

    @Override
    public int tbCount() {
        return dbRouterConfig.getTbCount();
    }

    @Override
    public void clear(){
        DBContextHolder.clearDBKey();
        DBContextHolder.clearTBKey();
    }

}
```

### DynamicDataSource

在动态数据源实现类中我们获取当前数据源路由key，如db02，实现数据源的切换，即分库。具体的数据源切换原理可以看核心代码分析。

```java
/**
 * 动态数据源获取，获取数据源时，都从这个里面进行获取
 */
public class DynamicDataSource extends AbstractRoutingDataSource {

    @Override
    protected Object determineCurrentLookupKey() {
        return "db" + DBContextHolder.getDBKey();
    }

}
```

### DynamicMybatisPlugin

![image-20231110105102321](https://github.com/Andrewmeo/images/blob/master/dbrouter/image-20231110105102321.png?raw=true)

> 在@Intercepts注解中定义具体拦截哪种类型，类中哪个方法，可能有多个一样的方法，拦截方法中的参数（更加一步的指定拦截的方法）：
>
> 1. type：具体拦截StatementHandler类型。
> 2. method：具体拦截prepare方法。
> 3. args：可能有多个一样的方法，拦截方法中的参数（更加一步的指定拦截的方法）。其中具体拦截了参数为数据库连接对象，以及Integer类型的transactionTimeout。该方法属于RoutingStatementHandler类，返回一个Statement对象。
>
> StatementHandler接口有一个BaseStatementHandler抽象实现类和RoutingStatementHandler实现类，抽象类下有三个子类。在进行SQL预编译之前进行拦截处理。

在intercept拦截方法中，通过Invocation获取StatementHandler对象，从对象中提取出SQL语句的元信息。然后从元信息中获取自定义注解判断是否进行分表操作，因为有可能只进行分库不分表的操作。

| 实现类                   | 说明                                             |
| ------------------------ | ------------------------------------------------ |
| RoutingStatementHandler  | 用于处理具体的组件。                             |
| BaseStatementHandler     | 用于是实现StatementHandler接口中子类公用的方法。 |
| CallableStatementHandler | 处理带有存储过程的SQL。                          |
| PreparedStatementHandler | 处理带有参数的SQL。                              |
| SimpleStatementHandler   | 处理不带参数的SQL。                              |

```java
/**
 * Mybatis 拦截器，通过对 SQL 语句的拦截处理，修改分表信息
 */
@Intercepts({@Signature(type = StatementHandler.class, method = "prepare", args = {Connection.class, Integer.class})})
public class DynamicMybatisPlugin implements Interceptor {

    private Pattern pattern = Pattern.compile("(from|into|update)[\\s]{1,}(\\w{1,})", Pattern.CASE_INSENSITIVE);

    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        // 获取StatementHandler
        StatementHandler statementHandler = (StatementHandler) invocation.getTarget();
        MetaObject metaObject = MetaObject.forObject(statementHandler, SystemMetaObject.DEFAULT_OBJECT_FACTORY, SystemMetaObject.DEFAULT_OBJECT_WRAPPER_FACTORY, new DefaultReflectorFactory());
        // MappedStatement 包含sql语句的元信息
        MappedStatement mappedStatement = (MappedStatement) metaObject.getValue("delegate.mappedStatement");

        // 获取自定义注解判断是否进行分表操作
        String id = mappedStatement.getId();
        String className = id.substring(0, id.lastIndexOf("."));
        Class<?> clazz = Class.forName(className);
        DBRouterStrategy dbRouterStrategy = clazz.getAnnotation(DBRouterStrategy.class);
        if (null == dbRouterStrategy || !dbRouterStrategy.splitTable()){
            return invocation.proceed();
        }

        // 获取SQL
        BoundSql boundSql = statementHandler.getBoundSql();
        String sql = boundSql.getSql();

        // 替换SQL表名 USER 为 USER_001
        Matcher matcher = pattern.matcher(sql);
        String tableName = null;
        if (matcher.find()) {
            tableName = matcher.group().trim();
        }
        assert null != tableName;

        // 生成替换的SQL语句，通过线程工作空间ThreadLocal获取到分表路由
        String replaceSql = matcher.replaceAll(tableName + "_" + DBContextHolder.getTBKey());

        // 通过反射修改SQL语句
        Field field = boundSql.getClass().getDeclaredField("sql");
        field.setAccessible(true);
        field.set(boundSql, replaceSql);
        field.setAccessible(false);

        return invocation.proceed();
    }

}
```

## 5 核心代码分析

### 5.1 数据源切换

#### DataSourceAutoConfig

```java
    /**
     * 用于配置 TargetDataSources 以及 DefaultTargetDataSource
     * TargetDataSources: 额外的数据源
     * 可以用指定的key获取其他的数据源来达到动态切换数据源
     * DefaultTargetDataSource: 默认的数据源
     * 如果没有要用的数据源就会使用默认的数据源
     * @return
     */
    @Bean
    public DataSource dataSource() {
        Map<Object, Object> targetDataSources = new HashMap<>();
        for (String dbInfo : dataSourceMap.keySet()) {
            Map<String, Object> objMap = dataSourceMap.get(dbInfo);
            targetDataSources.put(dbInfo, new DriverManagerDataSource(objMap.get("url").toString(), objMap.get("username").toString(), objMap.get("password").toString()));
        }

        // 设置数据源
        DynamicDataSource dynamicDataSource = new DynamicDataSource();
        dynamicDataSource.setTargetDataSources(targetDataSources);
        dynamicDataSource.setDefaultTargetDataSource(new DriverManagerDataSource(defaultDataSourceConfig.get("url").toString(), defaultDataSourceConfig.get("username").toString(), defaultDataSourceConfig.get("password").toString()));

        return dynamicDataSource;
    }
```

#### DynamicDataSource

数据源切换的重点在于DynamicDataSource，该动态数据源类继承了AbstractRoutingDataSource。

```java
/**
 * 动态数据源获取，获取数据源时，都从这个里面进行获取
 */
public class DynamicDataSource extends AbstractRoutingDataSource {

    @Override
    protected Object determineCurrentLookupKey() {
        return "db" + DBContextHolder.getDBKey();
    }
}
```

#### AbstractRoutingDataSource

跟进AbstractRoutingDataSource，其中就可以看到getConnection中调用类中的determineTargetDataSource方法返回数据源，然后获取到连接对象。我们继续进入determineTargetDataSource看看如何获取到数据源的。

![](https://github.com/Andrewmeo/images/blob/main/image-20231106200959642.png?raw=true)

在determineTargetDataSource中，首先做了一个断言，判断数据源是否存在。然后如果lookupKey为空，那么dataSource为空，默认的数据源就是resolvedDefaultDataSource。

![](https://github.com/Andrewmeo/images/blob/main/image-20231106200852517.png?raw=true)

所以数据源切换的原理就是通过重写determineCurrentLookupKey，传入数据源实现数据源的切换。

![](https://github.com/Andrewmeo/images/blob/main/image-20231106201039306.png?raw=true)

### 5.2 库表索引—扰动函数实现散列

单纯在Map设置索引很可能会出现哈希冲突。我们可以参考HashMap方法，使用扰动函数让hash值更加复杂，使得在数组上的映射更加散列。

### 5.3 HashMap中的hash值

首先我们要来了解HashMap的核心，hash值(key)。hash值是一段二进制数值，作为底层数组中一段数据的标识。那么HashMap是如何获得key的hash值？如何高效确认key映射的数组位置呢？

以JDK 1.8 版本查看HashMap中最为常用的方法之一：put(K key, V value)方法，我们进入HashMap中看看如何确认key的存储位置的。

![](https://github.com/Andrewmeo/images/blob/master/dbrouter/image-20231109201253648.png?raw=true)

这里key会先通过hash()方法进行处理。我们先进入该方法中。在hash()方法中先进行了key的空值判断，不为空后调用父类Object的hashCode()函数计算key的hash值，返回值为32位的int类型。然后右移16位，再和原来的进行异或运算。

![](https://github.com/Andrewmeo/images/blob/master/dbrouter/image-20231109201438578.png?raw=true)

这个过程相当于将计算出的hash值的高16位和低16位进行异或运算，为什么进行异或运算呢，异或运算相同为0，相异为1。高位和低位目的就是为了使得低16位的随机性增大。

![](https://github.com/Andrewmeo/images/blob/master/dbrouter/image-20231110005854674.png?raw=true)

所以hash()方法也叫扰动函数，目的就是为了增强随机性，使得存储的key更加均衡散列，减少哈希碰撞。其返回的是一个int类型的值，范围在[-2147483638, 2147483647]，所以说在hash()方法计算出的hash值可以映射到40亿长的数组中。

### 5.4 分库分表中库表路由计算

对来看数据库中库表的映射路由计算就好理解了。

路由计算：

```java
	    // 获取所有表
        int size = dbRouterConfig.getDbCount() * dbRouterConfig.getTbCount();

        // 扰动函数；在 JDK 的 HashMap 中，对于一个元素的存放，需要进行哈希散列。而为了让散列更加均匀，所以添加了扰动函数。
        // 因此在这里借鉴 HashMap 源码
        // 得到具体的数据库表序号，比如第16张表
        int h;
        int idx = (size - 1) & (h = dbKeyAttr.hashCode()) ^ (h >>> 16); // 按位"与"和"异或"。

        // 库表索引；相当于是把一个长条的桶，切割成段，对应分库分表中的库编号和表编号
        // 获取对应的库，库是从1开始算的，因此要在此基础上+1
        // 得到具体的数据库序号，比如2号数据库
        int dbIdx = idx / dbRouterConfig.getTbCount() + 1;
        // 获取当前库的第几张表
        int tbIdx = idx - dbRouterConfig.getTbCount() * (dbIdx - 1);
```

首先看右边部分，获取dbKeyAttr哈希运算结果哈希运算结果，进行位运算不足32位会向高位补零。将dbKeyAttr右移16位后再与原来的二进制数进行"异或运算"。

```java
int idx = (size - 1) & (dbKeyAttr.hashCode() ^ (dbKeyAttr.hashCode() >>> 16));
```

如下图调试过程的呈现，8张表，通过传入的路由字段属性 dbKeyAttr 计算出8张表中的第5张，然后再计算出路由数据库02，以及该数据库的第001张表。

![](https://github.com/Andrewmeo/images/blob/master/dbrouter/image-20231108221356887.png?raw=true)

## 6 Spring的SPI机制

在SPI机制（服务提供者接口）中，服务提供者实现了某个接口的类，并在resource/META-INF/services目录下，创建一个以该接口全限定名命名的文件，在此文件中列出具体的全限定实现类名。服务使用者获取服务的实现会查找相关的SPI配置文件，就可以加载并初始化相应的服务实现。

如下是整个分库分表路由组整合到SpringBoot装配的Bean及其功能。

![](https://github.com/Andrewmeo/images/blob/master/dbrouter/image-20231107233550344.png?raw=true)

分库分表组件作为中间件整合到SpringBoot中使用，都会使用SPI来进行自动装配。在SpringBoot中，自动装配的底层还是Spring SPI。

### Java SPI

#### 服务提供

首先我们使用原生的Java SPI实现：

定义一个接口：

```java
public interface HelloSPI {

    String getName();

    void handle();

}
```

定义两个实现类，

```java
public class HelloSPI_01 implements HelloSPI {
    @Override
    public String getName() {
        return "1";
    }

    @Override
    public void handle() {
        System.out.println("执行"+getName());
    }
}
```

```java
public class HelloSPI_02 implements HelloSPI {
    @Override
    public String getName() {
        return "2";
    }

    @Override
    public void handle() {
        System.out.println("执行"+getName());
    }
}
```

在META-INF.services下创建文本文件，文件名为实现的接口的全限定名。文件内容是两个实现类的全限定名。

![](https://github.com/Andrewmeo/images/blob/master/dbrouter/image-20231107160833191.png?raw=true)

#### 服务调用

在原生的SPI，服务使用者使用 ServiceLoader 加载和使用服务。

服务使用者需要查找相关的SPI配置文件，从SPI文件中读取出全限定类名。

```java
public class AppStart {
    public static void main(String[] args) {
        ServiceLoader<HelloSPI> load = ServiceLoader.load(HelloSPI.class);
        Iterator<HelloSPI> iterator = load.iterator();
        while (iterator.hasNext()) {
            HelloSPI next = iterator.next();
            System.out.println("扫描到"+next.getName());
            next.handle();
        }
        System.out.println("执行结束");
    }
}
```

#### ServiceLoader

**ServiceLoader**是一个简单的服务提供者加载工具。是JDK6引进的一个特性。提供load方法加载接口类对象。

> ServiceLoader 实现了 Iterable 接口，所以它有迭代器的属性，这里主要都是实现了迭代器的 hasNext 和 next 方法。Iterable中的 hasNext 和 next 调用的是 lookupIterator 的相应 hasNext 和 next 方法，lookupIterator 是懒加载的查询迭代器。
>
> 其次，LazyIterator 中的 nextProviderClass 方法，静态变量PREFIX就是"META-INF/services/"目录，这也就是为什么需要在 classpath 下的"META-INF/services/"目录里创建一个以服务接口命名的文件。
>
> 最后，通过反射方法 Class.forName() 加载类对象，并用 newInstance 方法将类实例化，并把实例化后的类缓存到 providers 对象中，(LinkedHashMap<String,S>类型） 然后返回实例对象。

load 方法是通过获取当前线程的 **线程上下文类加载器** 实例来加载的。Java应用运行的初始线程的上下文类加载器默认是系统类加载器。这里其实 **破坏了双亲委派模型**，因为 Java 应用收到类加载的请求时，按照双亲委派模型会向上请求父类加载器完成，这里并没有这么做（有些面试官会问到破坏双亲委派模型相关问题，简单了解）。

![](https://github.com/Andrewmeo/images/blob/master/dbrouter/image-20231107210956274.png?raw=true)

通过断点调试可以看到我们从ServiceLoader的迭代器中获取到第一个实现类对象，并调用其中的handle方法。

![](https://github.com/Andrewmeo/images/blob/master/dbrouter/image-20231107161723147.png?raw=true)

最终执行结果：

![](https://github.com/Andrewmeo/images/blob/master/dbrouter/image-20231107162102307.png?raw=true)

### Spring SPI

Spring Boot 有一个与 SPI 相似的机制，但它并不完全等同于 Java 的标准 SPI。

Spring Boot 的自动配置机制主要依赖于 spring.factories 文件。这个文件可以在多个 jar 中存在，并且 Spring Boot 会加载所有可见的 spring.factories 文件。我们可以在这个文件中声明一系列的自动配置类，这样当满足某些条件时，这些配置类会自动被 Spring Boot 应用。

#### 服务提供

接下来展示与 Spring Boot 紧密相关的Spring SPI。创建Maven工程，引入spring-boot依赖。然后就是和上面一样创建一个接口，定义两个实现类。

注册服务：在 resources/META-INF 下创建一个文件名为 spring.factories。这个文件里，可以注册 HelloSPI_01 以及 HelloSPI_02 实现类。

![](https://github.com/Andrewmeo/images/blob/master/dbrouter/image-20231107184357870.png?raw=true)

注意这里`com.bree.javaspi.HelloSPI`是接口的全路径，而`com.bree.javaspi.impl.HelloSPI_01,com.bree.javaspi.impl.HelloSPI_02`是实现类的全路径。如果有多个实现类，它们应当用逗号分隔。

> spring.factories 文件中的条目键和值之间不能有换行，即 key=value 形式的结构必须在同一行开始。但是，如果有多个值需要列出（如多个实现类），并且这些值是逗号分隔的，那么可以使用反斜杠（`\`）来换行。spring.factories 的名称是约定俗成的。如果试图使用一个不同的文件名，那么 Spring Boot 的自动配置机制将不会识别它。

#### 服务调用

在Spring SPI中，对于服务使用者使用 SpringFactoriesLoader 来加载服务，SpringFactoriesLoader.loadFactories 的第二个参数是类加载器，此处我们使用默认的类加载器，所以传递 null。

这种方式利用了 Spring 的 SpringFactoriesLoader，它允许开发者提供接口的多种实现，并通过 spring.factories 文件来注册它们。这与 JDK 的 SPI 思想非常相似，只是在实现细节上有所不同。这也是 Spring Boot 如何自动配置的基础，它会查找各种 spring.factories 文件，根据其中定义的类来初始化和配置 bean。

```java
public class AppStart {
    public static void main(String[] args) {
        List<HelloSPI> helloSPIS = SpringFactoriesLoader.loadFactories(HelloSPI.class, null);
        for (HelloSPI helloSPI : helloSPIS) {
            System.out.println("扫描到"+helloSPI.getName());
            helloSPI.handle();
        }
    }
}
```

#### SpringFactoriesLoader

## 7 SPI应用场景

SPI 扩展机制应用场景有很多，比如 Common-Logging，JDBC，Dubbo等等。

SPI 流程：

1. 有关组织和公式定义接口标准。
2. 第三方服务提供者具体实现接口具体方法，配置 META-INF/services/${interface_name} 文件。
3. 服务使用者引入第三方服务

比如 JDBC 场景下：

1. 首先 Java 定义了接口 java.sql.Driver，并没有具体的实现，具体的实现都是由不同数据库厂商提供实现。

2. 在 MySQL 的 jar 包 mysql-connector-java-6.0.6.jar 中，可以在 META-INF/services 目录下找到一个名字为 java.sql.Driver 的文件，文件内容是 com.mysql.cj.jdbc.Driver，这里面的内容就是针对 Java 中定义的接口的实现。

同样在 PostgreSQL 的 jar 包 PostgreSQL-42.0.0.jar 中，也可以找到同样的配置文件，文件内容是 org.postgresql.Driver，这是 PostgreSQL 对 Java 的 java.sql.Driver 的实现。

## 8 SPI和API的区别

- API （Application Programming Interface）在大多数情况下，都是实现方制定接口并完成对接口的实现，调用方仅仅依赖接口调用，且无权选择不同实现。 从使用人员上来说，API 直接被应用开发人员使用。
- SPI （Service Provider Interface）是调用方来制定接口规范，提供给外部来实现，调用方在调用时则选择自己需要的外部实现。  从使用人员上来说，SPI 被框架扩展人员使用。