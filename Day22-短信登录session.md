---
tags:
  - 黑马点评
  - 短信登录
  - Session
  - 拦截器
---
# Day05 - 短信登录Session

## 一、整体流程

三个流程串起来就是完整的 Session 登录：

| 流程 | 做什么 |
|------|--------|
| **① 发送短信验证码** | 提交手机号 → 校验格式 → 生成验证码 → **存到 Session** → "发送"短信 |
| **② 验证码登录/注册** | 提交手机号+验证码 → 校验验证码 → 查库有没有这个手机号 → 没有就新建用户 → **用户存到 Session** |
| **③ 校验登录状态** | 每个请求带 cookie → 拦截器从 Session 取用户 → 有就放行（存 ThreadLocal），没有就 401 拦截 |

## 二、发送短信验证码

| 项    | 值            |
| ---- | ------------ |
| 请求方式 | POST         |
| 请求路径 | `/user/code` |
| 请求参数 | phone（电话号码）  |
| 返回值  | 无            |

> **思考：`/api/user/code` nginx 反向代理**
> 
> 前端请求 `http://127.0.0.1:8080/api/user/code?phone=xxx`，nginx 监听 8080，把 `/api/**` 的请求**转发到后端 8081 并去掉 `/api` 前缀**，所以后端 Controller 里写的是 `/user/code`。回顾一下苍穹外卖学的 nginx 反向代理。

### 业务代码
```java
public Result sendCode(String phone, HttpSession session) {
    // 1. 校验手机号格式
    if (RegexUtils.isPhoneInvalid(phone)) {
        return Result.fail("手机号格式错误");
    }
    // 2. 符合，生成6位随机验证码
    String code = RandomUtil.randomNumbers(6);
    // 3. 验证码保存到会话中
    session.setAttribute("code", code);
    // 4. 发送验证码（模拟，真实场景调短信服务商API）
    log.debug("发送短信验证码成功，验证码：{}", code);
    // 5. 返回ok
    return Result.ok();
}
```
> **疑问：Session 是什么？为什么 Controller 方法参数直接传 HttpSession 就能用？**
>
> **Session 是服务器端的会话存储**：浏览器第一次访问服务器时，服务器创建一个 Session 对象，通过 Cookie（`JSESSIONID`）返回给浏览器；之后浏览器每次请求都带上这个 cookie，服务器就能找到对应的 Session。
> **SpringMVC 自动注入**：方法参数里写 `HttpSession session`，Spring 会自动把当前请求对应的 Session 对象传进来，不用自己从 request 里取。

## 三、验证码登录/注册

### 业务代码
```java
public Result login(LoginFormDTO loginForm, HttpSession session) {
    // 1. 校验手机号格式
    if (RegexUtils.isPhoneInvalid(loginForm.getPhone())) {
        return Result.fail("手机号格式错误");
    }
    // 2. 校验验证码
    String cacheCode = session.getAttribute("code").toString();
    String code = loginForm.getCode();
    if (cacheCode == null || !code.equals(cacheCode)) {
        return Result.fail("验证码错误");
    }
    // 3. 根据手机号查询用户
    User user = query().eq("phone", loginForm.getPhone()).one();
    // 4. 不存在，去注册（创建新用户）
    if (user == null) {
        user = createUserWithPhone(loginForm.getPhone());
    }
    // 5. 保存用户信息到会话中
    session.setAttribute("user", user);
    return Result.ok();
}

// 私有方法：创建新用户
private User createUserWithPhone(String phone) {
    User user = new User();
    user.setPhone(phone);
    // 昵称 = 前缀 + 10位随机数字（如 "user_1234567890"）
    user.setNickName(SystemConstants.USER_NICK_NAME_PREFIX + RandomUtil.randomNumbers(10));
    save(user);
    return user;
}
```
### 思考
> **为什么登录时还要再校验一次手机号格式？发验证码时不是校验过了吗？**
>
> 你说得对，**每个独立请求都要校验**。因为发验证码和登录是两次请求，中间用户完全可以在前端把手机号改成别的（比如抓包改请求参数）。如果只靠发验证码时的校验，用户改了手机号就绕过了。这就是"**每个独立接口自己保证安全**"，不能依赖上一个请求做过什么。

>  **我觉得有一个 bug：发完验证码后改了手机号，凑巧新手机号在数据库里有，岂不是登录成功了？存 key 用 phone 行不行？**
>
> **你这个思考非常对，这确实是个安全漏洞！** 现在的实现：
> - `session.setAttribute("code", code)` —— key 是固定的 `"code"`，不是和手机号绑定的
> - 流程：A 用户给手机号 138xxxx 发了验证码 → session 里存了 code=123456 → 用户在前端改成 139xxxx（恰好已注册）→ 提交 139xxxx + 123456 → 验证码校验通过（code 对）→ 按 139xxxx 查用户 → **直接登录了别人的账号！**
>
> **你说的解决思路是对的：key 应该和手机号绑定**，比如：
> ```java
> // 发验证码时
> session.setAttribute("code:" + phone, code);
> // 登录时
> String cacheCode = (String) session.getAttribute("code:" + loginForm.getPhone());
> ```
> 这样取的是"当前提交手机号对应的验证码"，改了手机号就取不到了，自然校验失败。

## 四、登录校验拦截器

### 为什么需要拦截器？

不可能每个 Controller 方法都写一遍"从 Session 取用户、判断是否登录"。所以用**拦截器统一处理**：所有请求先过拦截器，拦截器从 Session 取用户，有就放行（存到 ThreadLocal），没有就 401 拦截。

### 拦截器代码
```java
public class LoginInterceptor implements HandlerInterceptor {
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        // 1. 获取session
        HttpSession session = request.getSession();
        // 2. 获取session中的用户
        Object user = session.getAttribute("user");
        // 3. 判断用户是否存在
        if (user == null) {
            response.setStatus(401);  // 未登录，返回401
            return false;             // 拦截
        }
        // 4. 用户存在，保存到ThreadLocal
        UserHolder.saveUser((User) user);
        return true;                  // 放行
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {
        // 移除用户（防止Tomcat线程池复用导致数据污染！）
        UserHolder.removeUser();
    }
}
```

>  **注意 afterCompletion 要 removeUser**
>
> 苍穹外卖用过的 `BaseContext`（ThreadLocal），这里 `UserHolder` 同理。Tomcat 处理请求用的是**线程池**，请求结束线程会归还复用。如果不 remove，下一个请求复用到这个线程，`ThreadLocal` 里还留着上一个用户的数据——就会出现"A 用户登录了，B 用户发请求却看到 A 的信息"这种串号 bug。**preHandle 存，afterCompletion 删，必须成对出现。**
### 注册拦截器 + 排除路径
```java
@Configuration
public class MvcConfig implements WebMvcConfigurer {
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new LoginInterceptor())
                .excludePathPatterns(
                    "/user/login",      // 登录本身不能拦
                    "/user/code",       // 发验证码不能拦
                    "/blog/hot",        // 热门博客游客可看
                    "/upload/**",       // 上传
                    "/shop-type/**",    // 店铺类型
                    "/shop/**",         // 店铺信息
                    "/voucher/**"       // 优惠券
                );
    }
}
```
### 遇到的问题：前端返回了密码、创建时间等敏感字段

> **原因**：存 Session 的时候直接把整个 `User` 对象存进去了（含 password、createTime、updateTime 等数据库字段），后续从 Session 取出来返回给前端就全暴露了。
>
> **解决**：只存需要的字段（id、头像、昵称）到 Session：
> ```java
> // 登录成功后，把User转成UserDTO再存session
> UserDTO userDTO = BeanUtil.copyProperties(user, UserDTO.class);
> session.setAttribute("user", userDTO);
> ```

开始还以为给的资料是错的，还把UserDTO改成了User，最后又改回来了。无所谓，重点在后面redis替代。

---
## ⚠️ 避坑 Tips

### 1. 数据找不到的坑
- **控制台打印 insert 成功，数据库里却找不到？** 别急，**可能只是数据库分页只显示了第一页**！打印 `Updates: 1` 说明数据确实插进去了，但数据库数据太多，新插入的数据在**第二页**。
### 2. Session 的坑
- **Session 存的对象要精简**：别直接存整个 User（密码等敏感字段会带到前端），用 DTO
- **ThreadLocal 必须配对 remove**：preHandle 存、afterCompletion 删，线程池复用不删就串号
---

## 🎯 面试题

### Q1：Session 登录的原理是什么？

> **回答框架：**
> 1. **Session 是服务器端会话存储**：浏览器首次访问，服务器创建 Session，通过 Cookie（JSESSIONID）返回给浏览器
> 2. **后续请求**：浏览器自动带上 JSESSIONID，服务器根据 id 找到对应 Session
> 3. **登录流程**：发验证码时 Session 存 code → 登录校验 code 通过 → Session 存用户信息 → 后续请求拦截器从 Session 取用户判断登录态
> 4. **本质**：Session 存在服务器内存，Cookie 存 SessionID，两者配合识别用户

### Q2：拦截器怎么实现登录校验？ThreadLocal 是干嘛的？

> **回答框架：**
> 1. **拦截器**：实现 `HandlerInterceptor`，重写 preHandle（前置）和 afterCompletion（后置）
> 2. **preHandle**：从 Session 取用户，null 就返回 401 拦截；有就存到 ThreadLocal 放行
> 3. **ThreadLocal**：线程局部变量，同一个请求的 Controller/Service 都能取到当前用户，不用层层传参
> 4. **afterCompletion remove**：Tomcat 线程池复用，不 remove 会导致下一个请求拿到上一个用户的数据（串号）
> 5. **注册**：在 MvcConfig（实现 WebMvcConfigurer）里 addInterceptor，excludePathPatterns 配白名单



今天不是前置喽，第二十二天。
黑马点评 day05，今天开始实战了，配置了一下nginx，然后今天只写了用 Session 登录的部分，学校里事还不少，要写一些材料，坚持写了这部分。写过苍穹外卖后确实对这些基本的更加熟悉了，整个流程还比较清楚。
