---
title: Caffeine Cache under the hood
categories: [ 编程, Cache ]
tags: [ caffeine cache]
---

> Caffeine is a high performance Java caching library providing a near optimal hit rate.

- 自动加载value, 支持异步加载
- 基于size的eviction：frequency and recency
- 基于时间的过期策略：last access or last write
- 异步更新value
- key支持weak reference
- value支持weak reference和soft reference
- 淘汰（或移除）通知
- todo
- 缓存获取数据统计

Caffeine基于[JSR-107 JCache](https://www.jcp.org/en/jsr/detail?id=107)规范实现。
Caffeine的性能benchmark可参考[Caffeine Benchmarks](https://github.com/ben-manes/caffeine/wiki/Benchmarks)，
可以看到Caffeine在不同读写比的情况下， 吞吐量比Guava Cache和Jdk内置的HashMap都有较大提升。
接下来了解下Caffeine的使用和实现原理。

## 使用

```java
        // manual cache，Cache-Aside模式
        Cache<String, String> cache = Caffeine.newBuilder()
                                          .expireAfterWrite(Duration.ofHours(1))
                                          .build();
        cache.put("hello", "world");
        // null表示不存在key
        @Nullable String val = cache.getIfPresent("hello");
        // key不存在则走后面的loading function
        String val2 = cache.get("hello", key -> "world");


        // loading cache，Read-Through模式
        LoadingCache<String, String> loadingCache = Caffeine.newBuilder()
                                                 .expireAfterAccess(Duration.ofMinutes(10))
                                                 .maximumSize(2000)
                                                 .build(new CacheLoader<String, String>() {
                                                     @Override
                                                     public @Nullable String load(@NonNull String s) throws Exception {
                                                         // your loading logic
                                                         return "";
                                                     }
                                                 });
        // 不存在会自动loading
        String val3 = loadingCache.get("hello");
```

## 接口设计

CacheCache按不同维度划分：
- kv是否有限：Bounded or Unbounded，判断参见方法`com.github.benmanes.caffeine.cache.Caffeine#isBounded`，大部分场景都使用BoundedCache
- 是否自动loading: ManualCache和LoadingCache
- 是否异步: Cache和AsyncCache

对应接口UML如下（实现类只画出了BoundedCache），其接口和类均位于`com.github.benmanes.caffeine.cache`包下:

![](/assets/2024/09/29/caffeine_uml.png)

可以看到，核心的kv存储是放在了`LocalCache`接口（本质是一个`ConcurrentMap`）下，而其他的Loading和Async接口在存储基础上附加了自动加载value和异步加载的能力，以同步接口为例，接口能力如下：

- `Cache`: 提供缓存基础能力，包含设置、读取、失效、获取缓存统计数据等功能；
- `LoadingCache`: 提供自动加载能力，在`Cache`基础上新增不存在则loading value的读取以及refresh方法；
- `LocalCache`: 提供核心存储能力，线程安全和原子能力保证，继承自`java.util.concurrent.ConcurrentMap`；
- `LocalManualCache`: 基于`LocalCache`做存储的非自动Loading Cache；
- `LocalLoadingCache`: 继承自`LocalManualCache`，新增Loading功能。









