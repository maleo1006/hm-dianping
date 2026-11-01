---
tags:
  - 黑马点评
  - Redis
  - Java客户端
---
# Day04 - Jedis（了解）、SpringDataRedis

## 一、Jedis 入门（了解即可）

Java 操作 Redis 有很多客户端，Jedis 是最经典的一个，**方法名和 Redis 命令一模一样**，学过命令就能直接上手。

### 1. 使用基本步骤

1. **引入依赖**（jedis 坐标）
2. **创建 Jedis 对象，建立连接**
3. **使用 Jedis**，方法名与 Redis 命令一致（`jedis.set()`、`jedis.get()`）
4. **释放资源**（用完要归还/关闭）

### 2. Jedis 连接池（为什么必须用？）

**Jedis 本身是线程不安全的**，而且频繁创建和销毁连接有性能损耗，所以推荐用**连接池**代替直连。

>  **疑问：为什么线程不安全？**
>
> 一个 Jedis 对象内部持有一个 **Socket 连接**。如果多个线程共享同一个 Jedis 实例并发发命令，连接上的请求和响应会**互相错乱**（线程A发的 SET 可能收到线程B的响应）。所以 Jedis 实例**不能跨线程共享**，每个线程要用自己的实例。
>
> 连接池解决的就是这个问题：池子里放着多个 Jedis 实例，**每次 `getResource()` 借一个"私有"的实例给当前线程用**，用完 `close()` 还回池子（注意：连接池里 close 不是销毁，是归还）。每个线程拿到的都是独立的连接，线程安全 + 避免频繁创建销毁。

## 二、SpringDataRedis

### 1. 是什么

**SpringData 是 Spring 中数据操作的模块**，包含对各种数据库的集成，其中对 Redis 的集成模块就叫 **SpringDataRedis**。

**核心功能：**

| 功能 | 说明 |
|------|------|
| 整合客户端 | 提供对 Lettuce 和 Jedis 两种客户端的整合 |
| 统一 API | 提供 **RedisTemplate** 统一 API 操作 Redis |
| 发布订阅 | 支持 Redis 发布订阅模型 |
| 集群支持 | 支持 Redis 哨兵和 Redis 集群 |
| 响应式 | 支持基于 Lettuce 的响应式编程 |
| 序列化 | 支持基于 JDK、JSON、字符串、Spring 对象的序列化/反序列化 |

> RedisTemplate 就是"把操作 Redis 封装成了 Java 方法"：`opsForValue()`、`opsForHash()`、`opsForList()` `opsForSet` `opsForZSet` 对应五大数据类型。

### 2. 使用步骤

1. **引入依赖**：redis 起步依赖 + 连接池依赖（commons-pool2）
2. **配置文件**：application.yml 配 host、port、password、database
3. **注入 RedisTemplate**（Spring 自动装配）
4. **使用**：`redisTemplate.opsForXxx()...`

## 三、序列化方式（今天的重头戏）

### 1. 默认 JDK 序列化 —— 坑

RedisTemplate 可以接收任意 Object 作为值写入 Redis，写入前会把 Object 序列化为**字节形式**，**默认是 JDK 序列化**。存进去之后 Redis 里看到的是这样的：

```
\xAC\xED\x00\x05t\x00\x04name
```

**JDK 序列化的两个缺点：**

1. **可读性差**：全是乱码字节，人看不懂
2. **内存占用较大**：序列化结果很冗余

所以默认的 JDK 序列化基本不能用，要换序列化器。

> 💡 **key 为什么一定要换 String 序列化？** 字符串转字节直接 `getBytes()` 就行，而 key 和 hashKey 基本都是字符串类型，所以 **key 一律用 String 序列化器**（可读、省内存）。

### 2. 方案一：自定义 RedisTemplate + JSON 序列化

值是对象的话，一般用 **GenericJackson2JsonRedisSerializer**（自动序列化）：

- 存对象 → 自动序列化成 JSON 字符串
- 取出来 → 自动反序列化回对象

```java
@Configuration
public class RedisConfig {
    @Bean
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory factory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(factory);
        // key 用 String 序列化
        template.setKeySerializer(new StringRedisSerializer());
        // value 用 JSON 序列化（存对象自动变 JSON，取出来自动变对象）
        template.setValueSerializer(new GenericJackson2JsonRedisSerializer());
        // hash 的 key 和 value 同理
        template.setHashKeySerializer(new StringRedisSerializer());
        template.setHashValueSerializer(new GenericJackson2JsonRedisSerializer());
        return template;
    }
}
```

**但是有坑！** 看存进去的效果：

```
{
  "@class": "com.heima.redis.pojo.User",   ← 多出来的！
  "name": "虎哥",
  "age": 21
}
```

**为什么多了 `@class`？** 为了在反序列化时知道对象的类型，JSON 序列化器会把类的 class 类型**写进 JSON 结果**存入 Redis → **带来额外内存开销**。存一个对象就要带一个 `@class` 全路径。

### 3. 方案二：StringRedisTemplate（推荐）

为了节省内存空间，不用 JSON 序列化器处理 value，而是**统一用 String 序列化器**，key 和 value 都**只能存 String 类型**。

**那对象怎么办？手动序列化/反序列化：**

- 存：对象 → 手动转成 JSON 字符串 → `redisTemplate.opsForValue().set("user", jsonStr)`
- 取：`redisTemplate.opsForValue().get("user")` → 拿到 JSON 字符串 → 手动转回对象

```java
// 手动序列化（对象 → JSON 字符串）
String jsonStr = JSON.toJSONString(user);
stringRedisTemplate.opsForValue().set("user:100", jsonStr);

// 手动反序列化（JSON 字符串 → 对象）
String json = stringRedisTemplate.opsForValue().get("user:100");
User user = JSON.parseObject(json, User.class);
```

**为什么 Spring 直接提供了 StringRedisTemplate？** 因为用 RedisTemplate 还要自己配置 String 序列化器太麻烦，Spring 直接内置了一个 **key 和 value 序列化方式默认就是 String 的模板类**。

### 4. 两种方案怎么选？

| | 方案一：自定义 RedisTemplate + JSON | 方案二：StringRedisTemplate |
|---|------|------|
| 做法 | 改序列化器为 GenericJackson2JsonRedisSerializer | 手动序列化/反序列化 |
| 存对象 | 自动（省事） | 手动（多写两行代码） |
| 内存 | **存 @class 全路径，占内存** | **不存 @class，省内存** |
| 可读性 | 好（JSON） | 好（JSON） |
| 适用 | 简单场景图省事 | **生产更推荐** |

**根据需求选择**：想省事用方案一，要省内存/生产环境用方案二。

## 四、Hash 的 Java 操作（StringRedisTemplate）

Java 里的操作和命令不一样，**更像 HashMap**：`put`、`entries` 等。

```java
@Test
void testHash() {
    // HSET：put(key, hashKey, value)
    stringRedisTemplate.opsForHash().put("user:400", "name", "虎哥");
    stringRedisTemplate.opsForHash().put("user:400", "age", "21");

    // HGETALL：entries() 返回整个 Map
    Map<Object, Object> entries = stringRedisTemplate.opsForHash().entries("user:400");
    System.out.println("entries = " + entries);
}
```

> 其余类型（opsForList / opsForSet / opsForZSet）用法同理，后面实战里**用到干中学吧**。
---
## ⚠️ 避坑 Tips

### 1. Jedis 的坑
- **Jedis 线程不安全**：不要一个实例多线程共享（Socket 连接会错乱），用连接池 `getResource()` 借实例
- **连接池的 close() 是归还不是销毁**：从池子借的实例用完要 close()，不还会导致连接泄漏
- **maxTotal 别开太大**：Redis 本身有连接上限，8 个就够用，开太大浪费资源还容易把 Redis 拖垮
### 2. 序列化的坑
- **RedisTemplate 默认 JDK 序列化**：不换序列化器，Redis 里全是 `\xAC\xED` 乱码，还占内存
- **key 一定要用 String 序列化器**：key 基本都是字符串，`getBytes()` 就行，别让 key 也走 JDK 序列化
- **GenericJackson2JsonRedisSerializer 会存 @class**：反序列化需要知道类型，全路径写进 JSON，**内存开销 + 类路径改了旧数据就反序列化失败**
- **StringRedisTemplate 存对象必须手动 JSON**：忘了序列化会直接把对象 `toString()` 存进去，取出来就是废数据
- **改了实体类字段，Redis 里的旧 JSON 可能解析失败**：JSON 反序列化对字段名敏感，实体改名要兼容旧数据
### 3. 依赖的坑
- **用连接池别忘了 commons-pool2 依赖**：Lettuce 的池化依赖它，缺了配置不生效
- **SpringBoot 版本和 Redis 版本要匹配**：高版本 starter 默认 Lettuce，别混用

---

## 🎯 面试题

### Q1：Jedis 和 SpringDataRedis 有什么区别？

> **回答框架：**
> 1. **Jedis**：独立的 Redis Java 客户端，方法名和命令一致，**线程不安全**，一般配连接池用
> 2. **SpringDataRedis**：Spring 官方对 Redis 的集成，封装了 **RedisTemplate 统一 API**
> 3. **底层**：SpringDataRedis 底层可以整合 Jedis 或 Lettuce（高版本默认 Lettuce，线程安全、支持响应式）
> 4. **结论**：Spring 项目用 SpringDataRedis（RedisTemplate），省去自己管理连接；非 Spring 场景或想轻量可以用 Jedis

### Q2：为什么 Jedis 线程不安全？连接池怎么解决？

> **回答框架：**
> 1. **原因**：Jedis 实例内部持有 Socket 连接，多线程共享一个实例并发发命令，请求响应会错乱
> 2. **解决**：连接池（JedisPool），`getResource()` 每次借一个独立实例给当前线程
> 3. **归还**：用完 close() 归还池子，不是销毁连接，避免频繁创建销毁的性能损耗
> 4. **配置**：maxTotal（最大连接数）、maxIdle（最大空闲）、maxWaitMillis（等待超时）

### Q3：RedisTemplate 默认的序列化方式是什么？有什么问题？

> **回答框架：**
> 1. **默认 JDK 序列化**：对象序列化成字节存 Redis
> 2. **问题**：① 可读性差，Redis 里全是 `\xAC\xED` 乱码 ② 内存占用大
> 3. **解决**：key 用 StringRedisSerializer，value 用 GenericJackson2JsonRedisSerializer 或手动 JSON

### Q4：RedisTemplate 和 StringRedisTemplate 区别？选哪个？

> **回答框架：**
> 1. **序列化不同**：RedisTemplate 默认 JDK 序列化（可自定义）；StringRedisTemplate key/value 默认就是 String 序列化
> 2. **数据格式不同**：RedisTemplate 存的是二进制（JDK 序列化），StringRedisTemplate 存的是字符串
> 3. **存对象**：StringRedisTemplate 只能存 String，存对象要手动 JSON 序列化/反序列化
> 4. **推荐**：生产环境更推荐 StringRedisTemplate——省内存（不带 @class）、可读性好，多写两行 JSON 转换的代码很值得

记录第二十一天，点评前置day4。明天实战！
今天学了Java客户端操作Redis，对序列化的理解比前面苍穹外卖里学的时候更清楚了。字符串序列化只需要 getBytes ，所以 spring 直接内置了 StringRedisSerializer。值的话如果存对象，用Generic的那个序列化可以自动序列化反序列化，但是会多存一个识别地址，占内存。所以又推荐值也用字符串序列化，并手动进行序列化反序列化。明天开始实战！