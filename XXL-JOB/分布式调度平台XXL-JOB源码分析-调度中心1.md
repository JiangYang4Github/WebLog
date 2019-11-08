# 架构图

 ![img](https://raw.githubusercontent.com/JiangYang4Github/image-repository/master/XXL-JOB/6.%E5%85%B6%E4%BB%96/%E6%95%B4%E4%BD%93%E6%9E%B6%E6%9E%84.png) 

上图是我们要进行源码分析的2.1版本的整体架构图。其分为两大块，调度中心和执行器，本文先分析调度中心，也就是xxl-job-admin这个包的代码。

# 关键bean

在application.properties配置正确的数据库连接信息后，直接启动XxlJobAdminApplication即可。

配置类XxlJobAdminConfig，里面维护了一些调度中心端的配置数据。

XxlJobScheduler这个组件实现了InitializingBean接口，所以spring容器在初始化的时候会调用afterPropertiesSet方法，此方法如下：

```java
@Override
public void afterPropertiesSet() throws Exception {
    // init i18n
    initI18n();

    // admin registry monitor run
    JobRegistryMonitorHelper.getInstance().start();

    // admin monitor run
    JobFailMonitorHelper.getInstance().start();

    // admin-server
    initRpcProvider();

    // start-schedule
    JobScheduleHelper.getInstance().start();

    logger.info(">>>>>>>>> init xxl-job admin success.");
}
```

第一步国际化相关。

第二步执行器注册状态监控线程。

第三步调度任务失败重试线程。

第四步启动admin端服务，接收注册请求等。

第五步JobScheduleHelper调度器，死循环，在xxl_job_info表里取将要执行的任务，更新下次执行时间的，调用JobTriggerPoolHelper类，来给执行器发送调度任务的。

# JobScheduleHelper

这个类就是死循环从xxl_job_info表中取出未来5秒内要执行的任务，进行调度分发。

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

死循环内的代码如上图，首先利用for update语句进行获取任务的资格锁定，再去获取未来5秒内即将要执行的任务。

```java
public static final long PRE_READ_MS = 5000;    // pre read
```

展开遍历任务的逻辑代码，有三个分支

```java
// 时间轮刻度计算
if (nowTime > jobInfo.getTriggerNextTime() + PRE_READ_MS) {
	// 过期超5s：本地忽略，当前时间开始计算下次触发时间

	// fresh next
    jobInfo.setTriggerLastTime(jobInfo.getTriggerNextTime());
    jobInfo.setTriggerNextTime(
            new CronExpression(jobInfo.getJobCron())
            .getNextValidTimeAfter(new Date())
            .getTime()
	);
}
```

 第一个分支当前任务的触发时间已经超时5秒以上了，不在执行，直接计算下一次触发时间。

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

 第二个分支为触发时间已满足，利用JobTriggerPoolHelper这个类进行任务调度，之后判断下一次执行时间如果在5秒内，进行此任务数据的缓存，处理逻辑与第三个分支一样。

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

```java
while (!ringThreadToStop) {
    try {
        // second data
        List<Integer> ringItemData = new ArrayList<>();
        int nowSecond = Calendar.getInstance().get(Calendar.SECOND);   // 避免处理耗时太长，跨过刻度，向前校验一个刻度；
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
                // do trigger
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

根据当前秒数刻度和前一个刻度进行时间轮的任务获取，之后和上文一样，利用JobTriggerPoolHelper进行任务调度。

# 时序图

 ![img](https://raw.githubusercontent.com/JiangYang4Github/image-repository/master/XXL-JOB/2.%E4%BB%BB%E5%8A%A1%E8%B0%83%E5%BA%A6/%E8%B0%83%E5%BA%A6%E4%B8%AD%E5%BF%83%E8%B0%83%E5%BA%A6%E4%BB%BB%E5%8A%A1%E6%97%B6%E5%BA%8F%E5%9B%BE.png) 