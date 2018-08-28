# 任务调度
---
## 一 Timer
    private static Timer timer = new Timer();
    public static void main(String[] args) throws Exception {
        timer.schedule(new TimerTask(){
            public void run() {
                System.out.println("Timer");
            }
        }, 0, 500);
        Thread.sleep(5000);
        timer.schedule(new TimerTask(){
            public void run(){
                throw new RuntimeException();
            }
        }, 0,1000);
    }
|问题|
|:-| 
|(1)Timer对调度的支持是基于绝对时间,而不是相对时间的，因此任务对系统时钟的改变是敏感的|
|(2)一个Timer中的不同任务线程隔离不好，一旦其中有一个线程异常，所有的调度任务都结束|
## 二 ScheduledExecutorService
    public static void main(String[] args) {
        final ScheduledExecutorService scheduledExecutorService = new ScheduledThreadPoolExecutor(1);
        scheduledExecutorService.scheduleAtFixedRate(
                () -> System.out.println("ScheduledExecutorService"),
                0,
                1,
                TimeUnit.SECONDS);
    }
|与Timer对比的优点|
|:-| 
|(1)支持相对时间|
|(2)基于 ScheduledThreadPoolExecutor 隔离不同任务|
## 三 spring schedule
### 3.1 spring schedule使用
    @Component
    public class ScheduleJob {
     
        @Scheduled(cron = "0 */1 * * * ?")
        public void fixedRateJob() {
            System.out.println("spring schedule");
        }
     
    }
spring Schedule基于注解使用很简单如上代码使用cron表达式控制执行间隔，被注解的方法是具体任务类
### 3.2 spring schedule实现
（1）ScheduledAnnotationBeanPostProcessor实现ApplicationListener<ContextRefreshedEvent><br>
（2）在bean初始化的时候进行监听onApplicationEvent(ContextRefreshedEvent event)<br>
（3）onApplicationEvent方法中调用 finishRegistration()方法，在方法中调用ScheduleTaskRegisterar进行任务调度（scheduleAtFixedRate或scheduleWithFixedDelay）<br>
ScheduleTaskRegister的实现逻辑如下图
![springSchedule](../picture/schedule/springSchedule.PNG)
## 四 quartz
### 4.1 quartz使用
Quartz是一个完全由java编写的开源作业调度框架

    public static void main(String[] args) throws SchedulerException, InterruptedException {
        Scheduler scheduler = StdSchedulerFactory.getDefaultScheduler();
        Trigger trigger = cronSchedule("0 0/2 8-17 * * ?").build();
        JobDetail job = newJob(HelloQuartz.class)
                //定义name/group
                .withIdentity("job1", "group1")
                //定义属性
                .usingJobData("name", "quartz")
                .build();
        scheduler.scheduleJob(job, trigger);
        scheduler.start();
        Thread.sleep(10000);
        scheduler.shutdown(true);
    }
    
|quartz调度过程构成部分|描述|
|:-|:-|
|Scheduler|调度器，所有的调度都是由它控制|
|Trigger|定义触发的条件（支持复杂的条件比如cron表达式）|
|JobDetail & Job|JobDetail 定义任务数据，执行逻辑是在Job中，JobDetail & Job 方式，sheduler每次执行，根据JobDetail创建一个新的Job实例，避免直接使用Job并发访问的问题|
### 4.2 quartz实现
quartz的任务调度最底层是基于多线程实现的，在QuartzScheduler的构造方法中初始化QuartzSchedulerThread并把它放到线程池中执行，QuartzSchedulerThread 的run()方法代码节选如下

    public class QuartzSchedulerThread extends Thread {
        private QuartzScheduler qs;
        private QuartzSchedulerResources qsRsrcs;
        // 锁对象
        private final Object sigLock = new Object();
        private boolean signaled;
        private long signaledNextFireTime;
        private boolean paused;
        // 线程停止标识
        private AtomicBoolean halted;
        private Random random = new Random(System.currentTimeMillis());
        // When the scheduler finds there is no current trigger to fire, how long
        // it should wait until checking again...
        private static long DEFAULT_IDLE_WAIT_TIME = 30L * 1000L;
        private long idleWaitTime = DEFAULT_IDLE_WAIT_TIME;
        private int idleWaitVariablness = 7 * 1000;
        private final Logger log = LoggerFactory.getLogger(getClass());
         
        @Override
        public void run() {
            while (!halted.get()) {
                try {
                    int availThreadCount = qsRsrcs.getThreadPool().blockForAvailableThreads();
                    if(availThreadCount > 0) { // will always be true, due to semantics of blockForAvailableThreads...
                        try {
                            triggers = qsRsrcs.getJobStore().acquireNextTriggers(now + idleWaitTime, Math.min(availThreadCount, qsRsrcs.getMaxBatchSize()), qsRsrcs.getBatchTimeWindow());
                        } catch (JobPersistenceException jpe) { }
                        if (triggers != null && !triggers.isEmpty()) {
                            now = System.currentTimeMillis();
                            long triggerTime = triggers.get(0).getNextFireTime().getTime();
                            long timeUntilTrigger = triggerTime - now;
                            while(timeUntilTrigger > 2) {
                                synchronized (sigLock) {
                                    if (halted.get()) {
                                        break;
                                    }
                                    if (!isCandidateNewTimeEarlierWithinReason(triggerTime, false)) {
                                        try {
                                            // we could have blocked a long while then on 'synchronize', so we must recompute
                                            now = System.currentTimeMillis();
                                            timeUntilTrigger = triggerTime - now;
                                            if(timeUntilTrigger >= 1)
                                                sigLock.wait(timeUntilTrigger);
                                        } catch (InterruptedException ignore) {
                                        }
                                    }
                                }
                            }
                            // 执行调用job逻辑
                            // ...
                        }
                    } else { // if(availThreadCount > 0)
                        continue; // while (!halted)
                    }
                } catch(RuntimeException re) {
                    getLog().error("Runtime error occurred in main trigger firing loop.", re);
                }
            }
        }
sigLock是线程的锁对象，用锁对象的wait()方法控制具体任务的执行时间间隔
### 4.3 缺点
|缺点|
|:-|
|(1)quartz是支持分布式定时任务的，但是它是基于数据库的分布式锁机制来保证一次只有一个线程来操作：加锁  > 操作 > 释放锁。数据库的行锁是一种悲观锁，锁表时其他线程无法查询，因此，如果加锁后进行的操作时间过长，会带来集群间主线程等待的问题|
|(2)尽管quartz能够基于数据库实现作业的高可用，但是由于抢占式执行的模式，不支持分布式并行运行作业的功能|

## 五 tbsSchedule

## 六 elasticJo