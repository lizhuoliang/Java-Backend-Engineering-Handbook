# 全文检索Elasticsearch实现

## 1. 功能定义

全文检索要解决的问题是：

- 当数据量很大、文本很多、搜索体验要求较高时，系统如何比数据库模糊查询更快、更准地找到相关内容

最典型的方案就是：

- `Elasticsearch`

它适合的场景通常包括：

- 文章搜索
- 商品搜索
- 知识库搜索
- 日志检索
- 简历搜索

和数据库 `like` 查询相比，全文检索更关注：

- 分词
- 相关度排序
- 高亮
- 大规模检索性能

## 2. 为什么数据库模糊搜索不够

因为数据库搜索通常更擅长：

- 精确匹配
- 结构化过滤

而全文检索更擅长：

- 文本分词
- 多字段相关性
- 大规模文本检索

例如用户搜：

- `Java 并发 实战`

如果用数据库 `like`，通常只能按字面做简单模糊查。  
而搜索引擎可以更自然地处理：

- 分词
- 匹配程度
- 结果排序

## 3. 全文检索的典型架构

一个常见的全文检索架构通常是：

1. 业务数据仍然存 MySQL
2. 搜索索引存 Elasticsearch
3. 数据变更时同步到 ES
4. 搜索请求优先走 ES

也就是说：

- MySQL 是事务主库
- ES 是查询加速和检索引擎

不要把 ES 当成唯一业务真相来源。

## 4. 核心设计

全文检索实现时，建议重点考虑：

1. 哪些字段需要建索引
2. 索引数据怎么同步
3. 搜索结果按什么排序
4. 是否需要高亮
5. 搜索结果回源数据库还是直接返回索引内容

其中“索引同步”是整个 ES 系统最核心、也最容易出问题的一环。

## 5. 典型索引文档结构

以文章搜索为例，一个索引文档可能长这样：

```json
{
  "id": 1,
  "title": "Java并发编程实战",
  "summary": "介绍线程池、锁和并发工具",
  "content": "完整正文内容",
  "status": 1,
  "createTime": "2026-06-02 10:00:00"
}
```

通常不会把所有数据库字段都原样塞进 ES，而是只保留搜索真正需要的字段。

## 6. 典型流程

1. 文章发布或更新
2. 系统同步数据到 ES
3. 用户输入搜索词
4. 搜索请求打到 ES
5. ES 返回结果、高亮和总数
6. 前端展示搜索结果

## 7. 核心代码

## 7.1 建索引核心实现

这里用 Spring Data Elasticsearch 风格举例。

```java
@Document(indexName = "article_index")
public class ArticleIndex {

    @Id
    private Long id;

    private String title;

    private String summary;

    private String content;

    private Integer status;

    private LocalDateTime createTime;

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public String getTitle() {
        return title;
    }

    public void setTitle(String title) {
        this.title = title;
    }

    public String getSummary() {
        return summary;
    }

    public void setSummary(String summary) {
        this.summary = summary;
    }

    public String getContent() {
        return content;
    }

    public void setContent(String content) {
        this.content = content;
    }

    public Integer getStatus() {
        return status;
    }

    public void setStatus(Integer status) {
        this.status = status;
    }

    public LocalDateTime getCreateTime() {
        return createTime;
    }

    public void setCreateTime(LocalDateTime createTime) {
        this.createTime = createTime;
    }
}
```

## 7.2 数据同步到 ES

```java
@Service
public class ArticleIndexServiceImpl implements ArticleIndexService {

    private final ArticleMapper articleMapper;
    private final ArticleIndexRepository articleIndexRepository;

    public ArticleIndexServiceImpl(
            ArticleMapper articleMapper,
            ArticleIndexRepository articleIndexRepository) {
        this.articleMapper = articleMapper;
        this.articleIndexRepository = articleIndexRepository;
    }

    @Override
    public void syncById(Long articleId) {
        Article article = articleMapper.selectById(articleId);
        if (article == null) {
            articleIndexRepository.deleteById(articleId);
            return;
        }

        ArticleIndex index = new ArticleIndex();
        index.setId(article.getId());
        index.setTitle(article.getTitle());
        index.setSummary(article.getSummary());
        index.setContent(article.getContent());
        index.setStatus(article.getStatus());
        index.setCreateTime(article.getCreateTime());
        articleIndexRepository.save(index);
    }
}
```

这段代码体现的是最基本的索引同步思路：

- 主库改了
- 索引也要跟着更新

## 7.3 搜索核心实现

```java
@Service
public class ArticleEsSearchServiceImpl implements ArticleEsSearchService {

    private final ElasticsearchOperations elasticsearchOperations;

    public ArticleEsSearchServiceImpl(ElasticsearchOperations elasticsearchOperations) {
        this.elasticsearchOperations = elasticsearchOperations;
    }

    @Override
    public PageResult<ArticleSearchVO> search(KeywordSearchRequest request) {
        if (!StringUtils.hasText(request.getKeyword())) {
            throw new BizException("搜索关键字不能为空");
        }

        NativeQuery query = NativeQuery.builder()
                .withQuery(q -> q.bool(b -> b
                        .must(m -> m.multiMatch(mm -> mm
                                .query(request.getKeyword())
                                .fields("title", "summary", "content")))))
                .withPageable(PageRequest.of(
                        Math.max(request.getPageNum().intValue() - 1, 0),
                        (int) Math.min(request.getPageSize(), 20L)))
                .build();

        SearchHits<ArticleIndex> hits = elasticsearchOperations.search(query, ArticleIndex.class);

        List<ArticleSearchVO> list = hits.getSearchHits().stream().map(hit -> {
            ArticleIndex source = hit.getContent();
            ArticleSearchVO vo = new ArticleSearchVO();
            vo.setId(source.getId());
            vo.setTitle(source.getTitle());
            vo.setSummary(source.getSummary());
            return vo;
        }).toList();

        return new PageResult<>(list, hits.getTotalHits(), request.getPageNum(), request.getPageSize());
    }
}
```

这段代码的关键点在于：

- 多字段匹配
- 分页查询
- 结果总数返回

## 8. 索引同步是这类系统的难点

全文检索里最核心的工程问题，不是“会不会查 ES”，而是：

- 数据怎么同步过去

常见同步方式包括：

1. 数据写库后同步写 ES
2. 发 MQ 异步更新 ES
3. 定时全量/增量重建索引

很多正式项目会组合使用：

- 主流程走异步消息同步
- 后台再配一套重建索引任务兜底

## 9. 高亮和相关度排序怎么理解

全文检索和数据库搜索最大的体验差异之一，就是它可以：

1. 告诉你为什么这条结果相关
2. 把命中的词高亮显示

例如搜索 `Java 并发` 时，可以把标题里的相关词高亮，用户体验会好很多。  
这也是 ES 这种搜索引擎真正发挥价值的地方。

## 10. 常见坑点

### 10.1 把 ES 当主数据库

这是非常危险的思路。  
ES 更适合作为查询和检索引擎，不适合承载核心事务真相。

### 10.2 索引字段无限膨胀

会增加索引体积和维护成本。

### 10.3 只建索引，不考虑同步失败

最后很容易出现：

- MySQL 是新数据
- ES 还是旧数据

### 10.4 搜索结果和业务权限脱节

即使搜索快，也可能把用户无权看的内容搜出来。

## 11. 总结

全文检索的核心，不是“上一个 ES 集群”，而是把它作为 MySQL 之外的一层专业文本检索能力来建设。

一个成熟的全文检索实现至少应该做到：

- 索引和主库职责分离
- 搜索字段有边界
- 数据同步机制可用
- 搜索结果支持分页与相关性

这样系统才能从“能查”真正走向“好搜、快搜、搜得准”。
