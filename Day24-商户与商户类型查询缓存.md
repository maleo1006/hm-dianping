---
tags:
  - 黑马点评
  - Redis
  - 缓存
---
# Day07 - 商户与商户类型查询缓存

## 一、什么是缓存

**缓存（Cache）就是数据交换的缓冲区**，是存数据的临时地方，一般读写性能较高。

多级缓存：浏览器缓存 → 应用层缓存（Tomcat）→ 数据库缓存 → CPU 缓存 → 磁盘缓存。

| 缓存的作用 | 缓存的成本 |
|-----------|-----------|
| 降低后端负载 | 数据一致性成本 |
| 提高读写效率，降低响应时间 | 代码维护成本、运维成本 |

## 二、商户查询缓存

### 缓存模型

**流程：** 请求先查 Redis → 命中直接返回；未命中 → 查数据库 → 写入 Redis。

根据 id 查商铺的完整流程：
1. 提交商铺 id → 从 Redis 查缓存
2. **命中** → 反序列化返回
3. **未命中** → 根据 id 查数据库
4. 数据库里**不存在** → 返回 404（错误信息）
5. 数据库里**存在** → 把商铺数据写入 Redis → 返回

### 业务代码
```java
public Result queryById(Long id) {
    // 1. 从缓存中查询
    String key = RedisConstants.CACHE_SHOP_KEY + id;
    String shopJson = stringRedisTemplate.opsForValue().get(key);
    // 2. 命中，直接返回
    if (StrUtil.isNotBlank(shopJson)) {
        Shop shop = JSONUtil.toBean(shopJson, Shop.class);
        return Result.ok(shop);
    }
    // 3. 未命中，查询数据库
    Shop shop = getById(id);
    if (shop == null) {
        return Result.fail("商铺不存在！");
    }
    // 4. 写入缓存
    stringRedisTemplate.opsForValue().set(key, JSONUtil.toJsonStr(shop));
    return Result.ok(shop);
}
```

> **我最开始还疑惑：接口不是写完了吗，前端已经拿到返回数据了啊，还写啥？**
>
> 后来想通了：**MyBatis-Plus 自带的 `getById()` 只是查数据库**，缓存逻辑要自己写
> ——先查 Redis，没有再查数据库并写回 Redis。这才是"加缓存"的业务。

**测试效果：查询速度从 1.19s 变成 45ms**，数据量越大效果越明显。

## 三、商户类型查询缓存（练习）

返回的是一个 List 集合，缓存的数据类型怎么选？

- **String 类型**：整个 List 序列化成一个 JSON 字符串存进去（省事）
- **List 类型**：每个对象序列化成一个字符串，遍历 push 进去

### 方案一：String 类型
```java
public Result queryTypeList() {
    String key = RedisConstants.CACHE_SHOP_TYPE_KEY;
    // 1. 查缓存
    String typeListJson = stringRedisTemplate.opsForValue().get(key);
    if (StrUtil.isNotBlank(typeListJson)) {
        List<ShopType> typeList = JSONUtil.toList(typeListJson, ShopType.class);
        return Result.ok(typeList);
    }
    // 2. 查数据库
    List<ShopType> typeList = this.query().orderByAsc("sort").list();
    // 3. 写缓存
    stringRedisTemplate.opsForValue().set(key, JSONUtil.toJsonStr(typeList));
    return Result.ok(typeList);
}
```

**测试：1.21s → 28ms。**

这样省事吧，List 要遍历一个一个添加。

### 方案二：List 类型（写着玩玩，看 Redis 里存什么格式）
```java
// 1. 查缓存（range 取整个 List）
String key = RedisConstants.CACHE_SHOP_TYPE_KEY;
List<String> jsonList = stringRedisTemplate.opsForList().range(key, 0, -1);
if (jsonList != null && !jsonList.isEmpty()) {
    // List<String> 转 List<ShopType>
    List<ShopType> typeList = new ArrayList<>();
    for (String json : jsonList) {
        typeList.add(JSONUtil.toBean(json, ShopType.class));
    }
    return Result.ok(typeList);
}
// 2. 查数据库
List<ShopType> typeList = this.query().orderByAsc("sort").list();
if (typeList == null || typeList.isEmpty()) {
    return Result.fail("查询商铺类型列表失败");
}
// 3. 遍历写入 List
for (ShopType shopType : typeList) {
    stringRedisTemplate.opsForList().rightPush(key, JSONUtil.toJsonStr(shopType));
}
return Result.ok(typeList);
```

### Redis 里 List 长什么样？

树形结构 `cache:shop:type` 下面是一行一个 JSON 对象。

>  emm... 实际用还是 String 方便：一个 key 一个值，直接 `JSONUtil.toList()` 还原成 List，不用遍历。List 类型适合"需要逐个进出队列"的场景，整列表缓存用 String 更省事。

## 后面课程要解决

### 1. 数据一致性的坑
- **改了数据库，缓存怎么办？** 现在的代码没有更新缓存的逻辑——商铺信息改了，Redis 里还是旧数据。后面会讲缓存更新策略
### 2. 缓存穿透的坑
- **数据库里没有的数据也要缓存空值**：现在的代码如果商铺不存在，直接返回 404 不写缓存——有人恶意刷不存在的 id，每次都打数据库。后面课程会讲（缓存空值 / 布隆过滤器）

记录第二十四天，商户查询缓存。
黑马点评day7，今天学了商户查询缓存，业务没多难，不难理解，但花的时间有点儿多，研究了redis存list类型数据，list要遍历转成type对象返回给前端，SpringMVC把`List<ShopType>`转 JSON 返回给前端的时候，自动加上`[]`。

[[Day25-缓存更新策略]]