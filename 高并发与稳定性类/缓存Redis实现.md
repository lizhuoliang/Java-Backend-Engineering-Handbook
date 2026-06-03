# 缓存Redis实现

## 1. 功能定义

缓存要解决的问题是：

- 某些数据被频繁读取时，如何避免每次都直接打到数据库

在高并发系统里，数据库通常不是最先“扛不住”的唯一点，但它常常是最先感受到压力的点。  
尤其是下面这些场景：

- 热门商品详情
- 首页推荐列表
- 用户基础资料
- 配置类数据

如果每次请求都直接查数据库，系统很快就会在读压力下变慢甚至雪崩。  
所以缓存本质上是在“数据源”和“请求流量”之间加一层缓冲。

## 2. 为什么 Redis 会成为缓存标配

因为 Redis 有几个非常适合做缓存的特点：

1. 内存级访问速度快
2. 数据结构丰富
3. 支持过期时间
4. 支持原子操作
5. 生态成熟

它不只是“一个 key-value 工具”，而是高并发系统里最常见的基础组件之一。

## 3. 缓存最常见的价值

缓存通常用来解决三类问题：

1. 提升读性能  
减少数据库访问次数。

2. 降低后端压力  
让热点数据不每次都穿透到数据库。

3. 承担部分中间状态  
例如登录态、验证码、排行榜、限流计数器。

所以缓存并不只是“查库加速”，它还是很多系统能力的底层支撑。

## 4. 典型缓存流程

最常见的缓存读取流程通常如下：

1. 先查 Redis
2. Redis 有数据则直接返回
3. Redis 没数据则查数据库
4. 查到后写回 Redis
5. 返回结果

这就是经典的 `Cache Aside` 模式，也叫旁路缓存模式。

## 5. 核心设计

缓存实现时，建议先明确：

1. 哪些数据适合缓存
2. 缓存 key 怎么设计
3. 缓存多久过期
4. 数据更新后如何删缓存
5. 如何防止缓存穿透、击穿、雪崩

这 5 个问题基本决定了缓存系统能不能稳定工作。

## 6. 典型数据示例

例如缓存用户资料：

```json
{
  "key": "user:profile:101",
  "value": {
    "userId": 101,
    "nickname": "张三",
    "avatar": "https://cdn.example.com/a.png"
  },
  "ttl": 1800
}
```

一个好的 key 设计通常应该有这些特点：

- 可读
- 有业务前缀
- 能唯一定位对象

## 7. 核心代码

## 7.1 缓存查询核心实现

这里用“查询用户资料”举例。

```java
@Service
public class UserProfileCacheServiceImpl implements UserProfileCacheService {

    private static final String USER_PROFILE_KEY_PREFIX = "user:profile:";

    private final StringRedisTemplate stringRedisTemplate;
    private final UserMapper userMapper;

    public UserProfileCacheServiceImpl(
            StringRedisTemplate stringRedisTemplate,
            UserMapper userMapper) {
        this.stringRedisTemplate = stringRedisTemplate;
        this.userMapper = userMapper;
    }

    @Override
    public UserProfileVO getByUserId(Long userId) {
        String key = USER_PROFILE_KEY_PREFIX + userId;
        String cache = stringRedisTemplate.opsForValue().get(key);
        if (cache != null) {
            return JSON.parseObject(cache, UserProfileVO.class);
        }

        User user = userMapper.selectById(userId);
        if (user == null) {
            return null;
        }

        UserProfileVO vo = new UserProfileVO();
        vo.setUserId(user.getId());
        vo.setNickname(user.getNickname());
        vo.setAvatar(user.getAvatar());

        stringRedisTemplate.opsForValue().set(
                key,
                JSON.toJSONString(vo),
                Duration.ofMinutes(30)
        );

        return vo;
    }
}
```

这段代码展示了最基础的缓存旁路模式：

- 先查缓存
- 未命中查数据库
- 再回填缓存

## 7.2 更新后删缓存核心实现

```java
@Override
@Transactional(rollbackFor = Exception.class)
public void updateProfile(Long userId, UpdateProfileRequest request) {
    User update = new User();
    update.setId(userId);
    update.setNickname(request.getNickname());
    update.setAvatar(request.getAvatar());
    userMapper.updateById(update);

    stringRedisTemplate.delete(USER_PROFILE_KEY_PREFIX + userId);
}
```

这里最常见的策略是：

- 更新数据库后删除缓存

而不是直接强行更新缓存。  
因为大多数业务里，“删缓存”比“更缓存”更简单、更稳。

## 8. 为什么缓存和数据库一致性是难点

因为缓存和数据库是两个存储。  
只要是两个存储，就一定存在：

- 谁先更新
- 中间失败怎么办
- 并发下会不会读到旧值

大多数业务系统里，最常用、也最实用的策略通常是：

1. 先更新数据库
2. 再删除缓存

这不能保证绝对强一致，但对大多数读多写少场景已经足够实用。

## 9. 三个最经典的缓存问题

### 9.1 缓存穿透

查一个根本不存在的数据，每次都打到数据库。

常见解决方式：

- 空值缓存
- 布隆过滤器

### 9.2 缓存击穿

一个热点 key 过期瞬间，大量请求同时打到数据库。

常见解决方式：

- 热点 key 互斥重建
- 逻辑过期

### 9.3 缓存雪崩

大量 key 同时过期，导致大量请求一起打到数据库。

常见解决方式：

- 过期时间加随机值
- 多级缓存
- 降级兜底

## 10. 常见坑点

### 10.1 什么都缓存

缓存不是越多越好。  
频繁变化、命中率低的数据不一定适合缓存。

### 10.2 key 设计混乱

后面排查和维护会非常痛苦。

### 10.3 更新数据库后忘记删缓存

会导致长时间脏读。

### 10.4 不考虑空值缓存

容易被不存在的数据穿透数据库。

## 11. 总结

缓存实现的核心，不是“把数据放到 Redis”，而是围绕数据访问路径、过期策略、一致性和热点保护，建立一套真正能抗流量的读优化体系。

一个成熟的 Redis 缓存实现至少应该做到：

- key 设计规范
- 旁路缓存模式清晰
- 更新后删缓存
- 预防穿透、击穿、雪崩

这才是缓存真正的工程化价值。
