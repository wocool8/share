# 性能调优
---

## 1 减少refresh的次数
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
## 2 减少flush的次数
当translog的数据量达到512M或时常达到30分钟时，会触发一次刷新(Flush)，主要是把文件系统缓存的数据持久化到硬盘上，因为持久化是一个比较耗时的操作，可以通过修改
(index.translog.flush_threshold_size)增加Translog缓存的数据量，来减少刷新的次数
## 3 避免大结果集及深翻
查询from开始的size条数据，需要从每个分片中查询前from + size(n次查询后排序)条数据。协同节点需要收集n个分片的前from + size调数据，然后对数据再进行排序，返回从from + 1开始的
size条数据，如果from/size/n 有一个值很大，这样查询会很耗cpu资源，而且效率也很低，es为解决这种问题提供了scroll和scroll-scan两种方式
### 3.1 scroll
例如 批量查询，要查询1-100页的数据，每页有size条数据，如果使用search查询，重复排序太多次，
scroll得思路是从每个分片上查询100 * size调数据，协同节点处理n * 100 * size条数据，然后对数据进行合并排序，把这100 * size条数据快照起来，醉后使用类似于数据库游标
等形式逐次获取结果，这种做法减少了查询和排序的次数

    SearchResponse responseSearch = client.prepareSearch("mumu")
        .setScroll("1m")
        .setQuery(query)
        .setSize(100)
        .withSort(sort)
        .execute().actionGet();
### 3.2 scroll-scan
scroll-scan 与 scroll相似，只是增加search_type=scan，参数告诉集群不需要对第一次查询的文本进行显示度计算和排序，scroll-scan从2.1.0版本开始被移除了
        
    search_type=scan removeded
    The scan search type was deprecated since version 2.1.0 and is now removed. All benefits from this search type can now be achieved by doing a scroll request that sorts documents in _doc order, for instance:
    GET /my_index/_search?scroll=2m
    {
      "sort": [
        "_doc"
      ]
    }

## 4 选择合适的SearchType
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
### 4.1 QUERY_AND_FETCH 
向索引的所有分片（shard）都发出查询请求，各分片返回的时候把元素文档（document）和计算后的排名信息一起返回。因为只需要去shard查询一次，这种搜索方式是最快的。但是各个shard返回的结果的数量之和可能是用户要求的size的n倍
### 4.2 QUERY_THEN_FETCH  
第一步，先向所有的shard发出请求，各分片只返回排序和排名相关的信息（不包括文档document)，然后按照各分片返回的分数进行重新排序和排名，取前size个文档。第二步，去相关的shard取document。
### 4.3 DFS_QUERY_AND_FETCH 和 DFS_QUERY_THEN_FETCH

    $ curl -XGET 'localhost:9200/startswith/test/_search?pretty=true&search_type=dfs_query_then_fetch' -d '{
            "query": {
            "match_phrase_prefix": {
               "title": {
                 "query": "d",
                 "max_expansions": 5
               }
             }
           }
         }' | grep title
    
          "_score" : 1.9162908, "_source" : {"title":"dzone"}
          "_score" : 1.9162908, "_source" : {"title":"data"}
          "_score" : 1.9162908, "_source" : {"title":"drunk"}
          "_score" : 1.9162908, "_source" : {"title":"drive"}
          
DFS_前缀 多了一个初始化散发(initial scatter)步骤。官方文档描述DFS使用全局打分，对于match_phrase 或 match_phrase_prefix相似度匹配，会使得查询结果更精确。但是如果不使用打分相关则可以使用QUERY_AND_FETCH 和 QUERY_THEN_FETCH 优化性能

## 5 定期删除
Lucene中段具有不变性，所以删除文档后不会立即从从磁盘中删除该文档，而是产生一个.del文档纪录被删除的文档1.而在检索中被删除的文档还会参与检索，在最后阶段会被过滤，
如果删除的文档太多也会影响计算。机器空闲时执行如下命令删除这些文件
   
    curl -XPOST http://localhost:9200/_optimize?onlu_expunge_deletes=true   
## 6 堆大小设置
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
    
### 6.1 最好不要超过物理内存的50%
文件系统缓存存储refresh结果，且如果剩余内存很小也会影响全文检索的速度
### 6.2 最堆内存大小最好不要超过32G
java中，所有的对象都分配在堆上，每个对象头都通过Klass Pointer指针指向元数据，32位最大寻址空间为4GB（2的32次幂）,64位系统为（2的64次幂），在
64位系统上，因为指针变大导致更大的空间浪费在指针上，更大的问题是，更大的指针在主存和各级缓存之间移动会占用更大带宽。Java中使用内存指针压缩（compressed Oops）用来解决这个问题
，它的指针不在表示内存精确位置，而是表示偏移量这意味着32位的指针可以应用4GB的Byte，而不是bit。32GB的物理内存也可以用32位指针表示。
## 7 接入方式
### 7.1 Transport Client (传输客户端)
适合大批量的客户端连接，连接集群和销毁连接比较方便和高效。
### 7.2 Node Client (节点客户端)
节点客户端的本身也是es的node，所以接入和退出时比较复杂，还会影响集群的状态，但是同时也有更高的执行效率，适合持久连接的少量客户端
## 8 角色隔离和脑裂
### 8.1 角色隔离
候选主节点

    node.master: true
    node.date: false
数据节点

    node.master: false
    node.date: true
### 8.2 脑裂
设置选举主节点时需要参与选举的节点个数

    discovery.zen.minimum_master_nodes: （master集群节点数/2） + 1
    
## 9 设置合适的分片数量
分片数量选择2的n次幂(路由可以使用位运算)<br>
ES的index分成若干shard对数据量有硬性限制，每个分片最大能存20亿条数据。<br>
新建分片需要预估集群使用的数据量，建议每个分片存储5000万数据(预估数量上限计算分片数)。由于分片数量过大对分页查询性能有所影响，所以也不建议es分片数量过大

## 10 合理使用doc_values
倒排索引只有词对应的doc，但是并不知道每一个doc中的内容，那么如果想要排序的话每一个doc都去获取一次文档内容。倒排索引将词项映射到包含它们的文档，Doc values将文档映射到它们包含的词项，
Doc Values 和倒排索引一样，基于Segement生成并且是不可变的，Doc Values 本质上是一个序列化的 列式存储，这个结构非常适用于聚合、排序、脚本等操作，Doc Values 默认对所有字段启用，除了 analyzed strings
所以对于不需要排序的字段 禁用 Doc Values，可以优化性能

## 11 mapping

    {
        "mappings": {
            "mumu": {
                "properties": {
                    "name": {
                        // 不需要分词字段设置not_analyzed
                        "index": "not_analyzed",
                        不做检索字段设置 no
                        "index": "no",
                        // 对于不需要排序的字段 禁用 Doc Values
                        doc_values : false
                        "type": "string"
                    }
                }
            }
        }
    }

## 12 routing(routing就是hash key， 这个key决定写入/查询的分片id)
如果不设置routing，ES写入时使用默认文档的ID将文档平均的分布于所有的分片上，查询时查询所有分片。为了避免把query下发到所有的分片进而提升检索性能，可以在写入数据以带有业务意义的字段作为
routing 例如账号，把一个账号的所有插入操作都写到同一个分片上，查询时指定账号映射到对应分片查询。但是自定义routing可能会造成数据倾斜

## 13 使用filter查询
Elasticsearch 可以缓存非评分查询从而达到更快的访问，ES 2.0API改动幅度比较大 可以直接使用BoolQueryBuilder使用Filter

    boolQueryBuilder.filter("name", "mumu"));



