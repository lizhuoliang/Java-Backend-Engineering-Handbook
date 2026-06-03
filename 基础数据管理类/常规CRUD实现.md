# 常规CRUD实现

## 1. 功能定义

`CRUD` 是后台系统里最基础的一类能力：

- `Create`：新增
- `Read`：查询
- `Update`：修改
- `Delete`：删除

它看起来简单，但工程化项目里真正要解决的不只是“增删改查能跑”，还包括：

- 接口分层是否清晰
- 参数校验是否完整
- 更新是否只改允许修改的字段
- 删除是物理删除还是逻辑删除
- 查询返回是否做了对象隔离

## 2. 典型场景

最常见的场景包括：

- 用户管理
- 商品管理
- 文章管理
- 配置管理
- 字典管理

也就是说，后面很多业务模块的底层骨架其实都是 CRUD。

## 3. 核心设计

一个工程化的 CRUD 通常至少要满足这几点：

1. `Controller` 只接请求，不堆业务逻辑
2. `Service` 负责业务校验和流程控制
3. `DTO` 和数据库实体分离
4. 删除优先考虑逻辑删除
5. 查询返回用 `VO`，不要直接把数据库实体原样暴露出去

## 4. 请求示例

### 4.1 新增

```json
{
  "name": "测试分类",
  "code": "test_category",
  "status": 1
}
```

### 4.2 查询详情返回

```json
{
  "id": 1,
  "name": "测试分类",
  "code": "test_category",
  "status": 1
}
```

## 5. 核心代码

## 5.1 Controller

```java
@RestController
@RequestMapping("/category")
public class CategoryController {

    private final CategoryService categoryService;

    public CategoryController(CategoryService categoryService) {
        this.categoryService = categoryService;
    }

    @PostMapping
    public Result<Void> add(@RequestBody @Valid CategoryAddRequest request) {
        categoryService.add(request);
        return Result.success();
    }

    @GetMapping("/{id}")
    public Result<CategoryVO> detail(@PathVariable Long id) {
        return Result.success(categoryService.detail(id));
    }

    @PutMapping("/{id}")
    public Result<Void> update(@PathVariable Long id, @RequestBody @Valid CategoryUpdateRequest request) {
        categoryService.update(id, request);
        return Result.success();
    }

    @DeleteMapping("/{id}")
    public Result<Void> delete(@PathVariable Long id) {
        categoryService.delete(id);
        return Result.success();
    }
}
```

## 5.2 Service 接口

```java
public interface CategoryService {

    void add(CategoryAddRequest request);

    CategoryVO detail(Long id);

    void update(Long id, CategoryUpdateRequest request);

    void delete(Long id);
}
```

## 5.3 Service 核心实现

```java
@Service
public class CategoryServiceImpl implements CategoryService {

    private final CategoryMapper categoryMapper;

    public CategoryServiceImpl(CategoryMapper categoryMapper) {
        this.categoryMapper = categoryMapper;
    }

    @Override
    public void add(CategoryAddRequest request) {
        Category category = new Category();
        category.setName(request.getName());
        category.setCode(request.getCode());
        category.setStatus(request.getStatus());
        categoryMapper.insert(category);
    }

    @Override
    public CategoryVO detail(Long id) {
        Category category = categoryMapper.selectById(id);
        if (category == null) {
            throw new BizException("数据不存在");
        }
        CategoryVO vo = new CategoryVO();
        vo.setId(category.getId());
        vo.setName(category.getName());
        vo.setCode(category.getCode());
        vo.setStatus(category.getStatus());
        return vo;
    }

    @Override
    public void update(Long id, CategoryUpdateRequest request) {
        Category exist = categoryMapper.selectById(id);
        if (exist == null) {
            throw new BizException("数据不存在");
        }

        Category category = new Category();
        category.setId(id);
        category.setName(request.getName());
        category.setStatus(request.getStatus());
        categoryMapper.updateById(category);
    }

    @Override
    public void delete(Long id) {
        Category exist = categoryMapper.selectById(id);
        if (exist == null) {
            throw new BizException("数据不存在");
        }
        categoryMapper.deleteById(id);
    }
}
```

## 6. 为什么 CRUD 不能写成“一个万能接口”

很多初学者喜欢把新增、修改、查询、删除全堆在一个超大接口里。  
这种做法短期看像省事，长期维护成本很高。

因为 CRUD 的每个动作关注点不同：

- 新增更关注默认值和唯一性
- 修改更关注字段边界和并发覆盖
- 删除更关注引用关系和逻辑删除
- 查询更关注对象脱敏和返回结构

所以工程上更推荐拆成标准 REST 风格接口。

## 7. 常见坑点

### 7.1 直接把数据库实体暴露给前端

这样容易把不该暴露的字段一起返回出去。

### 7.2 更新接口全量覆盖

如果前端没传某个字段，可能会被错误地覆盖成 `null`。

### 7.3 直接物理删除

很多业务数据后面需要追溯，所以删除前要先确认是否应该逻辑删除。

## 8. 总结

CRUD 是最基础的后端能力，但工程化的重点从来不是“能查到数据”，而是把新增、查询、修改、删除拆成职责清晰、边界明确、可维护的接口与服务结构。
