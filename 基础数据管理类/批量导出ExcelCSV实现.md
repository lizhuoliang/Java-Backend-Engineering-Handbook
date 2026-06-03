# 批量导出ExcelCSV实现

## 1. 功能定义

批量导出要解决的问题是：

- 把系统中的列表数据导出成用户可下载的文件

最常见的导出格式有：

- `Excel`
- `CSV`

典型场景包括：

- 导出用户列表
- 导出订单明细
- 导出统计报表

## 2. 核心设计

导出功能真正重要的点有 4 个：

1. 导出字段要可控
2. 导出条件要和列表查询一致
3. 大数据量导出要考虑性能
4. 文件名和编码要兼容浏览器下载

## 3. 典型流程

1. 前端传查询条件
2. 后端按同样条件查出数据
3. 转成导出行对象
4. 写入 `Excel` 或 `CSV`
5. 输出到响应流

## 4. 核心代码

## 4.1 CSV 导出核心实现

这里先用更容易理解的 `CSV` 举例。

```java
@GetMapping("/export")
public void export(CategoryQueryRequest request, HttpServletResponse response) throws IOException {
    List<CategoryVO> list = categoryService.listForExport(request);

    response.setContentType("text/csv;charset=UTF-8");
    response.setCharacterEncoding("UTF-8");
    response.setHeader("Content-Disposition", "attachment; filename=category.csv");

    try (PrintWriter writer = response.getWriter()) {
        writer.println("ID,名称,编码,状态");
        for (CategoryVO item : list) {
            writer.printf("%d,%s,%s,%d%n",
                    item.getId(),
                    item.getName(),
                    item.getCode(),
                    item.getStatus());
        }
    }
}
```

## 4.2 导出查询核心实现

```java
@Service
public class CategoryServiceImpl implements CategoryService {

    private final CategoryMapper categoryMapper;

    public CategoryServiceImpl(CategoryMapper categoryMapper) {
        this.categoryMapper = categoryMapper;
    }

    @Override
    public List<CategoryVO> listForExport(CategoryQueryRequest request) {
        List<Category> list = categoryMapper.selectList(new LambdaQueryWrapper<Category>()
                .like(StringUtils.hasText(request.getName()), Category::getName, request.getName())
                .eq(request.getStatus() != null, Category::getStatus, request.getStatus())
                .orderByDesc(Category::getId));

        return list.stream().map(item -> {
            CategoryVO vo = new CategoryVO();
            vo.setId(item.getId());
            vo.setName(item.getName());
            vo.setCode(item.getCode());
            vo.setStatus(item.getStatus());
            return vo;
        }).toList();
    }
}
```

## 5. 为什么导出条件要和列表条件一致

因为用户一般是在列表页筛选完条件之后再点导出。  
如果导出逻辑和列表逻辑不一致，就会出现：

- 页面上看到的是 A
- 导出的却是 B

这会让用户非常困惑。

所以导出最好复用同一套查询条件对象。

## 6. 大数据量导出怎么处理

如果数据量不大，直接查库后写文件就可以。  
如果数据量很大，就要考虑：

1. 分批查询
2. 流式写出
3. 异步导出
4. 导出任务中心

否则一次性查几十万条数据，内存和响应时间都会有压力。

## 7. 常见坑点

### 7.1 导出不走筛选条件

会导致结果和页面不一致。

### 7.2 文件编码处理不好

中文导出容易乱码。

### 7.3 一次性全量查库

大数据量场景下很危险。

## 8. 总结

批量导出的工程重点，不在“能不能生成文件”，而在查询条件复用、字段可控、编码兼容和大数据量策略。一个成熟的导出功能，必须既正确，又可承受真实数据规模。
