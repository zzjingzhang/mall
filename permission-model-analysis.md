# Mall 项目权限模型分析

## 1. 数据库表关系

项目采用**基于角色的权限控制（RBAC）**模型，涉及以下核心表：

### 1.1 核心实体表

| 表名 | 说明 | 关键字段 |
|------|------|----------|
| `ums_admin` | 后台用户表 | id, username, password, status |
| `ums_role` | 后台角色表 | id, name, description, status |
| `ums_menu` | 后台菜单表 | id, parent_id, title, name, icon, level |
| `ums_resource` | 后台资源表（接口权限） | id, name, url, category_id |

### 1.2 关系表

| 表名 | 说明 | 关系 |
|------|------|------|
| `ums_admin_role_relation` | 用户-角色关系表 | 多对多：一个用户可以有多个角色，一个角色可以有多个用户 |
| `ums_role_menu_relation` | 角色-菜单关系表 | 多对多：一个角色可以有多个菜单，一个菜单可以分配给多个角色 |
| `ums_role_resource_relation` | 角色-资源关系表 | 多对多：一个角色可以有多个资源权限，一个资源可以分配给多个角色 |

### 1.3 其他表（历史遗留/辅助）

| 表名 | 说明 |
|------|------|
| `ums_permission` | 权限表（历史遗留，新版本主要使用 resource） |
| `ums_admin_permission_relation` | 用户-权限直接关系表（除角色外的加减权限） |
| `ums_role_permission_relation` | 角色-权限关系表（历史遗留） |
| `ums_resource_category` | 资源分类表 |

### 1.4 ER 关系图

```
ums_admin ──(N:N)── ums_admin_role_relation ──(N:N)── ums_role
                                                    │
                                                    ├──(N:N)── ums_role_menu_relation ──(N:N)── ums_menu
                                                    │
                                                    └──(N:N)── ums_role_resource_relation ──(N:N)── ums_resource
```

---

## 2. 实体类、Mapper、Service、Controller 层

### 2.1 实体类（Model）

由 MyBatis Generator 自动生成，位置：`mall-mbg/src/main/java/com/macro/mall/model/`

- `UmsAdmin.java` - 后台用户实体
- `UmsRole.java` - 角色实体
- `UmsMenu.java` - 菜单实体
- `UmsResource.java` - 资源实体
- `UmsAdminRoleRelation.java` - 用户角色关系实体
- `UmsRoleMenuRelation.java` - 角色菜单关系实体
- `UmsRoleResourceRelation.java` - 角色资源关系实体

### 2.2 Mapper 层

- **自动生成 Mapper**：`mall-mbg/src/main/java/com/macro/mall/mapper/`
  - `UmsAdminMapper`
  - `UmsRoleMapper`
  - `UmsMenuMapper`
  - `UmsResourceMapper`
  - 各关系表 Mapper

- **自定义 DAO**：`mall-admin/src/main/java/com/macro/mall/dao/`
  - `UmsAdminRoleRelationDao` - 自定义查询
    - `getRoleList(adminId)`: 获取用户的角色列表
    - `getResourceList(adminId)`: 获取用户的资源权限列表（通过角色关联）
    - `getAdminIdList(resourceId)`: 获取拥有某资源的所有用户ID

### 2.3 Service 层

位置：`mall-admin/src/main/java/com/macro/mall/service/`

| Service | 主要职责 |
|---------|----------|
| `UmsAdminService` | 用户管理、登录、加载用户详情、获取用户资源列表 |
| `UmsRoleService` | 角色管理、分配菜单/资源 |
| `UmsMenuService` | 菜单管理 |
| `UmsResourceService` | 资源管理、列出所有资源（动态权限数据源） |
| `UmsAdminCacheService` | 权限缓存管理 |

### 2.4 Controller 层

位置：`mall-admin/src/main/java/com/macro/mall/controller/`

- `UmsAdminController` - 用户管理、登录、刷新 token
- `UmsRoleController` - 角色管理、分配菜单/资源
- `UmsMenuController` - 菜单管理
- `UmsResourceController` - 资源管理

---

## 3. 登录后权限列表如何加载

### 3.1 完整流程

1. **用户登录**：`UmsAdminController.login()`
2. **调用 Service**：`UmsAdminServiceImpl.login(username, password)`
3. **加载 UserDetails**：`loadUserByUsername(username)`

### 3.2 核心代码路径

**`UmsAdminServiceImpl.loadUserByUsername()`** (位置: `mall-admin/src/main/java/com/macro/mall/service/impl/UmsAdminServiceImpl.java:265-273`)

```java
@Override
public UserDetails loadUserByUsername(String username){
    // 获取用户信息（先查缓存，再查数据库）
    UmsAdmin admin = getAdminByUsername(username);
    if (admin != null) {
        // 获取用户的资源权限列表（先查缓存，再查数据库）
        List<UmsResource> resourceList = getResourceList(admin.getId());
        // 封装成 Spring Security 需要的 UserDetails
        return new AdminUserDetails(admin, resourceList);
    }
    throw new UsernameNotFoundException("用户名或密码错误");
}
```

### 3.3 资源列表查询

**`UmsAdminServiceImpl.getResourceList()`** (位置: `mall-admin/src/main/java/com/macro/mall/service/impl/UmsAdminServiceImpl.java:226-239`)

```java
@Override
public List<UmsResource> getResourceList(Long adminId) {
    // 1. 先从缓存中获取
    List<UmsResource> resourceList = getCacheService().getResourceList(adminId);
    if(CollUtil.isNotEmpty(resourceList)){
        return resourceList;
    }
    // 2. 缓存中没有，从数据库查询
    resourceList = adminRoleRelationDao.getResourceList(adminId);
    if(CollUtil.isNotEmpty(resourceList)){
        // 3. 存入缓存
        getCacheService().setResourceList(adminId, resourceList);
    }
    return resourceList;
}
```

### 3.4 SQL 查询（通过角色关联获取资源）

**`UmsAdminRoleRelationDao.xml`** (位置: `mall-admin/src/main/resources/dao/UmsAdminRoleRelationDao.xml:17-35`)

```sql
SELECT ur.id, ur.create_time, ur.name, ur.url, ur.description, ur.category_id
FROM ums_admin_role_relation ar
LEFT JOIN ums_role r ON ar.role_id = r.id
LEFT JOIN ums_role_resource_relation rrr ON r.id = rrr.role_id
LEFT JOIN ums_resource ur ON ur.id = rrr.resource_id
WHERE ar.admin_id = #{adminId}
AND ur.id IS NOT NULL
GROUP BY ur.id
```

### 3.5 权限封装

**`AdminUserDetails.getAuthorities()`** (位置: `mall-admin/src/main/java/com/macro/mall/bo/AdminUserDetails.java:28-34`)

```java
@Override
public Collection<? extends GrantedAuthority> getAuthorities() {
    // 将资源列表转换为 Spring Security 的 GrantedAuthority
    // 格式："resourceId:resourceName"，例如 "5:商品管理"
    return resourceList.stream()
            .map(resource -> new SimpleGrantedAuthority(resource.getId() + ":" + resource.getName()))
            .collect(Collectors.toList());
}
```

---

## 4. Controller 接口如何声明权限

**项目不使用 `@PreAuthorize` 或 `@Secured` 注解**，而是采用**基于数据库的动态权限配置**。

### 4.1 权限声明方式

权限不是在 Controller 上声明的，而是通过**数据库中的资源表**来定义：

1. **在数据库中定义资源**：`ums_resource` 表
   - `name`: 资源名称，如"商品管理"
   - `url`: 资源 URL 模式，使用 Ant 通配符，如 `/product/**`
   - `category_id`: 资源分类

2. **数据示例**：
   ```sql
   INSERT INTO ums_resource VALUES (5, '2020-02-04 17:09:16', '商品管理', '/product/**', NULL, 1);
   INSERT INTO ums_resource VALUES (8, '2020-02-05 14:43:37', '订单管理', '/order/**', '', 2);
   INSERT INTO ums_resource VALUES (25, '2020-02-07 16:47:34', '后台用户管理', '/admin/**', '', 4);
   ```

3. **为角色分配资源**：通过 `ums_role_resource_relation` 表

### 4.2 白名单配置

无需权限认证的接口通过配置文件声明：

**`application.yml`** (位置: `mall-admin/src/main/resources/application.yml:33-51`)

```yaml
secure:
  ignored:
    urls: #安全路径白名单
      - /admin/login
      - /admin/register
      - /admin/info
      - /admin/logout
      # ... 其他白名单
```

---

## 5. 动态权限校验在哪里完成

### 5.1 核心组件

动态权限校验由以下组件配合完成，位置在 `mall-security` 模块：

| 组件 | 位置 | 职责 |
|------|------|------|
| `JwtAuthenticationTokenFilter` | 过滤器 | 从请求中解析 JWT，加载用户权限到 SecurityContext |
| `DynamicSecurityFilter` | 过滤器 | 动态权限拦截，在请求到达前进行权限校验 |
| `DynamicSecurityMetadataSource` | 元数据源 | 根据请求路径匹配所需的资源权限 |
| `DynamicAccessDecisionManager` | 决策管理器 | 判断用户是否拥有所需权限 |
| `DynamicSecurityService` | 服务接口 | 加载所有资源与权限的映射关系 |

### 5.2 完整校验流程

```
请求到达
    ↓
JwtAuthenticationTokenFilter (解析 JWT，加载 UserDetails 和 Authorities 到 SecurityContext)
    ↓
DynamicSecurityFilter (动态权限拦截)
    ↓
DynamicSecurityMetadataSource.getAttributes() (根据请求 URL 匹配所需资源权限)
    ↓
DynamicAccessDecisionManager.decide() (对比用户权限和所需权限)
    ↓
有权限 → 放行到 Controller
无权限 → 抛出 AccessDeniedException
```

### 5.3 关键代码分析

#### 5.3.1 动态权限数据源加载

**`MallSecurityConfig.dynamicSecurityService()`** (位置: `mall-admin/src/main/java/com/macro/mall/config/MallSecurityConfig.java:35-48`)

```java
@Bean
public DynamicSecurityService dynamicSecurityService() {
    return new DynamicSecurityService() {
        @Override
        public Map<String, ConfigAttribute> loadDataSource() {
            Map<String, ConfigAttribute> map = new ConcurrentHashMap<>();
            // 从数据库获取所有资源
            List<UmsResource> resourceList = resourceService.listAll();
            for (UmsResource resource : resourceList) {
                // key: URL 模式 (如 /product/**)
                // value: 权限标识 (如 "5:商品管理")
                map.put(resource.getUrl(), 
                    new SecurityConfig(resource.getId() + ":" + resource.getName()));
            }
            return map;
        }
    };
}
```

#### 5.3.2 根据请求路径获取所需权限

**`DynamicSecurityMetadataSource.getAttributes()`** (位置: `mall-security/src/main/java/com/macro/mall/security/component/DynamicSecurityMetadataSource.java:34-52`)

```java
@Override
public Collection<ConfigAttribute> getAttributes(Object o) throws IllegalArgumentException {
    if (configAttributeMap == null) this.loadDataSource();
    List<ConfigAttribute> configAttributes = new ArrayList<>();
    
    // 获取当前请求路径
    String url = ((FilterInvocation) o).getRequestUrl();
    String path = URLUtil.getPath(url);
    PathMatcher pathMatcher = new AntPathMatcher();
    
    // 遍历所有资源，匹配 URL 模式
    Iterator<String> iterator = configAttributeMap.keySet().iterator();
    while (iterator.hasNext()) {
        String pattern = iterator.next();
        // Ant 风格匹配，例如 /product/** 匹配 /product/list, /product/1 等
        if (pathMatcher.match(pattern, path)) {
            configAttributes.add(configAttributeMap.get(pattern));
        }
    }
    return configAttributes;
}
```

#### 5.3.3 权限决策

**`DynamicAccessDecisionManager.decide()`** (位置: `mall-security/src/main/java/com/macro/mall/security/component/DynamicAccessDecisionManager.java:20-39`)

```java
@Override
public void decide(Authentication authentication, Object object,
                   Collection<ConfigAttribute> configAttributes) 
                   throws AccessDeniedException {
    // 接口未配置资源权限，直接放行
    if (CollUtil.isEmpty(configAttributes)) {
        return;
    }
    
    Iterator<ConfigAttribute> iterator = configAttributes.iterator();
    while (iterator.hasNext()) {
        ConfigAttribute configAttribute = iterator.next();
        // 所需权限（如 "5:商品管理"）
        String needAuthority = configAttribute.getAttribute();
        
        // 遍历用户拥有的权限，逐一比对
        for (GrantedAuthority grantedAuthority : authentication.getAuthorities()) {
            if (needAuthority.trim().equals(grantedAuthority.getAuthority())) {
                return; // 有权限，放行
            }
        }
    }
    throw new AccessDeniedException("抱歉，您没有访问权限");
}
```

#### 5.3.4 Security 过滤器链配置

**`SecurityConfig.filterChain()`** (位置: `mall-security/src/main/java/com/macro/mall/security/config/SecurityConfig.java:38-73`)

```java
@Bean
SecurityFilterChain filterChain(HttpSecurity httpSecurity) throws Exception {
    // ... 其他配置
    
    // 添加 JWT 过滤器
    .addFilterBefore(jwtAuthenticationTokenFilter, UsernamePasswordAuthenticationFilter.class);
    
    // 有动态权限配置时，添加动态权限过滤器
    if(dynamicSecurityService != null){
        registry.and().addFilterBefore(dynamicSecurityFilter, FilterSecurityInterceptor.class);
    }
    return httpSecurity.build();
}
```

---

## 6. 修改角色权限后，已登录用户权限是否会立即生效

**不会立即生效，但下一次请求时会生效。**

### 6.1 缓存清理机制

当角色权限发生变化时，系统会**主动清除相关用户的权限缓存**：

#### 6.1.1 修改角色资源权限时

**`UmsRoleServiceImpl.allocResource()`** (位置: `mall-admin/src/main/java/com/macro/mall/service/impl/UmsRoleServiceImpl.java:103-118`)

```java
@Override
public int allocResource(Long roleId, List<Long> resourceIds) {
    // 先删除原有关系
    UmsRoleResourceRelationExample example = new UmsRoleResourceRelationExample();
    example.createCriteria().andRoleIdEqualTo(roleId);
    roleResourceRelationMapper.deleteByExample(example);
    
    // 批量插入新关系
    for (Long resourceId : resourceIds) {
        UmsRoleResourceRelation relation = new UmsRoleResourceRelation();
        relation.setRoleId(roleId);
        relation.setResourceId(resourceId);
        roleResourceRelationMapper.insert(relation);
    }
    
    // 关键：清除拥有该角色的所有用户的资源缓存
    adminCacheService.delResourceListByRole(roleId);
    return resourceIds.size();
}
```

#### 6.1.2 清除缓存的具体实现

**`UmsAdminCacheServiceImpl.delResourceListByRole()`** (位置: `mall-admin/src/main/java/com/macro/mall/service/impl/UmsAdminCacheServiceImpl.java:58-68`)

```java
@Override
public void delResourceListByRole(Long roleId) {
    // 查询拥有该角色的所有用户
    UmsAdminRoleRelationExample example = new UmsAdminRoleRelationExample();
    example.createCriteria().andRoleIdEqualTo(roleId);
    List<UmsAdminRoleRelation> relationList = adminRoleRelationMapper.selectByExample(example);
    
    if (CollUtil.isNotEmpty(relationList)) {
        // 批量删除这些用户的资源列表缓存
        String keyPrefix = REDIS_DATABASE + ":" + REDIS_KEY_RESOURCE_LIST + ":";
        List<String> keys = relationList.stream()
            .map(relation -> keyPrefix + relation.getAdminId())
            .collect(Collectors.toList());
        redisService.del(keys);
    }
}
```

### 6.2 生效时机

| 场景 | 是否立即生效 | 说明 |
|------|-------------|------|
| 修改角色权限后，已登录用户**正在请求中** | ❌ 不立即生效 | 当前请求使用的是 SecurityContext 中的旧权限 |
| 修改角色权限后，已登录用户**发起新请求** | ✅ 新权限生效 | JWT 过滤器会重新调用 `loadUserByUsername()`，缓存已被清除，会从数据库查询最新权限 |

---

## 7. Redis 或内存中是否缓存了权限信息

**是的，使用了 Redis 缓存。**

### 7.1 缓存内容

缓存了两类数据：

| 缓存类型 | Redis Key 模式 | 内容 | 过期时间 |
|----------|----------------|------|----------|
| 用户信息 | `mall:ums:admin:{username}` | `UmsAdmin` 对象 | 24 小时 |
| 用户资源列表 | `mall:ums:resourceList:{adminId}` | `List<UmsResource>` | 24 小时 |

### 7.2 缓存配置

**`application.yml`** (位置: `mall-admin/src/main/resources/application.yml:25-31`)

```yaml
redis:
  database: mall
  key:
    admin: 'ums:admin'           # 用户信息缓存 Key 前缀
    resourceList: 'ums:resourceList'  # 资源列表缓存 Key 前缀
  expire:
    common: 86400 # 24小时
```

### 7.3 缓存服务

**`UmsAdminCacheService`** 接口定义了缓存操作：

| 方法 | 说明 |
|------|------|
| `getAdmin(username)` | 从缓存获取用户 |
| `setAdmin(admin)` | 将用户存入缓存 |
| `delAdmin(adminId)` | 删除用户缓存 |
| `getResourceList(adminId)` | 从缓存获取用户资源列表 |
| `setResourceList(adminId, list)` | 将资源列表存入缓存 |
| `delResourceList(adminId)` | 删除用户资源缓存 |
| `delResourceListByRole(roleId)` | 按角色删除相关用户的资源缓存 |
| `delResourceListByRoleIds(roleIds)` | 批量按角色删除 |
| `delResourceListByResource(resourceId)` | 按资源删除相关用户的资源缓存 |

### 7.4 内存缓存

除了 Redis 缓存外，**`DynamicSecurityMetadataSource` 还使用了静态内存缓存**：

**`DynamicSecurityMetadataSource`** (位置: `mall-security/src/main/java/com/macro/mall/security/component/DynamicSecurityMetadataSource.java:20-32`)

```java
// 静态变量，内存缓存所有资源的 URL-权限映射
private static Map<String, ConfigAttribute> configAttributeMap = null;

@PostConstruct
public void loadDataSource() {
    // 应用启动时加载一次到内存
    configAttributeMap = dynamicSecurityService.loadDataSource();
}

public void clearDataSource() {
    configAttributeMap.clear();
    configAttributeMap = null;
}
```

**注意**：这个内存缓存只在**应用启动时加载一次**，如果修改了 `ums_resource` 表（如新增资源、修改 URL），需要重启应用或手动调用 `clearDataSource()` 才能生效。

---

## 8. 完整流程图总结

### 8.1 登录流程

```
用户输入用户名密码
    ↓
UmsAdminController.login()
    ↓
UmsAdminServiceImpl.login()
    ↓
loadUserByUsername(username)
    ├─ getAdminByUsername(username) → Redis 缓存 → 数据库
    └─ getResourceList(adminId) → Redis 缓存 → 数据库 (多表关联查询)
    ↓
封装 AdminUserDetails (包含用户和资源权限列表)
    ↓
生成 JWT Token
    ↓
返回 Token 给前端
```

### 8.2 请求校验流程

```
前端携带 JWT 发起请求
    ↓
JwtAuthenticationTokenFilter
    ├─ 解析 JWT 获取 username
    ├─ loadUserByUsername(username) → Redis 缓存 → 数据库
    └─ 将 Authentication (含 Authorities) 存入 SecurityContext
    ↓
DynamicSecurityFilter
    ├─ DynamicSecurityMetadataSource: 根据 URL 匹配所需权限
    └─ DynamicAccessDecisionManager: 比对用户权限 vs 所需权限
    ↓
有权限 → Controller
无权限 → 403 Access Denied
```

### 8.3 修改角色权限流程

```
管理员修改角色的资源权限
    ↓
UmsRoleServiceImpl.allocResource()
    ├─ 删除旧的角色-资源关系
    ├─ 插入新的角色-资源关系
    └─ adminCacheService.delResourceListByRole(roleId)
        └─ Redis: 删除拥有该角色的所有用户的 resourceList 缓存
    ↓
已登录用户发起新请求
    ↓
JwtAuthenticationTokenFilter.loadUserByUsername()
    └─ getResourceList() → Redis 缓存不存在 → 从数据库查询最新权限
    ↓
用户获得最新权限
```
