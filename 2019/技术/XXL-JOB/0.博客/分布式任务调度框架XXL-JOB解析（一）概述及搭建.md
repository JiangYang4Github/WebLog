## 一、概述

### 1.背景

定时作业是在我们项目开发过程中比较常见的需求，比如商品定时上下架， 统计昨日的用户，财务报表统计等等，Quartz作为开源作业调度中的佼佼者，是作业调度的首选。但是集群环境中Quartz采用API的方式对任务进行管理，从而可以避免上述问题，但是同样存在以下问题：

- 问题一：调用API的的方式操作任务，不人性化；

- 问题二：需要持久化业务QuartzJobBean到底层数据表中，系统侵入性相当严重。

- 问题三：调度逻辑和QuartzJobBean耦合在同一个项目中，这将导致一个问题，在调度任务数量逐渐增多，同时调度任务逻辑逐渐加重的情况下，此时调度系统的性能将大大受限于业务；

- 问题四：quartz底层以“抢占式”获取DB锁并由抢占成功节点负责运行任务，会导致节点负载悬殊非常大；而XXL-JOB通过执行器实现“协同分配式”运行任务，充分发挥集群优势，负载各节点均衡。

XXL-JOB弥补了quartz的上述不足之处。

### 2.设计思想

将调度行为抽象形成“调度中心”公共平台，而平台自身并不承担业务逻辑，“调度中心”负责发起调度请求。

将任务抽象成分散的JobHandler，交由“执行器”统一管理，“执行器”负责接收调度请求并执行对应的JobHandler中业务逻辑。

因此，“**调度**”和“**任务**”两部分可以相互解耦，提高系统整体稳定性和扩展性；

### 3.系统组成

- **调度模块（调度中心）**： 负责管理调度信息，按照调度配置发出调度请求，自身不承担业务代码。调度系统与任务解耦，提高了系统可用性和稳定性，同时调度系统性能不再受限于任务模块； 支持可视化、简单且动态的管理调度信息，包括任务新建，更新，删除，GLUE开发和任务报警等，所有上述操作都会实时生效，同时支持监控调度结果以及执行日志，支持执行器Failover。
- **执行模块（执行器）**： 负责接收调度请求并执行任务逻辑。任务模块专注于任务的执行等操作，开发和维护更加简单和高效； 接收“调度中心”的执行请求、终止请求和日志请求等。

### 4.自研调度模块

XXL-JOB最终选择自研调度组件（早期调度组件基于Quartz）；一方面是为了精简系统降低冗余依赖，另一方面是为了提供系统的可控度与稳定性；

XXL-JOB中的“调度模块”和“任务模块”完全解耦，调度模块进行任务调度时，将会解析不同的任务参数发起远程调用，调用各自的远程执行器服务。这种调用模型类似RPC调用，调度中心提供调用代理的功能，而执行器提供远程服务的功能。

## 二、快速开始

### 1.代码结构

好了，看完了官网的说明，接下来我们先把demo跑起来，先在[github]( https://github.com/xuxueli/xxl-job )上下载代码后解压，按照maven格式将源码导入IDEA，使用maven进行编译即可，源码结构如下：

- xxl-job-admin：调度中心

- xxl-job-core：公共依赖

-  xxl-job-executor-samples：提供了多种开箱即用的执行器示例，选择合适的版本执行器，可直接使用，也可以参考其并将现有项目改造成执行器。

### 2.数据库和配置

将/doc/db/tables_xxl_job.sql此脚本文件在数据库中执行即可。

修改配置文件application.properties，主要datasource的信息配置成正确的即可。

### 3.启动调度中心

我们这里使用XxlConfAdminApplication作为启动类启动，会发现启动的非常快，访问http://localhost:8080/xxl-job-admin，账号密码为admin/123456，登录后即可看到如下页面

![](https://github.com/JiangYang4Github/WebLog/blob/master/2019/%E6%8A%80%E6%9C%AF/XXL-JOB/6.%E5%85%B6%E4%BB%96/%E8%B0%83%E5%BA%A6%E4%B8%AD%E5%BF%83%E9%A6%96%E9%A1%B5.png?raw=true) 

调度中心默认内置了一个任务，如下所示

![](https://github.com/JiangYang4Github/WebLog/blob/master/2019/%E6%8A%80%E6%9C%AF/XXL-JOB/6.%E5%85%B6%E4%BB%96/%E9%BB%98%E8%AE%A4%E4%BB%BB%E5%8A%A1%E9%A1%B5%E9%9D%A2.png?raw=true) 

### 4.启动实例执行器

XXL-JOB提供了多种类型的执行器实例，我们这里使用springboot版本的执行器进行学习。

 ![](https://github.com/JiangYang4Github/WebLog/blob/master/2019/%E6%8A%80%E6%9C%AF/XXL-JOB/6.%E5%85%B6%E4%BB%96/%E5%8C%85%E7%BB%93%E6%9E%84%E6%88%AA%E5%9B%BE.png?raw=true) 

其中第二个handler——DemoJobHandler，即是刚刚看到的那个默认创建任务对应的Handler，其中的类注释已经非常详细了，不再累述。

```java
/**
 * 任务Handler示例（Bean模式）
 *
 * 开发步骤：
 * 1、继承"IJobHandler"：“com.xxl.job.core.handler.IJobHandler”；
 * 2、注册到Spring容器：添加“@Component”注解，被Spring容器扫描为Bean实例；
 * 3、注册到执行器工厂：添加“@JobHandler(value="自定义jobhandler名称")”注解，注解value值对应的是调度中心新建任务的JobHandler属性的值。
 * 4、执行日志：需要通过 "XxlJobLogger.log" 打印执行日志；
 *
 * @author xuxueli 2015-12-19 19:43:36
 */
@JobHandler(value="demoJobHandler")
@Component
public class DemoJobHandler extends IJobHandler {

    @Override
    public ReturnT<String> execute(String param) throws Exception {
        XxlJobLogger.log("XXL-JOB, Hello World.");

        for (int i = 0; i < 5; i++) {
            XxlJobLogger.log("beat at:" );
            TimeUnit.SECONDS.sleep(2);
        }
        return SUCCESS;
    }

}
```

修改配置

```properties
# web port
server.port=8081

# log config
logging.config=classpath:logback.xml

### 调度中心部署跟地址 [选填]：如调度中心集群部署存在多个地址则用逗号分隔。执行器将会使用该地址进行"执行器心跳注册"和"任务结果回调"；为空则关闭自动注册；
xxl.job.admin.addresses=http://127.0.0.1:8080/xxl-job-admin
 
### 执行器AppName [选填]：执行器心跳注册分组依据；为空则关闭自动注册
xxl.job.executor.appname=xxl-job-executor-sample
 
### 执行器运行日志文件存储磁盘路径 [选填] ：需要对该路径拥有读写权限；为空则使用默认路径；
xxl.job.executor.logpath=/app/xxl-job/jobhandler
 
### 执行器日志保存天数 [选填] ：值大于3时生效，启用执行器Log文件定期清理功能，否则不生效；
xxl.job.executor.logretentiondays=4

```

其中需要注意的配置项为**调度中心地址**和所属的**执行器**，配置正确后，启动XxlJobExecutorApplication，发现这台执行器已经成功注册到**示例执行器**下,可以接受Handler为demoJobHandler的任务调度请求了！

![](https://github.com/JiangYang4Github/WebLog/blob/master/2019/%E6%8A%80%E6%9C%AF/XXL-JOB/6.%E5%85%B6%E4%BB%96/%E6%89%A7%E8%A1%8C%E5%99%A8%E5%88%97%E8%A1%A8%E9%A1%B5%E9%9D%A2.png?raw=true) 

### 5.已有项目接入

如果已有的项目想要接入调度中心呢？完成下面六个简单的步骤即可完成接入。

#### 5.1引入maven依赖

```
<dependency>
    <groupId>com.xitu</groupId>
    <artifactId>xxl-job-core</artifactId>
    <version>2.1.0</version>
</dependency>
```

#### 5.2配置执行器信息

```properties
# web port
server.port=8081

# log config
logging.config=classpath:logback.xml

### 调度中心部署跟地址 [选填]：如调度中心集群部署存在多个地址则用逗号分隔。执行器将会使用该地址进行"执行器心跳注册"和"任务结果回调"；为空则关闭自动注册；
xxl.job.admin.addresses=http://127.0.0.1:8080/xxl-job-admin
 
### 执行器AppName [选填]：执行器心跳注册分组依据；为空则关闭自动注册
xxl.job.executor.appname=xxl-job-executor-sample
 
### 执行器运行日志文件存储磁盘路径 [选填] ：需要对该路径拥有读写权限；为空则使用默认路径；
xxl.job.executor.logpath=/app/xxl-job/jobhandler
 
### 执行器日志保存天数 [选填] ：值大于3时生效，启用执行器Log文件定期清理功能，否则不生效；
xxl.job.executor.logretentiondays=4
```

#### 5.3执行器组件配置

```java
@Configuration
public class XxlJobConfig {
    private Logger logger = LoggerFactory.getLogger(XxlJobConfig.class);

    @Value("${xxl.job.admin.addresses}")
    private String adminAddresses;

    @Value("${xxl.job.executor.appname}")
    private String appName;

    @Value("${xxl.job.executor.ip}")
    private String ip;

    @Value("${xxl.job.executor.port}")
    private int port;

    @Value("${xxl.job.accessToken}")
    private String accessToken;

    @Value("${xxl.job.executor.logpath}")
    private String logPath;

    @Value("${xxl.job.executor.logretentiondays}")
    private int logRetentionDays;


    @Bean(initMethod = "start", destroyMethod = "destroy")
    public XxlJobSpringExecutor xxlJobExecutor() {
        logger.info(">>>>>>>>>>> xxl-job config init.");
        XxlJobSpringExecutor xxlJobSpringExecutor = new XxlJobSpringExecutor();
        xxlJobSpringExecutor.setAdminAddresses(adminAddresses);
        xxlJobSpringExecutor.setAppName(appName);
        xxlJobSpringExecutor.setIp(ip);
        xxlJobSpringExecutor.setPort(port);
        xxlJobSpringExecutor.setAccessToken(accessToken);
        xxlJobSpringExecutor.setLogPath(logPath);
        xxlJobSpringExecutor.setLogRetentionDays(logRetentionDays);

        return xxlJobSpringExecutor;
    }
}
```

#### 5.4编写定时任务JobHandler

```
/**
 * 任务Handler示例（Bean模式）
 *
 * 开发步骤：
 * 1、继承"IJobHandler"：“com.xxl.job.core.handler.IJobHandler”；
 * 2、注册到Spring容器：添加“@Component”注解，被Spring容器扫描为Bean实例；
 * 3、注册到执行器工厂：添加“@JobHandler(value="自定义jobhandler名称")”注解，注解value值对应的是调度中心新建任务的JobHandler属性的值。
 * 4、执行日志：需要通过 "XxlJobLogger.log" 打印执行日志；
 *
 * @author xuxueli 2015-12-19 19:43:36
 */
@JobHandler(value="demoJobHandler")
@Component
public class DemoJobHandler extends IJobHandler {
 
   @Override
   public ReturnT<String> execute(String param) throws InterruptedException {
      //...你的业务代码...
      return SUCCESS;
   }
}
```

#### 5.5调度中心添加执行器 

![](https://github.com/JiangYang4Github/WebLog/blob/master/2019/%E6%8A%80%E6%9C%AF/XXL-JOB/%E7%AC%AC%E4%B8%80%E7%AF%87%E5%9B%BE%E7%89%87/%E6%96%B0%E5%BB%BA%E6%89%A7%E8%A1%8C%E5%99%A8.png?raw=true)  

AppName: 是每个执行器集群的唯一标示AppName, 执行器会周期性以AppName为对象进行自动注册。可通过该配置自动发现注册成功的执行器, 供任务调度时使用，上文我们执行器注册的AppName为xxl-job-executor-sample，是默认创建的;
名称: 执行器的名称, 因为AppName限制字母数字等组成,可读性不强, 名称为了提高执行器的可读性;
排序: 执行器的排序, 系统中需要执行器的地方,如任务新增, 将会按照该排序读取可用的执行器列表;
注册方式：调度中心获取执行器地址的方式；
    自动注册：执行器自动进行执行器注册，调度中心通过底层注册表可以动态发现执行器机器地址；
    手动录入：人工手动录入执行器的地址信息，多地址逗号分隔，供调度中心使用；
机器地址："注册方式"为"手动录入"时有效，支持人工维护执行器的地址信息；

#### 5.6调度中心添加任务

![](https://github.com/JiangYang4Github/WebLog/blob/master/2019/%E6%8A%80%E6%9C%AF/XXL-JOB/%E7%AC%AC%E4%B8%80%E7%AF%87%E5%9B%BE%E7%89%87/%E6%96%B0%E5%BB%BA%E4%BB%BB%E5%8A%A1.png?raw=true) 

最后我们在调度中心维护个任务，执行器使用我们上文执行器注册的示例执行器（也就是xxl-job-executor-sample），JobHandler为我们执行器代码中JobHandler注解配置的值（也就是demoJobHandler），cron表达式网上有在线生成的工具，然后我们点击**任务管理**->**操作**->**执行一次**，会发现执行器已经被调用了。

## 三、后续

按照本文可以快速启动一个可以完成任务调度的调度中心和可执行任务的执行器，那调度中心和执行器之间的注册心跳是如何实现的？调度中心如何进行任务调度以及任务分发？任务的执行结果又是如何上报给调度中心的？自己在工作中正好接触到了这个框架，并花时间深入的阅读了源码，拟写如下几篇博客，记录自己学习的成果。

- 注册心跳
- 任务调度
- 触发任务
- 执行任务
- 结果上报
- 失败重试