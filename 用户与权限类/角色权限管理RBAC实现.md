# 角色权限管理RBAC实现

## 1. 功能定义

`RBAC` 是 `Role-Based Access Control`，也就是基于角色的访问控制。

它要解决的问题不是“用户能不能登录”，而是：

- 一个用户拥有哪些角色
- 一个角色拥有哪些权限
- 一个用户最终拥有哪些权限

在工程化项目里，直接把权限挂到用户身上虽然也能做，但会很快失控。  
真正可维护的做法通常是：

1. 用户关联角色
2. 角色关联权限
3. 权限控制接口、菜单、按钮或数据范围

这就是最经典的 `RBAC` 模型。

## 2. RBAC 的核心关系

最基础的 `RBAC` 一般有 5 张核心表：

```json
{
  "sys_user": "用户表",
  "sys_role": "角色表",
  "sys_permission": "权限表",
  "sys_user_role": "用户角色关系表",
  "sys_role_permission": "角色权限关系表"
}
```

它们的关系可以理解成：

```text
用户 -> 角色 -> 权限
```

例如：

```json
{
  "user": "zhangsan",
  "roles": ["admin", "operator"],
  "permissions": ["user:list", "user:add", "order:list"]
}
```

## 3. 为什么要用 RBAC

因为它有几个明显好处：

1. 权限复用  
很多用户其实是同一类人，不需要每个人单独配权限。

2. 易于管理  
新增一个用户时，只要给他分配角色，不需要一条条分配权限。

3. 易于扩展  
后面要做菜单权限、按钮权限、数据权限时，也能继续挂在这个模型上。

## 4. 典型流程

一个 RBAC 功能通常会有下面几类操作：

1. 创建角色
2. 给角色分配权限
3. 给用户分配角色
4. 用户登录后加载角色和权限
5. 业务接口根据权限标识决定是否放行

所以这一篇最核心的能力其实有两个：

- 用户分配角色
- 根据用户查询权限

## 5. 核心数据示例

### 5.1 角色表

```json
[
  {
    "id": 1,
    "roleCode": "admin",
    "roleName": "管理员"
  },
  {
    "id": 2,
    "roleCode": "operator",
    "roleName": "运营人员"
  }
]
```

### 5.2 权限表

```json
[
  {
    "id": 1,
    "permissionCode": "user:list",
    "permissionName": "用户列表"
  },
  {
    "id": 2,
    "permissionCode": "user:add",
    "permissionName": "新增用户"
  }
]
```

### 5.3 用户角色关系表

```json
[
  {
    "userId": 1,
    "roleId": 1
  },
  {
    "userId": 1,
    "roleId": 2
  }
]
```

### 5.4 角色权限关系表

```json
[
  {
    "roleId": 1,
    "permissionId": 1
  },
  {
    "roleId": 1,
    "permissionId": 2
  }
]
```

## 6. 核心设计

在 Java 项目里，RBAC 最重要的设计点有 3 个：

1. 权限标识要稳定  
推荐使用 `user:list`、`order:create` 这种字符串标识。

2. 用户权限不要每次都层层查库  
登录后可以把角色和权限缓存到 Redis。

3. 修改角色或权限后，要刷新权限缓存  
否则用户实际权限和系统展示权限容易不一致。

## 7. 核心代码

## 7.1 给用户分配角色

这个功能通常出现在“用户管理”或“角色管理”后台页面。

### 请求示例

```json
{
  "userId": 1,
  "roleIds": [1, 2]
}
```

### Service 接口

```java
public interface UserRoleService {

    void assignRoles(Long userId, List<Long> roleIds);
}
```

### Service 核心实现

这段代码的核心思想是“先删后插”，保证最终角色关系是最新的。

```java
@Service
public class UserRoleServiceImpl implements UserRoleService {

    private final UserRoleMapper userRoleMapper;
    private final PermissionCacheService permissionCacheService;

    public UserRoleServiceImpl(
            UserRoleMapper userRoleMapper,
            PermissionCacheService permissionCacheService) {
        this.userRoleMapper = userRoleMapper;
        this.permissionCacheService = permissionCacheService;
    }

    @Override
    @Transactional(rollbackFor = Exception.class)
    public void assignRoles(Long userId, List<Long> roleIds) {
        userRoleMapper.deleteByUserId(userId);

        if (roleIds == null || roleIds.isEmpty()) {
            permissionCacheService.clearUserPermissionCache(userId);
            return;
        }

        for (Long roleId : roleIds) {
            userRoleMapper.insertRelation(userId, roleId);
        }

        permissionCacheService.clearUserPermissionCache(userId);
    }
}
```

### Mapper 核心方法

```java
public interface UserRoleMapper {

    void deleteByUserId(Long userId);

    void insertRelation(@Param("userId") Long userId, @Param("roleId") Long roleId);
}
```

这里不展开完整 XML，只强调核心逻辑：

- 删除原有角色关系
- 插入新的角色关系
- 清理用户权限缓存

## 7.2 给角色分配权限

这一步决定“某个角色能做什么”。

### 请求示例

```json
{
  "roleId": 1,
  "permissionIds": [1, 2, 3]
}
```

### Service 接口

```java
public interface RolePermissionService {

    void assignPermissions(Long roleId, List<Long> permissionIds);
}
```

### Service 核心实现

```java
@Service
public class RolePermissionServiceImpl implements RolePermissionService {

    private final RolePermissionMapper rolePermissionMapper;
    private final UserRoleMapper userRoleMapper;
    private final PermissionCacheService permissionCacheService;

    public RolePermissionServiceImpl(
            RolePermissionMapper rolePermissionMapper,
            UserRoleMapper userRoleMapper,
            PermissionCacheService permissionCacheService) {
        this.rolePermissionMapper = rolePermissionMapper;
        this.userRoleMapper = userRoleMapper;
        this.permissionCacheService = permissionCacheService;
    }

    @Override
    @Transactional(rollbackFor = Exception.class)
    public void assignPermissions(Long roleId, List<Long> permissionIds) {
        rolePermissionMapper.deleteByRoleId(roleId);

        if (permissionIds != null) {
            for (Long permissionId : permissionIds) {
                rolePermissionMapper.insertRelation(roleId, permissionId);
            }
        }

        List<Long> userIds = userRoleMapper.selectUserIdsByRoleId(roleId);
        permissionCacheService.clearUserPermissionCacheBatch(userIds);
    }
}
```

这段代码的重点不是“删除再新增”本身，而是最后一步：

- 角色权限一旦改变，所有挂在这个角色下的用户权限缓存都要刷新

这是很多人第一次做 RBAC 时容易漏掉的点。

## 7.3 根据用户查询权限

这一段会直接决定前一篇“认证鉴权”里权限判断是否准确。

### Service 接口

```java
public interface PermissionService {

    Set<String> getPermissionCodesByUserId(Long userId);
}
```

### Service 核心实现

推荐先查缓存，没有再查数据库。

```java
@Service
public class PermissionServiceImpl implements PermissionService {

    private static final String PERMISSION_KEY_PREFIX = "permission:user:";

    private final StringRedisTemplate stringRedisTemplate;
    private final PermissionMapper permissionMapper;

    public PermissionServiceImpl(
            StringRedisTemplate stringRedisTemplate,
            PermissionMapper permissionMapper) {
        this.stringRedisTemplate = stringRedisTemplate;
        this.permissionMapper = permissionMapper;
    }

    @Override
    public Set<String> getPermissionCodesByUserId(Long userId) {
        String key = PERMISSION_KEY_PREFIX + userId;
        Set<String> cacheSet = stringRedisTemplate.opsForSet().members(key);
        if (cacheSet != null && !cacheSet.isEmpty()) {
            return cacheSet;
        }

        Set<String> permissionSet = permissionMapper.selectPermissionCodesByUserId(userId);
        if (permissionSet != null && !permissionSet.isEmpty()) {
            stringRedisTemplate.opsForSet().add(key, permissionSet.toArray(new String[0]));
        }
        return permissionSet;
    }
}
```

### Mapper 核心方法

```java
public interface PermissionMapper {

    Set<String> selectPermissionCodesByUserId(Long userId);
}
```

这条 SQL 的核心思路其实就是多表关联：

```text
sys_user_role -> sys_role_permission -> sys_permission
```

最终查出一个用户拥有的所有 `permissionCode`。

## 7.4 权限缓存清理

这个类本身不复杂，但工程价值很高。

```java
public interface PermissionCacheService {

    void clearUserPermissionCache(Long userId);

    void clearUserPermissionCacheBatch(List<Long> userIds);
}
```

```java
@Service
public class PermissionCacheServiceImpl implements PermissionCacheService {

    private static final String PERMISSION_KEY_PREFIX = "permission:user:";

    private final StringRedisTemplate stringRedisTemplate;

    public PermissionCacheServiceImpl(StringRedisTemplate stringRedisTemplate) {
        this.stringRedisTemplate = stringRedisTemplate;
    }

    @Override
    public void clearUserPermissionCache(Long userId) {
        stringRedisTemplate.delete(PERMISSION_KEY_PREFIX + userId);
    }

    @Override
    public void clearUserPermissionCacheBatch(List<Long> userIds) {
        if (userIds == null || userIds.isEmpty()) {
            return;
        }
        List<String> keys = userIds.stream()
                .map(userId -> PERMISSION_KEY_PREFIX + userId)
                .toList();
        stringRedisTemplate.delete(keys);
    }
}
```

## 8. RBAC 与前一篇鉴权实现怎么衔接

前一篇里我们写过这种判断：

```java
boolean hasPermission = permissionService.hasPermission(loginUser.getUserId(), "user:list");
```

而这篇实际上是在解释这个权限是怎么来的：

1. 用户先关联角色
2. 角色再关联权限
3. 登录后或首次访问时，系统查出该用户的全部权限标识
4. 权限标识缓存到 Redis
5. 业务接口再基于这些权限做判断

也就是说：

- 认证鉴权篇负责“怎么拦截”
- RBAC 篇负责“权限数据怎么组织”

## 9. 常见扩展

RBAC 在真实项目里常常会继续扩展成下面几种类型。

### 9.1 菜单权限

控制一个菜单是否展示。

### 9.2 按钮权限

控制一个按钮是否可点击，例如“新增”“删除”“导出”。

### 9.3 接口权限

控制后端接口是否允许访问。

### 9.4 数据权限

控制“能看哪些数据”，例如只能看自己部门的数据。

通常前 3 种都可以挂在 `permissionCode` 上，数据权限则往往需要单独设计。

## 10. 常见坑点

### 10.1 只改数据库，不清缓存

这会导致权限明明改了，但用户仍然按旧权限运行。

### 10.2 一个用户一个权限硬编码

这样前期看着简单，后面一旦用户数量多、权限复杂，就很难维护。

### 10.3 权限标识不规范

建议统一成：

```text
模块:动作
```

例如：

- `user:list`
- `user:add`
- `order:refund`

### 10.4 角色和权限边界不清

角色是“身份集合”，权限是“动作能力”。  
不要把两者混成一张表。

### 10.5 超级管理员写死过多特殊逻辑

超级管理员可以特殊处理，但不要把各种权限判断都写成 `if admin then pass`，否则系统会越来越难维护。

## 11. 面试中常见问法

### 11.1 RBAC 是什么

回答重点：

- 基于角色的访问控制
- 用户不直接绑权限，而是先绑角色
- 角色再绑定权限

### 11.2 为什么要引入角色这一层

回答重点：

- 方便权限复用
- 方便管理
- 降低用户和权限的直接耦合

### 11.3 修改角色权限后为什么要清缓存

回答重点：

- 否则用户权限仍然是旧数据
- 会导致鉴权结果不准确

### 11.4 用户权限一般怎么查出来

回答重点：

- 用户角色关系表
- 角色权限关系表
- 权限表
- 多表关联查出 `permissionCode`

## 12. 总结

一个工程化的 `RBAC` 实现，核心不在于表有多少，而在于把“用户、角色、权限”三层关系梳理清楚，并让权限变更能正确影响到实际鉴权结果。

在 Java 项目里，最常见的做法就是：

- 用户绑定角色
- 角色绑定权限
- 登录后查询并缓存用户权限
- 接口通过权限标识做统一鉴权
- 角色或权限修改后刷新缓存

这套设计既足够清晰，也足够支撑大多数后台系统的权限管理需求。
