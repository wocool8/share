# 数据库连接池参数配置
---
## 一 案例分析
### 1.1 使用druid connection holder is null
#### 1.1.1 异常栈

    Caused by: java.sql.SQLException: connection holder is null
    at com.alibaba.druid.pool.DruidPooledConnection.checkState(DruidPooledConnection.java:1085)
    at com.alibaba.druid.pool.DruidPooledConnection.getMetaData(DruidPooledConnection.java:825)
    at org.springframework.jdbc.support.JdbcUtils.extractDatabaseMetaData(JdbcUtils.java:285)
    ... 70 more
    ERROR: (DruidDataSource.java:1815)   abandon connection, open stackTrace
    at java.lang.Thread.getStackTrace(Thread.java:1588)
    at com.alibaba.druid.pool.DruidDataSource.getConnectionDirect(DruidDataSource.java:942)
    at com.alibaba.druid.filter.FilterChainImpl.dataSource_connect(FilterChainImpl.java:4534)
    at com.alibaba.druid.filter.stat.StatFilter.dataSource_getConnection(StatFilter.java:661)
    at com.alibaba.druid.filter.FilterChainImpl.dataSource_connect(FilterChainImpl.java:4530)
    at com.alibaba.druid.pool.DruidDataSource.getConnection(DruidDataSource.java:880)
    at com.alibaba.druid.pool.DruidDataSource.getConnection(DruidDataSource.java:872)
    
#### 1.1.2 分析过程
由于connection holder is null，所以在druid的源码中找有关连接的holder，在com.alibaba.druid.pool包下有一个DruidConnectionHolder，源码如下

    public final class DruidConnectionHolder {
        private final static Log                      LOG                      = LogFactory.getLog(DruidConnectionHolder.class);
        public static boolean                         holdabilityUnsupported   = false;
    
        protected final DruidAbstractDataSource       dataSource;
        protected final long                          connectionId;
        protected final Connection                    conn;
        protected final List<ConnectionEventListener> connectionEventListeners = new CopyOnWriteArrayList<ConnectionEventListener>();
        protected final List<StatementEventListener>  statementEventListeners  = new CopyOnWriteArrayList<StatementEventListener>();
        protected final long                          connectTimeMillis;
        protected volatile long                       lastActiveTimeMillis;
        protected volatile long                       lastValidTimeMillis;
        private long                                  useCount                 = 0;
        private long                                  keepAliveCheckCount      = 0;
        private long                                  lastNotEmptyWaitNanos;
        private final long                            createNanoSpan;
        protected PreparedStatementPool               statementPool;
        protected final List<Statement>               statementTrace           = new ArrayList<Statement>(2);
        protected final boolean                       defaultReadOnly;
        protected final int                           defaultHoldability;
        protected final int                           defaultTransactionIsolation;
        protected final boolean                       defaultAutoCommit;
        protected boolean                             underlyingReadOnly;
        protected int                                 underlyingHoldability;
        protected int                                 underlyingTransactionIsolation;
        protected boolean                             underlyingAutoCommit;
        protected boolean                             discard                  = false;
        protected final Map<String, Object>           variables;
        protected final Map<String, Object>           globleVariables;

通过源码结构可以判定DruidConnectionHolder是连接和事物的关联，holder中持有一个连接。数据库事物commit之前的所有操作都要基于同一个Connection,所以可以推断出holder is null 就是连接被释放了,根据详细的异常堆信息    at com.alibaba.druid.pool.DruidDataSource.getConnectionDirect(DruidDataSource.java:942)查看源码

    public DruidPooledConnection getConnectionDirect(long maxWaitMillis) throws SQLException {
            int notFullTimeoutRetryCnt = 0;
            for (;;) {
                // handle notFullTimeoutRetry
                DruidPooledConnection poolableConnection;
                try {
                    poolableConnection = getConnectionInternal(maxWaitMillis);
                } catch (GetConnectionTimeoutException ex) {
                    if (notFullTimeoutRetryCnt <= this.notFullTimeoutRetryCount && !isFull()) {
                        notFullTimeoutRetryCnt++;
                        if (LOG.isWarnEnabled()) {
                            LOG.warn("get connection timeout retry : " + notFullTimeoutRetryCnt);
                        }
                        continue;
                    }
                    throw ex;
                }
                
            if (removeAbandoned) {
                StackTraceElement[] stackTrace = Thread.currentThread().getStackTrace();
                    poolableConnection.connectStackTrace = stackTrace;
                    poolableConnection.setConnectedTimeNano();
                    poolableConnection.traceEnable = true;
          
                    activeConnectionLock.lock();
                    try {
                         activeConnections.put(poolableConnection, PRESENT);
                    } finally {
                         activeConnectionLock.unlock();
                    }
             }
                
决定是否回收Connection的条件是removeAbandoned, 然后把连接放到activeConnections中，然后在DruidConnectionHolder的init()方法中发现连接销毁线程

    createAndLogThread();
    createAndStartCreatorThread();
    createAndStartDestroyThread();
       
    protected void createAndStartDestroyThread() {
        destroyTask = new DestroyTask();

        if (destroyScheduler != null) {
            long period = timeBetweenEvictionRunsMillis;
            if (period <= 0) {
                period = 1000;
            }
            destroySchedulerFuture = destroyScheduler.scheduleAtFixedRate(destroyTask, period, period,
                                                                          TimeUnit.MILLISECONDS);
            initedLatch.countDown();
            return;
        }

        String threadName = "Druid-ConnectionPool-Destroy-" + System.identityHashCode(this);
        destroyConnectionThread = new DestroyConnectionThread(threadName);
        destroyConnectionThread.start();
    }
    
    public class DestroyTask implements Runnable {

        @Override
        public void run() {
            shrink(true, keepAlive);

            if (isRemoveAbandoned()) {
                removeAbandoned();
            }
        }

    }    
            
    public int removeAbandoned() {
        int removeCount = 0;
        long currrentNanos = System.nanoTime();
        List<DruidPooledConnection> abandonedList = new ArrayList<DruidPooledConnection>();

        activeConnectionLock.lock();
        try {
            Iterator<DruidPooledConnection> iter = activeConnections.keySet().iterator();
            for (; iter.hasNext();) {
                DruidPooledConnection pooledConnection = iter.next();
                if (pooledConnection.isRunning()) {
                    continue;
                }
                long timeMillis = (currrentNanos - pooledConnection.getConnectedTimeNano()) / (1000 * 1000);
                if (timeMillis >= removeAbandonedTimeoutMillis) {
                    iter.remove();
                    pooledConnection.setTraceEnable(false);
                    abandonedList.add(pooledConnection);
                }
            }
        } finally {
            activeConnectionLock.unlock();
        }
    ...
    }
    
销毁连接的方法判断执行时间是否大于removeAbandonedTimeoutMillis，如果大于则销毁，然后发现项目中连接池参数设置配置了如下<br>

    <property name="removeAbandoned" value="true"/>
    <property name="removeAbandonedTimeout" value="60"/>

开启了removeAbandoned且设置了1分钟，由于项目中数据量增加，导致一个定时任务执行时间超过设置的阀值，连接被释放，触发上面的异常。
#### 1.1.3 解决方案
removeAbandoned和removeAbandonedTimeout的设计初衷是为了防止连接泄露的情况发生，所以一定要配置。首先调长removeAbandonedTimeout时间，重新上线把中断的业务跑过，然后优化对应的执行逻辑，缩减运行时间，最终把时长设定在180秒（视业务而定）

### 1.2 获取连接失败(active = maxActive)

#### 1.2.1 异常栈
    ### The error occurred while executing a query
    ### Error querying database. Cause: org.springframework.jdbc.CannotGetJdbcConnectionException: Could not get JDBC Connection; 
    ### nested exception is com.alibaba.druid.pool.GetConnectionTimeoutException: wait millis 30000, active 5, maxActive 5
#### 1.2.2 分析过程
由于最active等于maxActive而且在等待的30000ms中没有连接被释放，最大连接设置成5相对较小
#### 1.2.3 解决方案
maxActive 的值的设置规则一般为：1000 / 服务器数量，但是maxActive最大不要超过60，调整maxActive的值为50

### 1.3 获取连接失败(active < maxActive)

#### 1.3.1 异常栈
    ### Cause: org.springframework.jdbc.CannotGetJdbcConnectionException: 
    Could not get JDBC Connection; nested exception is com.alibaba.druid.pool.GetConnectionTimeoutException:
     wait millis 1000, active 3, maxActive 20, creating 1 
     at org.mybatis.spring.MyBatisExceptionTranslator.translateExceptionIfPossible(MyBatisExceptionTranslator.java:73)
     at org.mybatis.spring.SqlSessionTemplate
#### 1.3.2 分析过程
active < maxActive ，所以当获取连接的时候，要新建连接，此时获取连接方法需要等待，但是由于一些原因（例如网络因素），在1000ms内，新连接没有获取到造成获取连接超时
#### 1.3.3 解决方案
调整maxWait参数值为10000ms

### 1.4 socketTimeout

#### 1.4.1 异常栈
    Last packet sent to the server was 15000 ms ago.;
    nested exception is com.mysql.jdbc.exceptions.jdbc4.CommunicationsException : Communications link failure
    ...
    Caused by: java.net.SocketTimeoutException: Read timed out

#### 1.4.2 分析过程
    jdbc.mysql.connectionProperties=useUnicode=true;characterEncoding=utf8;connectTimeout=3000;socketTimeout=15000;rewriteBatchedStatements=true;autoReconnectForPools=true;failOverReadOnly=false;roundRobinLoadBalance=true;allowMultiQueries=true;
数据库参数配置了socketTimeout=15000 socket在15秒没有返回package
#### 1.4.3 解决方案
删除了 socketTimeout=15000 配置

### 1.5 获取不到数据库连接(部分请求异常)

#### 1.5.1 异常栈
    org.springframework.jdbc.CannotGetJdbcConnectionException: Could not get JDBC Connection
#### 1.5.2 分析过程    
由于不是所以有都获取不到连接，所以考虑连接泄露的问题 数据库配置入下，

        <property name="testWhileIdle" value="false"/>
        <property name="testOnBorrow" value="false"/>
由于数据库空闲连接回收参数未配置且mysql连接8小时不使用，数据库会关闭连接，在获取连接时不校验连接有效，每次都会获取到失效连接
#### 1.5.3 解决方案
        开启获取校验
        <property name="testWhileIdle" value="true"/>
        <property name="testOnBorrow" value="true"/>
        开启空闲连接被回收时间 mysql连接8小时不使用 数据库会关闭连接
        <property name="minEvictableIdleTimeMillis" value="1800000"/>
        <property name="timeBetweenEvictionRunsMillis" value="30000"/>
## 二 druid连接池推荐配置

    <!-- 初始化连接数量 -->
    <property name="initialSize" value="5"/>
    <!-- 最大活动连接数量 -->
    <property name="maxActive" value="50"/>
    <!-- 最小空闲连接数量 -->
    <property name="minIdle" value="3"/>
    <!-- 获取连接时等待时间，超出将抛异常，单位毫秒 -->
    <property name="maxWait" value="10000"/>
    <!-- 是否自动提交 -->
    <property name="defaultAutoCommit" value="false"/>
    <!-- 空闲连接被回收时间，回收空闲连接至minIdle指定数量，单位毫秒 -->
    <property name="minEvictableIdleTimeMillis" value="1800000"/>
    <!-- 检查空闲连接是否可被回收，如果小于等于0，不会启动检查线程，默认-1，单位毫秒 -->
    <property name="timeBetweenEvictionRunsMillis" value="30000"/>
    <!-- SQL查询,用来验证从连接池取出的连接 -->
    <property name="validationQuery" value="select 1"/>
    <!-- 指明连接是否被空闲连接回收器(如果有)进行检验.如果检测失败,则连接将被从池中去除 -->
    <property name="testWhileIdle" value="true"/>
    <!-- 指明是否在从池中取出连接前进行检验,如果检验失败-->
    <property name="testOnBorrow" value="true"/>
    <!-- 指明是否在归还到池中前进行检验-->
    <property name="testOnReturn" value="true"/>
    <!-- 标记是否删除泄露的连接，设置为true可以为写法糟糕的没有关闭连接的程序修复数据库连接. -->
    <property name="removeAbandoned" value="true"/>
    <!-- 泄露的连接可以被删除的超时值, 单位秒 -->
    <property name="removeAbandonedTimeout" value="180"/>
 