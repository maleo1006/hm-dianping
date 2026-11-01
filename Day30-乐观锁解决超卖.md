---
tags:
  - 黑马点评
  - 乐观锁
  - 超卖
---
# Day13 - 乐观锁解决超卖

## 一、添加优惠券

接口业务是已经写好的，写个测试接口发请求添加一个秒杀券就可以了。

> 我添加完后数据库有了，但是页面没有。排查发现：
> 1. **时间问题**：结束时间不能超过当前时间，改了改就可以了
> 2. **店铺id问题**：还没显示可能是店铺id写错了，写成其他店了

## 二、简单的下单业务

下单时需要判断两点：
1. 秒杀是否开始或结束（尚未开始 / 已经结束都无法下单）
2. 库存是否充足（不足则无法下单）

**最简单的写法（肯定有并发问题）：**

```java
@Transactional
@Override
public Result seckillVoucher(Long voucherId) {
    // 1. 查询优惠券
    SeckillVoucher seckillVoucher = seckillVoucherService.getById(voucherId);
    // 2. 尚未开始
    if (seckillVoucher.getBeginTime().isAfter(LocalDateTime.now())){
        return Result.fail("秒杀尚未开始!");
    }
    // 3. 已结束
    if (seckillVoucher.getEndTime().isBefore(LocalDateTime.now())){
        return Result.fail("秒杀已结束!");
    }
    // 4. 判断优惠券库存
    if (seckillVoucher.getStock() < 1){
        return Result.fail("库存不足!");
    }
    // 5. 减库存（新写法：.setSql .eq .update）
    boolean success = seckillVoucherService.update()
            .setSql("stock = stock - 1")
            .eq("voucher_id", voucherId).update();
    if (!success){
        // 扣减失败
        return Result.fail("库存不足!");
    }
    // 6. 创建订单
    VoucherOrder voucherOrder = new VoucherOrder();
    long orderId = redisIdWorker.nextId("order");   // 用 Day12 的全局唯一id
    voucherOrder.setId(orderId);
    Long userId = UserHolder.getUser().getId();
    voucherOrder.setUserId(userId);
    voucherOrder.setVoucherId(voucherId);
    save(voucherOrder);
    return Result.ok(orderId);
}
```

> **新学的写法：LambdaUpdateWrapper 链式更新**
> `seckillVoucherService.update().setSql("stock = stock - 1").eq("voucher_id", voucherId).update()`
> - `setSql`：直接写 SQL 片段做更新（`stock = stock - 1` 是在数据库里减，不是查出来减）
> - `eq`：where 条件
> - `.update()`：执行

## 三、并发测试，发现问题：超卖！

**测试方式：** 我没有安装 JMeter，用的是 **Apifox 的自动化测试**做性能测试。

**测试配置：**
- 接口：`POST http://localhost:8081/voucher-order/seckill/10`
- 并发用户数：**40 个线程**（太多容易卡）
- 库存：**20 个**

**注意：要在请求头上加上Authorization,参数值是生成的token的值**

**测试结果：**
- 总请求 1787 次，请求失败率 0.00%（接口层面没报错）
- 但看数据库——**超卖了！**

**库存变成了 -9，生成了 29 条订单数据。**

### 超卖是怎么发生的？

**根本原因：检查库存和扣减库存不是原子操作（两个动作之间有间隔）**

```
线程1: 查询库存 → 库存=1 > 0 → 准备扣减
线程2: 查询库存 → 库存=1 > 0 → 准备扣减   ← 线程1还没扣完！
线程1: 扣减 → 库存=0
线程2: 扣减 → 库存=-1                     ← 超卖！
```

> 每个线程都先"查库存"再"减库存"，两步之间别的线程插进来了。**多线程并发下，判断和操作不是一气呵成的**，就会重复扣减。

## 四、解决方案：加锁

超卖是典型的**多线程安全问题**，常见解决方案就是**加锁**，分两类：

| | 悲观锁 | 乐观锁 |
|---|--------|--------|
| **思想** | 认为线程安全问题**一定会发生**，操作数据前先获取锁，确保线程**串行执行** | 认为线程安全问题**不一定发生**，不加锁，**更新时判断**有没有其它线程改过数据 |
| **实现** | Synchronized、Lock | CAS法、版本号法 |
| **优点** | 简单粗暴 | 性能好 |
| **缺点** | 性能一般 | 存在**成功率低**的问题 |

> 本项目用**乐观锁**（秒杀场景追求性能）。

### 乐观锁的两种实现方式

**① CAS法（Compare And Swap，比较并交换）**

**核心：更新时带上"我查到的旧值"作为条件，如果值变了说明别人改过了，更新失败。**

```java
// 修改前：无条件减库存
seckillVoucherService.update()
        .setSql("stock = stock - 1")
        .eq("voucher_id", voucherId)
        .update();

// 修改后：CAS法，where 里带上查到的 stock 值
seckillVoucherService.update()
        .setSql("stock = stock - 1")
        .eq("voucher_id", voucherId).eq("stock", seckillVoucher.getStock())
        .update();
```

> 两个线程都查到 stock=1 → 线程1更新成功（stock=0）→ 线程2再更新时 `stock=1` 的条件不满足了（现在是0）→ **更新失败，0 行受影响** → 防止超卖。

**② 版本号法**

**核心：表里加一个 version 字段，每次更新 version+1，where 带上查到的 version。**

```sql
update tb_seckill_voucher
set stock = stock - 1, version = version + 1
where voucher_id = 10 and version = 1;   -- 查到的版本号
```

> 线程2 的 version 还是旧的 1，但表里已经是 2 了 → 更新失败。

### CAS 的坑：成功率太低了！

```java
// 用 eq("stock", seckillVoucher.getStock()) 测试后
// 发现成功率太低 —— "乐观锁怎么还悲观上了"
```

> **为什么成功率低？** 高并发下 40 个线程同时读库存，大家都读到同一个值（比如 20）。第一个线程扣成 19，剩下 39 个线程再用 `stock=20` 去更新——**全部失败**！一次秒杀可能只成功 1 个人，剩下全被拒，这比超卖还难受。

**优化：把条件从"等于"改成"大于0"**

```java
seckillVoucherService.update()
        .setSql("stock = stock - 1")
        .eq("voucher_id", voucherId).gt("stock", 0)
        .update();
```

> **只要库存 > 0 就能扣**，不用管具体是多少。这样：
> - 不会超卖（库存 0 了 `stock > 0` 不成立，更新失败）
> - 成功率大幅提升（不用等别人扣完，只要还有货就能抢）

>  **这就是超卖问题的终极答案：把"先查再扣"变成"条件扣减"**——用 SQL 的 where 条件（`stock > 0`）把判断和扣减合并成一个原子操作，从根上消灭竞态。

## 五、总结

**超卖这样的线程安全问题，解决方案：**

1. **悲观锁**：添加同步锁，让线程串行执行
   - 优点：简单粗暴
   - 缺点：性能一般
2. **乐观锁**：不加锁，在更新时判断是否有其它线程在修改
   - 优点：性能好
   - 缺点：存在成功率低的问题

> **本阶段方案：乐观锁 + `gt("stock", 0)` 条件扣减**——用数据库的原子更新解决并发问题，不用 JVM 锁、不依赖分布式锁，秒杀场景下性能最好。
---

## ⚠️ 避坑 Tips

### 1. 测试的坑
- **没装 JMeter 可以用 Apifox/HiFox**：自动化测试 → 场景用例 → 性能测试，配并发用户数就行
- **并发数别太大**：40 个线程就够复现问题，太多容易卡
- **一定要看数据库**：接口失败率 0% 不代表没超卖，要看库存有没有变负

### 2. 乐观锁的坑
- **CAS 用 eq 条件成功率太低**：高并发下大家都读到同一个值，只有第一个能成功，剩下全失败
- **别用"先查再改"**：`if (stock > 0) stock--` 这种 Java 代码判断+更新不是原子的，必须靠 SQL 条件
- **`gt("stock", 0)` 是优化后的写法**：保证不超卖 + 成功率，秒杀推荐

### 3. 数据库的坑
- **库存字段别用 int**：扣成负数还能继续扣，至少要有条件约束（`stock > 0`）
- **`@Transactional` 别忘了**：扣库存和创建订单是两个操作，要么都成功要么都失败

---

## 🎯 面试题

### Q1：什么是超卖问题？怎么产生的？

> **回答框架：**
> 1. **定义**：库存 20，却有 29 个人抢到了（库存被扣成负数）
> 2. **原因**：多线程并发下，**查库存和扣库存不是原子操作**——多个线程都查到库存 > 0，然后一起扣减，导致超扣
> 3. **本质**：经典的线程安全问题（竞态条件），判断和操作之间有间隔，别的线程插进来
> 4. **项目例子**：黑马点评秒杀，20 库存 40 并发，库存变 -9

### Q2：悲观锁和乐观锁的区别？秒杀用哪个？

> **回答框架：**
> 1. **悲观锁**：认为一定会冲突，先加锁让线程串行（Synchronized/Lock），简单但性能差
> 2. **乐观锁**：认为不一定冲突，更新时校验（CAS/版本号），性能好但可能失败
> 3. **秒杀场景选乐观锁**：并发量高，性能优先
> 4. **优化**：CAS 用 `eq(stock, 旧值)` 成功率太低，改成 `gt(stock, 0)` 条件扣减

### Q3：乐观锁怎么实现？CAS 和版本号有什么区别？

> **回答框架：**
> 1. **CAS法**：更新时 where 带上查到的旧值（`eq("stock", 20)`），值变了就更新失败
> 2. **版本号法**：表加 version 字段，更新时 version+1，where 带旧 version
> 3. **区别**：CAS 直接用业务字段做比较，版本号用专用字段；本质都是"更新前校验有没有被改"
> 4. **注意**：CAS 用业务字段做条件成功率低（并发下值很快变），秒杀优化成 `stock > 0`

### Q4：为什么 `gt("stock", 0)` 能解决超卖？

> **回答框架：**
> 1. **原子性**：`update ... set stock = stock - 1 where stock > 0` 是**一条 SQL**，数据库保证原子执行，不存在"判断和扣减的间隔"
> 2. **不超卖**：库存扣到 0 后，`stock > 0` 不成立，后续更新 0 行受影响
> 3. **成功率高**：不要求库存等于某个具体值，只要还有货就能抢
> 4. **本质**：把并发安全交给数据库的行锁/原子更新，而不是 Java 代码


记录第一个月了！今天是乐观锁解决超卖。
黑马点评 day13，今天先自己复现了超卖（20 库存变 -9），再一步步改：乐观锁 CAS → 成功率太低 → 最终 `stock > 0` 条件扣减，最后库存从 -9 到正常。研究了研究apifox并发测试，用它进行的测试，四十个线程抢20个券，至少测试都达到想要的效果了。听说后面更难啊，加油吧。
