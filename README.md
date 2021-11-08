# TimeTask
## 1. Timer
java.util.Timer是 JDK 1.3 开始就已经支持的一种定时任务的实现方式，维护了一个内部线程与队列，且在实例化Timer时初始化好。

	TimerThread:内部线程，继承自Thread;
	TaskQueue:任务队列，包含当前Timer的所有任务，采用数组的方式实现了一个基于最小堆实现的优先级队列。
    按照任务距离下一次执行时间的大小将任务排序，保证在堆顶的任务最先执行。这样在需要执行任务时，每次只需要取出堆顶的任务运行即可。
说明：

	1. 线程调度任务以供将来在后台线程中执行的功能，任务可以安排一次执行，或定期重复执行。 
	2. 每个Timer对象是单个后台线程，用于依次执行所有定时器的所有任务，导致同一时间只能有一个任务得到执行；
		如果一个定时器任务需要花费很多时间来完成，它会阻碍当前计时器的任务执行线程，可能会延迟随后任务的执行。
	3. 当最后一次对Timer对象的引用后，所有未完成的任务已完成执行，定时器的任务执行线程正常终止(并被收集到垃圾回收)。
		默认情况下，任务执行线程不作为守护程序线程运行，因此它能够使应用程序终止，如果需要快速终止定时器的任务执行线程，则调用者应该调用定时器的cancel方法。
	4. 如果定时器的任务执行线程意外终止，那么在计时器上安排任务的将会抛出IllegalStateException ，就像定时器的cancel方法被调用一样。
主要方法：

	1. schedule(task,time)、schedule(task,delay)、schedule(task,firstTime,period)、schedule(task,delay,period);
	2. scheduledAtFixedRate(task,firstTime,period)、scheduledAtFixedRate(task,delay,period)
示例：

	     TimerTask task=new TimerTask() {
            @Override
            public void run() {
                //具体业务逻辑
                System.out.println("time:"+new Date()+"thread:"+Thread.currentThread().getName());
            }
        };

        Timer timer=new Timer("myTimer");
        timer.schedule(task,1000L);
## 2. ScheduledExecutorService
一个接口，有多个实现类，比较常用的是 ScheduledThreadPoolExecutor；
![image](https://user-images.githubusercontent.com/41152743/140290056-035b5b84-d945-4c75-b054-6254978a9699.png)

说明：

![image](https://user-images.githubusercontent.com/41152743/140302602-24e60c87-3230-4a71-a154-3aa57e3022fc.png)

	1. ScheduledThreadPoolExecutor 本身就是一个线程池，支持任务并发执行。
		1.1 其中最大线程数为 Integer.MAX_VALUE，任务队列为java.util.concurrent.ScheduledThreadPoolExecutor.DelayedWorkQueue；
		1.2 该队列是定制的基于堆结构的优先级队列，只能用来存储RunnableScheduledFutures任务；基于数组实现的，初始容量为16；
			详情解析见:https://blog.csdn.net/nobody_1/article/details/99684009
	2. Schedule方法，将任务封装成ScheduledThreadPoolExecutor.ScheduledFutureTask，并添加到任务队列DelayedWorkQueue中
		其中，ScheduledFutureTask继承了FutureTask和RunnableScheduledFuture,主要包括
		sequenceNumber: 创建任务自增的序列号、
		time: 任务能够开始执行的时间、
		period: 任务周期执行的时间
			(正数表示固定周期去执行任务：nextRunTime = 本次runTime + delay、负数表示任务执行完成后，延时去执行任务：nextRunTime = now() + delay、0表示非重复任务)、
		outerTask: 表示再次执行的任务、
		heapIndex: 表示在任务队列中的索引位置,用来支持快速从队列中删除任务。
	   因此,延迟时间小的先执行，延迟时间一致的任务先创建的先执行
主要方法：

	1. #scheduleAtFixedRate(Runnable command, long initialDelay,long period,TimeUnit unit)
		在给定的初始延迟后，按照给定周期一直周期执行。
![image](https://user-images.githubusercontent.com/41152743/140305661-0da732c1-6af5-49c3-a465-74f0672f7710.png)

	2. #scheduleWithFixedDelay(Runnable command,long initialDelay,long delay,TimeUnit unit) 
		在给定的初始延迟后，本次任务执行完成后固定延迟一段时间开始下次任务的执行
![image](https://user-images.githubusercontent.com/41152743/140305759-fa5855d9-81c0-4c1c-9ed3-01688054a748.png)

	3. #schedule(Callable<V> callable,long delay,TimeUnit unit)、#schedule(Runnable command,long delay,TimeUnit unit)
		在给定初始延迟后，创建一个一次性执行的任务
## 3. spring task
使用 Timer 还是 ScheduledExecutorService 都无法使用 Cron 表达式指定任务执行的具体时间，Spring Task 支持 Cron 表达式 ，但是功能单一。

说明：

	1. 启动类上添加@EnableScheduling注解，并在其中的方法上添加@Scheduled注解启用；
	2. 支持的任务类型，三种：
		cron表达式任务：@Scheduled(cron=”${cron.expression}”),按cron表达式执行；
		固定延迟间隔任务：@Scheduled(fixedDelayString = "${task.fixedDelay}", initialDelayString = "${task.initialDelay}"
		固定频率任务：@Scheduled(fixedRateString = "${task.fixedRate}", initialDelayString = "${task.initialDelay}")		
原理：具体见：https://www.cnblogs.com/throwable/p/12616945.html#%E4%BB%BB%E5%8A%A1%E8%B0%83%E5%BA%A6%E5%99%A8

1. 启动类上@EnableScheduling注解，@Import引入的SchedulingConfiguration类，返回一个bean，引入了ScheduledAnnotationBeanPostProcessor类；
2. 该类实现了BeanPostProcessor接口(MergedBeanDefinitionPostProcessor的父接口)，用于Bean实例初始化前后分别回调，其中后回调postProcessAfterInitialization()方法就是用于解析@Scheduled和装载ScheduledTask；
3. 还实现了SmartInitializingSingleton接口，所有单例实例化完毕之后回调，作用是在持有的applicationContext为NULL的时候开始调度所有加载完成的任务，org.springframework.scheduling.config.ScheduledTaskRegistrar#scheduleTasks，封装TaskScheduler，或者使用默认的核心线程数为1的SheduledThreadPoolExecutor，或者使用自定义的线程池，主要包括三种ThreadPoolTaskScheduler、ConcurrentTaskScheduler、DefaultManagedTaskScheduler，其底层依赖于ScheduledThreadPoolExecutor
4. 还实现了BeanNameAware接口用于回调bean名称；BeanFactoryAware接口用于回调bean工厂实例，具体是DefaultListableBeanFactory，也就是熟知的IOC容器；ApplicationContextAware接口用于回调ApplicationContext实例，即Spring上下文，同时是事件广播器、资源加载器的实现等等。
5. 还实现了DisposableBean接口，当前Bean实例销毁时候回调，用于取消和清理所有的ScheduledTask。ScheduledAnnotationBeanPostProcessor#destroy

注(Bean的一生)：
![image](https://user-images.githubusercontent.com/41152743/140677509-dae16814-43f5-46d4-b522-9ec75fa7a442.png)



