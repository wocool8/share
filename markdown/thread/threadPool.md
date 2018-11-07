# Thread Pool
---
## 一 为什么要使用ThreadPool
|new Thread弊端|
|:-|
|每次new Thread新建对象性能差|
|线程缺乏统一管理，可能无限制新建线程，相互之间竞争，及可能占用过多系统资源导致死机或oom|
|缺乏更多功能，如定时执行、定期执行、线程中断|
## 二 ThreadPoolExecutor
    /**
     * @param corePoolSize the number of threads to keep in the pool, even
     *        if they are idle, unless {@code allowCoreThreadTimeOut} is set
     * @param maximumPoolSize the maximum number of threads to allow in the
     *        pool
     * @param keepAliveTime when the number of threads is greater than
     *        the core, this is the maximum time that excess idle threads
     *        will wait for new tasks before terminating.
     * @param unit the time unit for the {@code keepAliveTime} argument
     * @param workQueue the queue to use for holding tasks before they are
     *        executed.  This queue will hold only the {@code Runnable}
     *        tasks submitted by the {@code execute} method.
     * @throws IllegalArgumentException if one of the following holds:<br>
     *         {@code corePoolSize < 0}<br>
     *         {@code keepAliveTime < 0}<br>
     *         {@code maximumPoolSize <= 0}<br>
     *         {@code maximumPoolSize < corePoolSize}
     * @throws NullPointerException if {@code workQueue} is null
     */
    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue) {
        this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
             Executors.defaultThreadFactory(), defaultHandler);
    }    

处理任务的优先级为：核心线程corePoolSize、任务队列workQueue、最大线程maximumPoolSize，如果三者都满了，使用handler处理被拒绝的任务。
### 2.1 newCachedThreadPool（创建缓存线程池）
这个线程池根据需要（新任务到来时）创建新的线程，如果有空闲线程则会重复使用，线程空闲了60秒后会被回收

    public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, 
                                      Integer.MAX_VALUE,
                                      60L,
                                      TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
    }
maximumPoolSize为Integer.MAX_VALUE来保证可创建线程数量足够大，因此选择了[SynchronousQueue](/markdown/java/queue.md)不带缓存的队列，corePoolSize为0即不长期持有线程。
### 2.2 newFixedThreadPool（创建固定线程数量的线程池） 
    public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, 
                                      nThreads,
                                      0L, 
                                      TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
    }
设定线程池的corePoolSize和maximumPoolSize为nThreads不对空闲线程进行回收，所以keepAliveTime设置为0。使用[LinkedBlockingQueue](/markdown/java/queue.md)作为缓冲。
### 2.3 newScheduledThreadPool（创建一个定长线程池，支持定时及周期性任务执行）

    public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
        return new ScheduledThreadPoolExecutor(corePoolSize);
    }
    
    public ScheduledThreadPoolExecutor(int corePoolSize) {
        super(corePoolSize, 
              Integer.MAX_VALUE, 
              0, 
              NANOSECONDS,
              new DelayedWorkQueue());
    }

使用场景

    final ScheduledExecutorService scheduledExecutorService = Executors.newScheduledThreadPool(1);
    scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
        public void run() {
            try {
                // do something
            } catch (Exception e) {
                // handle exception
            }
        }
    }, 0, 1, TimeUnit.MINUTES);    
    
### 2.4 newSingleThreadExecutor （创建一个单线程化的线程池，它只会用唯一的工作线程来执行任务）
    public static ExecutorService newSingleThreadExecutor() {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 
                                    1,
                                    0L,
                                    TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));
    }
最大线程数量为1，缓冲池使用LinkedBlockingQueue    
## 三 线程池增长策略
    
    /*
     * 基于JDK1.8 源码
     * Proceed in 3 steps:
     * 1. 线程数量 < corePoolSize : 创建线程
     * 2. 线程数量 = corePoolSize 但是 workQueue 未满 : 任务放入缓冲队列
     * 3. 线程数量 > corePoolSize 且 workQueue满了，如果 线程数量 < maximumPoolSize : 创建新的线程处理任务
     */
    public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
            
        int c = ctl.get();
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            if (! isRunning(recheck) && remove(command))
                reject(command);
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        else if (!addWorker(command, false))
            reject(command);
    }
## 四 线程池拒绝策略
拒绝策略积累接口方法为RejectedExecutionHandler

    public interface RejectedExecutionHandler {
        void rejectedExecution(Runnable r, ThreadPoolExecutor executor);
    }
### 4.1 AbortPolicy
抛出rejectedExecution

    public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        throw new RejectedExecutionException("Task " + r.toString() +
                                              " rejected from " +
                                              e.toString());
    }
        
### 4.2 DiscardPolicy
舍弃任务

        /**
         * Does nothing, which has the effect of discarding task r.
         */
        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        }
### 4.3 DiscardOldestPolicy
如果执行程序尚未关闭，则位于工作队列头部的任务将被删除，然后重试执行程序

        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
            if (!e.isShutdown()) {
                e.getQueue().poll();
                e.execute(r);
            }
        }
### 4.4 CallerRunsPolicy 
这个策略放弃执行任务。但是由于池中已经没有任何资源了，那么就直接使用调用该execute的线程本身来执行

        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
            if (!e.isShutdown()) {
                r.run();
            }
        }       

<!--
核心池数量如何设定
如何根据场景选择不同的线程池
-->
