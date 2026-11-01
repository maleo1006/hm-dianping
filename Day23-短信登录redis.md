---
tags:
  - 黑马点评
  - Redis
  - 短信登录
  - 拦截器
---
# Day06 - 短信登录Redis

## 一、Session 的问题：集群 Session 共享

**Session 共享问题：多台 Tomcat 并不共享 Session 存储空间，当请求切换到不同 Tomcat 服务时，导致数据丢失。**

> **我一开始没理解："丢失数据的场景？分发到不一样的 Tomcat，刚登录了下一秒就没有数据了吗？用户体验感不好。"**
>
> **对，就是这样。** 流程：
> 1. 用户登录，请求被 nginx 分到 Tomcat-A，Session 存在 A 的内存里
> 2. 用户刷新页面，nginx 负载均衡把请求分到 Tomcat-B
> 3. Tomcat-B 自己的 Session 里没有这个用户 → 又要登录
>
> 用户感觉就是"我明明登录了，怎么又让我登录？"——Session 在单机下没问题，一上集群就崩。

**Session 的替代方案要满足：**

| 要求 | Redis 为什么满足 |
|------|----------------|
| 数据共享 | 所有 Tomcat 连同一个 Redis，数据统一存 |
| 内存存储 | Redis 就是内存数据库，快 |
| key-value 结构 | 正好是 key-value |
**正中下怀。Redis：终于到我了！**

## 二、Redis 登录整体流程

### Redis 里存什么？

| key                 | value          | 说明                               |
| ------------------- | -------------- | -------------------------------- |
| `phone:138...`      | `9527`         | 验证码（**以手机号为 key**，不再是写死的 "code"） |
| `token:fadfjklfweo` | `{name: lisi}` | 用户信息（**以随机 token 为 key**）        |

### 流程对比 Session 版

| 步骤    | Session 版                            | Redis 版                                         |
| ----- | ------------------------------------ | ----------------------------------------------- |
| 发验证码  | `session.setAttribute("code", code)` | `redis.set("login:code:" + phone, code, 2, 分钟)` |
| 登录校验  | 从 session 取 code                     | 从 Redis 取 `login:code:phone` 对应的 code           |
| 存用户   | `session.setAttribute("user", user)` | 生成随机 token，存 `login:token:xxx → 用户信息`           |
| 返回前端  | session 自动通过 cookie 回传               | **返回 token 字符串，前端存起来**                          |
| 后续请求  | 浏览器自动带 JSESSIONID cookie             | 前端把 token 放到**请求头 authorization**               |
| 拦截器校验 | 从 session 取 user                     | 从请求头取 token → 从 Redis 查用户                       |
|       |                                      |                                                 |

>  **key 不是 code 了，而是 phone**——就是昨天发现的那个 bug 的解决方案！验证码和手机号绑定，发完改手机号也没用，因为 key 里就带着手机号。

> **这也是为什么设计的时候不用 phone 作为登录凭证**：
> - phone 是敏感信息，放请求头不安全
> - phone 可能被改、被重放
> - **token 是随机的**，无意义、不可预测、随时可以让它失效（删 Redis key）
## 三、核心代码

### 1. 前端把 token 放到请求头
```js
// request 拦截器，将用户 token 放入头中
let token = sessionStorage.getItem("token");
axios.interceptors.request.use(
    config => {
        if(token) config.headers['authorization'] = token;
        return config;
    },
    error => { return Promise.reject(error); }
)
```

> 前端每次请求自动在 header 加 `authorization: token值`，后端从 header 取。

### 2. 登录成功存 Redis
```java
// 5. 保存用户信息到 redis 中
// 5.1 随机生成 token，作为登录令牌
String token = UUID.randomUUID().toString(true);
// 5.2 将 User 对象转为 Hash 存储
UserDTO userDTO = BeanUtil.copyProperties(user, UserDTO.class);
Map<String, Object> userMap = BeanUtil.beanToMap(userDTO, new HashMap<>(),
        CopyOptions.create()
                .setIgnoreNullValue(true)
                .setFieldValueEditor((fieldName, fieldValue) -> fieldValue.toString()));
// 5.3 存到 redis 中
stringRedisTemplate.opsForHash().putAll(RedisConstants.LOGIN_USER_KEY + token, userMap);
// 5.4 设置过期时间
stringRedisTemplate.expire(RedisConstants.LOGIN_USER_KEY + token, RedisConstants.LOGIN_USER_TTL, TimeUnit.MINUTES);
// 6. 返回 token 给前端
return Result.ok(token);
```
**关键设计：**
- **token = UUID 随机字符串**：不可预测，当门禁卡
- **用户信息用 Hash 存**：`opsForHash()`，对应昨天学的 Hash 类型
- **`beanToMap` + CopyOptions**：把 UserDTO 转成 Map，方便 Hash 存储
- **过期时间 30 分钟**：用户不活跃就过期

## 四、双拦截器方案
### 为什么需要两个拦截器？

> 设置过期时间时，如果用户没有进行操作（没有访问），三十分钟后 token 过期，重新登录。那怎么知道用户有没有进行操作呢？→ **拦截器**

**问题：** 如果拦截器只判断"Redis 里有没有用户"，那每次请求都要把 Redis 里的过期时间重置（刷新 token），但拦截器配置了 `excludePathPatterns` 白名单（登录、发验证码、店铺等），**白名单路径不经过拦截器，就不会刷新 token**——用户一直逛店铺页，30 分钟后还是被踢下线。

### 解决方法：两个拦截器

| 拦截器 | 职责 | 拦截范围 | 做什么 |
|--------|------|---------|--------|
| **① RefreshTokenInterceptor**（刷新拦截器） | 刷新 token + 存 ThreadLocal | **拦截所有路径**（不排除任何） | 从 header 取 token → 从 Redis 查用户 → 存 ThreadLocal → **重置 token 过期时间** → 放行 |
| **② LoginInterceptor**（登录校验拦截器） | 判断是否登录 | 只拦需要登录的路径（白名单排除） | 从 ThreadLocal 取用户 → 没有就 401，有就放行 |

> 两个拦截器配合：① 负责"所有请求都经过我，刷新 token、存用户"；② 负责"需要登录的路径，没登录就拦"。这样**即使用户逛白名单路径（店铺页），请求也会经过 ①，token 就被刷新了**。

### 拦截器怎么注入 StringRedisTemplate？

> **思考：拦截器里不能通过注解注入 StringRedisTemplate，因为是自己 new 的对象，没有被 Spring 管理，所以只能是通过构造方法注入，在配置类里通过注解注入 StringRedisTemplate，作为参数放到构造方法。**
>
> **为什么 `@Resource` 不生效？** 拦截器是在 MvcConfig 里 `new LoginInterceptor()` 创建的，**这个对象不在 Spring 容器里**，Spring 不会给它做依赖注入。

> **正确做法：**
```java
@Configuration  
public class MvcConfig implements WebMvcConfigurer {  
    @Resource  
    private StringRedisTemplate stringRedisTemplate;  
    @Override  
    public void addInterceptors(InterceptorRegistry registry) {  
        registry.addInterceptor(new LoginInterceptor())  
                .excludePathPatterns(  
                        "/user/login",  
                        "/user/code",  
                        "/blog/hot",  
                        "/upload/**",  
                        "/shop-type/**",  
                        "/shop/**",  
                        "/voucher/**"  
                )  
                .order(1);  
        registry.addInterceptor(new RefreshTokenInterceptor(stringRedisTemplate)).addPathPatterns("/**").order(0);  
    }  
}
```


> **`@Resource` vs `@Autowired` 有什么区别？**
> - `@Autowired`：Spring 提供，**默认按类型**注入
> - `@Resource`：Java 标准注解（JSR-250），**默认按名称**注入，找不到才按类型
> - 功能差不多，都是自动装配，项目里混着用没区别

## 五、今天踩的坑

### 1. 连接不到 Redis

写完测试的时候，报错：连接不到 Redis。我在虚拟机里启动 Redis，还是报错。**依靠 AI 解决了这个问题：就是没关防火墙**，导致项目跨机器访问 Redis 连接不上。

```bash
# 关闭防火墙（CentOS 7）
systemctl stop firewalld
systemctl disable firewalld  # 禁止开机自启
```

### 2. ClassCastException 类型转换错误

接着又报了 `java.lang.ClassCastException`。

**原因：** 从 Redis 用 `opsForHash().entries(key)` 取出来的是 `Map<Object, Object>`，直接强转成 UserDTO 会类型转换错误。

**解决：** 取出来后用 `BeanUtil.toBean(map, UserDTO.class)` 转换成对象，别直接强转。

---

## ⚠️ 避坑 Tips

### 1. Redis 连接的坑
- **跨机器连不上 Redis = 先查防火墙**：CentOS 默认防火墙开着，6379 端口外部访问不了
- **Redis 服务要启动**：虚拟机里 `redis-server redis.conf`，不是启动一次就永远在跑
- **密码要配对**：redis.conf 设了 requirepass，Java 配置里 password 要对应

### 2. 序列化/类型转换的坑
- **Hash 取出来是 Map<Object,Object>**：别直接强转成 UserDTO，用 `BeanUtil.toBean()` 转
- **存的时候用 Hash，取的时候也要用 Hash**：`opsForHash()`，别用 `opsForValue()`
- **beanToMap 时字段是 Long/Integer**：CopyOptions 里 `setFieldValueEditor` 转成 String，不然 Hash 存的类型和取的对不上
---
## 🎯 面试题

### Q1：Redis 代替 Session 解决了什么问题？

> **回答框架：**
> 1. **Session 的问题**：存在 Tomcat 内存，多台服务器不共享，负载均衡到不同服务器就要重新登录
> 2. **Redis 方案**：所有 Tomcat 连同一个 Redis，登录数据统一存 Redis
> 3. **流程**：登录成功生成随机 token → 用户信息存 Redis（key=token）→ 返回 token 给前端 → 前端后续请求带 token → 拦截器从 Redis 查用户
> 4. **额外好处**：① 内存数据库快 ② token 随机不可预测更安全 ③ 可以灵活设置过期时间
> 5. **项目例子**：黑马点评短信登录，Redis 存验证码（key=phone）和用户（key=token）

### Q2：双拦截器方案为什么？怎么实现？

> **回答框架：**
> 1. **问题**：单拦截器配了 excludePathPatterns 白名单，白名单路径不经过拦截器，token 过期时间不刷新，用户逛店铺页还是会被踢
> 2. **方案**：两个拦截器
>    - ① RefreshTokenInterceptor：拦截**所有路径**，从 header 取 token → 查 Redis → 存 ThreadLocal → **重置过期时间** → 放行
>    - ② LoginInterceptor：拦截需要登录的路径，从 ThreadLocal 取用户，没有就 401
> 3. **顺序**：① order(0) 先执行，先存好 ThreadLocal 并刷新 token；② 后执行，只判断 ThreadLocal
> 4. **好处**：所有请求都刷新 token（用户活跃就不过期），登录校验只在需要的路径做

### Q3：拦截器里怎么注入 StringRedisTemplate？为什么不能 @Resource？

> **回答框架：**
> 1. **原因**：拦截器是在 MvcConfig 里 `new LoginInterceptor()` 创建的，不在 Spring 容器里，Spring 不会自动注入
> 2. **做法**：在 MvcConfig（Spring 管理的配置类）里 `@Resource` 注入 StringRedisTemplate，然后 `new LoginInterceptor(stringRedisTemplate)` 通过构造方法传进去
> 3. **扩展**：只要是自己 new 的对象，都不能用 Spring 注解注入，要么构造方法传，要么改造成 Spring Bean



打卡第二十三天，短信登录 Redis。
黑马点评 day06，今天把剩下的重要部分写完了。先理解"登录状态怎么存"这个问题本身，才能理解 Redis 方案解决了什么。以为写完第一个拦截器就可以了，但是还有个 bug，就是目前拦截器拦不到所有的路径，用户可能在访问，只不过访问的是没有拦截的，这样还是会三十分钟超时。解决方法是在前面再加一个拦截器，只做拦截一切路径放行，保存信息，后面拦截器就只需要查询 ThreadLocal 用户，不存在就拦截，存在就继续。
新加一个刷新 token 拦截器，只进行信息保存，不做拦截，写好后，修改登录拦截器，只需要判断 ThreadLocal 中有没有用户信息，没有就设置 401 并拦截，有则继续。最后把新加的拦截器配置 mvc，设置优先级为最高。
