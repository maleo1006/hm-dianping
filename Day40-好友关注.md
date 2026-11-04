---
tags:
  - 黑马点评
  - 好友关注
  - Redis
  - Feed流
  - ZSet
  - 滚动分页
---
# Day20 - 好友关注（Redis Set 共同关注 + Feed 流 + 滚动分页）

## 一、关注和取关

两个接口：`follow`（关注/取关）、`getFollowOrNot`（查询是否已关注）。数据库里就一张 `follow` 表（user_id + follow_user_id）。

```java
@Override
public Result follow(Long followUserId, boolean isFollow) {
    // 获取登录用户
    Long userId = UserHolder.getUser().getId();
    // 判断是关注还是取关
    if (isFollow) {
        // 关注，新增数据
        Follow follow = new Follow();
        follow.setUserId(userId);
        follow.setFollowUserId(followUserId);
        save(follow);
    } else {
        // 取关，删除数据
        remove(new QueryWrapper<Follow>()
                .eq("user_id", userId).eq("follow_user_id", followUserId));
    }
    return Result.ok();
}

@Override
public Result getFollowOrNot(Long followUserId) {
    Long userId = UserHolder.getUser().getId();
    // 查询
    Integer count = query().eq("user_id", userId).eq("follow_user_id", followUserId).count();
    // 判断
    return Result.ok(count > 0);
}
```
## 二、共同关注（Redis Set 交集）

 关注/取关的时候，**顺手把"关注了谁"同步存进 Redis 的 Set**（`follows:{userId}`），查共同关注时直接**两个 Set 求交集**。
```java
// 关注/取关时同步维护 Redis（在 follow 方法里加两行）
@Override
public Result follow(Long followUserId, boolean isFollow) {
    Long userId = UserHolder.getUser().getId();
    String key = "follows:" + userId;
    if (isFollow) {
        Follow follow = new Follow();
        follow.setUserId(userId);
        follow.setFollowUserId(followUserId);
        boolean isSuccess = save(follow);
        // 关注成功 → 把对方 id 加入我的关注集合
        if (isSuccess) {
            stringRedisTemplate.opsForSet().add(key, followUserId.toString());
        }
    } else {
        boolean isSuccess = remove(new QueryWrapper<Follow>()
                .eq("user_id", userId).eq("follow_user_id", followUserId));
        // 取关成功 → 从集合里移除
        if (isSuccess) {
            stringRedisTemplate.opsForSet().remove(key, followUserId.toString());
        }
    }
    return Result.ok();
}
```

```java
@Override
public Result getFollowCommons(Long id) {
    // 获取当前用户id
    Long userId = UserHolder.getUser().getId();
    String key = "follows:" + userId;
    String key2 = "follows:" + id;
    // 查交集
    Set<String> intersect = stringRedisTemplate.opsForSet().intersect(key, key2);
    if (intersect == null || intersect.isEmpty()) {
        return Result.ok(Collections.emptyList());
    }
    // 解析id
    List<Long> ids = intersect.stream().map(Long::valueOf).collect(Collectors.toList());
    // 查询用户
    List<UserDTO> userDTOS = userService.listByIds(ids)
            .stream()
            .map(user -> BeanUtil.copyProperties(user, UserDTO.class))
            .collect(Collectors.toList());
    return Result.ok(userDTOS);
}
```

**为什么用 Set？** 关注关系是"我关注了谁"的**去重集合**，Redis 的 Set 天然支持：
- **去重**：同一个用户不会关注两次（sadd 重复自动忽略）
- **集合运算**：`intersect` 直接求交集，O(N) 搞定
- **O(1) 判断**：后面判断"是否关注"也可以 `sismember` 查

**注意：** 这个 Redis 集合是**数据库的冗余副本**，必须和数据库同步增删（关注 sadd、取关 srem），否则查出来是脏数据。

## 三、Feed 流（重点）

**Feed 流是什么？** 你打开抖音/微博看到的"关注的人发的动态时间线"。核心问题：**一个用户发笔记，怎么让所有关注他的人都能刷到？**
### 1. 三种实现模式：拉模式、推模式、推拉结合

**① 拉模式（读扩散）** — 发的时候省事，刷的时候费劲
- **发笔记**：什么都不用做，写完存库就行
- **刷动态**：查询自己关注的所有人 → 拉取他们发的笔记 → 合并排序展示
- **优点**：发省事
- **缺点**：读放大——关注了 N 个人，刷一次就要查 N 次再合并
- **适用**：**大V**（粉丝几百万，发一条推给所有粉丝，写放大比读放大更可怕）

**② 推模式（写扩散）** — 发的时候费劲，刷的时候快
- **发笔记**：推送给所有粉丝的收件箱（每个粉丝一个 Feed，key = `feed:{粉丝id}`）
- **刷动态**：直接读自己的收件箱，不用现查现算
- **优点**：刷的快，直接读自己收件箱一条命令
- **缺点**：写放大——粉丝 N 个就要写 N 份
- **适用**：**普通用户**（粉丝量小（千万级粉丝以下），写放大可接受，实现最简单）

**③ 推拉结合（混合模式）** — 平时推，大V拉
- **普通用户**：用**推**——发笔记推送给粉丝
- **大V**：**不推**（粉丝太多，写放大扛不住）——刷动态时**单独去拉**大V的内容
- **优点**：**平衡写放大和读放大**，两边都不炸
- **缺点**：实现复杂——要区分大V和普通用户、维护两套流程

|         | 拉模式（读扩散）     | 推模式（写扩散）   | 推拉结合（混合）     |
| ------- | ------------ | ---------- | ------------ |
| **谁干活** | 刷的时候拉取关注的人   | 发的时候推给粉丝   | 普通用户推、大V拉    |
| **收件箱** | 现查现算         | 每个粉丝一个     | 普通用户有、大V无    |
| **优点**  | 发省事、存储省      | 刷的快        | 读写混合，兼顾推拉优点  |
| **缺点**  | 读放大（查 N 次合并） | 写放大（写 N 份） | 实现复杂         |
| **适用**  | 很少用          | 普通用户（粉丝少）  | 过千万用户、有大V的平台 |

**本项目用推模式**：发笔记 → 查作者粉丝 → 给每个粉丝的收件箱 ZSet 里塞一条（score=时间戳，value=笔记id）。项目里的用户都是普通用户，粉丝量不大，写放大可以接受，实现也最简单。
### 2. 为什么收件箱用 ZSet？

收件箱要**按时间排序** + **分页拉取**，ZSet 刚好：
- **score = 时间戳**：按时间倒序排（`reverseRangeByScore`）
- **value = 笔记 id**：只存 id，不存笔记内容（内容查库），省内存
- **自带范围查询**：按 score 区间取，是滚动分页的基石

### 3. 滚动分页：为什么不用传统分页？

**传统分页（page/size）的致命问题：** 假设每页 2 条，你刷到第 1 页（笔记 9、8），此时博主发了新笔记 10，你再翻第 2 页（offset=2）→ 数据整体后移，**第 2 页变成了 8、7，你漏看了 7，还重复看到了 8**。Feed 流是不断有新数据插入的，页码分页会**重复和遗漏**。

**滚动分页的思路：** 不记"第几页"，**记"上次刷到哪了"**（最小时间戳 + 同时间戳偏移量）：
- 第一次查：`max = 当前时间戳`，取最新 2 条
- 记住这批里**最小的时间戳**（minTime）
- 下一次查：`max = minTime`，只取比它**更早**的 → 新插入的数据（时间戳更大）永远不会干扰，不重不漏

**思考：那最新的笔记怎么办？** 
下一次查询的 max 用的是"上一次的最小时间戳"，新笔记时间戳更大，不会出现在下一次结果里；回顶部（前端写了刷新）就能看到最新。

**同一时间戳怎么办？** 两个人同一毫秒发笔记，score 相同，ZSet 按 score 查会**把相同 score 的一起返回** → 需要 `offset` 参数：**如果当前时间和 minTime 相同，offset+1，跳过这批里已经看过的**。

## 四、推送到粉丝收件箱

不难，保存笔记成功后，查作者的粉丝，给每个粉丝的收件箱 ZSet 塞笔记 id：

```java
@Override
public Result saveBlog(Blog blog) {
    // 1.获取登录用户
    Long userId = UserHolder.getUser().getId();
    blog.setUserId(userId);
    // 2.保存探店笔记
    boolean isSuccess = save(blog);
    if (!isSuccess) {
        return Result.fail("新增笔记失败!");
    }
    // 3.查询笔记作者的所有粉丝（follow 表中 follow_user_id = 作者id 的记录）
    List<Follow> follows = followService.query()
            .eq("follow_user_id", userId).list();
    // 4.推送笔记id给所有粉丝
    for (Follow follow : follows) {
        // 4.1.获取粉丝id
        Long fanId = follow.getUserId();
        // 4.2.推送到粉丝收件箱：feed:{粉丝id}，score=时间戳，value=笔记id
        String key = FEED_KEY + fanId;
        stringRedisTemplate.opsForZSet().add(key, blog.getId().toString(), System.currentTimeMillis());
    }
    return Result.ok(blog.getId());
}
```
## 五、滚动分页查询

```java
@Override
public Result queryBlogOfFollow(Long max, Integer offset) {
    // 获取当前用户
    Long userId = UserHolder.getUser().getId();
    // 查询收件箱
    String key = FEED_KEY + userId;
    Set<ZSetOperations.TypedTuple<String>> typedTuples = stringRedisTemplate.opsForZSet()
            .reverseRangeByScoreWithScores(key, 0, max, offset, 2);
    // 判断
    if (typedTuples == null || typedTuples.isEmpty()) {
        return Result.ok();
    }
    // 解析数据
    List<Long> ids = new ArrayList<>();
    long minTime = 0;
    int os = 1;
    for (ZSetOperations.TypedTuple<String> typedTuple : typedTuples) {
        long time = typedTuple.getScore().longValue();
        // 获取id
        ids.add(Long.valueOf(typedTuple.getValue()));
        // 获取score（时间戳）
        if (time == minTime) {
            os++;   // 和最小时间戳相同 → 偏移量+1（跳过这批里已看过的）
        } else {
            minTime = time;  // 更小的时间戳 → 更新最小值，偏移量重置为1
            os = 1;
        }
    }
    // 根据id查询blog
    String idStr = StrUtil.join(",", ids);
    List<Blog> blogs = query().in("id", ids).last("order by field (id," + idStr + ")").list();
    for (Blog blog : blogs) {
        // 查询博客有关的用户
        queryBlogUser(blog);
        // 查询是否被点赞
        isBlogLiked(blog);
    }
    ScrollResult r = new ScrollResult();
    r.setList(blogs);
    r.setMinTime(minTime);
    r.setOffset(os);
    return Result.ok(r);
}
```
**逐段拆解：**
**① `reverseRangeByScoreWithScores(key, 0, max, offset, 2)`**
- `reverse`：按 score **从大到小**（最新的在前）
- `ByScore`：按**分数**查，区间是 `(0, max]`——只查时间戳 ≤ max 的（比上次更早的）
- `WithScores`：连 score 一起返回（要知道每条的时间戳）
- `offset, 2`：从第 offset 条开始取 2 条（滚动分页的"页码"就是 offset）

**② 计算 minTime 和 offset（滚动分页的精髓）**
- 遍历结果，找最小的时间戳 `minTime`
- 如果一条记录的 score **等于 minTime**，`os++`——说明和最小时间戳相同（同一毫秒发的），下次要从这批里**跳过已看过的**，所以偏移量要累计
- 如果更小，更新 minTime，os 重置为 1

**③ `order by field (id, ...)`**
- Redis 返回的 Set 是**无序**的，但查询出来要按"收件箱顺序（时间倒序）"展示
- `FIELD(id, 9, 8)` 让 MySQL 按指定 id 顺序返回 → 展示顺序和收件箱一致

**④ 返回 ScrollResult**
- `list`：这一批笔记
- `minTime` + `offset`：**下一批的查询凭证**，前端下一次请求时原样传回来

---
## ⚠️ 避坑 Tips

### 1. 滚动分页的 offset 是"同时间戳"的边界，不是页码
- 两个/多个笔记**同一毫秒**发（score 相同），`reverseRangeByScore` 会把它们**一起**返回 → 不加 offset 会重复加载
- offset 的语义：**"这批结果里有多少条和 minTime 相同"**，下次从这批的后面开始取
- 这是 `ScrollResult` 要返回 minTime + offset 两个值的原因，少一个都会出 bug
### 2. `order by field(id, ...)` 顺序必须保持
MyBatis-Plus 的 listByIds 默认是不保证顺序的，数据库查出来是主键索引顺序。 Feed 流这里，Redis ZSet 已经按时间排好序了，拿到的 ids 是有序集合，我们要让 SQL 查出的数据严格跟着 ids 顺序走

---
## 🎯 面试题
### Q1：什么是 Feed 流？推模式和拉模式的区别？本项目用的哪种？
> **回答框架：**
> 1. **Feed 流**：用户刷到的"关注的人发布的动态时间线"，核心是"一个人发内容，怎么高效地让所有粉丝看到"
> 2. **推模式（写扩散）**：发内容时把内容推送到每个粉丝的收件箱，**发的时候费劲、刷的时候快**，适合粉丝少的普通用户（千万级粉丝以下）
> 3. **拉模式（读扩散）**：不推，刷的时候去拉取所有关注对象的内容再合并，**发的时候轻松、刷的时候费劲**
> 4. **本项目**：推模式。发笔记时查粉丝列表，往每个粉丝的收件箱 ZSet 里塞笔记 id（score=时间戳）
### Q2：共同关注怎么实现？为什么用 Redis 的 Set？
> **回答框架：**
> 1. **数据库层面**：follow 表存关注关系，关注/取关时增删记录
> 2. **Redis 同步**：关注时把对方 id `sadd` 到 `follows:{userId}`，取关时 `srem`
> 3. **查询**：两个 Set 直接 `intersect` 求交集，拿到共同关注的用户 id 再去查用户详情
> 4. **为什么 Set**：① 去重（不会重复关注）；② 天然支持集合运算（交集差集并集）；③ 判断是否存在 O(1)；④ 把"查两次库 + Java 遍历求交集"降为"一次 Redis 集合运算"
### Q3：Feed 流为什么用滚动分页而不是传统分页？
> **回答框架：**
> 1. **传统分页的痛点**：Feed 流是"动态数据"，翻页期间有新内容插入会让数据整体偏移 → 下一页会**重复**（看到已看过的）或**遗漏**（跳过没看过的）
> 2. **滚动分页的思路**：不记"第几页"，记"上次刷到哪"——返回最小时间戳 minTime 和同时间戳偏移量 offset，下一次查 `score ≤ minTime` 且跳过 offset 条
> 3. **为什么新插入的数据不干扰**：新数据时间戳更大（> max），永远不在查询区间内，天然不重复
> 4. **offset 的意义**：同一毫秒发的多条笔记 score 相同，会一起返回，offset 用来跳过这批里已看过的。




记录第四十天，好友关注模块。
黑马点评 day20。今天写了关注/取关、共同关注、Feed 流。共同关注就是把关注列表同步到 Redis Set，两个集合求交集。Feed 流用的推模式，发笔记时推送到粉丝收件箱（ZSet 存笔记id+时间戳），刷的时候滚动分页（minTime + offset）查自己的收件箱。这个涉及很巧妙啊，Feed 流是动态数据，传统分页会重复和遗漏，minTime和offset，确定上次刷到哪了，同一毫秒发的多条笔记 score 相同，会一起返回，offset 用来跳过这批里已看过的，避免重复。
