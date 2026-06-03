# Webhook回调实现

## 1. 功能定义

Webhook 回调要解决的问题是：

- 当外部平台某个事件发生后，如何主动通知你的系统，而不是让你的系统不断轮询对方

最常见的 Webhook 场景包括：

- 支付成功回调
- 第三方物流状态回调
- GitHub 事件通知
- 企业平台审批结果回调
- 对接方业务状态异步通知

所以 Webhook 的核心不是“你去调用别人”，而是：

- 别人在关键事件发生时回调你

## 2. 为什么 Webhook 很常见

因为很多事件天然更适合“异步推送”而不是“你不停去问”。  
例如支付结果，如果你不断轮询平台：

- 既浪费资源
- 也不够及时

而 Webhook 的思路是：

- 一旦有结果，平台主动推给你

这是一种非常常见的开放平台协作方式。

## 3. Webhook 回调的核心风险

Webhook 虽然高效，但也有几个典型风险：

1. 回调来源不可信
2. 回调可能重复
3. 回调可能延迟
4. 回调处理失败后平台可能重试

所以 Webhook 处理最重要的不是“收到了”，而是：

- 如何安全、幂等、可恢复地处理收到的通知

## 4. 核心设计

Webhook 回调实现时，建议重点考虑：

1. 回调来源如何验证
2. 回调内容如何验签
3. 同一回调如何防重复处理
4. 回调处理失败怎么应对
5. 是否需要先落原始回调报文

其中非常关键的一点是：

- 回调报文原始内容最好先记录

因为后面排查问题时，这份原始回调数据非常有价值。

## 5. 典型流程

一个比较完整的 Webhook 回调处理流程通常如下：

1. 平台回调你的接口
2. 你的系统接收原始报文
3. 校验签名 / token / 来源
4. 按事件类型分发处理
5. 做幂等判断
6. 推进本地业务状态
7. 返回平台要求的响应

## 6. 核心代码

## 6.1 Webhook 接口入口

```java
@RestController
@RequestMapping("/webhook")
public class WebhookController {

    private final WebhookService webhookService;

    public WebhookController(WebhookService webhookService) {
        this.webhookService = webhookService;
    }

    @PostMapping("/event")
    public String handle(@RequestHeader Map<String, String> headers,
                         @RequestBody String body) {
        webhookService.handle(headers, body);
        return "success";
    }
}
```

这段代码看起来很薄，但它体现了 Webhook 入口的两个关键特征：

- 接收原始 headers
- 接收原始 body

因为签名校验很多时候就依赖原始报文。

## 6.2 Webhook 回调核心处理

```java
@Service
public class WebhookServiceImpl implements WebhookService {

    private final WebhookLogMapper webhookLogMapper;
    private final SignService signService;
    private final OrderService orderService;

    public WebhookServiceImpl(
            WebhookLogMapper webhookLogMapper,
            SignService signService,
            OrderService orderService) {
        this.webhookLogMapper = webhookLogMapper;
        this.signService = signService;
        this.orderService = orderService;
    }

    @Override
    @Transactional(rollbackFor = Exception.class)
    public void handle(Map<String, String> headers, String body) {
        String eventId = headers.get("x-event-id");
        String sign = headers.get("x-sign");

        if (!signService.verifyWebhook(body, sign)) {
            throw new BizException("Webhook验签失败");
        }

        if (webhookLogMapper.existsByEventId(eventId)) {
            return;
        }

        WebhookLog log = new WebhookLog();
        log.setEventId(eventId);
        log.setRawBody(body);
        log.setStatus("RECEIVED");
        log.setCreateTime(LocalDateTime.now());
        webhookLogMapper.insert(log);

        JSONObject json = JSON.parseObject(body);
        String eventType = json.getString("eventType");

        if ("ORDER_PAID".equals(eventType)) {
            orderService.handlePaidWebhook(json.getString("orderNo"));
        }

        WebhookLog update = new WebhookLog();
        update.setId(log.getId());
        update.setStatus("DONE");
        webhookLogMapper.updateById(update);
    }
}
```

这段代码体现了 Webhook 处理里最重要的几层：

1. 验签
2. 幂等
3. 原始报文留痕
4. 事件分发

## 7. 为什么 Webhook 一定要做幂等

因为很多平台在以下情况下会重试：

- 你的接口超时
- 你的接口返回失败
- 平台没收到你的确认响应

所以同一条回调可能会来多次。  
如果你不做幂等，就可能出现：

- 重复改状态
- 重复发货
- 重复发积分

最常见的幂等依据通常是：

- 外部事件 ID
- 外部业务流水号

## 8. 为什么原始回调报文最好落库

因为后续排查问题时，你经常需要回答：

- 平台当时到底发了什么
- 是我们验签失败，还是平台参数有问题
- 本地处理失败时原始报文是什么

如果回调只处理不留痕，后续排障会非常被动。

## 9. Webhook 和 MQ 的关系

很多成熟系统里，Webhook 入口通常不会直接做太重的业务逻辑。  
更常见的做法是：

1. 接收回调
2. 验签、落库
3. 投递内部 MQ
4. 异步处理业务

这样做的价值是：

- 回调接口更快响应
- 降低平台重试概率
- 把复杂业务从公网回调入口剥离出去

## 10. 常见坑点

### 10.1 不验签就直接处理业务

风险非常高。

### 10.2 不做幂等

重复回调会直接放大业务副作用。

### 10.3 回调入口做太重逻辑

容易超时，平台会不断重试。

### 10.4 不保留原始回调数据

后续排查和追溯会非常困难。

## 11. 总结

Webhook 回调实现的核心，不是“暴露一个接口给别人调”，而是围绕：

- 来源校验
- 报文验签
- 幂等处理
- 原始留痕
- 事件分发

建立一套安全、稳定、可恢复的异步集成入口。

一个成熟的 Webhook 实现至少应该做到：

- 先验签再处理
- 外部事件可幂等
- 原始报文可追踪
- 重逻辑尽量异步化

这才是真正可靠的回调集成方案。
