## JobTriggerPoolHelper

如前文所述，不管是scheduleThread还是ringThread，最后完成任务调度的都是JobTriggerPoolHelper.trigger方法，这个类有两个线程池fastTriggerPool和slowTriggerPool，顾名思义，分别是执行较快任务和较慢任务的，后查官方文档，如下：

![img](https://img2018.cnblogs.com/blog/753271/201909/753271-20190924104757901-1796015968.png) 

minTim属性，作用待明确

jobTimeoutCountMap属性，计数，key为jobId，value使用AtomicInteger计数。

helper静态变量指向自己本身，提供外部静态方法调用。

![img](https://img2018.cnblogs.com/blog/753271/201909/753271-20190924105720607-393540803.png)

重要方法，向两种线程池其中之一提交调度任务，进行调度，引出XxlJobTrigger这个类，一路跟进去

![img](https://img2018.cnblogs.com/blog/753271/201909/753271-20190924112835765-354928850.png)

![img](https://img2018.cnblogs.com/blog/753271/201909/753271-20190924113032158-120384122.png)

继续跟进

![img](https://img2018.cnblogs.com/blog/753271/201909/753271-20190924113055497-1907427470.png)

至此，完成执行器的任务调度。

## 时序图

![img](https://img2018.cnblogs.com/blog/753271/201909/753271-20190924213930801-512581316.png)

## 接收注册和心跳请求

![img](https://img2018.cnblogs.com/blog/753271/201909/753271-20190926153634669-808587215.png)