---
tags:
  - 黑马点评前置
  - Redis命令
---
# Day03 - Redis命令

## 一、通用命令

`help [command]` 可以查看一个命令的具体用法

| 命令                   | 作用                 | 注意             |
| -------------------- | ------------------ | -------------- |
| `KEYS pattern`       | 查看符合模板的所有 key      | **生产环境不建议用**   |
| `DEL key`            | 删除一个指定的 key        |                |
| `EXISTS key`         | 判断 key 是否存在        |                |
| `EXPIRE key seconds` | 给 key 设置有效期，到期自动删除 | 缓存标配           |
| `TTL key`            | 查看 key 的剩余有效期      | -1 永不过期，-2 不存在 |

> **思考：** EXPIRE + TTL 就是"缓存过期时间"的命令实现。项目里设置验证码 60 秒过期，就是 `SETEX`/`EXPIRE` 干的活。

## 二、Key 的结构

**Redis 的 key 允许有多个单词形成层级结构，单词之间用 `:` 隔开** `demo:user:1`   

**Value 是 Java 对象时，序列化为 JSON 字符串存储：**

| KEY            | VALUE                                     |
| -------------- | ----------------------------------------- |
| demo:user:1    | `{"id":1, "name": "Jack", "age": 21}`     |
| demo:product:1 | `{"id":1, "name": "小米11", "price": 4999}` |

> **思考：** `:` 分层有什么好处：① 可读性好，一眼看出是哪个项目哪个业务 ② 配合 `KEYS demo:user:*` 可以按前缀批量操作 ③ Redis 图形客户端会按 `:` 自动折叠成树形目录，这就是"图形化界面看很明显层级格式"的原因。

## 三、字符串 String

**String 的常见命令：**

| 命令 | 作用 |
|------|------|
| `SET key value` | 添加或修改一个 String 键值对 |
| `GET key` | 根据 key 获取 value |
| `MSET` | 批量添加多个键值对 |
| `MGET` | 批量获取多个 value |
| `INCR key` | 让一个整型 key 自增 1 |
| `INCRBY key num` | 让整型 key 自增指定步长（`incrby num 2` 就是 +2） |
| `INCRBYFLOAT key num` | 让浮点数字自增指定步长 |
| `SETNX key value` | **key 不存在才添加**，存在不执行 |
| `SETEX key seconds value` | 添加并指定有效期 |
## 四、哈希 Hash

### 1. 为什么要用 Hash？（对比 String 存对象）

**String 存对象：** 把整个对象序列化成 JSON，改一个字段要把整个 JSON 拿出来改完再存回去，不方便。

**Hash 存对象：** 每个字段独立存储，可以针对**单个字段**做增删改查：

| KEY         | field | value |
| ----------- | ----- | ----- |
| demo:user:1 | name  | Jack  |
| demo:user:1 | age   | 21    |
| demo:user:2 | name  | Rose  |
| demo:user:2 | age   | 18    |

> Hash 也叫散列，value 是一个无序字典，类似 Java 的 **HashMap**。

### 2. Hash 常见命令

| 命令                       | 作用                  |
| ------------------------ | ------------------- |
| `HSET key field value`   | 添加或修改 key 的 field 值 |
| `HGET key field`         | 获取 key 的 field 值    |
| `HMSET`                  | 批量添加多个 field        |
| `HMGET`                  | 批量获取多个 field        |
| `HGETALL key`            | 获取所有 field 和 value  |
| `HKEYS key`              | 获取所有 field          |
| `HVALS key`              | 获取所有 value          |
| `HINCRBY key field num`  | 让 field 值自增指定步长     |
| `HSETNX key field value` | field 不存在才添加        |
## 五、列表 List

### 1. List 是什么

**List 与 Java 的 LinkedList 类似，是双向链表结构**，支持正反向检索。

| 特征 | 说明 |
|------|------|
| 有序 | 按插入顺序排列 |
| 元素可以重复 | |
| 插入和删除快 | 链表特性，头尾 O(1) |
| 查询速度一般 | 要遍历，中间查询 O(n) |
**场景：** 存储有序数据，例如朋友圈点赞列表、评论列表。
### 2. List 常见命令

| 命令                      | 作用                                        |
| ----------------------- | ----------------------------------------- |
| `LPUSH key element ...` | 向列表**左侧**插入一个或多个元素                        |
| `LPOP key`              | 移除并返回**左侧**第一个元素，没有返回 nil                 |
| `RPUSH key element ...` | 向列表**右侧**插入元素                             |
| `RPOP key`              | 移除并返回**右侧**第一个元素                          |
| `LRANGE key start end`  | 返回一段角标范围内的所有元素                            |
| `BLPOP` / `BRPOP`       | 和 LPOP/RPOP 类似，但**没有元素时等待指定时间**，不直接返回 nil |
### 3. 用 List 模拟数据结构（思考）

| 数据结构     | 怎么做                                  |
| -------- | ------------------------------------ |
| **栈**    | 入口和出口在**同一边**（LPUSH + LPOP）          |
| **队列**   | 入口和出口在**不同边**（LPUSH + RPOP）          |
| **阻塞队列** | 入口和出口在不同边，出队用 **BLPOP/BRPOP**（没元素就等） |

## 六、集合 Set(分两组)

### 1. Set 常见命令

| 命令                     | 作用              |
| ---------------------- | --------------- |
| `SADD key member ...`  | 向 set 添加一个或多个元素 |
| `SREM key member ...`  | 移除指定元素          |
| `SCARD key`            | 返回 set 元素个数     |
| `SISMEMBER key member` | 判断元素是否存在        |
| `SMEMBERS key`         | 获取所有元素          |

### 2. 集合运算

| 命令                     | 作用      | 场景           |
| ---------------------- | ------- | ------------ |
| `SINTER key1 key2 ...` | 求**交集** | 共同关注、共同好友    |
| `SDIFF key1 key2 ...`  | 求**差集** | 推荐（关注了A没关注B） |
| `SUNION key1 key2 ...` | 求**并集** | 合并标签、全部好友    |
## 七、有序集合 Sorted Set

### 1. Sorted Set 是什么

**可排序的 set 集合**，和 Java 的 TreeSet 类似，但底层差别很大：
- 每个元素带一个 **score 属性**，按 score 排序
- 底层实现是 **跳表（SkipList）+ 哈希表**

| 特性 | 说明 |
|------|------|
| 可排序 | 按 score |
| 元素不重复 | |
| 查询速度快 | 跳表结构 |

**场景：** 因为可排序特性，**经常用来实现排行榜**。

### 2. Sorted Set 常见命令

| 命令                             | 作用                           |
| ------------------------------ | ---------------------------- |
| `ZADD key score member`        | 添加元素，已存在则更新 score            |
| `ZREM key member`              | 删除指定元素                       |
| `ZSCORE key member`            | 获取指定元素的 score                |
| `ZRANK key member`             | 获取指定元素的**排名**                |
| `ZCARD key`                    | 获取元素个数                       |
| `ZCOUNT key min max`           | 统计 score 在给定范围内的元素个数         |
| `ZINCRBY key increment member` | 让指定元素 score 自增步长             |
| `ZRANGE key min max`           | 按 score 排序后获取指定排名范围内的元素      |
| `ZRANGEBYSCORE key min max`    | 按 score 排序后获取指定 score 范围内的元素 |
| `ZDIFF` / `ZINTER` / `ZUNION`  | 求差集、交集、并集                    |

> **注意：** 所有排名默认**升序**；要降序就在命令的 Z 后面加 **REV**：`ZREVRANGE`、`ZREVRANK`。

记录第二十天，Linux系统敲了敲redis命令。
黑马点评前置day3，熟悉熟悉redis命令，苍穹外卖的时候就简单了解了个字符串类型，今天都跟着敲了敲，是要加快进度了。