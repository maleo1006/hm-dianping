---
tags:
  - 黑马点评
  - Redis
  - 全局唯一ID
  - 位运算
---
# Day12 - Redis 实现全局唯一ID

## 一、为什么要"全局唯一ID"？

**场景：** 用户抢购优惠券，每抢一次就生成一个订单，存到 `tb_voucher_order` 表。订单越来越多，id 从哪来？

**问题：如果用数据库自增 id：**
1. **id 规律太明显**：订单 id 是 1、2、3、4……别人通过订单号就能猜到你的订单量，不安全
2. **受单表数据量限制**：订单量大了要分表（拆成多张表），每张表都从 1 自增 → **id 就重复了**

> **全局唯一ID = 在分布式（多台机器/多张表）环境下，生成的 ID 不重复。**

## 二、全局唯一ID要满足什么特性？

| 特性 | 意思 | 大白话 |
|------|------|--------|
| **唯一性** | 全系统不重复 | 每个订单号只有一个 |
| **递增性** | 大致递增 | 后下的订单号比前一个大（方便排序） |
| **安全性** | 不能让人猜出规律 | 别是 1、2、3 这种连续的 |
| **高性能** | 生成要快 | 高并发下不能卡 |
| **高可用** | 不能挂 | 生成 id 的服务挂了系统就瘫了 |
本项目的选择：**用 Redis 来实现全局唯一ID**。
## 三、ID 的结构：时间戳 + 序列号

**ID 是一个 64 位的 long 类型数字（Java 的 long 就是 64 位），由三部分组成：**

```
0 - 00000000000000000000000000000000 00000000000000000000000000000000
符号位     时间戳（31 bit）               序列号（32 bit）
```

| 部分      | 位数     | 作用                                 |
| ------- | ------ | ---------------------------------- |
| **符号位** | 1 bit  | 永远是 0（保证数字是正数）                     |
| **时间戳** | 31 bit | 从开始时间到现在**经过的秒数**，够用 69 年          |
| **序列号** | 32 bit | 每秒内的计数器，每秒最多能生成 2^32（约 42 亿）个不同 ID |

> **什么是"时间戳"？**
>
> 时间戳就是"从某个固定时刻到现在过去了多少秒"。
> - 比如 2026年10月20日 12:00:00 的**绝对时间戳**是一个很大的数（约 17 亿秒，因为从 1970 年算起）
> - 但我们不直接用它，而是**自己定一个开始时间**（代码里的 `BEGIN_TIMESTAMP = 1792454400L`，就是 2026年10月20日 0 点），然后用 **现在的时间戳 - 开始时间戳**，得到一个小数
> - 这样时间戳部分占的位数少、能用 69 年，而且 **ID 里带时间 → 天然递增 → 满足"递增性"**
>
> **什么是"序列号"？**
>
> 同一秒内可能有无数个请求（高并发），时间戳都一样了，怎么区分？
> 用 **Redis 的 INCR 自增**：每来一个请求就 +1，得到 1、2、3、4……（`increment` 命令）。这就是"序列号"。
> **Redis 是单线程的，INCR 天然不会重复** → 满足"唯一性"。
>
> 时间戳负责"我是什么时候的"，序列号负责"这一秒里我是第几个"——两个拼起来，既唯一又递增。

## 四、核心代码逐行讲解

```java
@Component
public class RedisIdWorker {

    // 开始时间戳：2026-10-20 00:00:00 的秒数
    private static final long BEGIN_TIMESTAMP = 1792454400L;

    // 序列号的位数 = 32，因为 ID 里序列号占低 32 位
    private static final int COUNT_BITS = 32;

    private final StringRedisTemplate stringRedisTemplate;

    // 构造器注入（和 Day28 的 CacheClient 一样）
    public RedisIdWorker(StringRedisTemplate stringRedisTemplate) {
        this.stringRedisTemplate = stringRedisTemplate;
    }

    // keyPrefix：业务前缀，比如 order（订单）、voucher（优惠券）
    public long nextId(String keyPrefix) {
        // 1. 生成时间戳
        LocalDateTime now = LocalDateTime.now();
        long nowSecond = now.toEpochSecond(ZoneOffset.UTC);   // 现在的绝对时间戳（秒）
        long timeStamp = nowSecond - BEGIN_TIMESTAMP;          // 从开始时间到现在过了多少秒

        // 2. 生成序列号（Redis 自增长）
        // 2.1 获取当前日期（精确到天），比如 2026:10:20
        String date = now.format(DateTimeFormatter.ofPattern("yyyy:MM:dd"));
        // 2.2 自增长：key = "icr:order:2026:10:20"，每调一次 +1
        Long count = stringRedisTemplate.opsForValue().increment("icr:" + keyPrefix + ":" + date);

        // 3. 拼接时间戳和序列号并返回
        return timeStamp << COUNT_BITS | count;
    }
}
```

### 逐步拆解

**第 1 步：时间戳 = 现在 - 开始**

```java
long nowSecond = now.toEpochSecond(ZoneOffset.UTC);  // 现在的绝对秒数
long timeStamp = nowSecond - BEGIN_TIMESTAMP;        // 减去开始时间 → 一个小的秒数
```

> `toEpochSecond` = 把 LocalDateTime 转成秒数。`ZoneOffset.UTC` = 用 UTC 时区算（全球统一，避免时区差异）。
> 结果就是"从 2026-10-20 到现在过了多少秒"，这个数会**一直变大** → ID 递增。

**第 2 步：序列号 = Redis 自增**

```java
Long count = stringRedisTemplate.opsForValue().increment("icr:" + keyPrefix + ":" + date);
```

> `increment(key)` = Redis 的 **INCR 命令**：这个 key 不存在就设成 1，存在就 +1。
> **为什么 key 里带日期？** 每天一个 key：`icr:order:2026:10:20`、`icr:order:2026:10:21`……
> - 好处 ①：**方便统计当天订单量**（看看这个 key 涨到多少就知道今天多少单）
> - 好处 ②：每天从 1 开始，数字不会爆炸

**第 3 步：拼接 = 位运算（难点）**

```java
return timeStamp << COUNT_BITS | count;
```

> 这行是两个**位运算**，作用是"把时间戳放到高位、序列号放到低位，拼成一个 64 位数字"：
>
> **① `<<` 左移：把二进制位往左挪，右边补 0**
> ```java
> timeStamp << 32   // 时间戳往左移 32 位 → 时间戳占了高 32 位，低 32 位全是 0
> ```
> 比如 timeStamp = 1（二进制 `1`），左移 32 位后 = `1` 后面跟 32 个 0（相当于 2^32）。
>
> **② `|` 或运算：按位比较，只要有一个 1 结果就是 1**
> ```java
> 高32位(时间戳) | 低32位(序列号) → 拼成一个完整数字
> ```
> 左边低 32 位是 0，右边只有低 32 位有值 → 或运算 = **高 32 位放时间戳，低 32 位放序列号**。
>
> **举例：** timeStamp = 1，count = 2
> - `1 << 32` = 高32位是1，低32位全是0（值 = 2^32）
> - `| 2` = 把 2 填进低 32 位（值 = 2^32 + 2）
> - 结果就是一个 **64 位 long**：时间戳越大 ID 越大（递增），同一秒内 count 越大 ID 越大（唯一）

> 💡 **位运算为什么快？** 左移和或运算是 CPU 直接支持的操作，比字符串拼接（"1"+"2"）快得多，高并发下性能关键。

## 五、测试代码

```java
@Resource
private RedisIdWorker redisIdWorker;

// 线程池：最多 500 个线程同时干活
private ExecutorService es = Executors.newFixedThreadPool(500);

@Test
void testIdWorker() throws InterruptedException {
    // 倒计时锁：初始值 300，等它减到 0 才继续
    CountDownLatch latch = new CountDownLatch(300);

    // 任务：一个任务 = 循环 100 次生成 id
    Runnable task = () -> {
        for (int i = 0; i < 100; i++) {
            long id = redisIdWorker.nextId("order");
            System.out.println("id = " + id);
        }
        latch.countDown();   // 这个任务干完了，计数 -1
    };

    long begin = System.currentTimeMillis();
    // 提交 300 个任务到线程池（线程池用 500 线程并发执行）
    for (int i = 0; i < 300; i++) {
        es.submit(task);
    }
    // 主线程在这里等，直到 300 个任务全部 countDown 完（latch 变成 0）
    latch.await();
    long end = System.currentTimeMillis();
    System.out.println("time = " + (end - begin));
}
```

> 🤔 **这些概念是啥？**
>
> **① ExecutorService（线程池）**：自己手动开线程又慢又麻烦（`new Thread()` 用完就扔）。线程池 = 提前准备一堆线程，有任务就丢给空闲线程干，干完线程回收复用。`newFixedThreadPool(500)` = 固定 500 个线程。
>
> **② CountDownLatch（倒计时锁）**：让主线程"等所有任务干完再继续"。
> - `new CountDownLatch(300)` = 倒计时从 300 开始
> - 每个任务干完调 `latch.countDown()` = 计数减 1
> - 主线程 `latch.await()` = 阻塞等待，直到计数变 0
> - 为什么需要？因为主线程的 for 循环只负责"提交任务"，提交完任务还在线程池里跑，不加锁主线程就直接往下走了，测不到总时间
>
> **③ 为什么这么测？** 300 个任务 × 每个 100 次 = **3 万个 id 并发生成**。目的：
> - 验证**并发下 id 不重复**（唯一性）
> - 验证**生成速度**（性能）
>
> **测试结果：**
> - 没有重复 id
> - 1.481 秒生成 3 万个 id（每秒约 2 万+，Redis 单机轻松扛住）

## 六、四种全局唯一ID方案对比

| 方案                 | 特点                 | 优缺点                      |
| ------------------ | ------------------ | ------------------------ |
| **UUID**           | 十六进制一长串，返回 String  | 不单调递增、太长不友好，用的少          |
| **Redis 自增**（本项目）  | 时间戳 + 计数器，每天一个 key | 性能好、递增、好统计；依赖 Redis      |
| **雪花算法 Snowflake** | long 64 位，序列号用机器号  | 性能更好；**对时钟依赖高**（时钟回拨会重复） |
| **数据库自增**          | 单独一张表专门做自增，多张表共用   | 性能不如 Redis；可批量取 id 缓存    |

> **Redis 自增 ID 策略总结：**
> - **每天一个 key**，方便统计订单量
> - **ID 构造 = 时间戳 + 计数器**（Redis INCR 保证唯一，时间戳保证递增）

---

## ⚠️ 避坑 Tips

### 1. 时间戳的坑
- **`BEGIN_TIMESTAMP` 别乱设**：设太近（比如今天），时间戳很快会很大，69 年的寿命缩水；设太远，前期时间戳是负数
- **时区要统一**：`toEpochSecond(ZoneOffset.UTC)` 用 UTC，别一会 UTC 一会 +8，会差 8 小时
- **时间戳不是"现在的时间"**：是"现在 - 开始时间"的差值秒数，别搞混

### 2. 位运算的坑
- **`<< 32` 是左移 32 位不是乘以 32**：等价于乘 2^32
- **`|` 是或不是拼接字符串**：是按二进制位合并
- **序列号位数要和 `COUNT_BITS` 一致**：Redis 自增的 count 如果超过 32 位能表示的范围（2^32），会溢出，但每秒 42 亿个订单现实中不可能

### 3. 序列号的坑
- **key 必须带业务前缀和日期**：不带前缀，order 和 voucher 共用计数器会串；不带日期，key 无限膨胀
- **INCR 是原子的**：多线程并发也不会重复，这是唯一性的保障，不要自己用"读-改-写"实现

### 4. 测试的坑
- **`latch.await()` 一定要等**：不等就测不到真实耗时
- **线程池记得关闭**（生产环境）：测试里 `es.shutdown()` 可加可不加，生产要规范

---

## 🎯 面试题

### Q1：为什么要用全局唯一ID？数据库自增不行吗？

> **回答框架：**
> 1. **场景**：分布式系统/高并发下单，订单表数据量大会分表
> 2. **数据库自增的问题**：① id 规律明显，不安全 ② 分表后每个表都从 1 自增，id 重复
> 3. **全局唯一ID要求**：唯一、递增、安全、高性能、高可用
> 4. **方案选择**：Redis 自增（时间戳+计数器）/ UUID / 雪花算法 / 数据库自增，按需选

### Q2：Redis 生成全局唯一ID的原理？

> **回答框架：**
> 1. **结构**：64 位 long = 符号位(1) + 时间戳(31) + 序列号(32)
> 2. **时间戳**：现在秒数 - 开始秒数，保证递增，能用 69 年
> 3. **序列号**：Redis INCR 自增，单线程原子性保证唯一；**每天一个 key**（`icr:业务:日期`）方便统计
> 4. **拼接**：`timeStamp << 32 | count` 位运算合成，性能高
> 5. **测试**：3 万并发 id 无重复，1.481 秒生成完

### Q3：Redis 自增、UUID、雪花算法怎么选？

> **回答框架：**
> 1. **UUID**：无序、太长，基本不用
> 2. **Redis 自增**：简单、递增、可统计，依赖 Redis 可用性，够用且好实现
> 3. **雪花算法**：性能更好、不依赖网络，但对机器时钟敏感（回拨会重复），实现复杂
> 4. **项目选择**：黑马点评用 Redis 自增，因为项目本身就有 Redis，实现简单，还能统计订单量



唯一id干懵了啊，第二十九天。
黑马点评 day12，全局唯一ID，主要这些都没怎么见过，第一次接触位运算和并发测试。1.4 秒生成 3 万不重复 ID，还是挺有感觉的。"每天一个 key"这个太妙了我去，把数据按天切分，这样不但可以让单个 key 不超长，还方便统计数量。
