# Zookeeper
---
## 一 Znode
Zookeeper的命名空间是由一系列节点组成的节点树，每个数据节点都是有生命周期的，周期长短取决于节点类型
### 1.1 节点类型
|类型|描述|
|:-|:-|
|PERSISTENT|持久化节点上的数据被创建后，一直被保存直到主动删除节点|
|PERSISTENT_SEQUENTIAL|顺序自动编号持久化节点，假如我们在/lock/目录下创建节3个点，ZooKeeper集群会按照提起创建的顺序来创建节点，节点分别为/lock/0000000001、/lock/0000000002、/lock/0000000003|
|EPHEMERAL|临时节点， 客户端session超时这类节点就会被自动删除，zookeeper规定临时节点不能创建子节点|
|EPHEMERAL_SEQUENTIAL|与临时节点基本特性一致，根据客户端的创建顺序添加了顺序特征|
## 二 watcher
