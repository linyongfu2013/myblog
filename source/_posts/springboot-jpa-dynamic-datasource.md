---
title: SpringBoot Jpa 实现多数据源动态切换
tags:
- SpringBoot
- Jpa
categories:
- SpringBoot
---

在大型应用程序中，配置主从数据库并使用读写分离是常见的设计模式。常用的实现方式是使用数据库中间件，此文介绍如何通过编写代码的方式实现多数据源的配置和动态切换。核心是使用Spring 内置的 `AbstractRoutingDataSource` 这个抽象类，它可以把多个数据源配置成一个Map，然后，根据不同的key返回不同的数据源。

#### 环境介绍
* SpringBoot 1.5.10.RELEASE
* MySQL 5.7

#### 数据源配置
首先在 `application.yml` 里配置两个数据源：
```
spring:
  datasource:   #多数据源配置
    master:
      url: jdbc:mysql://localhost:3307/testdb?useUnicode=true&characterEncoding=utf8&useSSL=false
      username: root
      password: 123456
      driver-class-name: com.mysql.jdbc.Driver
    slave:
      url: jdbc:mysql://localhost:3308/testdb?useUnicode=true&characterEncoding=utf8&useSSL=false
      username: root
      password: 123456
      driver-class-name: com.mysql.jdbc.Driver
    type: com.alibaba.druid.pool.DruidDataSource

  jpa:
    show-sql: true
#    hibernate:
#      naming: 这个属性不知道为什么无法自动获取到，需要在代码赋值
#        physical-strategy: org.springframework.boot.orm.jpa.hibernate.SpringPhysicalNamingStrategy
server:
  port: 8080
  context-path: /imooc

```

#### 初始化数据源
编写数据源配置类，初始化数据源，并把两个物理数据源封装成一个`AbstractRoutingDataSource`：
```
@Configuration
public class DataSourceConfiguration {
    private final static String MASTER_DATASOURCE_KEY = "masterDataSource";
    private final static String SLAVE_DATASOURCE_KEY = "slaveDataSource";

    @Value("${spring.datasource.type}")
    private Class<? extends DataSource> dataSourceType;

    @Primary
    @Bean(value = MASTER_DATASOURCE_KEY)
    @Qualifier(MASTER_DATASOURCE_KEY)
    @ConfigurationProperties(prefix = "spring.datasource.master")
    public DataSource masterDataSource() {
        log.info("create master datasource...");
        return DataSourceBuilder.create().type(dataSourceType).build();
    }

    @Bean(value = SLAVE_DATASOURCE_KEY)
    @Qualifier(SLAVE_DATASOURCE_KEY)
    @ConfigurationProperties(prefix = "spring.datasource.slave")
    public DataSource slaveDataSource() {
        log.info("create slave datasource...");
        return DataSourceBuilder.create().type(dataSourceType).build();
    }

    @Bean(name = "routingDataSource")
    public AbstractRoutingDataSource routingDataSource(@Qualifier("masterDataSource") DataSource masterDataSource,
            @Qualifier("slaveDataSource") DataSource slaveDataSource) {
        DynamicDataSourceRouter proxy = new DynamicDataSourceRouter();
        Map<Object, Object> targetDataSources = new HashMap<>(2);
        targetDataSources.put("masterDataSource", masterDataSource);
        targetDataSources.put("slaveDataSource", slaveDataSource);

        proxy.setDefaultTargetDataSource(masterDataSource);
        proxy.setTargetDataSources(targetDataSources);
        return proxy;
    }
}

```
注意需要把其中一个数据源使用`@Primary` 注解标明为主数据源，并且这个主数据源不能是`AbstractRoutingDataSource`类型的，必须是`DataSource` 类型的。

#### 编写 `JpaEntityManager` 配置类
使用多数据源后，需要手动对 `Jpa` 的 `EntityManager` 进行初始化和配置，不能使用默认的自动配置，不然的话并不能实际创建两个不同的数据源。
```
@Configuration
@EnableConfigurationProperties(JpaProperties.class)
@EnableJpaRepositories(value = "com.imooc.dao.repository")
public class JpaEntityManager {

    @Autowired
    private JpaProperties jpaProperties;

    @Resource(name = "routingDataSource")
    private DataSource routingDataSource;

    //@Primary
    @Bean(name = "entityManagerFactoryBean")
    public LocalContainerEntityManagerFactoryBean entityManagerFactoryBean(EntityManagerFactoryBuilder builder) {
        // 不明白为什么这里获取不到 application.yml 里的配置
        Map<String, String> properties = jpaProperties.getProperties();
        //要设置这个属性，实现 CamelCase -> UnderScore 的转换
        properties.put("hibernate.physical_naming_strategy",
                "org.springframework.boot.orm.jpa.hibernate.SpringPhysicalNamingStrategy");

        return builder
                .dataSource(routingDataSource)//关键：注入routingDataSource
                .properties(properties)
                .packages("com.imooc.entity")
                .persistenceUnit("myPersistenceUnit")
                .build();
    }

    @Primary
    @Bean(name = "entityManagerFactory")
    public EntityManagerFactory entityManagerFactory(EntityManagerFactoryBuilder builder) {
        return this.entityManagerFactoryBean(builder).getObject();
    }

    @Primary
    @Bean(name = "transactionManager")
    public PlatformTransactionManager transactionManager(EntityManagerFactoryBuilder builder) {
        return new JpaTransactionManager(entityManagerFactory(builder));
    }
}
```

#### 编写动态保存数据源类型`key`的实现类
使用 `ThreadLocal` 来动态设置和保存数据源类型的`key`
```
public class DataSourceContextHolder {
    private static final ThreadLocal<String> holder = new ThreadLocal<>();

    public static void setDataSource(String type) {
        holder.set(type);
    }

    public static String getDataSource() {
        String lookUpKey = holder.get();
        return lookUpKey == null ? "masterDataSource" : lookUpKey;
    }

    public static void clear() {
        holder.remove();
    }
}
```

#### 实现`AbstractRoutingDataSource`
编写一个类继承`AbstractRoutingDataSource`，并重写 `determineCurrentLookupKey` 这个路由方法：
```
public class DynamicDataSourceRouter extends AbstractRoutingDataSource {
    @Override
    protected Object determineCurrentLookupKey() {
        return DataSourceContextHolder.getDataSource();
    }
}
```

#### 编写切面实现动态切换
日常工作中，通常是根据`Service` 层的方法签名，区分读写操作，最便捷的方式是使用 AOP 进行拦截：
```
@Slf4j
@Aspect
@Component
public class DynamicDataSourceAspect {

    @Pointcut("execution(* com.imooc.service..*.*(..))")
    private void aspect() {}

    @Around("aspect()")
    public Object around(ProceedingJoinPoint joinPoint) throws Throwable {
        String method = joinPoint.getSignature().getName();

        if (method.startsWith("find") || method.startsWith("select") || method.startsWith("query") || method
                .startsWith("search")) {
            DataSourceContextHolder.setDataSource("slaveDataSource");
            log.info("switch to slave datasource...");
        } else {
            DataSourceContextHolder.setDataSource("masterDataSource");
            log.info("switch to master datasource...");
        }

        try {
            return joinPoint.proceed();
        }finally {
            log.info("清除 datasource router...");
            DataSourceContextHolder.clear();
        }
    }
}
```

#### 总结
至此，核心的代码和细节已经讲解结束，其余的实体类、`Repository`接口、Service 方法、测试用例等，可以参考我上传到 [Github](https://github.com/linyongfu2013/springboot-multi-datasource.git) 上的完整工程.
