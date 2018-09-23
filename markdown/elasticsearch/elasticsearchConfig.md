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
    # ---------------------------------- transport -----------------------------------
    transport.tcp.port: 9300
    # 节点间传输数据时是否需要压缩
    transport.tcp.compress: true