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
