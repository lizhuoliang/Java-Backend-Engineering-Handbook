# 防XSS实现

## 1. 功能定义

防 XSS 要解决的问题是：

- 用户输入的内容在页面展示时，如何避免被当成恶意脚本执行

XSS 的全称是：

- Cross-Site Scripting

它本质上不是数据库层问题，也不是网络层问题，而是：

- 不可信输入被当成可执行内容输出到了页面里

所以防 XSS 的核心，是围绕：

- 输入
- 存储
- 输出

三层建立安全边界。

## 2. XSS 为什么危险

因为一旦恶意脚本在用户浏览器执行，可能造成：

1. 窃取 token 或 cookie
2. 冒充用户发请求
3. 篡改页面内容
4. 诱导用户操作

也就是说，XSS 不是“弹个 alert”这么简单，而是：

- 可以直接劫持用户侧上下文

## 3. XSS 最常见出现在哪里

最常见的高风险位置包括：

1. 富文本内容
2. 评论区
3. 用户昵称、签名、简介
4. 后台配置的公告内容

这些场景的共同特点是：

- 用户可输入
- 内容会被其他用户看到

所以 XSS 很多时候本质上是：

- 不可信内容跨用户传播

## 4. 常见 XSS 类型

### 4.1 存储型 XSS

恶意脚本先被存进数据库，再在页面展示时触发。  
这是业务系统里最常见、也最危险的一种。

### 4.2 反射型 XSS

恶意输入没入库，而是被接口立即回显到页面里。

### 4.3 DOM 型 XSS

前端脚本自己拼接 DOM 时引入风险。

在 Java 后端工程里，最常遇到的通常是：

- 存储型 XSS

## 5. 核心设计

防 XSS 实现时，建议重点考虑：

1. 哪些输入字段允许富文本
2. 哪些字段只允许纯文本
3. 富文本是否要白名单清洗
4. 展示时是否做转义
5. 接口返回给前端的内容是否可信

这里最重要的一点是：

- 不同字段的安全策略不能一样

例如：

- 用户昵称通常应该是纯文本
- 富文本正文则可能允许部分 HTML

## 6. 核心防护思路

XSS 防护通常有三层：

1. 输入校验  
限制明显不合理的输入。

2. 存储前清洗  
尤其是富文本场景。

3. 输出时转义或安全渲染  
确保不可信内容不被执行。

正式项目里，最常见的组合是：

- 普通文本字段做转义
- 富文本字段做白名单清洗

## 7. 核心代码

## 7.1 普通文本字段安全处理

对于昵称、标题、备注这类字段，通常不应该允许脚本标签直接进入。

```java
public class SafeTextUtil {

    public static String cleanPlainText(String text) {
        if (!StringUtils.hasText(text)) {
            return text;
        }
        return Jsoup.clean(text, Safelist.none());
    }
}
```

使用示例：

```java
@Service
public class UserProfileServiceImpl implements UserProfileService {

    private final UserMapper userMapper;

    public UserProfileServiceImpl(UserMapper userMapper) {
        this.userMapper = userMapper;
    }

    @Override
    public void updateNickname(Long userId, String nickname) {
        User update = new User();
        update.setId(userId);
        update.setNickname(SafeTextUtil.cleanPlainText(nickname));
        userMapper.updateById(update);
    }
}
```

这意味着：

- 用户昵称最终只保留纯文本

## 7.2 富文本白名单清洗

对于文章正文这类场景，不能简单全部去掉 HTML，因为它本来就要支持格式化内容。  
这时更合理的做法是：

- 允许一部分安全标签
- 清理掉危险标签和危险属性

```java
@Service
public class ArticleServiceImpl implements ArticleService {

    private final ArticleMapper articleMapper;

    public ArticleServiceImpl(ArticleMapper articleMapper) {
        this.articleMapper = articleMapper;
    }

    @Override
    public void save(ArticleSaveRequest request) {
        String safeContent = Jsoup.clean(
                request.getContent(),
                Safelist.relaxed()
                        .addTags("img")
                        .addAttributes("img", "src", "alt", "width", "height")
        );

        Article article = new Article();
        article.setTitle(SafeTextUtil.cleanPlainText(request.getTitle()));
        article.setContent(safeContent);
        articleMapper.insert(article);
    }
}
```

这段代码的关键思想是：

- 标题按纯文本处理
- 正文按富文本白名单处理

## 8. 为什么不能指望前端自己防 XSS

因为前端只能保护：

- 当前页面渲染方式

但真正决定“恶意内容有没有被存进去”的，仍然是后端。  
如果后端直接把危险内容原样入库，后续：

- 其他页面
- 其他前端
- 管理后台

仍然可能把它渲染出来。

所以防 XSS 必须包含：

- 后端输入/存储层治理

## 9. 输出转义为什么仍然重要

即使你做了输入清洗，也不意味着输出层完全可以不管。  
因为有些内容来源并不一定都经过了你当前服务的清洗逻辑。

所以更稳的思路是：

1. 存储前做清洗
2. 输出层按场景安全渲染

例如：

- 文本字段输出时默认 HTML 转义
- 富文本字段只在受控位置渲染

## 10. 常见坑点

### 10.1 所有内容都原样入库

这是存储型 XSS 最典型的来源。

### 10.2 富文本字段按纯文本策略处理

会导致功能不可用。

### 10.3 只信任前端过滤

后端仍然可能存入危险内容。

### 10.4 日志、预览、管理后台没有统一安全策略

同一份脏数据可能在别的页面再次触发。

## 11. 总结

防 XSS 的核心，不是“挡住 script 标签”这么简单，而是根据字段类型对输入、存储和输出建立不同的安全策略。

一个成熟的防 XSS 实现至少应该做到：

- 普通文本字段做纯文本清洗
- 富文本字段做白名单过滤
- 输出层按场景转义或安全渲染
- 不能只依赖前端防护

这才是真正可长期稳定执行的 XSS 防护体系。
