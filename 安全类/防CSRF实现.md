# 防CSRF实现

## 1. 功能定义

防 CSRF 要解决的问题是：

- 已登录用户在不知情的情况下，被诱导从浏览器自动发出一笔“合法身份但非法意图”的请求

CSRF 的全称是：

- Cross-Site Request Forgery

它的核心危险点在于：

- 攻击者未必拿到了你的账号密码
- 但他利用了你“已经登录”的身份上下文

所以 CSRF 并不是伪造身份，而是：

- 借用受害者浏览器里的合法身份发恶意请求

## 2. 什么场景最容易受到 CSRF 影响

CSRF 最容易影响的场景通常是：

- 基于 Cookie 自动带会话的系统

例如传统 Web 后台里：

- 用户登录后浏览器自动带 Cookie
- 页面请求不额外确认来源

这时如果用户访问了恶意页面，对方就可能诱导浏览器向你的系统自动发请求。

## 3. 为什么不是所有系统都同样怕 CSRF

这和认证方式有关。

### 更容易受 CSRF 影响

- Cookie Session 模式

因为浏览器会自动带上 Cookie。

### 风险相对较低

- token 放在 `Authorization` 请求头里，由前端脚本主动设置

因为浏览器不会像 Cookie 那样自动帮你带这个头。

不过要注意：

- 风险低不等于绝对没有风险

系统仍然要根据实际认证方式做判断。

## 4. 核心设计

防 CSRF 实现时，建议重点考虑：

1. 当前系统认证方式是什么
2. 请求是否依赖 Cookie 自动认证
3. 哪些请求属于高风险写操作
4. 是否需要 CSRF Token
5. 是否校验 `Origin` / `Referer`

## 5. 最经典的防护思路：CSRF Token

CSRF Token 的基本思想是：

1. 服务端生成一个随机 token
2. 下发给前端页面
3. 前端发起敏感请求时，必须把这个 token 一起带上
4. 服务端校验 token 是否匹配

这样攻击者虽然能诱导浏览器带上 Cookie，但通常拿不到你页面里真正的 CSRF Token。

## 6. 核心代码

## 6.1 生成和保存 CSRF Token

```java
@Service
public class CsrfTokenServiceImpl implements CsrfTokenService {

    private static final String CSRF_TOKEN_KEY = "csrf:token:";

    private final StringRedisTemplate stringRedisTemplate;

    public CsrfTokenServiceImpl(StringRedisTemplate stringRedisTemplate) {
        this.stringRedisTemplate = stringRedisTemplate;
    }

    @Override
    public String generateToken(Long userId) {
        String token = UUID.randomUUID().toString().replace("-", "");
        stringRedisTemplate.opsForValue().set(
                CSRF_TOKEN_KEY + userId,
                token,
                Duration.ofHours(2)
        );
        return token;
    }

    @Override
    public void verifyToken(Long userId, String token) {
        String cacheToken = stringRedisTemplate.opsForValue().get(CSRF_TOKEN_KEY + userId);
        if (!Objects.equals(cacheToken, token)) {
            throw new BizException("CSRF 校验失败");
        }
    }
}
```

## 6.2 写操作接口校验示例

```java
@RestController
@RequestMapping("/user")
public class UserController {

    private final CsrfTokenService csrfTokenService;
    private final UserService userService;

    public UserController(CsrfTokenService csrfTokenService, UserService userService) {
        this.csrfTokenService = csrfTokenService;
        this.userService = userService;
    }

    @PostMapping("/update")
    public Result<Void> update(
            @RequestHeader("X-CSRF-Token") String csrfToken,
            @RequestBody UpdateProfileRequest request) {
        Long userId = UserContext.get().getUserId();
        csrfTokenService.verifyToken(userId, csrfToken);
        userService.updateProfile(userId, request);
        return Result.success();
    }
}
```

这里体现的是最经典的一条规则：

- 敏感写操作必须额外提交一个服务端认可的随机 token

## 7. 为什么 CSRF Token 必须和用户上下文绑定

因为如果只是系统全局一个固定 token，就失去意义了。  
CSRF Token 有效的前提是：

- 和当前用户会话或当前用户身份绑定

这样攻击者即使知道自己的 token，也不能拿去伪造别人的请求。

## 8. `Origin` 和 `Referer` 校验怎么理解

除了 CSRF Token 外，很多系统还会辅助校验：

- `Origin`
- `Referer`

目的就是判断：

- 这个请求是不是从合法站点页面发出来的

这不是绝对完美方案，但在很多后台系统里是很有效的辅助防线。

## 9. 为什么 GET 请求风险通常低于 POST/PUT/DELETE

从设计上说，GET 请求应该尽量只做：

- 查询

而不做：

- 改状态
- 删除
- 提交

如果系统把敏感写操作也做成 GET，那 CSRF 风险会被明显放大。  
所以：

- 写操作用 POST/PUT/DELETE
- 查询用 GET

本身也是安全设计的一部分。

## 10. 常见坑点

### 10.1 使用 Cookie 会话却不做任何 CSRF 防护

这在传统后台系统里非常危险。

### 10.2 敏感写操作做成 GET

会让风险明显上升。

### 10.3 CSRF Token 不和用户上下文绑定

安全价值会大幅下降。

### 10.4 以为用了 JWT 就绝对不需要考虑 CSRF

还要看 token 的存放位置和请求发起方式。

## 11. 总结

防 CSRF 的核心，不是“多加一个 header”，而是针对浏览器自动带认证上下文的特点，为敏感写操作增加一层“请求来源真实性”校验。

一个成熟的防 CSRF 实现至少应该做到：

- 明确当前认证模式
- 对敏感写操作增加 CSRF Token 校验
- 必要时辅助校验 Origin/Referer
- 严格区分查询和写操作

这才是真正有针对性的 CSRF 防护。
