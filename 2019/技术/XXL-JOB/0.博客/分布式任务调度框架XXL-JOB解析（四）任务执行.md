### 一、前言

在上一篇文章中，我们完成了调度中心关于调度的实现，这一篇来看执行器端是怎么实现的

### 二、执行器端启动

记得在第一篇中，执行器端要有个配置项

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

去看 XxlJobSpringExecutor 这个类的 start 方法，

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

```java
public void start() throws Exception {
	// 初始化任务日志文件记录线程
	XxlJobFileAppender.initLogPath(logPath);
	// 初始化调度中心相关接口代理类
	initAdminBizList(adminAddresses, accessToken);
	// 初始化日志文件清除线程
	JobLogFileCleanThread.getInstance().start(logRetentionDays);
	// 初始化回调线程
	TriggerCallbackThread.getInstance().start();
	// 执行器端服务暴露
	port = port>0?port: NetUtil.findAvailablePort(9999);
	ip = (ip!=null&&ip.trim().length()>0)?ip: IpUtil.getIp();
	initRpcProvider(ip, port, appName, accessToken);
}
```

#### 1.调度中心接口代理

```java
private static List<AdminBiz> adminBizList;
private static Serializer serializer;
private void initAdminBizList(String adminAddresses, String accessToken) throws Exception {
    serializer = Serializer.SerializeEnum.HESSIAN.getSerializer();
    if (adminAddresses!=null && adminAddresses.trim().length()>0) {
        for (String address: adminAddresses.trim().split(",")) {
            if (address!=null && address.trim().length()>0) {
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

这样就将调度中心的代理类维护在了 adminBizList 字段中，我们来看 AdminBiz 接口提供了哪些方法

```java
public interface AdminBiz {
	//结果回调
    public ReturnT<String> callback(List<HandleCallbackParam> callbackParamList);

   	//注册
    public ReturnT<String> registry(RegistryParam registryParam);

    //注销
    public ReturnT<String> registryRemove(RegistryParam registryParam);

}
```

#### 2.执行器暴露服务

```java
private void initRpcProvider(String ip, int port, String appName, String accessToken) {

    // init, provider factory
    String address = IpUtil.getIpPort(ip, port);
    Map<String, String> serviceRegistryParam = new HashMap<String, String>();
    serviceRegistryParam.put("appName", appName);
    serviceRegistryParam.put("address", address);

    xxlRpcProviderFactory = new XxlRpcProviderFactory();
    xxlRpcProviderFactory.initConfig(NetEnum.NETTY_HTTP, Serializer.SerializeEnum.HESSIAN.getSerializer(), ip, port, accessToken, ExecutorServiceRegistry.class, serviceRegistryParam);

    // add services
    xxlRpcProviderFactory.addService(ExecutorBiz.class.getName(), null, new ExecutorBizImpl());

    // start
    xxlRpcProviderFactory.start();

}
```

我们发现暴露的接口实现服务，只有一个 ExecutorBizImpl ，这里用了 Dispatcher 模式。

### 三、任务执行

记得上一篇文章中我们调度中心使用 ExecutorBiz 的 run 方法完成任务分发的，我们之间看 run 方法

```java
public ReturnT<String> run(TriggerParam triggerParam) {
    // 先去任务执行线程池中看看有没有正在执行这个任务的线程
    JobThread jobThread = XxlJobExecutor.loadJobThread(triggerParam.getJobId());
    IJobHandler jobHandler = jobThread!=null?jobThread.getHandler():null;
    String removeOldReason = null;

    // valid：jobHandler + jobThread
    GlueTypeEnum glueTypeEnum = GlueTypeEnum.match(triggerParam.getGlueType());
    if (GlueTypeEnum.BEAN == glueTypeEnum) {

        // new jobhandler
        IJobHandler newJobHandler = XxlJobExecutor.loadJobHandler(triggerParam.getExecutorHandler());

        // valid old jobThread
        if (jobThread!=null && jobHandler != newJobHandler) {
            // change handler, need kill old thread
            removeOldReason = "change jobhandler or glue type, and terminate the old job thread.";

            jobThread = null;
            jobHandler = null;
        }

        // valid handler
        if (jobHandler == null) {
            jobHandler = newJobHandler;
            if (jobHandler == null) {
                return new ReturnT<String>(ReturnT.FAIL_CODE, "job handler [" + triggerParam.getExecutorHandler() + "] not found.");
            }
        }

    } else if (GlueTypeEnum.GLUE_GROOVY == glueTypeEnum) {
    	...
    } else if (glueTypeEnum!=null && glueTypeEnum.isScript()) {
        ...
    } else {
        ...
    }

    // executor block strategy
    if (jobThread != null) {
        ExecutorBlockStrategyEnum blockStrategy = ExecutorBlockStrategyEnum.match(triggerParam.getExecutorBlockStrategy(), null);
        if (ExecutorBlockStrategyEnum.DISCARD_LATER == blockStrategy) {
            // discard when running
            if (jobThread.isRunningOrHasQueue()) {
                return new ReturnT<String>(ReturnT.FAIL_CODE, "block strategy effect："+ExecutorBlockStrategyEnum.DISCARD_LATER.getTitle());
            }
        } else if (ExecutorBlockStrategyEnum.COVER_EARLY == blockStrategy) {
            // kill running jobThread
            if (jobThread.isRunningOrHasQueue()) {
                removeOldReason = "block strategy effect：" + ExecutorBlockStrategyEnum.COVER_EARLY.getTitle();

                jobThread = null;
            }
        } else {
            // just queue trigger
        }
    }

    // 如果没有执行此任务的线程，则创建、注册、和启动
    if (jobThread == null) {
        jobThread = XxlJobExecutor.registJobThread(triggerParam.getJobId(), jobHandler, removeOldReason);
    }

    // 将调度参数追加至任务执行线程中的待执行任务队列triggerQueue中
    ReturnT<String> pushResult = jobThread.pushTriggerQueue(triggerParam);
    return pushResult;
}
```

```java

triggerParam = triggerQueue.poll(3L, TimeUnit.SECONDS);

if (triggerParam.getExecutorTimeout() > 0) {
    //任务设置超时时间
    Thread futureThread = null;
    try {
        final TriggerParam triggerParamTmp = triggerParam;
        FutureTask<ReturnT<String>> futureTask = new FutureTask<ReturnT<String>>(new Callable<ReturnT<String>>() {
            @Override
            public ReturnT<String> call() throws Exception {
                return handler.execute(triggerParamTmp.getExecutorParams());
            }
        });
        futureThread = new Thread(futureTask);
        futureThread.start();

        executeResult = futureTask.get(triggerParam.getExecutorTimeout(), TimeUnit.SECONDS);
    } catch (TimeoutException e) {
        XxlJobLogger.log("<br>----------- xxl-job job execute timeout");
        XxlJobLogger.log(e);
        executeResult = new ReturnT<String>(IJobHandler.FAIL_TIMEOUT.getCode(), "job execute timeout ");
    } finally {
        futureThread.interrupt();
    }
} else {
    // 立即执行
    executeResult = handler.execute(triggerParam.getExecutorParams());
}
```

### 四、结果上报

#### 1.同步上报

#### 2.异步上报