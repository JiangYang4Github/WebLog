![](https://github.com/JiangYang4Github/WebLog/blob/master/2019/%E6%8A%80%E6%9C%AF/XXL-JOB/%E7%AC%AC%E4%BA%8C%E7%AF%87/index.jpg?raw=true)

正如第一篇文章所看到的，在我们启动一个执行器之后，我们会在一个延迟时间之后在调度中心看到这个注册上来的执行器，那在XXL-JOB框架中是如何实现的呢？我们先来看执行器这边。

## 一、执行器

我们在执行器端配置了个Bean，如下。

```java
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
```

显而易见，我们直奔XxlJobSpringExecutor类的start方法。

### 1.执行器初始化

```java
@Override
public void start() throws Exception {

	// init JobHandler Repository
	initJobHandlerRepository(applicationContext);

	// refresh GlueFactory
	GlueFactory.refreshInstance(1);

	// super start
	super.start();
}
```

第一步初始化JobHandler的数据，代码非常简单，从spring的上下文中取出JobHandler接口的全部实现类，然后调用registJobHandler方法，传入JobHandler注解配置的value值和bean实例。

```java
private void initJobHandlerRepository(ApplicationContext applicationContext){
    if (applicationContext == null) {
    	return;
    }

    // init job handler action
    Map<String, Object> serviceBeanMap = applicationContext.getBeansWithAnnotation(JobHandler.class);

    if (serviceBeanMap!=null && serviceBeanMap.size()>0) {
        for (Object serviceBean : serviceBeanMap.values()) {
            if (serviceBean instanceof IJobHandler){
                String name = serviceBean.getClass().getAnnotation(JobHandler.class).value();
                IJobHandler handler = (IJobHandler) serviceBean;
            	if (loadJobHandler(name) != null) {
            		throw new RuntimeException("xxl-job jobhandler naming conflicts.");
            	}
            	registJobHandler(name, handler);
            }
        }
    }
}
```

registJobHandler方法为XxlJobSpringExecutor的父类XxlJobExecutor实现，代码如下

```java
// ---------------------- job handler repository ----------------------
private static ConcurrentMap<String, IJobHandler> jobHandlerRepository = new ConcurrentHashMap<String, IJobHandler>();
public static IJobHandler registJobHandler(String name, IJobHandler jobHandler){
	return jobHandlerRepository.put(name, jobHandler);
}
public static IJobHandler loadJobHandler(String name){
	return jobHandlerRepository.get(name);
}
```

嗯，所谓的初始化JobHandler就是以JobHandler注解配置的value值为key，bean实例为value放入XxlJobExecutor的静态集合类jobHandlerRepository中。

第二步就是初始化执行glue脚本的工厂类，不细说。

第三步是才重点，我们来看父类的start方法到底干了什么。

### 2.创建代理

```java
public void start() throws Exception {
    // 日志相关
    XxlJobFileAppender.initLogPath(logPath);

    // 创建调度中心接口代理类
    initAdminBizList(adminAddresses, accessToken);

    // 日志相关
    JobLogFileCleanThread.getInstance().start(logRetentionDays);

    // 结果回调
    TriggerCallbackThread.getInstance().start();

    // 启动执行器服务
    port = port>0?port: NetUtil.findAvailablePort(9999);
    ip = (ip!=null&&ip.trim().length()>0)?ip: IpUtil.getIp();
    initRpcProvider(ip, port, appName, accessToken);
}
```

我们来看创建调度中心接口代理类和启动执行器服务的代码实现。

```java
// ---------------------- admin-client (rpc invoker) ----------------------
private static List<AdminBiz> adminBizList;
private static Serializer serializer;
private void initAdminBizList(String adminAddresses, String accessToken) throws Exception{
	serializer = Serializer.SerializeEnum.HESSIAN.getSerializer();
    if (adminAddresses!=null && adminAddresses.trim().length()>0) {
		for (String address: adminAddresses.trim().split(",")) {
			if (address!=null && address.trim().length()>0) {
                	//AdminBiz.MAPPING等于"/api"
                    String addressUrl = address.concat(AdminBiz.MAPPING);
                    AdminBiz adminBiz = (AdminBiz) new XxlRpcReferenceBean(
                            NetEnum.NETTY_HTTP,
                            serializer,
                            CallType.SYNC,
                            LoadBalance.ROUND,
                            AdminBiz.class,
                            null,
                            3000,
                            addressUrl,
                            accessToken,
                            null,
                            null
                    ).getObject();
                    if (adminBizList == null) {
                        adminBizList = new ArrayList<AdminBiz>();
                    }
                    adminBizList.add(adminBiz);
            }
		}
	}
}
```

利用XxlRpcReferenceBean创建了AdminBiz接口的多个代理类（调度中心支持集群部署，多地址），放入静态集合adminBizList中，XxlRpcReferenceBean这个类是他兄弟XXL-RPC框架的，熟悉RPC知识的同学应该明白，这里返回的是他的动态代理类，然后对方法调用进行拦截处理，组装参数发起远程调用。来看AdminBiz接口提供了哪些方法。

```java
/**
 * @author xuxueli 2017-07-27 21:52:49
 */
public interface AdminBiz {

    public static final String MAPPING = "/api";

    /**
     * 任务执行结果回调
     *
     * @param callbackParamList
     * @return
     */
    public ReturnT<String> callback(List<HandleCallbackParam> callbackParamList);

    /**
     * 注册
     *
     * @param registryParam
     * @return
     */
    public ReturnT<String> registry(RegistryParam registryParam);

    /**
     * 注销
     *
     * @param registryParam
     * @return
     */
    public ReturnT<String> registryRemove(RegistryParam registryParam);

}
```

好，现在我们利用XxlJobExecutor中的**adminBizList**，就可以完成执行器的任务结果回调，注册和注销操作了，那我们是在哪里调用的呢？我们去启动执行器服务这一步一探究竟。

### 3.发起注册or注销

```java
private void initRpcProvider(String ip, int port, String appName, String accessToken){
    // init, provider factory
    String address = IpUtil.getIpPort(ip, port);
    Map<String, String> serviceRegistryParam = new HashMap<String, String>();
    serviceRegistryParam.put("appName", appName);
    serviceRegistryParam.put("address", address);

    xxlRpcProviderFactory = new XxlRpcProviderFactory();
    xxlRpcProviderFactory.initConfig(NetEnum.NETTY_HTTP, Serializer.SerializeEnum.HESSIAN.getSerializer(), ip, port, accessToken, ExecutorServiceRegistry.class, serviceRegistryParam);

    // add services
    // 向Provider新增一个服务
    xxlRpcProviderFactory.addService(ExecutorBiz.class.getName(), null, new ExecutorBizImpl());

    // start
    // Provider 启动
    xxlRpcProviderFactory.start();

}
```

如果我们把任务调度这个动作看成RPC来说，那执行器相当于服务的提供端（完成任务的执行），调度中心相当于服务的消费端（负责任务的调用），所以我们这里创建了一个RpcProviderFactory，然后直接看他启动时干了什么。

```
public void start() throws Exception {
    // start server
    serviceAddress = IpUtil.getIpPort(this.ip, port);
    server = netType.serverClass.newInstance();
    server.setStartedCallback(new BaseCallback() {		// serviceRegistry started
        @Override
        public void run() throws Exception {
            // 注册
            if (serviceRegistryClass != null) {
                serviceRegistry = serviceRegistryClass.newInstance();
                serviceRegistry.start(serviceRegistryParam);
                if (serviceData.size() > 0) {
                    serviceRegistry.registry(serviceData.keySet(), serviceAddress);
                }
            }
        }
    });
    server.setStopedCallback(new BaseCallback() {		// serviceRegistry stoped
        @Override
        public void run() {
            // 注销
            if (serviceRegistry != null) {
                if (serviceData.size() > 0) {
                    serviceRegistry.remove(serviceData.keySet(), serviceAddress);
                }
                serviceRegistry.stop();
                serviceRegistry = null;
            }
        }
    });
    server.start(this);
}
```

调用serviceRegistry的start方法完成注册，他的实现类是谁？回到上一步看initConfig方法的传参，是ExecutorServiceRegistry，它的start方法如下。

```java
public void start(Map<String, String> param) {
	//利用线程池完成注册操作
	ExecutorRegistryThread.getInstance().start(param.get("appName"),param.get("address"));
}
```

跟进去发现向线程池registryThread提交了个任务，操作如下

```java
public void run() {
    while (!toStop) {
        try {
            RegistryParam registryParam = new RegistryParam(RegistryConfig.RegistType.EXECUTOR.name(), appName, address);
            for (AdminBiz adminBiz: XxlJobExecutor.getAdminBizList()) {
                try {
                    ReturnT<String> registryResult = adminBiz.registry(registryParam);
                    if (registryResult!=null && ReturnT.SUCCESS_CODE == registryResult.getCode()) {
                        registryResult = ReturnT.SUCCESS;
                        //相关日志打印;
                        break;
                    } else {
                        //相关日志打印;
                    }
                } catch (Exception e) {
                    //相关日志打印;
                }
            }
        } catch (Exception e) {
            if (!toStop) {
                //相关日志打印;
            }
        }

        try {
            if (!toStop) {
                TimeUnit.SECONDS.sleep(RegistryConfig.BEAT_TIMEOUT);
            }
        } catch (InterruptedException e) {
            if (!toStop) {
                //相关日志打印;
            }
        }
    }

    // registry remove
    try {
        RegistryParam registryParam = new RegistryParam(RegistryConfig.RegistType.EXECUTOR.name(), appName, address);
        for (AdminBiz adminBiz: XxlJobExecutor.getAdminBizList()) {
            try {
                ReturnT<String> registryResult = adminBiz.registryRemove(registryParam);
                if (registryResult!=null && ReturnT.SUCCESS_CODE == registryResult.getCode()) {
                    registryResult = ReturnT.SUCCESS;
                    //相关日志打印;
                    break;
                } else {
                    //相关日志打印;
                }
            } catch (Exception e) {
                if (!toStop) {
                    //相关日志打印;
                }
            }
        }
    } catch (Exception e) {
        if (!toStop) {
            logger.error(e.getMessage(), e);
        }
    }
    //相关日志打印;
}
```

其中toStop为volatile修饰的变量，可保证多线程执行下的可见性，此处起到跳转注销操作的作用。

接下来我们进入循环，遍历XxlJobExecutor的**adminBizList**，取出每一个AdminBiz代理类调用registry方法进行多调度中心的注册，时序图如下：

整体时序图如下：

![](https://github.com/JiangYang4Github/WebLog/blob/master/2019/%E6%8A%80%E6%9C%AF/XXL-JOB/5.%E6%B3%A8%E5%86%8C%E5%BF%83%E8%B7%B3/%E6%89%A7%E8%A1%8C%E5%99%A8%E6%B3%A8%E5%86%8C%E5%92%8C%E5%BF%83%E8%B7%B3%E5%A4%84%E7%90%86%E6%97%B6%E5%BA%8F%E5%9B%BE.png?raw=true)

## 二、调度中心

经过我们上文的分析，调度中心接受注册信息的关键在于两个，第一， AdminBiz 实现类，第二，服务暴露。

### 1.服务实现

实现类AdminBizImpl在com.xxl.job.admin.service.impl包下，注册和注销代码如下：

```java
@Override
public ReturnT<String> registry(RegistryParam registryParam) {
    int ret = xxlJobRegistryDao.registryUpdate(registryParam.getRegistGroup(), registryParam.getRegistryKey(), registryParam.getRegistryValue());
    if (ret < 1) {
        xxlJobRegistryDao.registrySave(registryParam.getRegistGroup(), registryParam.getRegistryKey(), registryParam.getRegistryValue());
    }
    return ReturnT.SUCCESS;
}

@Override
public ReturnT<String> registryRemove(RegistryParam registryParam) {
    xxlJobRegistryDao.registryDelete(registryParam.getRegistGroup(), registryParam.getRegistryKey(), registryParam.getRegistryValue());
    return ReturnT.SUCCESS;
}
```

代码非常简单，我们现在重点去看他是怎么暴露出去的。

### 2.服务暴露

和执行器的思路一样，我们直接看调度中心的conf包下有啥玩意。

- XxlJobAdminConfig：将一些配置值、dao、service利用InitializingBean的机制转存给静态变量中，之后的操作非常方便，值得学习。

- XxlJobScheduler：重点，调度中心业务逻辑启动的入口。

  ```java
  @Override
  public void afterPropertiesSet() throws Exception {
      // 国际化相关
      initI18n();
  
      // 执行器在线状态监听
      JobRegistryMonitorHelper.getInstance().start();
  
      // 失败任务重试
      JobFailMonitorHelper.getInstance().start();
  
      // 暴露调度中心服务，点进去
      initRpcProvider();
  
      // 任务调度
      JobScheduleHelper.getInstance().start();
  
      logger.info(">>>>>>>>> init xxl-job admin success.");
  }
  ```

  ```java
  private void initRpcProvider(){
  
      XxlRpcProviderFactory xxlRpcProviderFactory = new XxlRpcProviderFactory();
      xxlRpcProviderFactory.initConfig(
              NetEnum.NETTY_HTTP,
              Serializer.SerializeEnum.HESSIAN.getSerializer(),
              null,
              0,
              XxlJobAdminConfig.getAdminConfig().getAccessToken(),
              null,
              null);
  
      // 将AdminBizImpl添加至xxlRpcProviderFactory中
      xxlRpcProviderFactory.addService(AdminBiz.class.getName(), null, XxlJobAdminConfig.getAdminConfig().getAdminBiz());
  
      // servlet handler
      servletServerHandler = new ServletServerHandler(xxlRpcProviderFactory);
  }
  
  private void stopRpcProvider() throws Exception {
      XxlRpcInvokerFactory.getInstance().stop();
  }
  
  public static void invokeAdminService(HttpServletRequest request, HttpServletResponse response) throws IOException, ServletException {
      servletServerHandler.handle(null, request, response);
  }
  
  ```

  initRpcProvider方法的代码很熟悉，和执行器端的服务暴露一样，但是调度中心AdminBiz并没有用 xxlRpcProviderFactory.start()暴露，而是直接创建了一个ServletServerHandler，那他是怎么被调用的？看invokeAdminService是啥时候被调用的不就行了！利用IDEA的快捷键，找到如下代码：

  ```java
  @Controller
  public class JobApiController implements InitializingBean {
  
  
      @Override
      public void afterPropertiesSet() throws Exception {
  
      }
  
  
  	//AdminBiz.MAPPING等于"/api"
      @RequestMapping(AdminBiz.MAPPING)
      @PermissionLimit(limit=false)
      public void api(HttpServletRequest request, HttpServletResponse response) throws IOException, ServletException {
          XxlJobScheduler.invokeAdminService(request, response);
      }
  
  
  }
  ```

  至此，调度中心的接收注册和注销的代码实现以及分析完了，时序图如下：

![](https://github.com/JiangYang4Github/WebLog/blob/master/2019/%E6%8A%80%E6%9C%AF/XXL-JOB/5.%E6%B3%A8%E5%86%8C%E5%BF%83%E8%B7%B3/%E8%B0%83%E5%BA%A6%E4%B8%AD%E5%BF%83%E6%8E%A5%E6%94%B6%E6%B3%A8%E5%86%8C%E5%92%8C%E5%BF%83%E8%B7%B3%E5%A4%84%E7%90%86%E6%97%B6%E5%BA%8F%E5%9B%BE.png?raw=true) 

## 三、预告

本篇分析了调度中心和执行器之间的注册心跳是如何实现的，之后的两篇是这个调度框架比较重要的业务逻辑，我们来看看任务的调度和分发是如何实现的。