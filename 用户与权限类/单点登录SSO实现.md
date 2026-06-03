# 单点登录SSO实现

## 1. 功能定义

`SSO` 是 `Single Sign-On`，也就是单点登录。

它要解决的问题是：

- 用户在一个系统登录后，访问其他关联系统时不需要重复登录

例如公司里有：

- 用户中心
- 订单系统
- 财务系统
- 后台管理系统

如果每个系统都单独维护一套登录状态，用户体验和系统维护成本都会很差。  
这时候就需要 `SSO`。

## 2. 核心思想

单点登录的核心不是“多个系统共用一张用户表”这么简单，而是：

1. 认证集中到一个统一身份中心
2. 业务系统不自己做登录
3. 用户登录一次后，可以在多个系统之间共享登录状态

可以把它理解成：

```text
认证归中心，业务归各系统
```

## 3. 典型架构

一个简化版 SSO 架构通常有 3 类角色：

```json
{
  "ssoServer": "统一认证中心",
  "clientA": "业务系统A",
  "clientB": "业务系统B"
}
```

认证关系大致如下：

1. 用户访问业务系统 A
2. 业务系统发现用户未登录
3. 跳转到 SSO 认证中心
4. 用户在认证中心完成登录
5. 认证中心签发登录票据或授权码
6. 业务系统拿票据换取用户身份
7. 用户访问业务系统 B 时复用同一套登录状态

## 4. 常见实现方案

工程里常见有两种理解：

### 4.1 简化版内部 SSO

适用于公司内部系统，通常自己维护：

- 统一登录中心
- 票据或 token 校验接口

### 4.2 标准协议方案

例如：

- `OAuth2`
- `OpenID Connect`
- `CAS`

如果系统规模更大，或者要对接第三方，通常更推荐标准协议。

本文先讲最容易理解的“简化版内部 SSO”。

## 5. 典型流程

### 5.1 首次访问业务系统

1. 用户访问业务系统
2. 系统发现本地未登录
3. 跳转到 SSO 登录页，并带上回调地址
4. 用户在 SSO 中登录成功
5. SSO 生成一次性 `ticket`
6. 浏览器跳回业务系统回调地址
7. 业务系统拿 `ticket` 向 SSO 换取用户信息
8. 业务系统创建自己的本地会话

### 5.2 访问第二个业务系统

1. 用户访问系统 B
2. 系统 B 也跳到 SSO
3. SSO 发现浏览器里已有登录态
4. 无需再次输入密码，直接签发新的 `ticket`
5. 系统 B 换取用户信息，完成本地登录

## 6. 核心代码

## 6.1 SSO 登录成功后生成 ticket

这里不展开整个认证中心，只保留最关键的一步：生成一次性票据并缓存。

```java
@Service
public class SsoTicketService {

    private static final String TICKET_PREFIX = "sso:ticket:";

    private final StringRedisTemplate stringRedisTemplate;

    public SsoTicketService(StringRedisTemplate stringRedisTemplate) {
        this.stringRedisTemplate = stringRedisTemplate;
    }

    public String createTicket(Long userId, String username) {
        String ticket = UUID.randomUUID().toString().replace("-", "");
        String value = JSON.toJSONString(Map.of(
                "userId", userId,
                "username", username
        ));
        stringRedisTemplate.opsForValue().set(
                TICKET_PREFIX + ticket,
                value,
                Duration.ofMinutes(3)
        );
        return ticket;
    }
}
```

这里的 `ticket` 一般要满足两个特点：

- 短时有效
- 一次性使用

## 6.2 业务系统通过 ticket 换取用户信息

```java
@Service
public class SsoClientService {

    private final StringRedisTemplate stringRedisTemplate;

    public SsoClientService(StringRedisTemplate stringRedisTemplate) {
        this.stringRedisTemplate = stringRedisTemplate;
    }

    public LoginUser validateTicket(String ticket) {
        String key = "sso:ticket:" + ticket;
        String value = stringRedisTemplate.opsForValue().get(key);
        if (value == null) {
            throw new BizException("ticket无效或已过期");
        }

        stringRedisTemplate.delete(key);

        JSONObject jsonObject = JSON.parseObject(value);
        return new LoginUser(
                jsonObject.getLong("userId"),
                jsonObject.getString("username")
        );
    }
}
```

这里最关键的一步是：

- 校验成功后立刻删除 `ticket`

这样可以保证一次性使用，避免票据被重放。

## 6.3 业务系统回调处理

```java
@RestController
@RequestMapping("/sso")
public class SsoCallbackController {

    private final SsoClientService ssoClientService;
    private final AuthService authService;

    public SsoCallbackController(
            SsoClientService ssoClientService,
            AuthService authService) {
        this.ssoClientService = ssoClientService;
        this.authService = authService;
    }

    @GetMapping("/callback")
    public Result<Map<String, Object>> callback(@RequestParam String ticket) {
        LoginUser loginUser = ssoClientService.validateTicket(ticket);
        return Result.success(authService.createLocalSession(loginUser));
    }
}
```

这一步的含义是：

- SSO 负责证明“你是谁”
- 当前业务系统负责建立自己的本地登录态

## 7. 为什么业务系统还要创建本地会话

因为单点登录并不意味着所有业务请求都直接回认证中心处理。

真实项目里通常是：

1. SSO 完成统一认证
2. 各业务系统基于认证结果建立自己的本地会话或本地 token

这样后续访问效率更高，也方便各系统做自己的权限控制。

## 8. 单点退出的核心思路

单点登录往往还要配套“单点退出”。

也就是：

- 用户在任意一个系统退出
- 其他系统也要同步失效

常见做法有两种：

1. 前端跳转通知所有系统退出
2. 认证中心广播登出事件，由各系统清理本地会话

如果没有单点退出，就会出现“明明退出了 A，B 里还在线”的问题。

## 9. 常见坑点

### 9.1 把共享用户表当成 SSO

共用用户表只能说明账号一致，不等于单点登录。

### 9.2 ticket 不做一次性消费

这样可能被重复使用，存在安全风险。

### 9.3 ticket 有效期过长

票据越长时间有效，泄露风险越高。

### 9.4 业务系统不建立本地会话

会导致每次访问都强依赖认证中心，性能和可用性都会受影响。

## 10. 面试中常见问法

### 10.1 SSO 是什么

回答重点：

- 单点登录
- 一次登录，多系统通行
- 认证集中到统一身份中心

### 10.2 SSO 的核心流程是什么

回答重点：

- 未登录跳转认证中心
- 登录成功签发 ticket
- 客户端系统拿 ticket 换用户信息
- 再建立本地会话

### 10.3 为什么 ticket 要一次性使用

回答重点：

- 防止重放攻击
- 缩短票据暴露风险

## 11. 总结

单点登录的核心，不是“多个系统都能查到同一个用户”，而是把认证能力统一收敛到身份中心，再让各业务系统基于认证结果建立自己的登录态。

一个可落地的 SSO 方案，至少要把这几件事处理好：

- 统一登录入口
- ticket 或授权码机制
- 业务系统换取用户身份
- 本地会话创建
- 单点退出联动

这也是多系统架构里非常典型的一项工程化能力。
