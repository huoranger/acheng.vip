---
title: 使用 Guava 实现本地缓存
date: 2024-08-22
categories: ['Tech']
draft: false
---

引入 Guava 所需要的依赖包

```xml
<dependency>
    <groupId>com.google.guava</groupId>
    <artifactId>guava</artifactId>
    <version>32.1.2-jre</version>
</dependency>
```

使用「旁路模型构」建缓存

```java
private final LoadingCache<Long, User> storage = CacheBuilder.newBuilder()
            // 设置初始容量
            .initialCapacity(10)
            // 限制数量，最大值（也可限制对象栈内存总大小）
            .maximumSize(100)
            // 并发数量
            .concurrencyLevel(5)
            // 写入10分钟后过期
            .expireAfterWrite(10, TimeUnit.MINUTES)
            // 统计缓存命中率
            .recordStats()
            // 使用旁路型缓存（数据不存在，回源查找，放入缓存）
            .build(
                    new CacheLoader<>() {
                        @Override
                        public User load(Long key) {
                            log.info("用户:{}缓存不存在，尝试CacheLoader回源查找并回填...", key);
                            return new User().setId(1L).setName("蓝皮猫");
                        }
                    }
            );


    @SneakyThrows
    public User getUser(Long key) {
        User user = null;
        try {
            user = storage.get(key);
        } finally {
            System.out.println(new GsonBuilder().setPrettyPrinting().create().toJson(storage.stats()));
        }
        return user;
    }

    public void putUser(Long key, User value) {
        storage.put(key, value);
    }
```

