# Quartz框架实现定时任务

- [Quartz框架实现定时任务](#quartz框架实现定时任务)
  - [需求描述](#需求描述)
  - [Quartz框架](#quartz框架)
  - [Spring Boot 结合Quartz](#spring-boot-结合quartz)
  - [Quartz在Spring中集群](#quartz在spring中集群)
  - [具体代码实现](#具体代码实现)
  - [Quartz的XML实现](#quartz的xml实现)
  - [Quartz VS. Spring Scheduler](#quartz-vs-spring-scheduler)
  - [参考资料](#参考资料)

> created by 邢文怡

## 需求描述

基于SpringBoot框架的工程，需要使用Quartz实现定时任务，且支持**持久化**和**分布式集群**应用。

> 原有定时任务使用Spring自身的Scheduler实现，但实际应用中存在一些问题，故用Quartz进行替换。
>
> Spring Scheduler 与 Quartz Scheduler的区别参见后文。

## Quartz框架

参考：[quartz（从原理到应用）详解篇](https://blog.csdn.net/lkl_csdn/article/details/73613033)

**quartz调度核心元素**：

1. Scheduler:任务调度器，是实际执行任务调度的控制器。在spring中通过SchedulerFactoryBean封装起来。Quartz调度器的SchedulerFactoryBean提供了两种方式：内存RAMJobStore和数据库方式
    - 内存RAMJobStore：job的相关信息存储在内存里，每个节点存储各自的，互相隔离
    - 数据库方式：job的相关信息存储在数据库中，所有节点共用数据库，每个节点通过数据库来通信，保证一个job同一时间只会在一个节点上执行，并且如果某个节点挂掉，job会被分配到其他节点执行。
2. Trigger：触发器，用于定义任务调度的时间规则，有SimpleTrigger,CronTrigger,DateIntervalTrigger和NthIncludedDayTrigger，其中CronTrigger用的比较多，本文主要介绍这种方式。CronTrigger在spring中封装在CronTriggerFactoryBean中。
3. Calendar:它是一些日历特定时间点的集合。一个trigger可以包含多个Calendar，以便排除或包含某些时间点。
4. JobDetail:用来描述Job实现类及其它相关的静态信息，如Job名字、关联监听器等信息。在spring中有JobDetailFactoryBean和 MethodInvokingJobDetailFactoryBean两种实现，如果任务调度只需要执行某个类的某个方法，就可以通过MethodInvokingJobDetailFactoryBean来调用。
5. Job：是一个接口，只有一个方法void execute(JobExecutionContext context),开发者实现该接口定义运行任务，JobExecutionContext类提供了调度上下文的各种信息。Job运行时的信息保存在JobDataMap实例中。实现Job接口的任务，默认是无状态的，若要将Job设置成有状态的，在quartz中是给实现的Job添加@DisallowConcurrentExecution注解（以前是实现StatefulJob接口，现在已被Deprecated）,在与spring结合中可以在spring配置文件的job detail中配置concurrent参数。

## Spring Boot 结合Quartz

* Spring Boot 结合了Quartz Scheduler框架，提供`spring-boot-starter-quartz`依赖。

> 而**无需**像其它参考资料上所写的添加如下quartz依赖：
>
> ```xml
> <dependency>
>     <groupId>org.quartz-scheduler</groupId>
>     <artifactId>quartz</artifactId>
>     <version>x.x.x</version>
> </dependency>
> ```

* 可以使用 `spring.quartz`属性和`spring.quartz.properties.*`自定义高级Quartz配置属性。

> 如果需要自定义任务执行程序，请考虑实现`SchedulerFactoryBeanCustomizer`。

默认情况下，使用内存中的 `JobStore` 。但是，如果应用程序中有 `DataSource` bean，并且相应地配置了 `spring.quartz.job-store-type` 属性，则可以配置基于JDBC的存储，如以下示例所示：

```yaml
spring.quartz.job-store-type=jdbc
```

使用JDBC存储时，可以在启动时初始化架构，如以下示例所示：

```yaml
spring.quartz.jdbc.initialize-schema=always
```

> 默认情况下，使用Quartz库提供的标准脚本检测并初始化数据库。这些脚本删除现有表，在每次重启时删除所有触发器。也可以通过设置spring.quartz.jdbc.schema属性来提供自定义脚本。
>
> 可参见下方[具体代码实现]()中的涉及内容。

## Quartz在Spring中集群

一个 Quartz 集群中的每个节点是一个独立的 Quartz 应用，它又管理着其他的节点。意思是你必须对每个节点分别启动或停止。不像许多应用服务器的集群，独立的 Quartz 节点并不与另一其的节点或是管理节点通信。Quartz 应用是通过数据库表来感知到另一应用的。

图：表示了每个节点直接与数据库通信，若离开数据库将对其他节点一无所知

1. **创建Quartz数据库表**

   Quartz集群依赖于数据库，所以必须在数据库中创建相关库表。Quartz包括了所有被支持的数据库平台的SQL脚本。不同Quartz版本，所需数据库表个数不同。当前最新2.3.0版本默认数据库表包含如下11张表：

   | 表名                     | 描述                                                         |
      | ------------------------ | ------------------------------------------------------------ |
   | QRTZ_CALENDARS           | 以Blob类型存储Quartz的Calendar信息                           |
   | QRTZ_FIRED_TRIGGERS      | 存储与已触发的Trigger相关的状态信息，以及相联Job的执行信息   |
   | QRTZ_BLOB_TRIGGERS       | Trigger作为Blob类型存储（用于Quartz用户用JDBC创建他们自己定制的Trigger类型，JobStore并不知道如何存储实例的时候） |
   | QRTZ_CRON_TRIGGERS       | 存储CronTrigger，包括Cron表达式和时区信息                    |
   | QRTZ_SIMPLE_TRIGGERS     | 存储简单的Trigger，包括重复次数、间隔、以及已触发的次数      |
   | QRTZ_SIMPROP_TRIGGERS    | 存储CalendarintervalTrigger和DailyTimeIntervalTrigger两种类型的触发器 |
   | QRTZ_TRIGGERS            | 存储已配置的Trigger的信息                                    |
   | QRTZ_JOB_DETAILS         | 存储每一个已配置的Job的详细信息                              |
   | QRTZ_PAUSED_TRIGGER_GRPS | 存储已暂停的Trigger组的信息                                  |
   | QRTZ_LOCKS               | 存储程序的悲观锁的信息                                       |
   | QRTZ_SCHEDULER_STATE     | 存储少量的有关 Scheduler 的状态信息，和别的 Scheduler实例（假如是用于一个集群中） |

   > Quartz库表更多详情请参考：
   >
   > [Quartz 定时任务相关介绍表](https://blog.csdn.net/Q772363685/article/details/85329006)
   >
   > https://flylib.com/books/en/2.65.1/creating_the_quartz_database_structure.html

2. **配置数据库连接池**

   quartz配置文件中需设置如下。

   > 此类格式为`quartz.properties`的写法，若使用与Spring Boot结合的`spring.quartz.*`写法（见下文[具体代码实现]()），内容大致相同。

   ```yaml
   # 选择JDBC连接方式（JobStoreTX或JobStoreCMT）
   org.quartz.jobStore.class = org.quartz.impl.jdbcjobstore.JobStoreTX
   # 选定JDBC代理类（StdJDBCDelegate、OracleDelegate、PostgreSQLDelegate等），通常使用StdJDBCDelegate即可
   org.quartz.jobStore.driverDelegateClass = org.quartz.impl.jdbcjobstore.StdJDBCDelegate
   # 指定数据库表前缀（默认QRTZ_）
   org.quartz.jobStore.tablePrefix = QRTZ_
   # 指定数据源名称：
   org.quartz.jobStore.dataSource = myDS
   # 配置数据源属性
   org.quartz.dataSource.myDS.driver: com.mysql.jdbc.Driver
   org.quartz.dataSource.myDS.url: jdbc:mysql:***:@***:***:***
   org.quartz.dataSource.myDS.user: root
   org.quartz.dataSource.myDS.password: root
   ```

3. **配置 Quartz 使用集群**

   群集仅适用于JDBC-Jobstore（JobStoreTX或JobStoreCMT），通过将`org.quartz.jobStore.isClustered`属性设置为`true`来启用聚类。集群中的每个节点必须具有唯一的instanceId，通过将“AUTO”作为此属性的值，可以轻松完成。有关更多信息请参考[使用JDBC-JobStore配置群集](https://www.w3cschool.cn/quartz_doc/quartz_doc-3x7u2doc.html)

   ```yaml
   #============================================================================
   # Configure Main Scheduler Properties  
   #============================================================================
   org.quartz.scheduler.instanceName = MyClusteredScheduler
   org.quartz.scheduler.instanceId = AUTO
   
   #============================================================================
   # Configure ThreadPool  
   #============================================================================
   org.quartz.threadPool.class = org.quartz.simpl.SimpleThreadPool
   org.quartz.threadPool.threadCount = 25
   org.quartz.threadPool.threadPriority = 5
   
   #============================================================================
   # Configure JobStore  
   #============================================================================
   org.quartz.jobStore.misfireThreshold = 60000
   
   org.quartz.jobStore.isClustered = true
   org.quartz.jobStore.clusterCheckinInterval = 20000
   ```

Quartz实际并不关心你是在相同的还是不同的机器上运行节点。当集群是放置在不同的机器上时，通常称之为水平集群。节点是跑在同一台机器是，称之为垂直集群。

当你运行水平集群时，时钟应当要同步，以免出现离奇且不可预知的行为。假如时钟没能够同步，Scheduler实例将对其他节点的状态产生混乱。最简单的同步计算机时钟的方式是使用某一个Internet时间服务器(Internet Time Server ITS)。

若在相同环境中使用集群的和非集群的 Quartz 应用。唯一要注意的是这两个环境不可混用相同的数据库表。意思是非集群环境不要使用与集群应用相同的一套数据库表；否则将得到希奇古怪的结果，集群和非集群的 Job 都会遇到问题。

> 参考资料：
>
> [Quartz在Spring中集群](https://my.oschina.net/flythisway/blog/607459)

## 具体代码实现

1. 添加Quartz框架的依赖

   在工程的pom.xml文件中添加如下依赖：

   ```xml
   <dependency>
       <groupId>org.springframework.boot</groupId>
       <artifactId>spring-boot-starter-quartz</artifactId>
   </dependency>
   ```

2. 创建相关数据库表

   在pom.xml中添加依赖后，依赖包`Maven:org.quartz-scheduler:quartz:2.3.0\quartz-2.3.0.jar\org\quartz\impl\jdbcjobstore`中包含了默认的库表初始化脚本，例如`tables_oracle.sql`等。根据项目数据库情况，选择并执行对应脚本。

   本项目使用Oracle数据库，则采用`tables_oracle.sql`脚本。

3. 在application.yml配置文件中添加Quartz配置：

   ```yaml
   spring:
     quartz:
       job-store-type: jdbc # 数据库方式
       jdbc:
         initialize-schema: always # 数据库表结构初始化模式
         schema: classpath:quartzschema/initialize_tables_oracle.sql # 自定义数据库表初始化脚本（脚本内容见下一步骤）
       properties: # 附加属性
         org:
           quartz:
             scheduler:
               instanceName: clusteredScheduler
               instanceId: AUTO
             jobStore:
               class: org.quartz.impl.jdbcjobstore.JobStoreTX # 持久化配置
               driverDelegateClass: org.quartz.impl.jdbcjobstore.StdJDBCDelegate # 特定于数据库的代理
               tablePrefix: QRTZ_ # 数据库表前缀
               isClustered: true # 打开群集功能
               clusterCheckinInterval: 20000 # 设置此实例“检入”*与群集的其他实例的频率（以毫秒为单位）。影响检测失败实例的速度。
               useProperties: true # 以指示JDBCJobStore将JobDataMaps中的所有值都作为字符串，因此可以作为 名称-值对 存储而不是在BLOB列中以其序列化形式存储更多复杂的对象。从长远来看，这是更安全的，因为避免了将非String类序列化为BLOB的类版本问题。
             threadPool: # 连接池
               class: org.quartz.simpl.SimpleThreadPool
               threadCount: 10
               threadPriority: 5
               threadsInheritContextClassLoaderOfInitializingThread: true
   ```

   > 注意：配置文件中其它位置已有数据库相关配置如下。故无需再配置数据源（若不存在则请添加）
   >
   > ```yaml
   > spring:
   >     datasource:
   >        username: USERNAME
   >        password: PASSWORD
   >        url: jdbc:oracle:thin:@***:***:***
   > ```

3. 创建自定义的Quartz数据库表初始化脚本`initialize_tables_oracle.sql`

   ```sql
   delete from qrtz_fired_triggers;
   delete from qrtz_simple_triggers;
   delete from qrtz_simprop_triggers;
   delete from qrtz_cron_triggers;
   delete from qrtz_blob_triggers;
   delete from qrtz_triggers;
   delete from qrtz_job_details;
   delete from qrtz_calendars;
   delete from qrtz_paused_trigger_grps;
   delete from qrtz_locks;
   delete from qrtz_scheduler_state;
   ```

   > 此步骤并非必须，根据各自项目需求决定是否使用自定义脚本。默认情况下，使用Quartz库提供的标准脚本检测并初始化数据库。

4. 创建具体的作业类（Job类），继承`QuartzJobBean `，并重写`executeInternal`方法，该方法中的逻辑即为定时任务真正执行的业务逻辑。

   原有`BillingProcessing `类，使用Spring自身的Scheduler实现定时任务，其业务逻辑为`checkBillingProcessing()`方法。

   > 原Spring Scheduler实现如下：
   >
   > ```java
   > @Component
   > @Slf4j
   > @ConditionalOnProperty(name = "schedules.enabled.billingProcessing", havingValue = "true")
   > public class BillingProcessing {
   >     @Autowired
   >     private OmWLFlowtrackMapper omWlFlowtrackMapper; // OmWLFlowtrackMapper为接口类型
   > 
   >     @Scheduled(cron = "${schedules.billingProcessing.cron:0/30 * * * * ?}")
   >     public void checkBillingProcessing() {
   >         log.debug("checkBillingProcessing begin...");
   >         …… // 略去具体业务逻辑
   >         log.debug("checkBillingProcessing end.");
   >     }
   > }
   > ```

   将`BillingProcessing `类改为Quartz的Job类，实现如下：

   ```java
   @Slf4j
   @DisallowConcurrentExecution
   @PersistJobDataAfterExecution
   public class BillingProcessing extends QuartzJobBean {
       @Autowired
       private OmWLFlowtrackMapper omWlFlowtrackMapper; // OmWLFlowtrackMapper为接口类型
       
       private void checkBillingProcessing() {
           log.debug("checkBillingProcessing begin...");
           …… // 略去具体业务逻辑
           log.debug("checkBillingProcessing end.");
       }
   
       @Override
       public void executeInternal(JobExecutionContext jobExecutionContext) throws JobExecutionException {
           checkBillingProcessing();
       }
   }
   ```

5. JUnit单元测试

   ```java
   @RunWith(SpringRunner.class)
   @SpringBootTest
   public class ApplicationTest {
       @Autowired
       private Scheduler scheduler;
   
       @Value("${schedules.billingProcessing.cron:0/30 * * * * ?}")
       private String cronBillingProcessing;
   
       @Test
       public void testBillingProcessing() throws Exception {
   
           JobDetail jobDetail = JobBuilder.newJob(BillingProcessing.class)
                   .withIdentity("billingProcessingJob")
                   .storeDurably(true)
                   .build();
   
           Trigger trigger = TriggerBuilder.newTrigger()
                   .forJob(jobDetail)
                   .withIdentity("billingProcessingTrigger")
                   .withSchedule(cronSchedule(cronBillingProcessing))
                   .build();
   
           scheduler.scheduleJob(jobDetail, trigger);
   
           Thread.sleep(120000);
       }
   }
   ```

6. 创建JobDetail和Trigger触发器。JobDetail描述Job类及其相关的静态信息，如Job名字等；Trigger触发器，用于定义任务调度的时间规则。

   ```java
   @Configuration
   @ConditionalOnProperty(name = "schedules.enabled.billingProcessing", havingValue = "true")
   public class BillingProcessingJobConfig {
       @Value("${schedules.billingProcessing.cron:0/30 * * * * ?}")
       private String cronBillingProcessing;
   
       @Bean
       public JobDetail billingProcessingJobDetail() {
           return JobBuilder.newJob(BillingProcessing.class)
                   .withIdentity("billingProcessingJob")
                   .storeDurably(true)
                   .build();
       }
   
       @Bean
       public Trigger billingProcessingJobTrigger() {
           return TriggerBuilder.newTrigger()
                   .forJob(billingProcessingJobDetail())
                   .withIdentity("billingProcessingTrigger")
                   .withSchedule(cronSchedule(cronBillingProcessing))
                   .build();
       }
   }
   ```

   > 如果Quartz可用，`spring-boot-starter-quartz`会自动配置`Scheduler`（通过 `SchedulerFactoryBean` 抽象）。
   >
   > 自动拾取以下类型的Bean并与 `Scheduler` 关联：
   >
   > - `JobDetail` ：定义一个特定的作业。可以使用 `JobBuilder` API构建 `JobDetail` 实例
   > - `Calendar`：日历/日期计划
   > - `Trigger` ：定义何时触发特定作业

## Quartz的XML实现

实现步骤和示例：

[Spring+quartz集群配置，Spring定时任务集群，quartz定时任务集群](https://www.cnblogs.com/fanshuyao/p/6227116.html)

[Quartz-Spring集成Quartz通过XML配置的方式](https://blog.csdn.net/yangshangwei/article/details/78505730#示例-jobdetailfactorybean)

**存在问题：**

当前项目定时任务类中，使用了对接口类型的自动装配如下：

```java
public class BillingProcessing {
 @Autowired
 private OmWLFlowtrackMapper omWlFlowtrackMapper; // OmWLFlowtrackMapper为接口类型

 public void checkBillingProcessing() {
     log.debug("checkBillingProcessing begin...");
     …… // 略去具体业务逻辑
     log.debug("checkBillingProcessing end.");
 }
}
```

若需使用XML方式进行实现，则需要在quartz的xml配置文件中配置bean：

```xml
<bean id="omWlFlowtrackMapper" class="com.huawei.esop.esopscheduleservice.mapper.OmWLFlowtrackMapper">
```

但接口类型不允许直接在XML中配置，会报错。因此本项目不适用xml的实现方法。

## Quartz VS. Spring Scheduler

Spring Scheduler是一个轻量级，可满足简单的调度需求的方案。如果您使用的是Spring 3.0，则它为任务调度和异步方法执行提供注释支持。Spring Scheduler提供了对

- 使用[**FixedRate**](http://docs.spring.io/spring/docs/current/spring-framework-reference/html/scheduling.html) （即使先前执行未完成也以特定间隔定期运行） 和[**FixedDelay**](http://docs.spring.io/spring/docs/current/spring-framework-reference/html/scheduling.html) （在上一次执行完成后将任务的下一次执行延迟特定时间范围）的任务计划
- 基于 **Cron**表达式的调度

相反，Quartz Scheduler是功能完善的开源库，为Job Scheduling提供支持。它比Spring Scheduler相对复杂，但是为诸如JTA和集群之类的企业级功能提供支持。

- 通过使用JDBCJobStore，所有配置为“不易丢失”的作业和触发器都将通过JDBC存储在关系数据库中。
- 通过使用RAMJobStore，所有作业和触发器都存储在RAM中，因此不会在程序执行之间持久存在，但这具有不需要外部数据库的优点。

Quartz的群集功能可用于故障安全或负载平衡目的。

本质上，如果您的目标是实现一种快速，基本的作业/任务调度形式，那么Spring Scheduler将是理想的选择。另一方面，如果您需要集群以及JobPersistence支持，那么Quartz可能会更好。

参考：[Quartz Scheduler vs. Spring Scheduler](https://khalidsaleem.blogspot.com/2015/03/quartz-scheduler-vs-spring-scheduler.html)

## 参考资料

[Spring Boot Quartz Scheduler](https://www.docs4dev.com/docs/zh/spring-boot/2.1.1.RELEASE/reference/boot-features-quartz.html#quartz-scheduler)

[Spring整合Quartz分布式调度](https://zhuanlan.zhihu.com/p/35506135)

[quartz储存方式之JDBC JobStoreTX](https://blog.csdn.net/Uhzgnaw/article/details/46358333)

[Quartz快速入门指南](https://www.w3cschool.cn/quartz_doc/quartz_doc-2put2clm.html)

[quartz （从原理到应用）详解篇](https://blog.csdn.net/lkl_csdn/article/details/73613033)

[SpringBoot2.0.3整合Quartz2.3.0实现定时任务](https://blog.csdn.net/qq_39241251/article/details/84817741)

[springBoot整合Quartz定时任务（持久化到数据库）](https://blog.csdn.net/HXNLYW/article/details/95055601)

[QuartzAutoConfigurationTests.java](https://github.com/spring-projects/spring-boot/blob/master/spring-boot-project/spring-boot-autoconfigure/src/test/java/org/springframework/boot/autoconfigure/quartz/QuartzAutoConfigurationTests.java)

扩展：

[简单 Quartz 微服务，不支持分布式](https://segmentfault.com/a/1190000011228626)

[简单 Quartz-Cluster 微服务，支持集群分布式，并支持动态修改 Quartz 任务的 cronExpression 执行时间](https://gitee.com/ylimhhmily/SpringCloudTutorial/tree/master/springms-simple-quartz-cluster)