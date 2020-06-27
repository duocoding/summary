# caffeine
## caffeine简介  
Caffeine是基于Java 8的高性能缓存库，可提供接近最佳的命中率。
缓存与ConcurrentMap相似，但并不完全相同。最根本的区别是ConcurrentMap会保留添加到其中的所有元素，直到将其明确删除为止。Cache另一方面，通常可以配置为自动退出条目，以限制其内存占用量。在某些情况下，由于LoadingCacheor AsyncLoadingCache会自动退出缓存，即使不驱逐条目也可能有用。  
## 使用方式  
1. 引入依赖  
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-cache</artifactId>
</dependency>

<dependency>
    <groupId>com.github.ben-manes.caffeine</groupId>
    <artifactId>caffeine</artifactId>
    <version>2.6.2</version>
</dependency>
```  
2. 配置信息
```xml
spring.cache.cache-names=cache1
spring.cache.caffeine.spec=initialCapacity=50,maximumSize=500
```  

3. 使用  
    - 手动加载
在每次get key的时候指定一个同步的函数，如果key不存在就调用这个函数生成一个值。
        ```java
        public Object manulOperator(String key) {
            Cache<String, Object> cache = Caffeine.newBuilder()
                .expireAfterWrite(1, TimeUnit.SECONDS)
                .expireAfterAccess(1, TimeUnit.SECONDS)
                .maximumSize(10)
                .build();
            //如果一个key不存在，那么会进入指定的函数生成value
            Object value = cache.get(key, t -> setValue(key).apply(key));
            cache.put("hello",value);

            //判断是否存在如果不存返回null
            Object ifPresent = cache.getIfPresent(key);
            //移除一个key
            cache.invalidate(key);
            return value;
        }

        public Function<String, Object> setValue(String key){
            return t -> key + "value";
        }
        ```  
    - 同步加载
    构造Cache时候，build方法传入一个CacheLoader实现类。实现load方法，通过key加载value。
        ```java
        /**
            * 同步加载
            * @param key
            * @return
            */
        public Object syncOperator(String key){
            LoadingCache<String, Object> cache = Caffeine.newBuilder()
                .maximumSize(100)
                .expireAfterWrite(1, TimeUnit.MINUTES)
                .build(k -> setValue(key).apply(key));
            return cache.get(key);
        }

        public Function<String, Object> setValue(String key){
            return t -> key + "value";
        }
        ``` 
    - 异步加载
AsyncLoadingCache是继承自LoadingCache类的，异步加载使用Executor去调用方法并返回一个CompletableFuture。异步加载缓存使用了响应式编程模型。
如果要以同步方式调用时，应提供CacheLoader。要以异步表示时，应该提供一个AsyncCacheLoader，并返回一个CompletableFuture。
        ```java
            /**
            * 异步加载
            *
            * @param key
            * @return
            */
        public Object asyncOperator(String key){
            AsyncLoadingCache<String, Object> cache = Caffeine.newBuilder()
                .maximumSize(100)
                .expireAfterWrite(1, TimeUnit.MINUTES)
                .buildAsync(k -> setAsyncValue(key).get());

            return cache.get(key);
        }

        public CompletableFuture<Object> setAsyncValue(String key){
            return CompletableFuture.supplyAsync(() -> {
                return key + "value";
            });
        }
        ```  
## caffeine常用配置说明
```xml
initialCapacity=[integer]: 初始的缓存空间大小

maximumSize=[long]: 缓存的最大条数

maximumWeight=[long]: 缓存的最大权重

expireAfterAccess=[duration]: 最后一次写入或访问后经过固定时间过期

expireAfterWrite=[duration]: 最后一次写入后经过固定时间过期

refreshAfterWrite=[duration]: 创建缓存或者最近一次更新缓存后经过固定的时间间隔，刷新缓存

weakKeys: 打开key的弱引用

weakValues：打开value的弱引用

softValues：打开value的软引用

recordStats：开发统计功能
```  
### !注意
```
expireAfterWrite和expireAfterAccess同时存在时，以expireAfterWrite为准。

maximumSize和maximumWeight不可以同时使用

weakValues和softValues不可以同时使用
```  
