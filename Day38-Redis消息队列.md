---
tags:
  - 黑马点评
  - Redis
  - 消息队列
  - Stream
  - 消费者组
---
# Day18 - Redis消息队列（Stream + 消费者组实现异步秒杀）

> 承接 Day17：JDK 阻塞队列做异步下单有三个硬伤——**内存限制**（1024*1024 有上限）、**服务宕机数据丢失**、**任务取出后异常丢失**。今天的方案：**用 Redis 的 Stream 消息队列替代 JDK 队列**，它带持久化、消息确认、消费者组，把异步秒杀做"稳"。

## 一、消息队列入门

### 1. 什么是消息队列？

**消息队列（Message Queue）**，字面意思就是"存放消息的队列"。最简单的消息队列模型有 **3 个角色**：

| 角色 | 作用 |
| --- | --- |
| **消息队列** | 存储和管理消息，也叫消息代理（Message Broker） |
| **生产者** | 发送消息到消息队列 |
| **消费者** | 从消息队列获取消息并处理消息 |

**套到秒杀场景：** 生产者 = 秒杀业务的前置处理（判断秒杀时间和库存、校验一人一单）→ 抢购成功就把优惠券 id 和用户 id 发到消息队列；消费者 = 接收消息、完成下单。

### 2. Redis 实现消息队列的三种方式

Redis 提供了三种不同的方式来实现消息队列：
1. **list 结构**：基于 List 结构模拟消息队列
2. **PubSub**：基本的点对点消息模型
3. **Stream**：比较完善的消息队列模型
## 二、方案一：List 结构模拟消息队列

### 1. 实现方式

Redis 的 list 是双向链表，很容易模拟队列（**入口和出口不在一边**）：
- **LPUSH + RPOP**：左边进、右边出
- **RPUSH + LPOP**：右边进、左边出

**关键点——阻塞读取：** 当队列中没有消息时，`RPOP` / `LPOP` 会返回 null，**并不会像 JVM 的阻塞队列那样阻塞等待**。所以应该使用 **BRPOP** 或 **BLPOP** 实现阻塞效果。

### 2. 优缺点

| 优点                        | 缺点                               |
| ------------------------- | -------------------------------- |
| 利用 Redis 存储，不受限于 JVM 内存上限 | **无法避免消息丢失**（消息被取走后消费失败就没了）      |
| 基于 Redis 的持久化机制，数据安全性有保证  | **只支持单消费者**（一个消息只能被一个消费者取走，无法分流） |
| 可以满足消息有序性                 |                                  |
## 三、方案二：PubSub 发布订阅

### 1. 三个核心命令

PubSub（发布订阅）是 **Redis 2.0** 引入的消息传递模型：消费者订阅一个或多个 channel，生产者向 channel 发消息，**所有订阅者都能收到**。

| 命令 | 作用 |
| --- | --- |
| `SUBSCRIBE channel [channel...]` | 订阅一个或多个频道 |
| `PUBLISH channel msg` | 向一个频道发送消息 |
| `PSUBSCRIBE pattern [pattern...]` | 订阅与 pattern 格式匹配的所有频道（如 `order.*`） |

### 2. 优缺点

| 优点                     | 缺点                  |
| ---------------------- | ------------------- |
| 采用发布订阅模型，**支持多生产、多消费** | **不支持数据持久化**        |
|                        | **无法避免消息丢失**        |
|                        | **消息堆积有上限**，超出时数据丢失 |

> **结论：** 数据完全不落地，几乎每条缺点都和"丢"有关——比 List 还不可靠，pass。

## 四、方案三：Stream 消息队列（Redis 5.0）⭐

**Stream 是 Redis 5.0 引入的一种新数据类型，可以实现一个功能非常完善的消息队列。** 前面的问题它都能解决。

### 1. 发送消息：XADD

```bash
XADD key [NOMKSTREAM] [MAXLEN|MINID [=|~] threshold [LIMIT count]] *|ID field value [field value ...]
```
参数说明：
- **key**：队列名；`NOMKSTREAM` 表示队列不存在时**不自动创建**（默认是自动创建）
- **MAXLEN**：设置消息队列的最大消息数量
- **\*|ID**：消息的唯一 id。`*` 代表由 Redis 自动生成，格式是"**时间戳-递增数字**"，例如 `1644804662707-0`
- **field value**：消息内容叫 Entry，就是**多个 key-value 键值对**

示例——创建名为 users 的队列，发送 `{name=jack, age=21}`，id 自动生成：

```bash
<127.0.0.1:6379> XADD users * name jack age 21
"1644805700523-0"
```

### 2. 读取消息：XREAD
```bash
XREAD [COUNT count] [BLOCK milliseconds] STREAMS key [key ...] ID [ID ...]
```

- **COUNT**：每次读取消息的最大数量
- **BLOCK**：没有消息时是否阻塞、阻塞时长（毫秒）
- **STREAMS key**：从哪个队列读取
- **ID**：起始 id，只返回大于该 ID 的消息——`0` 代表从第一个消息开始，`$` 代表从最新的消息开始

示例——读取第一条消息：

```bash
<127.0.0.1:6379> XREAD COUNT 1 STREAMS users 0
1) 1) "users"
   2) 1) 1) "1644805700523-0"
         2) 1) "name"
            2) "jack"
            3) "age"
            4) "21"
```

### 3. XREAD 的 bug：漏读消息

**阻塞方式读取最新消息**：`XREAD COUNT 1 BLOCK 1000 STREAMS users $`，可以循环调用实现持续监听队列。**但是有 bug：
** 当指定起始 ID 为 `$`（读取最新的消息）时，如果我们在**处理一条消息的过程中，又有超过 1 条以上的消息到达队列**，下次获取时只能获取到最新的一条，**中间的消息全部漏读**。
### 4. 解决：消费者组 Consumer Group

**消费者组（Consumer Group）：将多个消费者划分到一个组中，监听同一个队列。** 具备三个核心特点：

| | 特点 | 说明 |
| --- | --- | --- |
| 01 | **消息分流** | 队列中的消息会分流给组内的不同消费者，而不是重复消费，加快消息处理速度 |
| 02 | **消息标示** | 消费者组维护一个标示，记录**最后一个被处理的消息**；哪怕消费者宕机重启，也会从标示之后继续读，**确保每一条消息都会被消费** |
| 03 | **消息确认** | 消费者获取消息后，消息处于 **pending** 状态，并存入 **pending-list**；处理完成后通过 **XACK** 确认，标记为已处理，才会从 pending-list 移除 |

> 三个特点正好对症下药：**分流**解决单消费者/漏读（组内每人读不同的消息），**标示**解决"重启后从哪读"，**确认**解决"处理失败的消息怎么找回"。这就是 XREAD 没有的可靠性。

### 5. 消费者组相关命令

**创建消费者组：**

```bash
XGROUP CREATE key groupName ID [MKSTREAM]
```
- key：队列名称
- groupName：消费者组名称
- ID：起始 ID 标示——`$` 代表队列中最后一个消息，`0` 代表队列中第一个消息
- MKSTREAM：队列不存在时自动创建队列

**其它常见命令：**
```bash
# 删除指定的消费者组
XGROUP DESTROY key groupName

# 给指定的消费者组添加消费者
XGROUP CREATECONSUMER key groupName consumerName

# 删除消费者组中的指定消费者
XGROUP DELCONSUMER key groupName consumerName
```

**从消费者组读取消息：**

```bash
XREADGROUP GROUP group consumer [COUNT count] [BLOCK milliseconds] [NOACK] STREAMS key [key ...] ID [ID ...]
```

- **group**：消费组名称
- **consumer**：消费者名称，**不存在会自动创建**
- **count**：本次查询的最大数量
- **BLOCK**：没有消息时最长等待时间
- **NOACK**：无需手动 ACK，获取到消息后自动确认
- **ID** 起始 id 的两种用法：
  - `">"`：从**下一个未消费的消息**开始（读新消息）
  - **其它值（如 0）**：根据指定 id 从 **pending-list** 中获取"已消费但未确认"的消息（读异常遗留消息）
### 6. 三种实现方式对比

| 特性 | List | PubSub | Stream |
| --- | --- | --- | --- |
| **消息持久化** | 支持 | 不支持 | 支持 |
| **阻塞读取** | 支持 | 支持 | 支持 |
| **消息堆积处理** | 受限于内存空间，可利用多消费者加快处理 | 受限于消费者缓冲区 | 受限于队列长度，可利用消费者组提高消费速度、减少堆积 |
| **消息确认机制** | 不支持 | 不支持 | **支持** |
| **消息回溯** | 不支持 | 不支持 | **支持** |

> **结论：** 秒杀场景选 **Stream**。业务庞大、对消息队列要求严格时，就使用**专业的 MQ**（如 RabbitMQ、RocketMQ、Kafka）。

## 五、Java 实现：Stream 实现异步秒杀下单

### 1. 创建消息队列

```bash
xgroup create stream.orders g1 0 mkstream
```
创建名为 `stream.orders` 的队列和消费者组 `g1`，`0` 表示从第一个消息开始，`mkstream` 表示队列不存在时自动创建。

### 2. 修改 Lua 脚本：抢购成功后发消息

**加一个参数 `local orderId = ARGV[3]`**，抢购成功后用 XADD 发送消息到队列：

```lua
-- 1.3.订单id（新增）
local orderId = ARGV[3]
...
-- 3.5.发送消息到stream队列（新增）
redis.call('xadd', 'stream.orders', '*', 'userId', userId, 'voucherId', voucherId, 'id', orderId)
return 0
```

> **小细节：** 订单 id 字段名**没有写 orderId，而是写的 id**——因为业务操作的是订单表，主键就是 id，消息里的字段名要和 `VoucherOrder` 实体类的字段对应，后面转对象才能对上。

### 3. seckillVoucher：生成订单 id 传入脚本

消息里的 orderId 要提前生成，作为第三个参数传给 Lua 脚本）：

```java
@Override
public Result seckillVoucher(Long voucherId) {  
    Long userId = UserHolder.getUser().getId();  
    // 2.3订单id  
    long orderId = redisIdWorker.nextId("order");  
    // 1.执行lua脚本  
    Long result = stringRedisTemplate.execute(  
            SECKILL_SCRIPT, Collections.emptyList(), voucherId.toString(), userId.toString(),String.valueOf(orderId)  
    );  
    // 2.判断结果是否为0  
    int r = result.intValue();  
    if (r != 0){  
        // 2.1不为0，没有下单资格  
        return Result.fail(r == 1 ? "库存不足!": "不能重复下单!");  
    }  
    // 3.获取代理对象（事务）  
    proxy = (IVoucherOrderService) AopContext.currentProxy();  
    return Result.ok(orderId);  
}
```
### 4. 线程任务消费下单（含异常重试）

开启线程任务下单的思路（对应草稿的总结）：
- **获取消息**：`XREADGROUP` 从队列读消息（阻塞模式）→ **获取失败（null）就 continue 继续获取**
- **解析消息**：count(1) 是一条一条读，所以 `list.get(0)` 拿到这条消息 → 从消息里拿出 userId、voucherId、id 键值对 → 转成 `VoucherOrder` 对象（**订单 id 字段是 id**，对应实体类里的订单 id）
- **创建订单 + ACK**：处理完消息后 XACK 确认
- **异常处理**：下单抛异常 → 去 **pending-list**（ID=0）获取待处理列表里的消息重试 → 再异常就继续循环（不用递归），可以休眠避免频繁重试，减轻CPU占用
---
## ⚠️ 避坑 Tips

### 1. List / PubSub 的坑
- **LPOP/RPOP 空队列返回 null，不阻塞**：要阻塞等待必须用 BLPOP / BRPOP
- **List 无法避免消息丢失、只支持单消费者**：消息被取走消费失败就没了，也不支持多消费者分流
- **PubSub 数据不落地**：不支持持久化、无法避免消息丢失、消息堆积有上限（消费者缓冲区），超出直接丢——**消息可靠性要求高的场景别用 PubSub**

### 2. Stream 的坑
- **XREAD 用 `$` 会漏读**：处理一条消息期间到达的多条消息，下次只能读到最新一条，中间的丢了——用消费者组 XREADGROUP 解决
- **XGROUP CREATE 记得加 MKSTREAM**：队列不存在时自动创建，否则报错
- **XADD 的字段名要和实体类字段对上**：订单 id 字段写 `id` 而不是 `orderId`，不然 BeanUtil.toBean 转换后拿不到订单 id
- **ID 的含义别搞混**：`0` = 从第一个消息读（也用于读 pending-list），`$` = 从最新消息读，`>` = 消费者组里读"下一个未消费的"

### 3. 消费者组的坑
- **处理完必须 XACK**：不确认消息就一直留在 pending-list，重启后会被**重复消费**，可能重复下单（后端有判断就不会重复）
- **异常重试别用递归**：pending-list 里消息多时递归会栈溢出，用 while(true) 循环；每次失败可以 Thread.sleep(20) 避免频繁重试占用 CPU
- **pending-list 是"已消费未确认"的消息**：`XREADGROUP ... ID 0` 读的是它，不是队列里的新消息

---

## 🎯 面试题

### Q1：Redis 实现消息队列有哪几种方式？各自有什么优缺点？

> **回答框架：**
> 1. **三种方式**：List 模拟、PubSub、Stream
> 2. **List**：LPUSH+RPOP/BRPOP；优点是不受 JVM 内存上限、有持久化、消息有序；缺点是无法避免消息丢失、只支持单消费者
> 3. **PubSub**：SUBSCRIBE/PUBLISH/PSUBSCRIBE，发布订阅模型支持多生产多消费；缺点是不支持持久化、无法避免消息丢失、消息堆积有上限
> 4. **Stream**（Redis 5.0）：XADD 发消息、XREAD/XREADGROUP 读消息；支持持久化、阻塞读取、消息确认、消息回溯、消费者组分流，是目前 Redis 里最完善的消息队列
> 5. **结论**：要求高就用 Stream；业务庞大、要求严格就用专业的 MQ（RabbitMQ/RocketMQ/Kafka）

### Q2：XREAD 有什么问题？为什么用消费者组解决？

> **回答框架：**
> 1. **XREAD 的问题**：用 `$` 作起始 ID 只读最新消息，处理一条消息期间到达的多条消息会漏读
> 2. **消费者组三个特性**：① 消息分流——消息分给组内不同消费者，不重复消费，加快处理；② 消息标示——记录最后处理的消息，宕机重启后从标示之后继续读，每条都会被消费；③ 消息确认——消息取走后进 pending-list，XACK 确认后才移除，失败的消息能捞回来
> 3. **一句话**：XREAD 是"读了就完事"，消费者组是"分流 + 断点 + 确认"，解决漏读和丢失

### Q3：消费者组怎么处理消息确认和异常重试？

> **回答框架：**
> 1. **正常消费**：XREADGROUP GROUP g1 c1 COUNT 1 BLOCK 2000 STREAMS stream.orders `>`（读未消费消息）→ 处理 → XACK 确认
> 2. **异常重试**：catch 到异常后，用 `XREADGROUP ... STREAMS stream.orders 0` 去 pending-list 读"已消费未确认"的消息，重新处理；处理成功就 ACK，pending-list 空了就 break
> 3. **再异常**：记录日志继续循环，不递归；可以 Thread.sleep 避免频繁重试、减轻 CPU 压力
> 4. **为什么可靠**：没 ACK 的消息一直在 pending-list，服务重启后也能捞回来，不会丢订单

### Q4：为什么用 Stream 替代 JDK 阻塞队列？解决了哪些问题？

> **回答框架：**
> 1. **JDK 队列的问题**：① 内存限制——JVM 内存，容量有限，满了放不进去；② 服务宕机数据全丢——队列在内存里，预减了库存但订单没创建，数据不一致；③ 任务取出后异常丢失——take 出队后执行失败不再重试
> 2. **Stream 的解决**：① 队列存在 Redis，不受 JVM 内存上限；② Redis 持久化，宕机不丢；③ 消费者组 pending-list + XACK，失败的消息留在"已消费未确认"列表，可以重试捞回，不会丢任务
> 3. **额外收益**：消费者组支持多消费者分流，并发消费能力更强

**一条主线串起来：**
- 异步秒杀的下单队列演进：**JDK 阻塞队列（Day17）** 能用但不稳（内存 + 丢数据）→ **Redis 消息队列（Day18）**：List 太弱（丢消息、单消费者）→ PubSub 更弱（不持久化、堆积丢数据）→ **Stream 最终胜出**（持久化 + 阻塞 + 确认 + 回溯 + 消费者组）
- **本质是"把队列从 JVM 内存搬到 Redis"**：能持久化、能确认、能分流，可靠性和扩展性都上来了
- 秒杀全链路也彻底定型：**Redis 前置校验（Lua）→ Stream 消息队列 → 异步线程下单**，前端秒回订单 id，后端慢慢落库，一条都不丢

记录第三十八天，Redis消息队列。
黑马点评 day18。今天解决昨天遗留的两个硬伤（内存限制、数据安全），把 JDK 阻塞队列换成了 Redis 的 Stream 消息队列。先把 Redis 三种消息队列方式过了一遍：List 模拟（LPUSH+BRPOP，丢消息、单消费者）、PubSub（发布订阅，不持久化、堆积丢数据）、Stream（Redis 5.0 最完善，持久化、确认、回溯、消费者组）。Stream 的 XREAD 用 $ 读最新消息会漏读，用消费者组来解决，就是消息分流、消息标示、消息确认三个特性刚好对症。最后用Java 实现就是把 Lua 脚本加一个 orderId 参数，抢购成功直接 xadd 到 stream.orders，线程任务 XREADGROUP 消费下单、处理完 XACK，异常就去 pending-list 重试。还有个小细节，消息字段名要写 id 而不是 orderId，对应订单表主键，这个真的很细啊。
