---
layout: post
title: Spring与SpringBoot学习笔记
categories: [lang, java]
tags: [Spring, SpringBoot]
date: 2020-12-18 20:00:00 +0800
---

## 前言
Spring是一个开源框架，它的核心是控制反转（Inversion of Control，简称IoC），它由Rod Johnson创建。Spring框架目标是简化Java企业级应用开发，简化开发复杂性。
SpringBoot是Spring的一个子项目，它的核心是自动配置（auto configuration），它由Pivotal团队提供。SpringBoot是简化Spring应用的初始搭建以及开发过程。

可见，学习Spring是学习SpringBoot的基础，SpringBoot只是在熟练使用Spring的基础上，进一步简化了Spring应用的搭建和开发过程。

## Spring学习

Spring Framework主要包括几个模块：

* 支持IoC和AOP的容器；
* 支持JDBC和ORM的数据访问模块；
* 支持声明式事务的模块；
* 支持基于Servlet的MVC开发；
* 支持基于Reactive的Web开发；
* 以及集成JMS、JavaMail、JMX、缓存等其他模块。

Spring框架不同版本依赖也有差异，列举如下：

| | Spring 5.x |	Spring 6.x |
| :--- | :--- | :--- |
| JDK版本	    | >= 1.6	        | >= 8 |
| Tomcat版本	| 7.x	            | 9.x |
| Annotation包	| javax.annotation	| jakarta.annotation |
| Servlet包	    | javax.servlet	    | jakarta.servlet |
| JMS包	        | javax.jms	        | jakarta.jms |
| JavaMail包	| javax.mail	    | jakarta.mail |

这里我们使用Spring 6.x版本，如果使用Spring的其他版本，则需要根据需要调整代码。下面依次介绍Spring的核心模块。

### IoC容器

容器：一个为某种特定组件的运行提供必要支持的一个软件系统。例如，Tomcat就是一个Servlet容器，它可以为Servlet的运行提供运行环境。
Spring的核心就是提供了一个**IoC容器**，它可以管理所有轻量级的JavaBean组件，提供的底层服务包括组件的生命周期管理、配置和组装服务、AOP支持，以及建立在AOP基础上的声明式事务服务等。
IoC（Inversion of Control，控制反转）是Spring的核心，它将对象的**创建**和**依赖注入**交给Spring容器来管理，从而实现了对象之间的解耦。在IoC模式下，控制权发生了反转，即从应用程序转移到了IoC容器，所有组件不再由应用程序自己创建和配置，而是由IoC容器负责，这样，**应用程序只需要直接使用已经创建好并且配置好的组件**。

IoC容器就是ApplicationContext（继承自BeanFactory，很少直接使用BeanFactory）的实例，ApplicationContext是一个接口，主要的实现类有ClassPathXmlApplicationContext（使用XML文件配置Bean）、FileSystemXmlApplicationContext和AnnotationConfigApplicationContext（常用，使用注解配置Bean）。通常我们直接使用AnnotationConfigApplicationContext使用注解配置。主要注解：
* @Configuration：用于定义配置类，相当于传统的XML配置文件
* @ComponentScan：用于指定要扫描的包，相当于XML配置文件中的<context:component-scan>元素
* @Component：用于将某个类定义Bean，相当于XML配置文件中的<bean>元素。注意：
    * 默认情况下：当我们把一个Bean标记为@Component后，它就会自动为我们创建一个**单例**（Singleton），即容器初始化时创建Bean，容器关闭前销毁Bean。
    * Prototype（原型）类型的Bean：每次调用getBean(Class)，容器都返回一个新的实例。可以通过@Scope注解来指定，如：`@Scope("prototype")`。
* @Bean：用于创建第三方的Bean，该Bean不在`@ComponentScan`配置扫描的包里，在`@Configuration`类中编写一个Java方法创建并返回它。相当于XML配置文件中的<bean>元素。如：
    ```java
    @Configuration
    public class AppConfig {
        @Bean
        public UserService createUserService() {
            return new UserService();
        }
    }
    ```
* @Autowired：用于自动注入Bean，相当于把指定类型的Bean注入到指定的字段。相当于XML配置文件中的<property>元素
* @Qualifier：用于指定要注入的Bean的别名，相当于XML配置文件中的<property>元素的name属性。`Bean+@Qualifier("name")`相当于`@Bean("name”)`
* @Value("classpath:/logo.txt")：用于注入资源文件到Resource成员变量，注入后直接调用Resource.getInputStream()就可以获取到输入流，避免了自己搜索文件的代码。
* @Value("${jdbc.url}")：用于注入配置文件中的属性值，注入后可以直接使用该属性值。相当于XML配置文件中的<property>元素的value属性。
* @Value("#{smtpConfig.host}")：注意观察#{}这种注入语法，它和${key}不同的是，**#{}表示从JavaBean读取属性**（即读取`smtpConfig.host`的值）进行注入。
* @PropertySource("app.properties")：用于注入配置文件的位置，相当于XML配置文件中的<context:property-placeholder>。注入后可以直接使用`@Value("${jdbc.url}")`方式直接注入配置文件的值到指定的成员变量中。
* @Profile("dev")：用于指定Bean的生效环境，相当于XML配置文件中的<beans profile="dev">。
* @Conditional：用于指定Bean的生效条件，相当于XML配置文件中的<beans>元素的condition属性。为便于使用，SpringBoot提供了更多扩展注解，如：@ConditionalOnProperty、@ConditionalOnClass、@ConditionalOnMissingBean等。

### AOP
不必关心过于复杂的AOP概念，只需要理解AOP本质上只是一种代理模式的实现方式，在Spring的容器中实现AOP特别方便。AOP**主要是在具体方法的执行前后，插入一些统一的预处理或收尾处理操作**，如：拦截器、过滤器。这里总结两种方法：

1. 使用AspectJ语法：通过Maven引入Spring对AOP的支持：`org.springframework:spring-aspects:6.0.0`。使用AspectJ实现AOP比较方便，只需：
    * 定义执行方法，并在方法上通过AspectJ的注解告诉Spring应该在何处调用此方法；
        * @Before("execution(* com.itranswarp.learnjava.service.UserService.*(..))")：表示在UserService类的所有方法执行前执行该方法；
        * @After("execution(* com.itranswarp.learnjava.service.UserService.*(..))")：表示在UserService类的所有方法执行后执行该方法；
        * @Around("execution(* com.itranswarp.learnjava.service.UserService.*(..))")：表示在UserService类的所有方法执行前后执行该方法；
        * @AfterReturning("execution(* com.itranswarp.learnjava.service.UserService.*(..))")：表示在UserService类的所有方法只有当目标代码正常返回时，才执行拦截器代码；
        * @AfterThrowing("execution(* com.itranswarp.learnjava.service.UserService.*(..))")：表示在UserService类的所有方法只有当目标代码抛出了异常时，才执行拦截器代码；
    * 在Bean类上标记@Component和@Aspect；
    * 在@Configuration类上标注@EnableAspectJAutoProxy。
2. 配合Annotation实现AOP：使用Annotation实现AOP可以做到更精准的控制AOP代理注入的范围，避免不需要AOP代理的Bean也被自动代理。实现上稍微复杂一些，参考：https://liaoxuefeng.com/books/java/spring/aop/annotation-config/index.html

无论是使用AspectJ语法，还是配合Annotation，使用AOP，实际上就是让Spring自动为我们创建一个Proxy，使得调用方能无感知地调用指定方法，但运行期却动态“织入”了其他逻辑，因此，AOP本质上就是一个代理模式。

注意：AOP有一些使用上的坑点，这里总结下正确使用AOP避坑指南（参考：https://liaoxuefeng.com/books/java/spring/aop/notice/index.html）：
* 访问被注入的Bean时，总是调用方法而非直接访问字段；
* 编写Bean时，如果可能会被代理，就不要编写public final方法。

### 访问数据库

#### 使用JdbcTemplate
Java程序使用JDBC接口访问关系数据库的时候，需要以下几步：

1. 创建全局DataSource实例，表示数据库连接池；
2. 在需要读写数据库的方法内部，按如下步骤访问数据库：
    * 从全局DataSource实例获取Connection实例；
    * 通过Connection实例创建PreparedStatement实例；
    * 执行SQL语句，如果是查询，则通过ResultSet读取结果集，如果是修改，则获得int结果。

正确编写JDBC代码的关键是使用try ... finally释放资源，涉及到事务的代码需要正确提交或回滚事务。

在Spring中使用JDBC，首先是通过IoC容器创建并管理一个DataSource实例，然后**使用该DataSource创建一个JdbcTemplate**就能方便地操作JDBC了。这里DataSource实例选用HikariDataSource，这是一个快速、简单、可靠的 JDBC 连接池，添加依赖包`com.zaxxer:HikariCP:5.0.1`即可。
本地测试时，涉及数据库的操作推荐使用**HSQLDB**这个数据库，添加依赖包`org.hsqldb:hsqldb:2.7.1`即可。它是一个用Java编写的关系数据库，可以以内存模式或者文件模式运行，本身只有一个jar包，非常适合演示代码或者测试代码。这样避免安装MySQL等重量级的数据库。

JdbcTemplate的用法总结如下是：

* 针对简单查询，优选query()和queryForObject()，因为只需提供SQL语句、参数和RowMapper；
* 针对更新操作，优选update()，因为只需提供SQL语句和参数；
* 任何复杂的操作，最终也可以通过execute(ConnectionCallback)实现，因为拿到Connection就可以做任何JDBC操作。

实际上我们使用最多的仍然是各种查询。如果在设计表结构的时候，能够和JavaBean的属性一一对应，那么直接使用BeanPropertyRowMapper就很方便。如果表结构和JavaBean不一致怎么办？那就需要稍微改写一下查询，使结果集的结构和JavaBean保持一致。

#### 使用声明式事务

Spring提供PlatformTransactionManager来表示事务管理器，所有的事务都由它负责管理。而事务由TransactionStatus表示。事务实际上是*配合Annotation实现的AOP代理*，在Spring中使用声明式事物，只需：
1. 在配置类中创建第三方Bean：PlatformTransactionManager。这里使用实际类型是DataSourceTransactionManager。
2. 在配置类添加@EnableTransactionManagement注解（不用额外添加@EnableAspectJAutoProxy），开启声明式事务。
3. 在需要事务的方法上添加@Transactional注解即可。

使用声明式事务的注意事项：
1. 声明式事务是基于AOP的，所以它只能作用于public方法，对于private方法无效。
2. 声明式事务默认是**只读**的，如果需要写操作，需要显式声明@Transactional(readOnly = false)。
3. 声明式事务默认是**回滚**的，如果需要不回滚，需要显式声明@Transactional(noRollbackFor = Exception.class)。
4. 事务的传播行为默认是REQUIRED，即在一个事务执行时如果调用了另一个标注了事务的方法，不会新建事务，而是加入已存在的事务中。如果需要其他行为，需要显式声明如：@Transactional(propagation = Propagation.REQUIRES_NEW)。事务的传播在实现上是使用ThreadLocal。Spring总是把JDBC相关的Connection和TransactionStatus实例绑定到ThreadLocal。如果一个事务方法从ThreadLocal未取到事务，那么它会打开一个新的JDBC连接，同时开启一个新的事务，否则，它就直接使用从ThreadLocal获取的JDBC连接以及TransactionStatus。

#### DAO 与 ORM
DAO即Data Access Object的缩写，它是一个面向对象的数据库访问层，本质上就是封装JdbcTemplate提供对数据进行增删改查的接口。DAO没啥复杂的，也不是必须的，很多时候直接在Service层操作数据库也是完全没有问题的。

ORM(Object-Relational Mapping)是一种把关系数据库的表记录映射为Java对象的过程。ORM框架有很多种，如Hibernate、MyBatis等。ORM框架的作用是把数据库表记录映射为Java对象，这样我们就可以直接操作Java对象，而不用直接操作数据库表了。ORM框架的核心是映射，即把数据库表记录映射为Java对象，以及把Java对象映射为数据库表记录。
Hibernate或JPA属于全自动ORM框架，这种框架通常提供了缓存，并且还分为一级缓存和二级缓存，增大了数据的不一致性。因此当前**更流行的是使用MyBatis**，它属于半自动ORM框架，它只负责把ResultSet自动映射到Java Bean，或者自动填充Java Bean参数，但仍需自己写出SQL确保每次操作一定是数据库操作而不是缓存。

## SpringBoot学习

SpringBoot是由Pivotal团队提供的全新框架，其设计目的是用来简化新Spring应用的初始搭建以及开发过程。本质上，SpringBoot是在Spring基础上封装的一套简化Spring开发的框架，它简化了Spring应用的初始搭建以及开发过程。它消除了大量的配置文件，使开发人员能够快速上手，并且SpringBoot还提供了大量的开箱即用的功能，如嵌入式服务器、安全、数据访问、监控等。其特点列举：

- 独立运行
- 简化配置
- 自动配置
- 无代码生成和xml配置
- 遵循“约定优于配置”的原则
- 提供了一系列大型项目通用的非业务性功能，如内嵌服务器、安全管理、运行数据监控、运行状况检查和外部化配置等
- 完全兼容Spring框架

Spring是SpringBoot的基础，是必须掌握的，SpringBoot则是实际开发的主力，加速开发。

### 编写SpringBoot程序

有了前面的Spring基础，编写SpringBoot程序其实是非常简单也非常容易的，可以直接参考：https://liaoxuefeng.com/books/java/springboot/index.html 。这里就不做展开了。

编写过程中有几个坑点记录一下：
1. 创建SpringBoot初始项目，通常有两种方式：
    * 使用“start.spring.io”官网创建。通常我们选用这种该方式，但是官网访问可能受限，此时如果有网络环境可以替换成阿里云的start.aliyun.com。
    * 使用idea创建maven项目，自行引入SpringBoot依赖。如果实在没网络，推荐考虑使用这种方式，不建议自行搭建startspring网站，因为部署时有各种依赖问题。

2. 国内环境IDEA获取maven依赖会失败，需要修改settings.xml文件，添加阿里云的maven仓库地址如下，并重启发起同步：
    ```
    <?xml version="1.0" encoding="UTF-8"?>
    <settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
            xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0 http://maven.apache.org/xsd/settings-1.0.0.xsd">
        <mirrors>
            <mirror>
                <id>alimaven</id>
                <name>aliyun maven</name>
                <url>http://maven.aliyun.com/nexus/content/groups/public/</url>
                <mirrorOf>central</mirrorOf>
            </mirror>
        </mirrors>
    </settings>
    ```

### 常用工具、组件

常用工具：
1. 开发环境实现修改代码后自动生效：直接添加 spring-boot-devtools 即可，无需改动代码
    ```
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-devtools</artifactId>
    </dependency>
    ```

2. 应用打包：
    * 加入SpringBoot自带的spring-boot-maven-plugin插件用来打包：
        ```
        <build>
            <plugins>
                <plugin>
                    <groupId>org.springframework.boot</groupId>
                    <artifactId>spring-boot-maven-plugin</artifactId>
                    <configuration>
                        <excludeDevtools>true</excludeDevtools>
                    </configuration>
                </plugin>
            </plugins>
        </build>
        ```
    * 打包执行：`mvn clean package`；生成文件在`target`目录下。
    * 运行：`java -jar target/xxx.jar`

3. 应用程序状态监控：SpringBoot已经内置了一个监控功能Actuator，正常启动应用程序，Actuator会把它能收集到的所有信息都暴露给JMX。
    * 添加依赖：
        ```
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        ```
    * 启动应用程序，访问`http://localhost:8080/actuator`，可以看到所有可用的监控信息。
    * 使用jconsole连接应用程序，可以看到所有可用的监控信息。

4. 使用Profiles和Conditional注解实现多环境配置。
5. @Configuration + @ConfigurationProperties("storage.local") 可以用于将配置文件中的属性映射到Java类中，方便获取配置。

常用组件：
1. Open API：Open API是一个标准，它的主要作用是描述REST API，同时让机器根据这个描述自动生成客户端文档。依赖`org.springdoc:springdoc-openapi-starter-webmvc-ui:2.5.0`

2. 访问Redis作为生产环境缓存：Spring框架提供SpringCache用于缓存，SpringCache中提供的两个主要接口是CacheManager和Cache，CacheManager管理Cache，Cache负责缓存具体的键值对。
    生产环境中，我们使用Redis作为缓存，引入`spring-boot-starter-data-redis`依赖，SpringBoot会自动将Redis整合到SpringCache框架中，我们只需调整必要配置文件即可，参考：[Springboot 整合 SpringCache 使用 Redis 作为缓存](https://www.cnblogs.com/hanzhe/p/16935954.html)。
    Redisy依赖中主要用到了这几个组件：
    * Lettuce：一个基于Netty的高性能Redis客户端；
    * RedisTemplate：一个类似于JdbcTemplate的接口，用于简化Redis的操作。

3. 使用消息队列：在微服务场景，**消息队列是必不可少的组件**，它用于实现微服务之间的解耦，提高系统的可扩展性，此外一些涉及消息预处理（分类、限流等）的工作也可以在消息队列实现。常见的消息队列有：
    * Artemis实现了JMS接口，仅用于java语言，不跨语言。有两种类型的消息通道：
        * Queue
        * Topic：一对多的Topic，即Producer发送消息到指定的Topic，任意多个在线的接收方均可从Topic获得一份完整的消息副本。
    * RabbitMQ实现了AMQP接口，即高级消息队列协议，它定义了一种二进制格式的消息流，任何编程语言都可以实现该协议。有两种类型的消息通道：
        * Queue
        * Exchange：当Producer想要发送消息的时候，它将消息发送给Exchange，由Exchange将消息根据各种规则投递到一个或多个Queue
    * Kafka实现了Kafka协议，即Kafka协议，它定义了一种二进制格式的消息流，任何编程语言都可以实现该协议。特点：
        * 分区消费：Kafka只有一种类似JMS的Topic的消息通道，一个Topic可以有一个至多个Partition，并且可以分布到多台机器上，实现高可用和高吞吐量。注意消费者从多个Partition消费消息，并不保证消息有序。
        * 批处理：消息发送和接收都尽量使用批处理，一次处理几十甚至上百条消息，比一次一条效率要高很多。
        * 持久性：Kafka总是将消息写入Partition对应的文件，消息保存多久取决于服务器的配置，可以按照时间删除。
    * RabbitMQ、Kafka、redis实现消息队列对比。
        * redis也可以用于实现消息队列，不过redis的持久化机制是异步的，并且redis的持久化机制是内存映射，所以redis的消息队列无法解决消息丢失问题。因此，redis更适合作为缓存，而不是消息队列。
        * RabbitMQ吞吐量是每秒几万条消息，通常用于实时的、对可靠性要求较高的消息传递场景。它以broker为中心，有消息的确认机制，确保消息被正确处理。RabbitMQ适合于需要复杂路由、消息优先级、延迟队列等高级特性的场景。如果需要**依赖消息队列实现比较复杂的业务功能，推荐使用RabbitMQ**。如：
            * 营销系统需要涉及复杂的匹配规则对广告进行分发，使用RabbitMQ在消息中添加 routing_key 或者自定义消息头就可以轻松实现。
            * 超时自动取消订单：这种场景使用消息队列是为了让超时的状态能被多个其他微服务组件感知，使用RabbitMQ的延迟队列功能，可以轻松实现订单超时自动取消的功能。
        * Kafka每秒几十万条消息吞吐，性能比RabbitMQ高很多，通常用于实时的、对吞吐量要求较高的消息传递场景，如**大数据埋点收集**等。它以partition为中心，有消息的持久化机制，确保消息不会丢失。不过Kafka不支持过于复杂的高级特性，如消息优先级、延迟队列等，在这些需求场景下需要做很多开发工作。

4. 使用ElasticSearch实现实时搜索：ElasticSearch是一个分布式搜索引擎，它提供全文搜索功能，并且支持分布式集群。SpringBoot提供了SpringDataElasticSearch，可以方便的将ElasticSearch整合到SpringBoot中。

5. 使用Dubbo实现RPC：Dubbo是一个高性能的RPC框架，它提供了一种简单易用的方式来实现远程调用。Dubbo支持多种协议，如Dubbo协议、Hessian协议、HTTP协议等。Dubbo还支持多种注册中心，如Zookeeper、Consul、Etcd等。SpringBoot提供了SpringBootDubbo，可以方便的将Dubbo整合到SpringBoot中。调用关系说明：
    * 服务容器负责启动，加载，运行服务提供者。
    * 服务提供者在启动时，向注册中心注册自己提供的服务。
    * 服务消费者在启动时，向注册中心订阅自己所需的服务。
    * 注册中心返回服务提供者地址列表给消费者，如果有变更，注册中心将基于长连接推送变更数据给消费者。
    * 服务消费者，从提供者地址列表中，基于软负载均衡算法，选一台提供者进行调用，如果调用失败，再选另一台调用。
    * 服务消费者和提供者，在内存中累计调用次数和调用时间，定时每分钟发送一次统计数据到监控中心。


## SpringCloud学习

Spring Cloud 为开发人员提供工具，以便快速构建分布式系统中一些常见的**模式**（例如配置管理、服务发现、断路器、智能路由、微代理、控制总线、短暂微服务和契约测试）。分布式系统的协调会导致**样板模式**，而使用 Spring Cloud，开发人员可以快速启动实现这些模式的服务和应用程序。

SpringBoot是SpringCloud的基础，SpringCloud是**基于SpringBoot实现**的。SpringBoot专注于快速、方便地开发单个个体微服务，SpringCloud是关注全局的微服务协调整理治理框架，它将SpringBoot开发的一个个单体微服务整合并管理起来，为各个微服务之间提供，配置管理、服务发现、断路器、路由、微代理、事件总线、全局锁、决策竞选、分布式会话等等集成服务。
SpringBoot专注于快速、方便的开发单个微服务个体，SpringCloud关注全局的服务治理框架。

SpringCloud是一个包含独立项目的伞形项目，这些项目可以一起工作，也可以单独使用，以构建分布式系统。SpringCloud项目包括：
* Spring Cloud Config：集中配置管理工具，支持本地存储、Git存储等。
* Spring Cloud Netflix：Netflix OSS的集成，包括Eureka、Hystrix、Zuul、Archaius等组件。
* Spring Cloud Azure：用于连接到Azure云服务的组件。
* Spring Cloud Alibaba：用于连接到阿里巴巴的分布式解决方案
* Spring Cloud for Amazon Web Services：用于与AWS云服务集成的组件。
* Spring Cloud Bus：事件总线，用于在微服务之间进行通信。
* Spring Cloud Gateway：API网关，用于路由和过滤API请求。
* Spring Cloud Security：基于Spring Security的安全框架，用于保护微服务。
* Spring Cloud Sleuth：分布式追踪系统，用于跟踪微服务之间的调用链。
* Spring Cloud Stream：基于消息驱动的微服务框架，用于构建消息驱动的微服务。
* Spring Cloud Task：用于构建短生命周期的微服务，如批处理任务。
* Spring Cloud Zookeeper：使用Zookeeper作为注册中心的微服务框架。
* Spring Cloud Consul：使用Consul作为注册中心的微服务框架。
* Spring Cloud Kubernetes：使用Kubernetes作为容器编排系统的微服务框架。
* ...


应用实例：[Spring Cloud开发](https://liaoxuefeng.com/books/java/springcloud/index.html)