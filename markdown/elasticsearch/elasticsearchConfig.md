# elasticsearch 参数配置
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
查询from开始的size条数据，需要从每个分片中查询前from + size条数据。协同节点需要收集n个分片的前 from + size调数据，然后对数据再进行排序，返回从from + 1开始的
size条数据，如果from/size/n 有一个值很大，这样查询会很耗cpu资源，而且效率也很低，es为解决这种问题提供了scroll和scroll-scan两种方式
#### 2.3.1 scroll
#### 2.3.2 scroll-scan

