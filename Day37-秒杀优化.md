---
tags:
  - 黑马点评
  - 秒杀优化
  - Redis
  - Lua脚本
  - 阻塞队列
  - 异步下单
---
# Day17 - 秒杀优化（Redis + Lua 预减库存 + 阻塞队列异步下单）

> 承接 Day16：前几天的思路一直是"并发不安全就加锁"（乐观锁、分布式锁），但加锁等于排队，会牺牲并发。今天的思路换了个方向——**把"判断秒杀库存、校验一人一单"这两个高频校验前置到 Redis，用 Lua 脚本保证原子性；下单再改成异步**。核心一句话：**校验前置 + 下单异步**。

## 一、优化思路：从"同步串行"到"Redis 前置校验 + 异步下单"

**原流程：** 手机 → Nginx → Tomcat → MySQL。秒杀的 6 步业务（查询优惠券 → 判断秒杀库存 → 查询订单 → 校验一人一单 → 减库存 → 创建订单）**全部在 Tomcat 串行完成**，其中查询订单、减库存、创建订单每一步都要访问 MySQL。

**比喻（饭店优化前）：** 最开始小姐姐一个人干所有活——顾客进来，她点餐、她做饭、她上菜。一次只能服务一个顾客，来的人多了就堵在门口。

**问题本质：**
- 所有请求都打到数据库，数据库扛不住
- 校验（库存、一人一单）和写库（减库存、创建订单）都在请求线程里串行做，**单个请求耗时长，并发上不去**

**优化后：** 手机 → Nginx → 先到 **Redis**（判断秒杀库存、校验一人一单）→ 通过后 Tomcat 只做**查询优惠券**这种轻量操作，把「优惠券 id、用户 id、订单 id」存入**阻塞队列**，立刻返回订单 id；阻塞队列由**独立线程**消费，异步完成查询订单、减库存、创建订单 → MySQL。

**比喻（饭店优化后）：** 小姐姐只负责接待顾客、开小票，把单子交给后厨；后厨照着小票做菜，还要有号码牌知道先做谁的。接待的速度一下就上来了。

**核心变化：**
- 判断秒杀库存、校验一人一单从 MySQL 挪到 Redis，**通过 Lua 脚本保证原子性**
- 判断库存充足且用户没有下过单后，在 Redis 里**预减库存**
- 同步写数据库变成**异步操作**：缩短了秒杀请求的处理链路，**大幅提高并发，同时减轻数据库压力**
### Redis + Lua 完整流程

**Redis 里存两个 key：**
- `seckill:stock:{voucherId}`（库存）→ 100
- `seckill:order:{voucherId}`（已下单用户 set）→ 1，2，3，5，7，8

**Lua 脚本流程：** 判断库存是否充足 → 不足返回 1；充足再判断用户是否已下单 → 已下单返回 2；都没问题 → 扣减库存 + 把 userId 存入 set 集合 → 返回 0。

**业务侧流程：** 执行 Lua 脚本 → 判断结果是否为 0 → 是：把优惠券 id、用户 id、订单 id 存入阻塞队列 → 返回订单 id，异步下单；否：返回异常信息（库存不足 / 不能重复下单）。

**思考：**
**为什么要"预减库存"？**
**答：** 下单已经改成异步了，如果不在 Redis 里先减库存，就无法在"放行请求"这一刻实时知道库存还有没有——等异步线程再查库存，可能已经放出去成千上万个请求了。

预减库存的作用：
- **前置拦截**：库存不足的请求在 Redis 层直接被拦下（返回 1），根本到不了队列和数据库
- **保证不超卖**：Redis 库存是"秒杀窗口期内的权威计数器"，**放行成功的请求数 ≤ 库存数**
- **最终一致性**：真正的数据库扣减放在异步线程里做（`stock = stock - 1 and stock > 0`），失败就说明已经没了，日志记录即可
## 二、实现一：Lua 脚本完成校验 + 预减库存

### 1. 新增秒杀券时同步保存库存到 Redis

新增秒杀优惠券的请求进来，除了存数据库，还要把库存同步到 Redis：

```java
public void addSeckillVoucher(Voucher voucher) {
    // 1.保存优惠券到数据库
    save(voucher);
    // 2.保存库存到 Redis，key = seckill:stock:{voucherId}
    stringRedisTemplate.opsForValue().set(SECKILL_STOCK_KEY + voucher.getId(), voucher.getStock().toString());
}
```

> **注意：** 库存 key 必须和 Lua 脚本里拼的 key 完全一致（`seckill:stock:` 前缀），写错就 get 不到库存，脚本会出问题（下面避坑区细说）。

### 2. Lua 脚本

把「判断库存 + 判断一人一单 + 扣库存 + 记录用户」四步**合成一个原子操作**，Redis 执行脚本期间不会插入其他命令：

```lua
-- 1.参数列表
-- 1.1.优惠券id
local voucherId = ARGV[1]
-- 1.2.用户id
local userId = ARGV[2]

-- 2.数据key
-- 2.1.库存key
local stockKey = 'seckill:stock:' .. voucherId
-- 2.2.订单key
local orderKey = 'seckill:order:' .. voucherId

-- 3.脚本业务
-- 3.1.判断库存是否充足 get stockKey
if(tonumber(redis.call('get', stockKey)) <= 0) then
    -- 3.2.库存不足，返回1
    return 1
end
-- 3.2.判断用户是否下单 sismember orderKey userId
if(redis.call('sismember', orderKey, userId) == 1) then
    -- 3.3.存在，说明是重复下单，返回2
    return 2
end
-- 3.3.扣库存 incrby stockKey -1
redis.call('incrby', stockKey, -1)
-- 3.4.下单 sadd orderKey userId
redis.call('sadd', orderKey, userId)
return 0
```

> **为什么这个脚本能替代加锁？** 因为 Redis 执行 Lua 脚本是**原子**的——判断 + 扣减 + 记录一气呵成，多个请求串行执行脚本，不存在"先判断后扣减"之间的空档，超卖和一人多单在源头就被堵死了。

### 3. 写脚本踩的坑（自测记录）

把阻塞队列以外的部分写完测试，发现 Lua 脚本错了好几处：
- `if(tonumber(redis.call('get', stockKey) <= 0)` —— **漏了个 ")"**，正确写法：`if(tonumber(redis.call('get', stockKey)) <= 0)`
- 前缀 `seckill:` 写成了 `seckil:`，拼错 key 导致取不到库存
- `return 2` 连到了一块写成了 `return2`，语法错误
## 三、实现二：阻塞队列 + 线程池异步下单

### 1. 为什么用阻塞队列？

**订单是"有人下单才有，没人下单就没有"**——消费线程不知道什么时候会有订单。用**阻塞队列**：队列空了 `take()` 就阻塞等待，有订单放进来立刻唤醒消费，不会空转浪费 CPU。

### 2. 线程池 + 消费线程（VoucherOrderHandler）

```java
private static final ExecutorService SECKILL_ORDER_EXECUTOR = Executors.newSingleThreadExecutor();

@PostConstruct
private void init() {
    SECKILL_ORDER_EXECUTOR.submit(new VoucherOrderHandler());
}

private class VoucherOrderHandler implements Runnable {
    @Override
    public void run() {
        while (true) {
            try {
                // 1.获取队列中的订单信息
                VoucherOrder voucherOrder = orderTasks.take();
                // 2.创建订单
                handleVoucherOrder(voucherOrder);
            } catch (Exception e) {
                log.error("创建订单异常", e);
            }
        }
    }
}
```

**流程：** 创建单线程线程池 → `@PostConstruct` 在类初始化（Bean 创建、依赖注入完成后）时自动执行一次 → 提交消费任务 → 消费线程死循环 `take()` 阻塞等待 → 拿到订单信息就调用 `handleVoucherOrder` 创建订单。

> **`@PostConstruct` 的作用：** 保证在**第一个请求到来之前**线程池就已经就绪，随时准备取订单下单。不用它的话，就得等第一次请求时才初始化，可能把第一个订单拖慢。

### 3. seckillVoucher 完整流程（我的理解）

```java
@Override
public Result seckillVoucher(Long voucherId) {
    Long userId = UserHolder.getUser().getId();
    // 1.执行lua脚本
    Long result = stringRedisTemplate.execute(
            SECKILL_SCRIPT, Collections.emptyList(), voucherId.toString(), userId.toString()
    );
    // 2.判断结果是否为0
    int r = result.intValue();
    if (r != 0){
        // 2.1不为0，没有下单资格
        return Result.fail(r == 1 ? "库存不足!" : "不能重复下单!");
    }
    // 2.2为0，有购买资格，把下单信息保存到阻塞队列
    VoucherOrder voucherOrder = new VoucherOrder();
    // 2.3订单id
    long orderId = redisIdWorker.nextId("order");
    voucherOrder.setId(orderId);
    // 2.4用户id
    voucherOrder.setUserId(userId);
    // 2.5代金券id
    voucherOrder.setVoucherId(voucherId);
    // 2.6放入阻塞队列
    orderTasks.add(voucherOrder);
    // 3.获取代理对象（事务）
    proxy = (IVoucherOrderService) AopContext.currentProxy();
    return Result.ok(orderId);
}
```

**我的理解：** 执行 seckillVoucher → 通过 Lua 脚本判断能否下单 → 有资格就把下单信息保存到阻塞队列 → 阻塞队列这边先写好一个单线程线程池，内部类实现 Runnable 执行任务，获取队列信息、创建订单 → `@PostConstruct` 在类加载时把线程池准备好，随时获取订单信息并下单。

**答疑①：原始写法"获取代理对象"为什么不生效？**

因为 `AopContext.currentProxy()` 拿代理对象是**依赖线程的**——Spring 只有在"通过代理对象进入的方法"里才会往当前线程的 ThreadLocal 里放代理上下文。`seckillVoucher` 是被 Controller 调用、经过 Spring 代理进入的，所以**主线程（请求线程）里能取到**；而线程池里的子线程是直接执行 `Runnable`，根本没经过 Spring 代理调用，**在子线程里取 `currentProxy()` 是取不到的（返回 null）**。

所以必须**在主线程（seckillVoucher）里把代理对象取好，存到成员变量 proxy**，子线程再用。取不到代理 → 无法通过代理调用事务方法 → `@Transactional` 失效。

> 补充：成员变量 proxy 是所有请求线程共享的，但无论哪个线程写，写进去的都是同一个 Spring 单例代理对象，所以不存在并发问题，官方代码也是这么写的。

**答疑②：更稳妥的写法？**

先取代理、再放队列更保险：`proxy = (IVoucherOrderService) AopContext.currentProxy();` 放在 `orderTasks.add(...)` **之前**，避免极端情况下消费线程已经把订单取走、而 proxy 还没来得及赋值（NPE）。

### 4. handleVoucherOrder：加锁 + 调用代理创建订单

```java
private void handleVoucherOrder(VoucherOrder voucherOrder) {
    Long userId = voucherOrder.getUserId();
    // 创建锁对象
    RLock lock = redissonClient.getLock("lock:order:" + userId);
    // 获取锁
    boolean isLock = lock.tryLock();
    if (!isLock){
        log.error("不允许重复下单");
        return;
    }
    try {
        proxy.creatVoucherOrder(voucherOrder);
    } finally {
        // 释放锁
        lock.unlock();
    }
}
```

**几乎不会发生并发安全问题**。因为"一人一单"的校验已经在 Lua 脚本里**原子地完成**了——重复用户会在 `sismember` 判断时返回 2 被拦截，根本走不到异步下单这一步，Redis 的 set 就是秒杀窗口内"一人一单"的权威。异步线程里再查一次数据库只是**双保险**。加锁（按用户维度）几乎永远不会冲突，属于防御性写法，**加上无妨、不加也安全**。

### 5. creatVoucherOrder：异步下单

```java
@Transactional
public void creatVoucherOrder(VoucherOrder voucherOrder) {
    // 一人一单
    Long userId = voucherOrder.getUserId();
    // 查询订单
    int count = query().eq("user_id", userId).eq("voucher_id", voucherOrder.getVoucherId()).count();
    // 判断是否存在
    if (count > 0){
        log.error("用户已经购买过一次");
        return;
    }
    // 减库存
    boolean success = seckillVoucherService.update()
            .setSql("stock = stock - 1") // set stock = stock - 1
            .eq("voucher_id", voucherOrder.getVoucherId()) // where voucher_id = ? and stock > 0
            .gt("stock", 0)
            .update();
    if (!success){
        // 扣减失败
        log.error("库存不足");
        return;
    }
    // 创建订单
    save(voucherOrder);
}
```

- **返回类型 Result → void**：新开异步线程，不需要给前端返回任何东西（前端早就在 `seckillVoucher` 里拿到订单 id 了），失败只记日志
- **入参 Long voucherId → VoucherOrder**：子线程拿不到 `UserHolder`（ThreadLocal 是线程绑定的，登录用户信息只在请求线程里），所以用户 id 必须随订单对象一起传进来

测试完成，没有报错、没有问题。

## 四、基于阻塞队列的异步秒杀存在的问题

**优化思路回顾：** ① 先利用 Redis 完成库存余量、一人一单判断，完成抢单业务；② 再将下单业务放入阻塞队列，利用独立线程异步下单。

**但"同步变异步"之后，JDK 阻塞队列本身暴露了两个问题：**

### 1. 内存限制问题

- 阻塞队列用的是 **JDK 自带的、基于 JVM 内存**的队列（`ArrayBlockingQueue`）
- 队列里数据越来越多，**可能内存溢出（OOM）**
- 所以必须传一个队列容量，我们传了 `1024 * 1024`——**但这只是把"内存溢出"推迟，并没有消除**：容量固定了，队列满了之后新的下单任务放不进去（`add()` 直接抛异常），一样是问题

### 2. 数据安全问题

- **服务宕机，信息全部丢失**：队列数据保存在 JVM 内存里，服务一宕机队列就没了 → Redis 里已经预减库存、但订单还没来得及创建 → **数据不一致**
- **任务取出后执行异常，任务永久丢失**：消费线程 `take()` 把任务**从队列里取出来**，如果执行 `handleVoucherOrder` 时出异常，任务已经不在队列里了，以后**再也不会被执行** → 订单丢失，**数据不一致**
---
## ⚠️ 避坑 Tips

### 1. 事务与代理的坑
- **AopContext.currentProxy() 是线程绑定的**：只能在"经过 Spring 代理进入的方法"所在线程取到；子线程（线程池）里取不到 → 必须在主线程取好存成员变量
- **事务必须走代理**：`creatVoucherOrder` 带 `@Transactional`，直接 this 调用会事务失效（Day14 的坑）
- **子线程没有 UserHolder**：异步线程里拿不到登录用户，用户 id 必须从 `VoucherOrder` 对象里取
- **先取代理、再入队**：把 `proxy = AopContext.currentProxy()` 放到 `orderTasks.add()` 之前，避免消费线程先执行而 proxy 还没赋值

---

## 🎯 面试题
### Q1：秒杀业务的优化思路是什么？

> **回答框架：**
> 1. **问题**：原流程 6 步全在 Tomcat 串行做，查询订单、减库存、创建订单都打 MySQL，请求慢、数据库扛不住，并发上不去
> 2. **校验前置**：把"判断秒杀库存、校验一人一单"挪到 Redis，用 Lua 脚本原子完成（判断 + 扣库存 + 记用户一气呵成），库存不足/重复下单在 Redis 层直接拦截
> 3. **下单异步**：抢购成功的用户，把订单信息放入阻塞队列，立刻返回订单 id；独立线程消费队列异步下单（查订单 → 减库存 → 创建订单）
> 4. **收益**：缩短请求链路，提高并发；只有成功抢购的请求才写数据库，减轻数据库压力
### Q2：为什么要用 Redis + Lua 脚本？为什么用户 id 用 set 保存？

> **回答框架：**
> 1. **为什么 Lua**：判断库存 + 判断一人一单 + 扣库存 + 记录用户要保证"要么全做、要么全不做"（原子性）。Redis 执行 Lua 脚本期间不插入其他命令，天然原子；用 Java 分步做会留出并发空档（Day13 超卖、Day14 一人多单就是这么来的）
> 2. **为什么 Redis 前置**：校验从数据库挪到内存，速度快、扛得住高并发
> 3. **为什么 set 存用户 id**：需求是"可存多值 + 不可重复"，set 天然去重，`sismember` O(1) 判断，正好做一人一单
> 4. **两个 key**：`seckill:stock:{id}` 存库存、`seckill:order:{id}` 存已下单用户集合
### Q3：为什么要预减库存？Redis 库存和数据库库存怎么保持一致？

> **回答框架：**
> 1. **为什么预减**：下单异步了，必须先在 Redis 扣库存，才能在"放行请求"这一刻实时拦截库存不足的请求；Redis 库存是秒杀窗口期的权威计数器，放行数 ≤ 库存数，从源头保证不超卖
> 2. **怎么一致**：Redis 预减负责"放不放人"，数据库在异步线程里做真正的扣减（`stock = stock - 1` 且 `stock > 0`，条件更新兜底）；数据库扣减失败说明库存已尽，记日志即可
> 3. **取舍**：秒杀窗口内以 Redis 为准（快），最终以数据库为准（准），两者是"最终一致"而不是"强一致"
### Q4：基于阻塞队列的异步秒杀存在哪些问题？怎么解决？

> **回答框架：**
> 1. **内存限制问题**：JDK 阻塞队列基于 JVM 内存，容量固定（1024*1024），队列满放不进去，数据多会 OOM
> 2. **数据安全问题**：① 基于内存，服务宕机队列数据全丢 → 已预减库存但订单没创建，数据不一致；② 任务被 take 出队后执行异常，任务永久丢失、不再执行，订单丢失
> 3. **解决思路**：用具备"持久化 + 可靠消费 + 消费者组"的消息队列替代 JDK 队列，比如 Redis 的 Stream / 专业的 MQ


记录第三十七天，秒杀优化。
黑马点评 day17。今天把秒杀从数据库串行判断优化成Redis 前置校验 + 异步下单，库存和一人一单的判断挪到 Redis，用 Lua 脚本保证原子性，抢购成功直接返回订单 id，下单交给阻塞队列 + 独立线程异步执行。Lua 脚本自己写的时候测试发现错了好多，漏括号、seckil 拼错、return 连写，debug 完通了。代理对象要在主线程获取，AopContext 是线程绑定的，子线程取不到。
