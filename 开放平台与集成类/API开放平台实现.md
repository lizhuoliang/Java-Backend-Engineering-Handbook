# API开放平台实现

## 1. 功能定义

API 开放平台要解决的问题是：

- 你的系统如何把一部分能力规范、安全、可管理地开放给外部调用方使用

最常见的开放对象包括：

- 商户
- 合作方系统
- 第三方开发者
- 内部其他业务线

它和普通内部接口最大的区别在于：

- 调用方不再是你自己完全可控的前端或服务
- 而是外部主体

所以开放平台本质上是在做：

- 对外能力产品化
- 对外访问治理化

## 2. 为什么开放平台不能等同于“把接口暴露出去”

因为一旦对外开放，系统马上要面对很多内部接口不需要考虑的问题：

1. 调用方身份管理
2. 鉴权与签名
3. 权限与配额
4. 接口版本管理
5. 调用日志与审计
6. 限流与风控

所以开放平台不是“多几个接口”，而是：

- 多一层平台治理

## 3. 一个开放平台通常包含什么

一个相对完整的 API 开放平台，通常至少包括：

1. 应用管理  
谁来调用你的接口。

2. `appId / appSecret` 体系
3. 接口文档与调试能力
4. 调用鉴权和签名
5. 限流与配额管理
6. 调用日志和监控

如果规模更大，还会继续扩展：

7. 沙箱环境
8. 版本治理
9. 订阅和回调能力

## 4. 核心设计

开放平台实现时，建议重点考虑：

1. 平台开放哪些能力
2. 调用方如何注册和审核
3. 每个调用方能访问哪些接口
4. 签名和限流规则怎么设计
5. 出问题后如何审计和封禁

这里最关键的一点通常是：

- 开放平台必须把“调用方”当成一等公民建模

也就是说，你不能只关注接口本身，还必须关注：

- 谁在调
- 调得怎么样
- 调用权限和额度是多少

## 5. 典型数据结构

### 5.1 调用方应用信息

```json
{
  "appId": "merchant_1001",
  "appSecret": "xxxxxx",
  "appName": "某合作商户",
  "status": "ENABLE"
}
```

### 5.2 应用接口授权关系

```json
{
  "appId": "merchant_1001",
  "apiCode": "order.query",
  "status": "ENABLE"
}
```

这两层非常关键：

1. 谁能调用
2. 能调用哪些接口

## 6. 典型调用流程

一个标准开放平台调用流程通常如下：

1. 调用方构造请求
2. 带上 `appId`、时间戳、nonce、sign
3. 平台校验签名
4. 校验应用状态
5. 校验接口权限
6. 校验限流和配额
7. 执行业务接口
8. 记录调用日志

## 7. 核心代码

## 7.1 开放平台鉴权拦截器

```java
@Component
public class OpenApiAuthInterceptor implements HandlerInterceptor {

    private final AppAuthService appAuthService;
    private final ApiPermissionService apiPermissionService;
    private final OpenApiLogService openApiLogService;

    public OpenApiAuthInterceptor(
            AppAuthService appAuthService,
            ApiPermissionService apiPermissionService,
            OpenApiLogService openApiLogService) {
        this.appAuthService = appAuthService;
        this.apiPermissionService = apiPermissionService;
        this.openApiLogService = openApiLogService;
    }

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) {
        String appId = request.getHeader("X-App-Id");
        String sign = request.getHeader("X-Sign");
        String timestamp = request.getHeader("X-Timestamp");
        String nonce = request.getHeader("X-Nonce");

        appAuthService.verify(appId, sign, timestamp, nonce, request);

        String apiCode = request.getRequestURI();
        apiPermissionService.checkPermission(appId, apiCode);

        openApiLogService.recordAccess(appId, apiCode, request.getRemoteAddr());
        return true;
    }
}
```

这段代码体现的是开放平台入口治理的核心骨架：

1. 先鉴权
2. 再校验接口权限
3. 再留调用日志

## 7.2 平台应用权限校验

```java
@Service
public class ApiPermissionServiceImpl implements ApiPermissionService {

    private final OpenApiPermissionMapper openApiPermissionMapper;

    public ApiPermissionServiceImpl(OpenApiPermissionMapper openApiPermissionMapper) {
        this.openApiPermissionMapper = openApiPermissionMapper;
    }

    @Override
    public void checkPermission(String appId, String apiCode) {
        boolean allowed = openApiPermissionMapper.existsEnabledPermission(appId, apiCode);
        if (!allowed) {
            throw new BizException("当前应用无权访问该接口");
        }
    }
}
```

## 8. 为什么开放平台一定要有调用日志

因为外部调用问题通常比内部接口问题更难排。  
你经常需要知道：

- 哪个 appId 调了什么接口
- 调用了多少次
- 成功率如何
- 出错时传了哪些关键参数

所以开放平台调用日志通常不是“可选项”，而是：

- 治理核心数据

## 9. 为什么开放平台几乎一定要限流

因为外部调用方的行为比内部前端更不可控。  
一旦某个合作方写错代码、疯狂重试、或者被攻击，就可能导致：

- 你的平台被打爆

所以开放平台最常见的一条规则就是：

- 按 appId 做限流

甚至进一步做：

- 按接口限流
- 按日调用量配额

## 10. 开放平台为什么需要版本治理

因为对外接口一旦发布，就不像内部接口那样说改就改。  
外部调用方可能已经深度接入了你的字段和语义。

所以开放平台通常更强调：

1. 向后兼容
2. 接口版本号
3. 废弃通知机制

否则你改一个字段，可能会影响很多对接方。

## 11. 常见坑点

### 11.1 把内部接口直接暴露成开放接口

安全和治理都会非常薄弱。

### 11.2 只有 appId / secret，没有接口权限控制

调用边界会过大。

### 11.3 没有调用日志和配额治理

外部调用问题很难排查。

### 11.4 接口一发布就随意改参数

会严重影响合作方接入稳定性。

## 12. 总结

API 开放平台的核心，不是“对外开放几个接口”，而是围绕：

- 调用方身份
- 签名鉴权
- 接口权限
- 限流配额
- 调用审计

建立一整套对外能力治理体系。

一个成熟的开放平台实现至少应该做到：

- appId/appSecret 模型清晰
- 接口权限可配置
- 签名规则统一
- 调用日志可追踪
- 限流和版本治理到位

这才是真正可长期演进的 API 开放平台架构。
