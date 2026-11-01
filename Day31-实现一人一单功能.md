---
tags:
  - 黑马点评
  - 一人一单
  - 悲观锁
  - 事务失效
---
# Day14 - 实现一人一单功能

> **多个概念连环套**：并发 + 锁粒度 + 字符串 intern + Spring 事务代理 + 集群。我按视频的思路：问题一步步暴露、方案一步步演进的顺序写的思路，最后整理。

## 一、需求：一人一单

**需求：修改秒杀业务，要求同一个优惠券，一个用户只能下一单。**

流程在原来的基础上加一步：判断库存充足后，**根据优惠券id和用户id查询订单**，订单已存在就报错。

**第一版实现（在减库存前加判断）：**

```java
// 一人一单
Long userId = UserHolder.getUser().getId();
// 查询订单：这个用户有没有买过这张券
int count = query().eq("user_id", userId).eq("voucher_id", voucherId).count();
// 判断是否存在
if (count > 0){
    return Result.fail("您已购买过该代金券!");
}
```

**测试结果：依然会出现一人多单的情况。**

> 为什么？**和超卖一模一样的并发问题**：先查询 → 查完再减库存、增订单。两个请求同时查到"都没买过" → 都往下走 → 一人两单。
>
> 本质：**"查询 + 插入"不是原子操作**

## 二、为什么这里不能用乐观锁？

**答：乐观锁是在"更新"时用的，这里是要"新增"（插入订单）。**

|       | 超卖                     | 一人一单                 |
| ----- | ---------------------- | -------------------- |
| 操作类型  | **更新**库存（stock - 1）    | **新增**订单             |
| 乐观锁依据 | 判断 stock 有没有被改过（有旧值可比） | 判断订单**存不存在**（压根没有旧值） |
| 用什么锁  | 乐观锁（条件扣减）              | **悲观锁**（保证串行查询+插入）   |

> 乐观锁的前提是"数据已存在、能比较旧值"。而一人一单是"检查记录是否存在 + 插入"，**不存在的东西没法比**，所以只能靠悲观锁让同一用户的请求串行执行——**一个人查完、插入完，下一个人才能查**。

## 三、悲观锁方案演进（4步，每步解决一个问题）

### 第 1 步：直接在方法上加 synchronized —— 锁粒度太大

把下单逻辑封装成方法（IDEA 快捷键 `Ctrl+Alt+M` 抽取），加上事务和锁：

```java
@Transactional
public synchronized Result creatVoucherOrder(Long voucherId) {}
```

> 问题：**所有用户共用一把锁**。用户 A 和用户 B 根本不冲突（不同人买同一张券是允许的），但 A 持锁时 B 也要干等 → 性能差。
>
> 我的理解：**业务只需要对同一个用户加锁**，不同用户应该拿不同的锁。

### 第 2 步：把锁放进方法体，按 userId 加锁 —— 锁粒度精确到用户

```java
// 同一个用户加同一把锁，不同用户加不同锁
Long userId = UserHolder.getUser().getId();
synchronized (userId.toString().intern()){
    return creatVoucherOrder(voucherId);
}
```

> **`userId.toString().intern()` 是什么意思？**
>
> **① `userId.toString()`**：Long 转成字符串。但注意——**每次调用都会 new 一个内容相同、对象不同的 String**。
> 比如用户 1010 两次请求，第一次 `toString()` 产生字符串对象 A（内容是"1010"），第二次产生对象 B（内容也是"1010"）。**A 和 B 不是同一个对象**。
>
> **② `.intern()`**：去**字符串常量池**里找。如果池里已有相同内容的字符串，**直接返回池里那个对象**；没有就把自己放进池子。
> - 第一次 "1010".intern() → 池里没有 → 放进去，返回对象 X
> - 第二次 "1010".intern() → 池里有 → **返回同一个对象 X** 
>
> **③ 为什么必须 intern？** `synchronized(对象)` 锁的是"对象本身"。如果不 intern，两次请求锁的是 A 和 B **两个不同的对象** → 等于没锁！intern 保证**相同 userId 永远拿到同一个对象** → 同一用户串行、不同用户（不同字符串）互不干扰。

### 第 3 步：发现问题 —— 锁先释放、事务后提交

> 问题：这样写是先释放锁，再提交事务。如果这时候别的线程进入，还是会出现并发安全问题。
>
> 场景：线程 A 在锁里执行完 creatVoucherOrder（插入订单），**锁释放了，但事务还没提交**（订单还没真正写进数据库）。线程 B 拿到锁，查订单 → **查不到**（A 的事务未提交，B 不可见）→ B 也插入 → **一人两单**。

**解决：把加锁放到调用外层，锁的范围包住整个事务方法调用：**

```java
Long userId = UserHolder.getUser().getId();
synchronized (userId.toString().intern()){
    return creatVoucherOrder(voucherId);   // 事务方法调用结束 = 事务已提交，才释放锁
}
```

### 第 4 步：事务失效！—— this 调用 vs 代理对象

> **最后！这样写事务是失效的！**
>
> 原因：`creatVoucherOrder(voucherId)` 是 **this 调用**——在同一个类里直接调自己的方法，**不是通过代理对象调用**，Spring 的 @Transactional 拦不到，事务不生效。

> **为什么同一个类里调用事务就失效？**
> Spring 的事务是**基于 AOP 代理**实现的：你注入的 service 实际是"代理对象"，调方法时代理先开事务、再调真实方法、返回后提交事务。
> 但 `this.creatVoucherOrder()` 是直接调真实对象的方法，**绕过了代理** → 代理的开事务/提交逻辑根本没执行 → 事务失效。

**解决：用 AopContext 拿到代理对象再调用**

```java
Long userId = UserHolder.getUser().getId();
synchronized (userId.toString().intern()){
    // 获取代理对象（事务）——必须是强转成接口
    IVoucherOrderService proxy = (IVoucherOrderService) AopContext.currentProxy();
    return proxy.creatVoucherOrder(voucherId);   // 走代理 → 事务生效
}
```

**配套配置：**

```xml
<dependency>
    <groupId>org.aspectj</groupId>
    <artifactId>aspectjweaver</artifactId>
</dependency>
```

```java
// 启动类上开启：暴露代理对象（否则 AopContext.currentProxy() 拿不到）
@EnableAspectJAutoProxy(exposeProxy = true)
```

>  **最终链条：** 锁包住"代理方法调用" → 代理方法执行完 = 事务已提交 → 才释放锁。三个坑全部解决：
> ① 锁粒度：只锁同一用户 
> ② 锁释放时机：事务提交后才释放 
> ③ 事务生效：走代理调用 

## 四、集群下，锁又失效了！

**单机（一个 JVM）加锁可以解决一人一单，但集群（多台服务器）下不行。**

**模拟集群：**
1. 把服务启动两份，端口 8081 和 8083（模拟两台服务器，两个 JVM）
2. 修改 nginx 的 `nginx.conf`，配置反向代理 + 负载均衡：

```
upstream backend {
    server 127.0.0.1:8081 max_fails=5 fail_timeout=10s weight=1;
    server 127.0.0.1:8083 max_fails=5 fail_timeout=10s weight=1;
}
```

3. 用户请求会在这两个节点间负载均衡，再次测试线程安全问题

**为什么锁失效？** synchronized 锁是 **JVM 内部**的（锁监视器在每个 JVM 里）。JVM1 里锁了用户 1010，但请求被 nginx 轮询到 JVM2 时——**JVM2 的锁监视器根本不知道 JVM1 锁过谁** → 同一用户又买了两单。

>  **结论：** 单机锁（synchronized / Lock）只能保证**单 JVM 内**的线程安全。集群/分布式环境下，需要一把**所有 JVM 共享的锁**——这就是后面要学的**分布式锁**。

---

## ⚠️ 避坑 Tips

### 1. 锁的坑
- **锁粒度宁小勿大**：synchronized 直接加方法 = 所有用户互相等，性能差。按 userId 加锁，只锁冲突的那部分人
- **synchronized(对象) 必须保证锁的是同一个对象**：Long 转 String 每次都是新对象，必须 `.intern()` 才保证相同 id 锁同一把
- **锁要包住整个事务方法**：先释放锁再提交事务 = 白锁（别人查不到你未提交的数据）

### 2. 事务的坑
- **同类内 this 调用 @Transactional 不生效**：事务靠 AOP 代理，this 调用绕过了代理
- **必须 `@EnableAspectJAutoProxy(exposeProxy = true)`**：不开这个，`AopContext.currentProxy()` 抛异常
- **`AopContext.currentProxy()` 要强转成接口类型**，不是实现类

### 3. 集群的坑
- **synchronized 是 JVM 级别的锁**：集群下每台机器一把锁，互相不知道，锁失效
- **测试集群**：启动两份服务（改端口）+ nginx upstream 轮询，同一用户打两单即复现
- 集群下的出路：**Redis 分布式锁**

---

## 🎯 面试题

### Q1：一人一单为什么用悲观锁不用乐观锁？

> **回答框架：**
> 1. **乐观锁适用场景**：更新已有数据时，判断旧值是否被改（CAS/版本号）
> 2. **一人一单是新增**：检查"订单记录是否存在 + 插入"，不存在的数据没法比较旧值
> 3. **所以用悲观锁**：synchronized 让同一用户的查询+插入串行执行

### Q2：userId.toString().intern() 是干什么的？不写行不行？

> **回答框架：**
> 1. **问题**：Long.toString() 每次产生新对象，synchronized 锁的是对象，两次请求锁不同对象 = 锁失效
> 2. **intern()**：去字符串常量池找相同内容，有就返回同一个对象 → 相同 userId 锁同一把锁
> 3. **不写不行**：不 intern 等于没锁，一人一单照样失效
> 4. **锁粒度**：锁的 key 是 userId，同一用户串行、不同用户并发

### Q3：事务为什么会失效？AopContext.currentProxy() 是干嘛的？

> **回答框架：**
> 1. **Spring 事务原理**：基于 AOP 代理，注入的 service 是代理对象，代理负责开事务/提交
> 2. **失效原因**：同类内 `this.creatVoucherOrder()` 直接调真实方法，绕过代理 → 事务不生效
> 3. **解决**：`AopContext.currentProxy()` 拿到代理对象再调用，走代理 → 事务生效
> 4. **配套**：需要 aspectjweaver 依赖 + `@EnableAspectJAutoProxy(exposeProxy = true)`

### Q4：集群下 synchronized 为什么失效？怎么解决？

> **回答框架：**
> 1. **synchronized 是 JVM 级锁**：锁监视器在每个 JVM 内部，跨 JVM 不认识
> 2. **集群场景**：nginx 负载均衡，同一用户请求打到不同服务器，各锁各的 → 一人多单
> 3. **解决**：用**分布式锁**，让所有 JVM 共享同一把锁（Redis setnx / Redisson），下一篇


绕！记录第三十一天，实现一人一单。
黑马点评 day14，今天是真迷糊啊，synchronized 从方法上挪到方法里、intern() 保证同一用户一把锁、锁包住事务、AopContext.currentProxy() 拿代理，每一步都根据暴露的问题进行解决。先是锁粒度太大，锁的对象是this（也就是整个service实现类），这样性能太差，所以又关键字，锁userId，锁粒度小了，但是事务还没提交就释放锁，造成并发安全问题，又调整锁范围，提交完事务再释放锁，自己能梳理下来了，可以。最后集群一测锁又废了，Nginx 轮询两台服务器同一用户买了两单。
