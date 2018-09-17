# Guava
---
## 一 Guava 的创建方式
### 1.1 CacheLoader 方式

    @SLF4J
    @Date
    public abstract class GuavaLoadAbstract<K, V> {
    
        /**
         * 最大缓存条数
         */
        private int maximumSize = 1000000;
        /**
         * 数据存活时长
         */
        private int expireAfterWriteDuration = 60;
        /**
         * 数据存活时长单位
         */
        private TimeUnit timeUnit = TimeUnit.MINUTES;
        /**
         * cache instance
         */
        private LoadingCache<K, V> cache;
    
        /**
         * @param
         * @return cache
         * @Description 过调用getCache().get(key)来获取数据
         */
        public LoadingCache<K, V> getCache() {
            if (null == cache) {
                synchronized (this) {
                    if (null == cache) {
                        cache = CacheBuilder.newBuilder().maximumSize(maximumSize)
                                .expireAfterWrite(expireAfterWriteDuration, timeUnit)
                                .recordStats()
                                .build(new CacheLoader<K, V>() {
                                    @Override
                                    public V load(K key) {
                                        return fetchData(key);
                                    }
                                });
                        log.info("Local cache {} initialization successfully", this.getClass().getSimpleName());
                    }
                }
            }
            return cache;
        }
    
        /**
         * @param key
         * @return value, 连同key一起被加载到缓存中的
         * @Description 根据key从数据库或其他数据源中获取一个value，并被自动保存到缓存中。
         */
        protected abstract V fetchData(K key);
    }
        
### 1.1 CallAble 方式
