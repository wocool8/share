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
                        cache = CacheBuilder.newBuilder()
                                // 基于缓存数量删除，如果缓存数量接近最大值会把不常用的删除（LRU）
                                .maximumSize(maximumSize)
                                // 给予过期时间删除（类似于FIFO）
                                .expireAfterWrite(expireAfterWriteDuration, timeUnit)
                                // key最后访问过期时间（LRU）
                                .expireAfterAccess(expireAfterWriteDuration,timeUnit)
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
        
### 1.2 CallAble 方式

    public abstract class GuavaLoadAbstract<K, V> {
        /**
         * cache instance
         */
        private Cache<K, V> callableCache;
        
        public LoadingCache<K, V> getCallAbleCache() {
            if(null == callableCache){
                synchronized (this) {
                    if(null == callableCache){
                        callableCache = CacheBuilder.newBuilder().maximumSize(maximumSize)
                                .expireAfterWrite(expireAfterWriteDuration,timeUnit)
                                .recordStats()
                                .build();
                    }
                }
            }
            return cache;
        }
        
        abstract V call(K k) throws ExecutionException;
    }    
    
    // 查询不到通过自定义方法构建缓存 并更新 相对于CacheLoader更加灵活
    public class MumuLocalCache extends GuavaLoadAbstract<String, String> {
        List<Packets> call(String s) throws ExecutionException {
            return super.getCallAbleCache().get(s , new Callable<String>() {
                public List<Packets> call() {
                    // do something to get value
                    return "";
                }
            });
        }    
    }
    
## 二 阻塞机制
### 2.1 阻塞 LOAD

        V get(K key, int hash, CacheLoader<? super K, V> loader) throws ExecutionException {
            Preconditions.checkNotNull(key);
            Preconditions.checkNotNull(loader);

            Object var15;
            try {
                if (this.count != 0) {
                    LocalCache.ReferenceEntry<K, V> e = this.getEntry(key, hash);
                    if (e != null) {
                        long now = this.map.ticker.read();
                        V value = this.getLiveValue(e, now);
                        if (value != null) {
                            this.recordRead(e, now);
                            this.statsCounter.recordHits(1);
                            Object var17 = this.scheduleRefresh(e, key, hash, value, now, loader);
                            return var17;
                        }

                        LocalCache.ValueReference<K, V> valueReference = e.getValueReference();
                        if (valueReference.isLoading()) {
                            Object var9 = this.waitForLoadingValue(e, key, valueReference);
                            return var9;
                        }
                    }
                }

                var15 = this.lockedGetOrLoad(key, hash, loader);
            } catch (ExecutionException var13) {
                Throwable cause = var13.getCause();
                if (cause instanceof Error) {
                    throw new ExecutionError((Error)cause);
                }

                if (cause instanceof RuntimeException) {
                    throw new UncheckedExecutionException(cause);
                }

                throw var13;
            } finally {
                this.postReadCleanup();
            }

            return var15;
        }

在get方法实现中会调用 scheduleRefresh方法 如下代码

        V scheduleRefresh(LocalCache.ReferenceEntry<K, V> entry, K key, int hash, V oldValue, long now, CacheLoader<? super K, V> loader) {
            if (this.map.refreshes() && now - entry.getWriteTime() > this.map.refreshNanos && !entry.getValueReference().isLoading()) {
                V newValue = this.refresh(key, hash, loader, true);
                if (newValue != null) {
                    return newValue;
                }
            }

            return oldValue;
        }
        
 此处会阻塞多个请求加载新的缓存值