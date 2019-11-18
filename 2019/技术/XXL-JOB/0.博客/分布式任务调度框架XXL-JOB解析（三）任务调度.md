![](https://github.com/JiangYang4Github/WebLog/blob/master/2019/%E6%8A%80%E6%9C%AF/XXL-JOB/%E7%AC%AC%E4%B8%89%E7%AF%87/index.jpg?raw=true)

### 一、前言

前篇我们跟读了代码，了解调度中心和执行器之间的注册心跳是如何实现的，接下来两篇我们来看看这个框架中非常重要的逻辑——任务的调度和分发是如何实现的。

### 二、调度中心初始化

在application.properties配置正确的数据库连接信息后，直接启动XxlJobAdminApplication即可。

配置类XxlJobAdminConfig，里面维护了一些调度中心端的配置数据。

XxlJobScheduler这个组件实现了InitializingBean接口，所以spring容器在初始化的时候会调用afterPropertiesSet方法，此方法如下：

```java
@Override
public void afterPropertiesSet() throws Exception {
    // 国际化相关
    initI18n();

    // 执行器在线状态监听
    JobRegistryMonitorHelper.getInstance().start();

    // 失败任务重试
    JobFailMonitorHelper.getInstance().start();

    // 暴露调度中心服务
    initRpcProvider();

    // 任务调度，点进去
    JobScheduleHelper.getInstance().start();

    logger.info(">>>>>>>>> init xxl-job admin success.");
}
```

第一步国际化相关。

第二步执行器注册状态监控线程。

第三步调度任务失败重试线程。

第四步我们上一篇已经分析过了，启动调度中心服务，接收注册请求等。

第五步JobScheduleHelper调度器，我们跟进去。

### 三、调度线程

```
public void start(){

        // schedule thread
        scheduleThread = new Thread(new Runnable() {...});
        scheduleThread.setDaemon(true);
        scheduleThread.setName("xxl-job, admin JobScheduleHelper#scheduleThread");
        scheduleThread.start();
        
        // ring thread
        ringThread = new Thread(new Runnable() {...});
        ringThread.setDaemon(true);
        ringThread.setName("xxl-job, admin JobScheduleHelper#ringThread");
        ringThread.start();
    }
```

 启动了两个守护线程，先来看scheduleThread。

```java
while (!scheduleThreadToStop) {
    // 扫描任务
    long start = System.currentTimeMillis();
    PreparedStatement preparedStatement = null;
    try {
        if (conn==null || conn.isClosed()) {
            conn = XxlJobAdminConfig.getAdminConfig().getDataSource().getConnection();
        }
        conn.setAutoCommit(false);
        preparedStatement = conn.prepareStatement(  "select * from xxl_job_lock where lock_name = 'schedule_lock' for update" );
        preparedStatement.execute();
        // tx start
        // 1、预读5s内调度任务
        long nowTime = System.currentTimeMillis();
        List<XxlJobInfo> scheduleList = XxlJobAdminConfig.getAdminConfig().getXxlJobInfoDao().scheduleJobQuery(nowTime + PRE_READ_MS);
        if (scheduleList!=null && scheduleList.size()>0) {
            // 2、推送时间轮，有三个分支处理逻辑，稍后讲解
            for (XxlJobInfo jobInfo: scheduleList) {...}
            // 3、更新trigger信息
            for (XxlJobInfo jobInfo: scheduleList) {...}
        }
        // tx stop
    } catch (Exception e) {
        if (!scheduleThreadToStop) {
            logger.error(">>>>>>>>>>> xxl-job, JobScheduleHelper#scheduleThread error:{}", e);
        }
    } finally {
        // commit
        try {
            conn.commit();
        } catch (SQLException e) {
            if (!scheduleThreadToStop) {
                logger.error(e.getMessage(), e);
            }
        }

        // close PreparedStatement
        if (null != preparedStatement) {
            try {
                preparedStatement.close();
            } catch (SQLException ignore) {
                if (!scheduleThreadToStop) {
                    logger.error(ignore.getMessage(), ignore);
                }
            }
        }
    }
    long cost = System.currentTimeMillis()-start;
    // next second, align second
    try {
        if (cost < 1000) {
            TimeUnit.MILLISECONDS.sleep(1000 - System.currentTimeMillis()%1000);
        }
    } catch (InterruptedException e) {
        if (!scheduleThreadToStop) {
            logger.error(e.getMessage(), e);
        }
    }

                }
```

这个类就是死循环从xxl_job_info表中取出未来5秒内要执行的任务，进行调度分发。死循环内的代码如上图，首先利用**for update**语句进行获取任务的资格锁定，再去获取未来5秒内即将要执行的任务。

```java
public static final long PRE_READ_MS = 5000;    // pre read
```

展开遍历任务的逻辑代码，有三个分支，我们先来看第一个分支。

```java
if (nowTime > jobInfo.getTriggerNextTime() + PRE_READ_MS) {
	// 过期超5s：直接忽略，当前时间开始计算下次触发时间
    jobInfo.setTriggerLastTime(jobInfo.getTriggerNextTime());
    jobInfo.setTriggerNextTime(
            new CronExpression(jobInfo.getJobCron())
            .getNextValidTimeAfter(new Date())
            .getTime()
	);
}
```

当前任务的触发时间已经超时5秒以上了，不在执行，直接计算下一次触发时间，之后更新。

第二个分支代码如下：

```java
// 过期5s内 ：立即触发一次，当前时间开始计算下次触发时间；
CronExpression cronExpression = new CronExpression(jobInfo.getJobCron());
long nextTime = cronExpression.getNextValidTimeAfter(new Date()).getTime();

// 1、trigger
JobTriggerPoolHelper.trigger(jobInfo.getId(), TriggerTypeEnum.CRON, -1, null, null);
logger.debug(">>>>>>>>>>> xxl-job, shecule push trigger : jobId = " + jobInfo.getId() );

// 2、fresh next
jobInfo.setTriggerLastTime(jobInfo.getTriggerNextTime());
jobInfo.setTriggerNextTime(nextTime);

// 下次5s内：预读一次；
if (jobInfo.getTriggerNextTime() - nowTime < PRE_READ_MS) {...}
```

任务的触发时间已满足，利用JobTriggerPoolHelper这个类进行任务调度，之后判断下一次执行时间如果在5秒内，进行此任务数据的缓存，处理逻辑与第三个分支一样，我们看过去。

```java
// 未过期：正常触发，递增计算下次触发时间

// 1、make ring second
int ringSecond = (int)((jobInfo.getTriggerNextTime()/1000)%60);

// 2、push time ring
pushTimeRing(ringSecond, jobInfo.getId());

// 3、fresh next
jobInfo.setTriggerLastTime(jobInfo.getTriggerNextTime());
jobInfo.setTriggerNextTime(
        new CronExpression(jobInfo.getJobCron())
        .getNextValidTimeAfter(new Date(jobInfo.getTriggerNextTime()))
        .getTime()
);
```

对触发时间秒数进行60取模，跟进pushTimeRing方法

```java
private void pushTimeRing(int ringSecond, int jobId){
    // push async ring
    List<Integer> ringItemData = ringData.get(ringSecond);
    if (ringItemData == null) {
        ringItemData = new ArrayList<Integer>();
        ringData.put(ringSecond, ringItemData);
    }
    ringItemData.add(jobId);

}
```

ringData是以0到59的整数为key，以jobId集合为value的Map集合。这个集合数据的处理逻辑，就在我们第二个守护线程ringThread中。

### 四、时间轮算法处理任务

第二个线程采用了个简易的时间轮算法的实现，时间轮的思想应用范围非常广泛，各种操作系统的定时任务调度、延迟消息&队列的实现。什么是时间轮算法呢，我画了一张图，简单来说，我只需要把任务放到它需要被执行的时刻，然后等着时针转到这个时刻时，取出该时刻放置的任务，执行就可以了，这就是时间轮算法最核心的思想。

![](https://github.com/JiangYang4Github/WebLog/blob/master/2019/%E6%8A%80%E6%9C%AF/XXL-JOB/%E7%AC%AC%E4%B8%89%E7%AF%87/%E6%97%B6%E9%97%B4%E8%BD%AE%E9%92%9F%E5%9B%BE.png?raw=true)

我们来看实现代码，也就是我们第二个守护线程**ringThread**。

```java
while (!ringThreadToStop) {
    try {
        // second data
        List<Integer> ringItemData = new ArrayList<>();
        int nowSecond = Calendar.getInstance().get(Calendar.SECOND);   
        // 避免处理耗时太长，跨过刻度，向前校验一个刻度；
        for (int i = 0; i < 2; i++) {
            List<Integer> tmpData = ringData.remove( (nowSecond+60-i)%60 );
            if (tmpData != null) {
                ringItemData.addAll(tmpData);
            }
        }
        // ring trigger
        logger.debug(">>>>>>>>>>> xxl-job, time-ring beat : " + nowSecond + " = " + Arrays.asList(ringItemData) );
        if (ringItemData!=null && ringItemData.size()>0) {
            // do trigger
            for (int jobId: ringItemData) {
                // 任务分发
                JobTriggerPoolHelper.trigger(jobId, TriggerTypeEnum.CRON, -1, null, null);
            }
            // clear
            ringItemData.clear();
        }
    } catch (Exception e) {
        if (!ringThreadToStop) {
            logger.error(">>>>>>>>>>> xxl-job, JobScheduleHelper#ringThread error:{}", e);
        }
    }
    // next second, align second
    try {
        TimeUnit.MILLISECONDS.sleep(1000 - System.currentTimeMillis()%1000);
    } catch (InterruptedException e) {
        if (!ringThreadToStop) {
            logger.error(e.getMessage(), e);
        }
    }
}
```

根据当前秒数刻度和前一个刻度进行时间轮的任务获取，之后和上文一样，然后调用**JobTriggerPoolHelper**的trigger方法进行任务的分发。任务调度的整体时序图如下：

![](https://github.com/JiangYang4Github/WebLog/blob/master/2019/%E6%8A%80%E6%9C%AF/XXL-JOB/2.%E4%BB%BB%E5%8A%A1%E8%B0%83%E5%BA%A6/%E8%B0%83%E5%BA%A6%E4%B8%AD%E5%BF%83%E8%B0%83%E5%BA%A6%E4%BB%BB%E5%8A%A1%E6%97%B6%E5%BA%8F%E5%9B%BE.png?raw=true) 

### 五、任务路由分发

如前文所述，不管是scheduleThread还是ringThread，最后完成任务调度的都是JobTriggerPoolHelper.trigger方法，这个类的调度线程池进行了隔离，拆分为"Fast"和"Slow"两个线程池，1分钟窗口期内任务耗时达500ms超过10次，该窗口期内判定为慢任务，慢任务自动降级进入"Slow"线程池，避免耗尽调度线程，提高系统稳定性。

看代码：

```java
public static void trigger(int jobId, TriggerTypeEnum triggerType, int failRetryCount, String executorShardingParam, String executorParam) {
        helper.addTrigger(jobId, triggerType, failRetryCount, executorShardingParam, executorParam);
}
```

helper静态变量指向自己本身，提供外部静态方法调用。

```java
public void addTrigger(final int jobId, final TriggerTypeEnum triggerType, final int failRetryCount, final String executorShardingParam, final String executorParam) {

    // choose thread pool
    ThreadPoolExecutor triggerPool_ = fastTriggerPool;
    AtomicInteger jobTimeoutCount = jobTimeoutCountMap.get(jobId);
   	//timeoutCount的慢调用次数大于10次，那就采用slow线程池
    if (jobTimeoutCount!=null && jobTimeoutCount.get() > 10) {
        triggerPool_ = slowTriggerPool;
    }

    // 向线程池中提交任务
    triggerPool_.execute(new Runnable() {
        @Override
        public void run() {

            long start = System.currentTimeMillis();

            try {
                // 重要方法
                XxlJobTrigger.trigger(jobId, triggerType, failRetryCount, executorShardingParam, executorParam);
            } catch (Exception e) {
                logger.error(e.getMessage(), e);
            } finally {

                // check timeout-count-map
                long minTim_now = System.currentTimeMillis()/60000;
                if (minTim != minTim_now) {
                    minTim = minTim_now;
                    jobTimeoutCountMap.clear();
                }
                // incr timeout-count-map
                long cost = System.currentTimeMillis()-start;
                if (cost > 500) {
                    //任务执行时间大于500毫秒，timeoutCount自增一
                    AtomicInteger timeoutCount = jobTimeoutCountMap.putIfAbsent(jobId, new AtomicInteger(1));
                    if (timeoutCount != null) {
                        timeoutCount.incrementAndGet();
                    }
                }
            }
        }
    });
}
```

代码段中我注释的重要方法，即向两种线程池其中之一提交调度任务，进行调度，引出**XxlJobTrigger**这个类，跟进去。

```java
public static void trigger(int jobId, TriggerTypeEnum triggerType, int failRetryCount, String executorShardingParam, String executorParam) {
    XxlJobInfo jobInfo = XxlJobAdminConfig.getAdminConfig().getXxlJobInfoDao().loadById(jobId);
    if (jobInfo == null) {
        logger.warn(">>>>>>>>>>>> trigger fail, jobId invalid，jobId={}", jobId);
        return;
    }
    if (executorParam != null) {
        jobInfo.setExecutorParam(executorParam);
    }
    int finalFailRetryCount = failRetryCount>=0?failRetryCount:jobInfo.getExecutorFailRetryCount();
    XxlJobGroup group = XxlJobAdminConfig.getAdminConfig().getXxlJobGroupDao().load(jobInfo.getJobGroup());

    // 分片任务参数
    int[] shardingParam = null;
    if (executorShardingParam!=null){
        String[] shardingArr = executorShardingParam.split("/");
        if (shardingArr.length==2 && isNumeric(shardingArr[0]) && isNumeric(shardingArr[1])) {
            shardingParam = new int[2];
            shardingParam[0] = Integer.valueOf(shardingArr[0]);
            shardingParam[1] = Integer.valueOf(shardingArr[1]);
        }
    }
    if (ExecutorRouteStrategyEnum.SHARDING_BROADCAST==ExecutorRouteStrategyEnum.match(jobInfo.getExecutorRouteStrategy(), null)
            && group.getRegistryList()!=null && !group.getRegistryList().isEmpty()
            && shardingParam==null) {
        //如果是分片任务，进行循环分发
        for (int i = 0; i < group.getRegistryList().size(); i++) {
            processTrigger(group, jobInfo, finalFailRetryCount, triggerType, i, group.getRegistryList().size());
        }
    } else {
        if (shardingParam == null) {
            shardingParam = new int[]{0, 1};
        }
        //执行任务触发
        processTrigger(group, jobInfo, finalFailRetryCount, triggerType, shardingParam[0], shardingParam[1]);
    }
}
```

processTrigger方法的代码很长，但是逻辑并不复杂，代码块的注释一读便懂。

```java
private static void processTrigger(XxlJobGroup group, XxlJobInfo jobInfo, int finalFailRetryCount, TriggerTypeEnum triggerType, int index, int total){

    // 查询调度参数
    ExecutorBlockStrategyEnum blockStrategy = ExecutorBlockStrategyEnum.match(jobInfo.getExecutorBlockStrategy(), ExecutorBlockStrategyEnum.SERIAL_EXECUTION);  // block strategy
    ExecutorRouteStrategyEnum executorRouteStrategyEnum = ExecutorRouteStrategyEnum.match(jobInfo.getExecutorRouteStrategy(), null);    // route strategy
    String shardingParam = (ExecutorRouteStrategyEnum.SHARDING_BROADCAST==executorRouteStrategyEnum)?String.valueOf(index).concat("/").concat(String.valueOf(total)):null;

    // 1、保存任务调用日志
    XxlJobLog jobLog = new XxlJobLog();
    jobLog.setJobGroup(jobInfo.getJobGroup());
    jobLog.setJobId(jobInfo.getId());
    jobLog.setTriggerTime(new Date());
    XxlJobAdminConfig.getAdminConfig().getXxlJobLogDao().save(jobLog);
    logger.debug(">>>>>>>>>>> xxl-job trigger start, jobId:{}", jobLog.getId());

    // 2、初始化调度参数，此triggerParam将通过RPC发给执行器
    TriggerParam triggerParam = new TriggerParam();
    triggerParam.setJobId(jobInfo.getId());
    triggerParam.setExecutorHandler(jobInfo.getExecutorHandler());
    triggerParam.setExecutorParams(jobInfo.getExecutorParam());
    triggerParam.setExecutorBlockStrategy(jobInfo.getExecutorBlockStrategy());
    triggerParam.setExecutorTimeout(jobInfo.getExecutorTimeout());
    triggerParam.setLogId(jobLog.getId());
    triggerParam.setLogDateTim(jobLog.getTriggerTime().getTime());
    triggerParam.setGlueType(jobInfo.getGlueType());
    triggerParam.setGlueSource(jobInfo.getGlueSource());
    triggerParam.setGlueUpdatetime(jobInfo.getGlueUpdatetime().getTime());
    triggerParam.setBroadcastIndex(index);
    triggerParam.setBroadcastTotal(total);

    // 3、策略模式，根据任务配置的路由算法，拿到执行器地址
    String address = null;
    ReturnT<String> routeAddressResult = null;
    if (group.getRegistryList()!=null && !group.getRegistryList().isEmpty()) {
        if (ExecutorRouteStrategyEnum.SHARDING_BROADCAST == executorRouteStrategyEnum) {
            if (index < group.getRegistryList().size()) {
                address = group.getRegistryList().get(index);
            } else {
                address = group.getRegistryList().get(0);
            }
        } else {
            routeAddressResult = executorRouteStrategyEnum.getRouter().route(triggerParam, group.getRegistryList());
            if (routeAddressResult.getCode() == ReturnT.SUCCESS_CODE) {
                address = routeAddressResult.getContent();
            }
        }
    } else {
        routeAddressResult = new ReturnT<String>(ReturnT.FAIL_CODE, I18nUtil.getString("jobconf_trigger_address_empty"));
    }

    // 4、触发远程调用，发送任务
    ReturnT<String> triggerResult = null;
    if (address != null) {
    	//关键方法，执行任务
        triggerResult = runExecutor(triggerParam, address);
    } else {
        triggerResult = new ReturnT<String>(ReturnT.FAIL_CODE, null);
    }

    // 5、收集调用返回值
    StringBuffer triggerMsgSb = new StringBuffer();
    triggerMsgSb.append(I18nUtil.getString("jobconf_trigger_type")).append("：").append(triggerType.getTitle());
    triggerMsgSb.append("<br>").append(I18nUtil.getString("jobconf_trigger_admin_adress")).append("：").append(IpUtil.getIp());
    triggerMsgSb.append("<br>").append(I18nUtil.getString("jobconf_trigger_exe_regtype")).append("：")
            .append( (group.getAddressType() == 0)?I18nUtil.getString("jobgroup_field_addressType_0"):I18nUtil.getString("jobgroup_field_addressType_1") );
    triggerMsgSb.append("<br>").append(I18nUtil.getString("jobconf_trigger_exe_regaddress")).append("：").append(group.getRegistryList());
    triggerMsgSb.append("<br>").append(I18nUtil.getString("jobinfo_field_executorRouteStrategy")).append("：").append(executorRouteStrategyEnum.getTitle());
    if (shardingParam != null) {
        triggerMsgSb.append("("+shardingParam+")");
    }
    triggerMsgSb.append("<br>").append(I18nUtil.getString("jobinfo_field_executorBlockStrategy")).append("：").append(blockStrategy.getTitle());
    triggerMsgSb.append("<br>").append(I18nUtil.getString("jobinfo_field_timeout")).append("：").append(jobInfo.getExecutorTimeout());
    triggerMsgSb.append("<br>").append(I18nUtil.getString("jobinfo_field_executorFailRetryCount")).append("：").append(finalFailRetryCount);

    triggerMsgSb.append("<br><br><span style=\"color:#00c0ef;\" > >>>>>>>>>>>"+ I18nUtil.getString("jobconf_trigger_run") +"<<<<<<<<<<< </span><br>")
            .append((routeAddressResult!=null&&routeAddressResult.getMsg()!=null)?routeAddressResult.getMsg()+"<br><br>":"").append(triggerResult.getMsg()!=null?triggerResult.getMsg():"");

    // 6、更新调用日志
    jobLog.setExecutorAddress(address);
    jobLog.setExecutorHandler(jobInfo.getExecutorHandler());
    jobLog.setExecutorParam(jobInfo.getExecutorParam());
    jobLog.setExecutorShardingParam(shardingParam);
    jobLog.setExecutorFailRetryCount(finalFailRetryCount);
    jobLog.setTriggerCode(triggerResult.getCode());
    jobLog.setTriggerMsg(triggerMsgSb.toString());
    XxlJobAdminConfig.getAdminConfig().getXxlJobLogDao().updateTriggerInfo(jobLog);

    logger.debug(">>>>>>>>>>> xxl-job trigger end, jobId:{}", jobLog.getId());
}
```

话不多说，直奔**runExecutor**方法。

```java
public static ReturnT<String> runExecutor(TriggerParam triggerParam, String address){
    ReturnT<String> runResult = null;
    try {
    	//此处拿到的executorBiz是XxlRpcReferenceBean.getObject()获取到的代理类，上一篇已经分析过这个XxlRpcReferenceBean类了，这里就不细说了。
        ExecutorBiz executorBiz = XxlJobScheduler.getExecutorBiz(address);
        //此处执行的就是XxlRpcReferenceBean的invoke方法，其发起远程调用，完成了任务的发送
        runResult = executorBiz.run(triggerParam);
    } catch (Exception e) {
        logger.error(">>>>>>>>>>> xxl-job trigger error, please check if the executor[{}] is running.", address, e);
        runResult = new ReturnT<String>(ReturnT.FAIL_CODE, ThrowableUtil.toString(e));
    }
    StringBuffer runResultSB = new StringBuffer(I18nUtil.getString("jobconf_trigger_run") + "：");
    runResultSB.append("<br>address：").append(address);
    runResultSB.append("<br>code：").append(runResult.getCode());
    runResultSB.append("<br>msg：").append(runResult.getMsg());
    runResult.setMsg(runResultSB.toString());
    return runResult;
}
```

至此，完成任务的路由分发，这个步骤的时序图如下：

![](https://img2018.cnblogs.com/blog/753271/201909/753271-20190924213930801-512581316.png)

### 六、最后

本篇我们分析了调度中心是如何进行任务调度和路由分发的，下一篇我们就去执行器看看，它是如何完成任务的接收、执行和结果上报的。

喜欢的可以关注我的公众号「**江飞杰**」第一时间阅读（会更新的比较快），里面也有自己的一些和技术无关的读书笔记与生活随感，欢迎大家来关注。

![](https://user-gold-cdn.xitu.io/2019/11/17/16e793daa95189b7?w=430&h=430&f=jpeg&s=40918)
