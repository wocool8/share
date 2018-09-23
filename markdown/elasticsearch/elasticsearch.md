# elasticsearch 参数配置及性能调优
---
## 一 config
elasticsearch安装目录下的config文件夹包含了配置文件elasticsearch.yml

    # ---------------------------------- Cluster -----------------------------------
    #
    # Use a descriptive name for your cluster:
    #
    cluster.name: mumu
    #
    # ------------------------------------ Node ------------------------------------
    #
    # Use a descriptive name for the node:
    #
    node.name: node-1
    #
    # Add custom attributes to the node:
    #
    node.attr.rack: r1
    # 指定该节点是否有资格被选举为Master节点，默认是true，如果被设置成true，自是有资格成为主节点的被选举机会，不一定会成为主节点
    node.master: true
    # 指定节点是否存储索引数据
    node.data: true    
    #
    # ----------------------------------- Paths ------------------------------------
    #
    # Path to directory where to store the data (separate multiple locations by comma):
    #
    path.data: /path/to/data
    #
    # Path to log files:
    #
    path.logs: /path/to/logs
    #配置文件路径
    path.config: /path/to/config
    # ----------------------------------- Memory -----------------------------------
    #
    # Lock the memory on startup:
    #
    bootstrap.memory_lock: true
    #
    # Make sure that the heap size is set to about half the memory available
    # on the system and that the owner of the process is allowed to use this
    # limit.
    #
    # Elasticsearch performs poorly when the system is swapping the memory.
    #
    # ---------------------------------- Network -----------------------------------
    #
    # Set the bind address to a specific IP (IPv4 or IPv6):
    #
    network.host: 192.168.0.1
    #
    # Set a custom port for HTTP:
    #
    http.port: 9200
    http.cors.enabled: true
    http.cors.allow-origin: "*"
    # --------------------------------- Discovery ----------------------------------
    #
    # Pass an initial list of hosts to perform discovery when new node is started:
    # The default list of hosts is ["127.0.0.1", "[::1]"]
    #
    discovery.zen.ping.unicast.hosts: ["host1", "host2"]
    #
    # Prevent the "split brain" by configuring the majority of nodes (total number of master-eligible nodes / 2 + 1):
    #
    discovery.zen.minimum_master_nodes: 3
    # ---------------------------------- Gateway -----------------------------------
    #
    # Block initial recovery after a full cluster restart until N nodes are started:
    #
    gateway.recover_after_nodes: 3
    # ---------------------------------- Various -----------------------------------
    #
    # Require explicit names when deleting indices:
    #
    action.destructive_requires_name: true
    # ---------------------------------- index -----------------------------------
    # 索引数据的默认分片数
    index.number_of_shardings: 5
    # 默认的索引副本数
    index.number_of_replicas: 1
    # 触发refresh时间间隔
    index.refresh.interval: 30ms
    # 系统文件缓存刷新到磁盘的数据量大小，默认值是512M
    index.translog.flush_threshold_size
    # ---------------------------------- transport -----------------------------------
    transport.tcp.port: 9300
    # 节点间传输数据时是否需要压缩
    transport.tcp.compress: true
    
## 二 性能优化

### 2.1 减少refresh的次数
Lucene在新增数据时为了提高写性能，采用的是延迟写入的策略，即先将数据写入内存中，当延时超过一秒时(index.refresh.interval参数配置)，会触发一次refresh，refresh会把内存中的数据以
段德兴市刷到文件系统的缓存中

    org.springframework.data.repository.CrudRepository得save方法
        public <S extends T> S save(S entity) {
            Assert.notNull(entity, "Cannot save 'null' entity.");
            this.elasticsearchOperations.index(this.createIndexQuery(entity));
            this.elasticsearchOperations.refresh(this.entityInformation.getIndexName());
            return entity;
        }
在一次大数据量数据存储实现中，采用MQ + ES，由于es使用了 org.springframework.data.repository.CrudRepository.save()方法过于频繁refresh造成MQ消费大量积压

        @Override
        public <T> void createSameTypeDocs(String index, String type, List<T> objects) {
            List<IndexQuery> indexQueries = new ArrayList<>();
            if (!CollectionUtils.isEmpty(objects)) {
                objects.forEach(object -> indexQueries.add(buildIndexQuery(index, type, object)));
            }
            if (!CollectionUtils.isEmpty(indexQueries)) {
                elasticsearchTemplate.bulkIndex(indexQueries);
            }
        }
由于存储的数据实时查询要求不高，所以采用以默认1s执行refresh
### 2.2 减少flush的次数
当translog的数据量达到512M或时常达到30分钟时，会触发一次刷新(Flush)，主要是把文件系统缓存的数据持久化到硬盘上，因为持久化是一个比较耗时的操作，可以通过修改
(index.translog.flush_threshold_size)增加Translog缓存的数据量，来减少刷新的次数
### 2.3 避免大结果集及深翻
查询from开始的size条数据，需要从每个分片中查询前from + size(n次查询后排序)条数据。协同节点需要收集n个分片的前from + size调数据，然后对数据再进行排序，返回从from + 1开始的
size条数据，如果from/size/n 有一个值很大，这样查询会很耗cpu资源，而且效率也很低，es为解决这种问题提供了scroll和scroll-scan两种方式
#### 2.3.1 scroll
例如 批量查询，要查询1-100页的数据，每页有size条数据，如果使用search查询，重复排序太多次，
scroll得思路是从每个分片上查询100 * size调数据，协同节点处理n * 100 * size条数据，然后对数据进行合并排序，把这100 * size条数据快照起来，醉后使用类似于数据库游标
等形式逐次获取结果，这种做法减少了查询和排序的次数

    SearchResponse responseSearch = client.prepareSearch("mumu")
        .setScroll("1m")
        .setQuery(query)
        .setSize(100)
        .withSort(sort)
        .execute().actionGet();
#### 2.3.2 scroll-scan
scroll-scan 与 scroll相似，只是增加search_type=scan，参数告诉集群不需要对第一次查询的文本进行显示度计算和排序，scroll-scan从2.1.0版本开始被移除了
        
    search_type=scan removeded
    The scan search type was deprecated since version 2.1.0 and is now removed. All benefits from this search type can now be achieved by doing a scroll request that sorts documents in _doc order, for instance:
    GET /my_index/_search?scroll=2m
    {
      "sort": [
        "_doc"
      ]
    }

### 2.4 选择合适的SearchType
    public enum SearchType {
        DFS_QUERY_THEN_FETCH((byte)0),
        QUERY_THEN_FETCH((byte)1),
        DFS_QUERY_AND_FETCH((byte)2),
        QUERY_AND_FETCH((byte)3),
        /** @deprecated */
        @Deprecated
        SCAN((byte)4),
        /** @deprecated */
        @Deprecated
        COUNT((byte)5);   
    }
DFS_QUERY_THEN_FETCH 和 QUERY_THEN_FETCH 区别是DFS_QUERY_THEN_FETCH包含一个额外的阶段，在初始查询中执行词频计算，从而进行更精确的打分
### 2.5 定期删除
Lucene中段具有不变性，所以删除文档后不会立即从从磁盘中删除该文档，而是产生一个.del文档纪录被删除的文档1.而在检索中被删除的文档还会参与检索，在最后阶段会被过滤，
如果删除的文档太多也会影响计算。机器空闲时执行如下命令删除这些文件
   
    curl -XPOST http://localhost:9200/_optimize?onlu_expunge_deletes=true   
### 2.6 堆大小设置
config文件夹下的jvm.option文件

    ## JVM configuration
    ################################################################
    ## IMPORTANT: JVM heap size
    ################################################################
    ##
    ## You should always set the min and max JVM heap
    ## size to the same value. For example, to set
    ## the heap to 4 GB, set:
    ##
    -Xms4g
    -Xmx4g
    ################################################################
    ## Expert settings
    ################################################################
    ##
    ## All settings below this section are considered
    ## expert settings. Don't tamper with them unless
    ## you understand what you are doing
    ##
    ################################################################
    ## GC configuration
    -XX:+UseConcMarkSweepGC
    -XX:CMSInitiatingOccupancyFraction=75
    -XX:+UseCMSInitiatingOccupancyOnly
    ## optimizations
    # pre-touch memory pages used by the JVM during initialization
    -XX:+AlwaysPreTouch
    ## basic
    # force the server VM (remove on 32-bit client JVMs)
    -server
    # explicitly set the stack size (reduce to 320k on 32-bit client JVMs)
    -Xss1m
    # set to headless, just in case
    -Djava.awt.headless=true
    # ensure UTF-8 encoding by default (e.g. filenames)
    -Dfile.encoding=UTF-8
    # use our provided JNA always versus the system one
    -Djna.nosys=true
    # use old-style file permissions on JDK9
    -Djdk.io.permissionsUseCanonicalPath=true
    # flags to configure Netty
    -Dio.netty.noUnsafe=true
    -Dio.netty.noKeySetOptimization=true
    -Dio.netty.recycler.maxCapacityPerThread=0
    # log4j 2
    -Dlog4j.shutdownHookEnabled=false
    -Dlog4j2.disable.jmx=true
    -Dlog4j.skipJansi=true
    ## heap dumps
    # generate a heap dump when an allocation from the Java heap fails
    # heap dumps are created in the working directory of the JVM
    -XX:+HeapDumpOnOutOfMemoryError
    # specify an alternative path for heap dumps
    # ensure the directory exists and has sufficient space
    #-XX:HeapDumpPath=${heap.dump.path}
    ## GC logging
    #-XX:+PrintGCDetails
    #-XX:+PrintGCTimeStamps
    #-XX:+PrintGCDateStamps
    #-XX:+PrintClassHistogram
    #-XX:+PrintTenuringDistribution
    #-XX:+PrintGCApplicationStoppedTime
    # log GC status to a file with time stamps
    # ensure the directory exists
    #-Xloggc:${loggc}
    # By default, the GC log file will not rotate.
    # By uncommenting the lines below, the GC log file
    # will be rotated every 128MB at most 32 times.
    #-XX:+UseGCLogFileRotation
    #-XX:NumberOfGCLogFiles=32
    #-XX:GCLogFileSize=128M
    # Elasticsearch 5.0.0 will throw an exception on unquoted field names in JSON.
    # If documents were already indexed with unquoted fields in a previous version
    # of Elasticsearch, some operations may throw errors.
    # WARNING: This option will be removed in Elasticsearch 6.0.0 and is provided
    # only for migration purposes.
    #-Delasticsearch.json.allow_unquoted_field_names=true
    
#### 2.6.1 最好不要超过物理内存的50%
文件系统缓存存储refresh结果，且如果剩余内存很小也会影响全文检索的速度
#### 2.6.2 最堆内存大小最好不要超过32G
java中，所有的对象都分配在堆上，每个对象头都通过Klass Pointer指针指向元数据，32位最大寻址空间为4GB（2的32次幂）,64位系统为（2的64次幂），在
64位系统上，因为指针变大导致更大的空间浪费在指针上，更大的问题是，更大的指针在主存和各级缓存之间移动会占用更大带宽。Java中使用内存指针压缩（compressed Oops）用来解决这个问题
，它的指针不在表示内存精确位置，而是表示偏移量这意味着32位的指针可以应用4GB的Byte，而不是bit。32GB的物理内存也可以用32位指针表示。
### 2.7 接入方式
#### 2.7.1 Transport Client (传输客户端)
适合大批量的客户端连接，连接集群和销毁连接比较方便和高效。
#### 2.7.2 Node Client (节点客户端)
节点客户端的本身也是es的node，所以接入和退出时比较复杂，还会影响集群的状态，但是同时也有更高的执行效率，适合持久连接的少量客户端
### 2.8 角色隔离和脑裂
#### 2.8.1 角色隔离
候选主节点

    node.master: true
    node.date: false
数据节点

    node.master: false
    node.date: true
#### 2.8.2 脑裂
设置选举主节点时需要参与选举的节点个数

    discovery.zen.minimum_master_nodes: （master集群节点数/2） + 1
