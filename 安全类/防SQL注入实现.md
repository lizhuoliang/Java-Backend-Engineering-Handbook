# 防SQL注入实现

## 1. 功能定义

防 SQL 注入要解决的问题是：

- 用户输入不可信数据时，系统如何避免这些输入被拼接进 SQL 语句，进而篡改查询逻辑或执行恶意数据库操作

SQL 注入是 Web 安全里最经典的问题之一。  
它之所以危险，是因为攻击者利用的不是数据库漏洞，而是：

- 开发者把用户输入直接拼进 SQL

所以防 SQL 注入的核心，不在数据库，而在应用代码层。

## 2. SQL 注入为什么会发生

最根本的原因通常是：

- 把用户输入当成 SQL 代码的一部分来执行了

例如这种危险写法：

```java
String sql = "select * from user where username = '" + username + "'";
```

如果用户输入：

- `' or 1=1 --`

那原本的 SQL 含义就被完全改掉了。

## 3. SQL 注入会造成什么后果

最常见的风险包括：

1. 越权查询数据
2. 修改或删除数据
3. 绕过登录条件
4. 批量探测数据库结构

所以 SQL 注入不是“小问题”，而是能够直接伤害数据安全的高风险漏洞。

## 4. 核心设计

防 SQL 注入实现时，建议至少抓住这几个原则：

1. 不手拼 SQL 参数
2. 一律使用参数化查询
3. 动态排序字段做白名单
4. 高风险关键字不过度信任前端
5. ORM 和 SQL 框架使用方式要规范

这里最关键的一个原则就是：

- 数据和 SQL 结构必须分离

## 5. 最基础的正确做法：参数化查询

在 Java 生态里，无论你是用：

- JDBC
- MyBatis
- JPA
- MyBatis-Plus

都应该优先使用参数化方式，而不是字符串拼接。

## 6. 核心代码

## 6.1 错误示例

```java
public List<User> queryByUsername(String username) {
    String sql = "select * from user where username = '" + username + "'";
    return jdbcTemplate.query(sql, userRowMapper);
}
```

这就是典型的注入风险代码。

## 6.2 正确示例：JDBC 参数化

```java
public List<User> queryByUsername(String username) {
    String sql = "select * from user where username = ?";
    return jdbcTemplate.query(sql, userRowMapper, username);
}
```

这里的关键就在于：

- `username` 不再参与 SQL 结构拼接
- 它只是参数

## 6.3 MyBatis 正确示例

```java
@Select("select * from user where username = #{username}")
User selectByUsername(@Param("username") String username);
```

这里用的是：

- `#{}` 占位

这通常是安全的参数绑定方式。

## 7. 为什么 MyBatis 里的 `${}` 很危险

这是 Java 后端里非常典型的一个坑。

### `#{}` 的含义

- 作为参数绑定
- 通常更安全

### `${}` 的含义

- 直接字符串替换

如果把用户输入直接放进 `${}`，就很容易造成注入风险。

例如：

```java
@Select("select * from user order by ${sortField}")
```

如果 `sortField` 来自前端，而且不做限制，就可能非常危险。

## 8. 排序字段为什么是注入高发区

很多系统在做分页排序时，会让前端传：

- 排序字段
- 排序方向

而开发者又很容易写成动态 SQL 字符串拼接。  
这时候即使业务查询条件本身用了参数化，也仍然可能在：

- `order by`

这里被注入。

正确做法通常是：

- 后端白名单映射允许排序的字段

例如：

```java
private static final Map<String, String> SORT_FIELD_MAP = Map.of(
    "createTime", "create_time",
    "id", "id"
);
```

## 9. 为什么 ORM 也不能让你完全放心

很多人以为：

- 用了 ORM 就绝对不会有 SQL 注入

这并不对。  
ORM 能帮你规避大部分参数拼接问题，但如果你：

- 手写动态 SQL
- 手拼原生 SQL
- 动态拼排序字段

依然可能把注入风险带回来。

所以真正的关键不是“用了什么框架”，而是：

- 你有没有把不可信输入直接拼进 SQL 结构

## 10. 常见防护策略

除了参数化查询之外，正式项目里通常还会做：

1. 输入校验  
例如字段长度、格式、枚举值范围。

2. 排序字段白名单
3. 最小权限数据库账号
4. 安全测试和代码审查

这意味着防 SQL 注入不是一条语句能彻底解决，而是：

- 编码规范 + 输入控制 + 权限最小化

的组合。

## 11. 常见坑点

### 11.1 动态 SQL 里混用 `${}` 和用户输入

这是最经典的风险之一。

### 11.2 排序字段完全信任前端

很多系统就是在这里出问题。

### 11.3 觉得用了 ORM 就不用管注入

手写 SQL 的地方依然可能有漏洞。

### 11.4 没有输入长度和格式约束

虽然不一定直接导致注入，但会增加攻击面。

## 12. 总结

防 SQL 注入的核心，不是“记住几个危险关键字”，而是从编码层面坚决做到：

- 用户输入永远只是参数
- 永远不要让不可信输入参与 SQL 结构拼接

一个成熟的防 SQL 注入实现至少应该做到：

- 全面参数化查询
- 动态字段白名单控制
- 高风险 SQL 编码规范化
- 关键模块定期审查

这才是真正稳定可落地的防注入实践。
