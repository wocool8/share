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
(1)线程数量 < corePoolSize : 创建新的线程处理任务<br>
(2)线程数量 = corePoolSize 但是 workQueue 未满 : 任务放入缓冲队列<br>
(3)线程数量 > corePoolSize 且 workQueue满了 且 线程数量 < maximumPoolSize : 创建新的线程处理任务<br>
处理任务的优先级为：核心线程corePoolSize、任务队列workQueue、最大线程maximumPoolSize，如果三者都满了，使用handler处理被拒绝的任务。
### 2.1 newCachedThreadPool
    public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, 
                                      Integer.MAX_VALUE,
                                      60L,
                                      TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
    }
maximumPoolSize为Integer.MAX_VALUE来保证可创建线程数量足够大，因此选择了[SynchronousQueue](/markdown/java/queue.md)不带缓存的队列，corePoolSize为0即不长期持有线程。
### 2.2 newFixedThreadPool 
### 2.3 newScheduledThreadPool
### 2.4 newSingleThreadExecutor 


## 二 线程池增长策略
## 三 线程池排队策略
