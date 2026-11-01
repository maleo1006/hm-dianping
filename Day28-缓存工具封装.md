---
tags:
  - 黑马点评
  - Redis
  - 缓存工具
  - 封装
---
# Day11 - 缓存工具封装

## 一、为什么要封装？

前面写的缓存代码逻辑太复杂（穿透判空、空值缓存、互斥锁、逻辑过期、DoubleCheck……），后面每个业务都要再写一遍这些逻辑，**开发成本很高**。所以把这几套方案**封装成工具类**，以后任何业务查缓存都直接调用，传个参数就行。

**基于 StringRedisTemplate 封装一个缓存工具类，满足四个需求：**

| 方法 | 需求 | 解决的问题 |
|------|------|-----------|
| `set` | 对象序列化为 JSON 存入 String key，可设 TTL | 基础写缓存 |
| `setWithLogicalExpire` | 对象序列化为 JSON 存入，可设**逻辑过期**时间 | 为击穿方案做准备 |
| `queryWithPassThrough` | 查缓存反序列化，**缓存空值**方式 | 缓存穿透 |
| `queryWithLogicalExpire` | 查缓存反序列化，**逻辑过期**方式 | 缓存击穿 |

## 二、构造器注入

```java
@Component
public class CacheClient {

    private final StringRedisTemplate stringRedisTemplate;

    // 构造器注入（Lombok @RequiredArgsConstructor 也行）
    public CacheClient(StringRedisTemplate stringRedisTemplate) {
        this.stringRedisTemplate = stringRedisTemplate;
    }

    private static final Long CACHE_NULL_TTL = 2L;   // 空值 TTL（短）
    private static final String LOCK_SHOP_KEY = "lock:shop:";
    // 逻辑过期重建用的线程池
    private static final ExecutorService CACHE_REBUILD_EXECUTOR = Executors.newFixedThreadPool(10);
}
```

>  **为什么用构造器注入？**
> - CacheClient 是个 `@Component`（Spring 管理的 Bean），Spring 会自动把容器里的 StringRedisTemplate 通过构造方法传进来
> - 相比 `@Resource` 字段注入，**构造器注入更推荐**：final 保证不可变、依赖一目了然、方便测试
## 三、四个方法

### 方法1：`set` —— 对象序列化 + TTL

```java
public void set(String key, Object value, Long time, TimeUnit unit) {
    stringRedisTemplate.opsForValue().set(key, JSONUtil.toJsonStr(value), time, unit);
}
```

> 把任意对象转成 JSON 字符串存进 Redis，设置过期时间。就是"普通缓存"的封装。

### 方法2：`setWithLogicalExpire` —— 逻辑过期写入

```java
public void setWithLogicalExpire(String key, Object value, Long time, TimeUnit unit) {
    // 设置逻辑过期
    RedisData redisData = new RedisData();
    redisData.setExpireTime(LocalDateTime.now().plusSeconds(unit.toSeconds(time)));
    redisData.setData(value);
    // 写入Redis
    stringRedisTemplate.opsForValue().set(key, JSONUtil.toJsonStr(redisData));
}
```

> 把"数据 + 逻辑过期时间"一起包进 RedisData 存进 Redis（**不设真实 TTL**）。供逻辑过期方案用。

### 方法3：`queryWithPassThrough` —— 缓存空值解决穿透

```java
public <T, ID> T queryWithPassThrough(
        String keyPrefix, ID id, Class<T> type,
        Function<ID, T> dbFallback, Long time, TimeUnit unit) {
    // 1. 从缓存中查询
    String key = keyPrefix + id;
    String json = stringRedisTemplate.opsForValue().get(key);
    if (StrUtil.isNotBlank(json)) {
        return JSONUtil.toBean(json, type);        // 命中正常数据
    }
    if (json != null) {
        return null;                                // 命中空值（缓存过空串）
    }
    // 2. 未命中，查数据库（通过函数式参数传入）
    T t = dbFallback.apply(id);
    if (t == null) {
        // 3. 缓存空值 + 短TTL
        stringRedisTemplate.opsForValue().set(key, "", CACHE_NULL_TTL, TimeUnit.MINUTES);
        return null;
    }
    // 4. 写入缓存
    this.set(key, t, time, unit);
    return t;
}
```

**核心亮点：`Function<ID, T> dbFallback` 函数式参数**

> 把"查数据库"这个动作作为**参数**传进来——工具类不需要知道具体查什么表，调用者告诉它怎么查就行。
>
> 调用方式：
> ```java
> // 方法引用（更简洁）
> Shop shop = cacheClient.queryWithPassThrough(
>         CACHE_SHOP_KEY, id, Shop.class, this::getById, 30L, TimeUnit.MINUTES);
>
> // 或 Lambda 表达式（等价）
> Shop shop = cacheClient.queryWithPassThrough(
>         CACHE_SHOP_KEY, id, Shop.class, id2 -> getById(id2), 30L, TimeUnit.MINUTES);
> ```

> **泛型 `<T, ID>` 为什么两个？** T 是返回类型（Shop），ID 是 id 类型（Long）。工具类通用，任何实体、任何 id 类型都能用。

### 方法4：`queryWithLogicalExpire` —— 逻辑过期解决击穿

```java
public <T, ID> T queryWithLogicalExpire(
        String keyPrefix, ID id, Class<T> type,
        Function<ID, T> dbFallback, Long time, TimeUnit unit) {
    // 1. 从缓存中查询
    String key = keyPrefix + id;
    String json = stringRedisTemplate.opsForValue().get(key);
    if (StrUtil.isBlank(json)) {
        return null;                       // 未命中（没预热），返回 null
    }
    // 2. 反序列化
    RedisData redisData = JSONUtil.toBean(json, RedisData.class);
    T t = JSONUtil.toBean((JSONObject) redisData.getData(), type);
    LocalDateTime expireTime = redisData.getExpireTime();
    // 3. 未过期，直接返回
    if (expireTime.isAfter(LocalDateTime.now())) {
        return t;
    }
    // 4. 已过期，尝试获取互斥锁
    String lockKey = LOCK_SHOP_KEY + id;
    boolean isLock = tryLock(lockKey);
    if (isLock) {
        // 5. 成功：开启新线程重建缓存（不阻塞当前请求）
        CACHE_REBUILD_EXECUTOR.submit(() -> {
            try {
                // 查数据库（还是函数式参数！）
                T t1 = dbFallback.apply(id);
                // 写入缓存 + 重置逻辑过期时间
                this.setWithLogicalExpire(key, t1, time, unit);
            } catch (Exception e) {
                throw new RuntimeException(e);
            } finally {
                unlock(lockKey);           // 释放锁
            }
        });
    }
    // 6. 失败：返回过期的旧数据
    return t;
}
```
---

## ⚠️ 避坑 Tips

### 1. 泛型的坑
- **`<T, ID>` 泛型要写全**：T 是返回类型，ID 是 id 类型，方法签名里都要出现才能推断
- **`JSONUtil.toBean(json, type)` 的 type 是 Class<T>**：泛型擦除后运行时需要真实的 Class 才能反序列化，所以方法必须传 `Class<T> type` 参数

### 2. 函数式参数的坑
- **`Function<ID, T>` 是"传入 ID 返回 T"**：`dbFallback.apply(id)` 就是调用传入的查询方法
- **方法引用 `this::getById` 和 Lambda `id2 -> getById(id2)` 等价**：能写方法引用就别写 Lambda，更简洁
- **空值判定的顺序不能乱**：isNotBlank → != null → 查库

### 3. 注入的坑
- **构造器注入要加 `@Component`**：不加这个注解，Spring 不创建 CacheClient，注入无从谈起
- **final 字段 + 构造器**：配合 Lombok 的 `@RequiredArgsConstructor` 可以少写构造方法
- **空值 TTL 要短**：`CACHE_NULL_TTL` 2 分钟，别和正常缓存一样
---
## 🎯 面试题

### Q1：为什么要封装缓存工具类？解决了什么问题？

> **回答框架：**
> 1. **背景**：缓存穿透/击穿的解决方案（空值、互斥锁、逻辑过期）逻辑复杂，每个业务都要写一遍
> 2. **封装**：把四套方案抽成工具类——set（TTL）、setWithLogicalExpire（逻辑过期）、queryWithPassThrough（防穿透）、queryWithLogicalExpire（防击穿）
> 3. **好处**：① 业务代码一行调用 ② 换业务不用改工具类 ③ 逻辑统一，不会每个地方写歪

### Q2：为什么用 Function 函数式参数？泛型怎么设计的？

> **回答框架：**
> 1. **Function<ID, T>**：把"查数据库"的动作作为参数传入，工具类不知道具体业务，只负责缓存逻辑——**依赖倒置**，调用者传方法引用 `this::getById` 或 Lambda
> 2. **泛型 `<T, ID>`**：T 返回类型（Shop）、ID 主键类型（Long），任何实体通用
> 3. **Class<T> type 参数**：泛型运行时会擦除，反序列化必须要有真实的 Class 才能 `JSONUtil.toBean`
> 4. 这是**策略模式**的体现：算法骨架（缓存流程）固定，具体查询策略由调用方注入

### Q3：缓存工具类里的线程池是干嘛的？要注意什么？

> **回答框架：**
> 1. **用途**：逻辑过期方案中，拿到锁的线程开启**独立线程**后台重建缓存，不阻塞用户请求
> 2. **实现**：`Executors.newFixedThreadPool(10)` 固定 10 线程
> 3. **注意**：生产环境不推荐 Executors 默认线程池（无界队列有风险），要用 ThreadPoolExecutor 自定义；不过这里演示够用

---

第一次封装工具，感觉和之前那种封装的感觉确实不一样：

**① 参数要考虑很多** —— 以前封装是"把重复代码抽出来"，这次是"把可变的点全部变成参数"：key 前缀、id、类型、查询方法、TTL 全部参数化，一个方法应对所有业务。

**② 使用泛型** —— `<T, ID>` 让工具类彻底通用。以前写死 Shop，现在传什么类型返回什么类型。

**③ 函数式作为参数** —— 之前只见过 `Function`、`Consumer` 在 Java 8 Stream 里用，这次发现**业务代码里也能用**：把"查数据库"这个动作当参数传，通过 `this::getById` 方法引用或 Lambda 传参。这是理解"把行为当数据传递"的关键一步，后面学设计模式、学函数式编程都会用到。

**本质思考：** 封装不是"代码少写了"，而是**把变化的部分交给调用者，把不变的部分留在工具里**。缓存流程（判空 → 查库 → 写缓存 → 锁）是不变的骨架，查哪个库、缓存多久是变化的细节——工具类留骨架，调用者填细节。

有点儿感觉了，记录第二十八天。
黑马点评 day11，今天把前面几天的缓存方案封装成了工具类。说实话第一次写这种"工具类 + 泛型 + 函数式参数"的代码，有点开窍的感觉。一直用着各种的工具类觉得方便，今天跟着视频写了写，才知道参数设计、泛型推断这些门道。
