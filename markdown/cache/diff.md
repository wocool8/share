## Redis和Memcached的区别

### 1、数据类型支持不同
- [Redis五种数据结构及指令](/markdown/cache/redisDataStructrue.md)
- Memcached仅支持简单的key-value结构的数据
### 2、内存管理机制不同
- redis 并不是所有的数据都一直存储在内存中的

    当物理内存用完时，Redis可以将一些很久没用到的value交换到磁盘。Redis只会缓存所有的key的信息，如果Redis发现内存的使用量超过了某一个阀值，将触发swap的操作，Redis根据“swappability = age*log(size_in_memory)”计算出哪些key对应的value需要swap到磁盘。然后再将这些key对应的value持久化到磁盘中，同时在内存中清除。

- Memcached所有数据都存储在内存
### 3、数据持久化支持
- [Redis持久化](/markdown/cache/redisPersistence.md)
- Memcached不支持持久化
### 4、集群管理的不同
- Memcached本身并不支持分布式

    因此只能在客户端通过像一致性哈希这样的分布式算法来实现Memcached的分布式存储

- Redis更偏向于在服务器端构建分布式存储

    [Redis集群](/markdown/cache/redisCluster.md)

### 5、线程模型    
- Memcached是多线程，非阻塞IO复用的网络模型 
- Redis使用[单线程多路复用](/markdown/cache/redisSingleThread.md) 

### 统计及正则表达式
- Memcached不支持统计和key的正则表达式查询
- Redis支持简单的统计，和key的正则表达式查询 
