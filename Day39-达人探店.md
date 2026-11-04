---
tags:
  - 黑马点评
  - 达人探店
  - 点赞
  - RedisSet
  - RedisSortedSet
---
# Day19 - 达人探店（发、看笔记 + 点赞 + 点赞排行榜）

## 一、发布探店笔记

### 1. 功能

点击首页最下方菜单栏中的 **+** 按钮，即可发布探店图文。两个请求：

| 请求                 | 方法   | 说明                  |
| ------------------ | ---- | ------------------- |
| `/api/upload/blog` | POST | 上传图片，返回图片地址         |
| `/api/blog`        | POST | 保存笔记（标题、内容、图片、关联商户） |

# 避雷！！！必须看到这！！！
### 2. 集群部署踩坑：Nginx 轮询，一台开一台没开会出问题？


**我遇到的问题：** 登录的时候出了很大问题。大家一定要把之前多开的服务器都打开，或者改 nginx 配置只转发到一台服务器。

**为什么会这样？**（有无大神讲一讲，问ai，感觉这个比较靠谱）

> **Nginx 的轮询策略是"一视同仁"的——它不感知后端节点是否存活。** 两台 Tomcat 一台开着、一台没开，nginx 还是会按轮询把请求轮流分给两台：分到没开的那台，连接直接被拒绝，前端表现就是**时好时坏**（同一个接口刷新一下成功、刷新一下失败）。

更具体地说，虽然登录态在 Redis 里是共享的（token 查 Redis，哪台 Tomcat 都能取到用户），但**登录接口、刷新 token 的拦截器、上传图片，任何一个请求被分到宕机节点都会失败**。
## 二、查看探店笔记

这个功能很简单：**先查博客（Blog），再查博客有关的用户**——根据博客里的用户 id 查出用户名和头像，赋给博客返回。

### 1. Blog 实体加两个扩展字段

博客表（tb_blog）里没有用户名和头像这两列，所以在 Blog 实体里加字段，并用 `@TableField(exist = false)` 告诉 MyBatis-Plus 这俩字段**不是数据库字段**：

```java
/**
 * 用户图标
 */
@TableField(exist = false)
private String icon;
/**
 * 用户姓名
 */
@TableField(exist = false)
private String name;
```

### 2. 填充用户信息

```java
private void queryBlogUser(Blog blog) {
    Long userId = blog.getUserId();
    User user = userService.getById(userId);
    blog.setName(user.getNickName());
    blog.setIcon(user.getIcon());
}
```

### 3. 坑：商铺信息要预热

依旧是前面造成的问题：查看笔记会关联展示商铺信息，而商铺查询走的是**逻辑过期解决缓存击穿**的方案（Day26、27）。逻辑过期方案在**缓存 miss 时直接返回 null**（不阻塞重建），所以**商铺信息没预热进 Redis，第一次查就返回 null**，前端看不到商铺信息，直接提示"商铺不存在"。

> **教训：** 用逻辑过期的 key，上线前必须**预热**（先把商铺数据写进 Redis），否则冷启动期间查什么都是 null。

## 三、点赞功能

- 同一个用户只能点赞一次，再次点击则取消点赞
- 如果当前用户已经点赞，则点赞按钮高亮显示（前端已实现，判断 Blog 类的 `isLike` 字段）
### 1. 实现步骤

1. 给 Blog 类中添加一个 `isLike` 字段，标示是否被当前用户点赞
2. 修改点赞功能：利用 Redis 的 **set 集合**判断是否点赞过，未点赞则点赞数 +1，已点赞则点赞数 -1
3. 修改根据 id 查询 Blog 的业务，判断当前登录用户是否点赞过，赋值给 `isLike` 字段
4. 修改分页查询 Blog 业务，同样判断当前登录用户是否点赞过，赋值给 `isLike` 字段

### 2. likeBlog：点赞 / 取消点赞（Set 版）

```java
@Override
public Result likeBlog(Long id) {
    // 获取登录用户
    Long userId = UserHolder.getUser().getId();
    // 没有点赞，点赞+1
    String key = "blog:isliked:"+ id;
    Boolean isMember = stringRedisTemplate.opsForSet().isMember(key, userId.toString());
    if (BooleanUtil.isFalse(isMember)) {
        boolean isSuccess = update().setSql("liked = liked + 1").eq("id", id).update();
        // 加到缓存
        if (isSuccess) {
            stringRedisTemplate.opsForSet().add(key, userId.toString());
        }
    }else {
        // 已点赞，点赞-1
        boolean isSuccess = update().setSql("liked = liked - 1").eq("id", id).update();
        // 缓存中删除
        if (isSuccess) {
            stringRedisTemplate.opsForSet().remove(key, userId.toString());
        }
    }
    return Result.ok();
}
```

**核心思路：** 每篇笔记一个 Set，里面装所有点过赞的用户 id。
- `isMember` 判断用户在不在集合里 → 不在：数据库 `liked + 1` + `sadd` 把用户 id 加进集合
- 已在集合里：数据库 `liked - 1` + `srem` 把用户 id 移出集合

> **为什么用 Set？** 用户 id 天然要"去重"（一个用户只能点一次），Set 正好满足。

### 3. 查询笔记时填充 isLike

查询博客详情、分页查询热门博客时，都要判断**当前登录用户**是否点过赞，把结果赋给 `isLike` 字段，前端根据它决定按钮高亮状态：

```java
private void isBlogLiked(Blog blog) {
    // 1.获取登录用户
    Long userId = UserHolder.getUser().getId();
    // 2.用户未登录，无需判断点赞状态（不然空指针！见第四节坑①）
    if (userId == null) {
        return;
    }
    // 3.判断当前登录用户是否点赞过
    String key = "blog:isliked:" + blog.getId();
    Boolean isMember = stringRedisTemplate.opsForSet().isMember(key, userId.toString());
    blog.setIsLike(BooleanUtil.isTrue(isMember));
}
```

> **注意：** `isLike` 是"当前用户视角"的字段——同一篇笔记，你看是高亮、没点赞的人看就是灰色，所以每次查询都要按登录用户现查现赋值。

## 四、点赞排行榜（Set → SortedSet 升级）

### 1. 需求

在探店笔记详情页，把给这篇笔记点赞的人显示出来，比如**最早点赞的 TOP5**，形成点赞排行榜：

| 说明 | 内容 |
| --- | --- |
| 请求方式 | GET |
| 请求路径 | `/blog/likes/{id}` |
| 请求参数 | id：blog 的 id |
| 返回值 | `List<UserDTO>` 给这个笔记点赞的 TopN 用户集合 |

### 2. 为什么换成 SortedSet？
**新需求是"按点赞时间先后排序"，原来的 Set 满足不了：**

| | List | Set | SortedSet |
| --- | --- | --- | --- |
| **排序方式** | 按添加顺序排序 | 无法排序 | **根据 score 值排序** |
| **唯一性** | 不唯一 | 唯一 | 唯一 |
| **查找方式** | 按索引查找或首尾查找 | 根据元素查找 | 根据元素查找 |

**结论：既要"用户 id 去重"，又要"按时间排序" → 选 SortedSet。** 点赞时把**点赞时间戳当 score** 存进去：`ZADD key userId timestamp`。

### 3. 点赞逻辑升级：ZSet + 时间戳 score

`likeBlog` 改成 ZSet 操作，用 `zscore` 判断用户是否点过赞,**有值就返回 score，没有就 nil**

对应的 `isBlogLiked` 也从 `isMember` 改成 `zscore != null` 判断。

### 4. queryBlogLikes：查询点赞 Top5

```java
@Override
public Result queryBlogLikes(Long id) {
    String key = BLOG_LIKED_KEY + id;
    // 查top5点赞用户（按score升序，即最早点赞的前5个）
    Set<String> top5 = stringRedisTemplate.opsForZSet().range(key, 0, 4);
    if (top5 == null || top5.isEmpty()) {
        return Result.ok(Collections.emptyList());
    }
    // 解析用户id
    List<Long> ids = top5.stream().map(Long::valueOf).collect(Collectors.toList());
    // 查用户信息
    List<UserDTO> userDTOS = userService.listByIds(ids)
            .stream()
            .map(user -> BeanUtil.copyProperties(user, UserDTO.class))
            .collect(Collectors.toList());
    // 返回
    return Result.ok(userDTOS);
}
```

**分页查询博客时给每条笔记填充用户信息 + 点赞状态**（又复习了 lambda，`this::queryBlogUser`）：

```java
records.forEach(blog -> {
    this.queryBlogUser(blog);
    this.isBlogLiked(blog);
});
```

### 5. 坑①：开第二个窗口登录时空指针报错
```
Cannot invoke "com.hmdp.dto.UserDTO.getId()" because the return value of "com.hmdp.utils.UserHolder.getUser()" is null
```
**原因：** 第二个浏览器窗口没登录，`UserHolder.getUser()` 是 null，`isBlogLiked` 里直接取 id 就空指针了。

**解决：没有登录就不查询是否被点赞**——`isBlogLiked` 开头先判 `user == null` 直接 return
### 6. 坑②：in 查询结果不按传入顺序

**现象：** Redis 里 top5 是按点赞时间排好序的（传参也是按顺序），但 `listByIds` 查出来顺序不对。

**原因：** 用 `in(id, ...)` 查询时，MySQL 返回结果**不保证按传入 id 的顺序**（按主键索引/存储顺序），排行榜顺序就乱了。

**解决：** 手动 `order by field(id, ...)` 指定按传入顺序排：

```java
String idStr = StrUtil.join(",", ids);
List<UserDTO> userDTOS = userService.query()
        .in("id", ids)
        .last("ORDER BY FIELD(id," + idStr + ")")
        .list()
        .stream()
        .map(user -> BeanUtil.copyProperties(user, UserDTO.class))
        .collect(Collectors.toList());
```

> 自己拼 SQL ：`last("order by field(id, 5,1)")` 这种格式，保证查出来的顺序就是 Redis top5 的顺序。

---
## ⚠️ 避坑 Tips

### 1. 集群部署的坑
- **Nginx 轮询不感知节点存活**：一台 Tomcat 没开，请求照样会被分到它 → 请求时好时坏；要么都开，要么 nginx 配置只留一台
- **逻辑过期的商铺缓存要预热**：缓存 miss 直接返回 null，不预热前端就提示"商铺不存在"
### 2. 查询细节的坑
- **未登录用户 UserHolder 是 null**：填充 isLike 前先判 `user == null` 直接 return，否则空指针（UserDTO.getId() on null）
- **`in()` 查询不按传入 id 顺序**：排行榜类需要保序的场景，用 `.last("ORDER BY FIELD(id, ...)")` 手动指定
- **Blog 的 icon/name 要加 `@TableField(exist = false)`**：不然 MyBatis-Plus 会去表里找这两列，报"字段不存在"
---
## 🎯 面试题

### Q1：点赞功能怎么实现？为什么用 Redis 而不是直接写数据库？

> **回答框架：**
> 1. **需求**：一个用户只能点赞一次，再点取消；已点赞则按钮高亮
> 2. **为什么 Redis**：判断"该用户有没有点过赞"是高频操作，每次查数据库受不了；每篇笔记一个 Set（或 ZSet）存点赞用户 id，内存判断 O(1)
> 3. **流程**：Redis 判断在不在集合 → 不在就 `liked+1` + sadd；在就 `liked-1` + srem
> 4. **isLike 字段**：查询博客时按当前登录用户现查现赋值，前端决定按钮高亮——它是"用户视角"的，不是博客表字段
### Q2：点赞排行榜为什么从 Set 升级成 SortedSet？

> **回答框架：**
> 1. **新需求**：按点赞时间先后排 Top5 用户
> 2. **Set 无法排序**：三种结构对比——List 不唯一、Set 唯一但排不了序、SortedSet 唯一且按 score 排序
> 3. **方案**：ZSet 的 score 存点赞时间戳（System.currentTimeMillis()），`zrange(key,0,4)` 按 score 升序取最早点赞的 5 个用户
> 4. **判断点赞**：`zscore` 返回 score 值=已赞，返回 nil=没赞过
### Q3：MyBatis-Plus 的 in() 查询结果顺序和传入 id 顺序不一致，怎么解决？

> **回答框架：**
> 1. **现象**：top5 用户 id 在 Redis 里是排好序的，`listByIds(ids)` 查出来却乱序
> 2. **原因**：MySQL 的 `in` 查询不保证按传入 id 的顺序返回
> 3. **解决**：手动拼排序 SQL，`.last("ORDER BY FIELD(id,5,1,3...)")` 指定按传入顺序输出
> 4. **本质**：数据从 Redis 拿出来的是"业务顺序"，数据库查出来是"存储顺序"，中间必须用 ORDER BY FIELD 对齐
### Q4：Nginx 轮询负载均衡，一台后端宕机为什么会出问题？登录态不是共享的吗？

> **回答框架：**
> 1. **Nginx 轮询不感知节点存活**：宕机的节点照样会被分到请求，分到就连接失败，表现为时好时坏
> 2. **登录态确实共享**：token 存 Redis，哪台 Tomcat 都能取到用户——共享的只是"登录状态"
> 3. **应对**：要么后端节点全部在线，要么 nginx 配置里只保留存活节点


达人探店模块，记录第三十九天。
黑马点评 day19。今天开了新模块，感觉还是简单了有心情做下去啊也学得清楚明白。做了发布笔记、查看笔记、点赞、点赞排行榜四个功能。避雷啊，之前集群部署的坑，nginx 轮询一台开一台没开就有毛病，问ai感觉靠谱的是轮询不感知节点存活，有无大神讲一下。查看笔记还是之前因为逻辑过期的商铺缓存没预热，查的商铺都不存在。点赞功能用 Set 判断去重，到排行榜用 SortedSet，score 存点赞时间戳。还要两个 bug：未登录时空指针（UserHolder.getUser() 是 null，要先判空 return）、in 查询不按传入顺序（order by field 手动指定）。
