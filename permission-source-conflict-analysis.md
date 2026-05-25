# mall 项目接口权限注解与动态资源权限一致性分析

## 1. 使用 @PreAuthorize 或类似注解的接口

### 1.1 查找结果

**结论：项目中没有使用任何注解式权限控制。**

经过全面搜索，项目中**不存在**以下注解：
- `@PreAuthorize`
- `@PostAuthorize`
- `@Secured`
- `@RolesAllowed`

### 1.2 代码验证

查看了多个核心 Controller，如：
- `PmsBrandController.java`（商品品牌管理）
- `UmsAdminController.java`（后台用户管理）
- `UmsRoleController.java`（角色管理）

这些 Controller 的方法上只使用了：
- `@ApiOperation`（Swagger 文档注解）
- `@RequestMapping/@GetMapping/@PostMapping`（路由映射）
- `@ResponseBody`

**没有任何权限注解。**

---

## 2. 数据库资源权限加载逻辑

### 2.1 核心架构

项目采用的是**完全基于数据库的动态权限系统**，核心组件如下：

| 组件 | 文件路径 | 职责 |
|------|----------|------|
| `DynamicSecurityFilter` | `mall-security/src/main/java/com/macro/mall/security/component/DynamicSecurityFilter.java` | 动态权限过滤器，拦截请求 |
| `DynamicSecurityMetadataSource` | `mall-security/src/main/java/com/macro/mall/security/component/DynamicSecurityMetadataSource.java` | 从数据源加载权限配置 |
| `DynamicAccessDecisionManager` | `mall-security/src/main/java/com/macro/mall/security/component/DynamicAccessDecisionManager.java` | 权限决策管理器 |
| `DynamicSecurityService` | `mall-security/src/main/java/com/macro/mall/security/component/DynamicSecurityService.java` | 动态权限服务接口 |

### 2.2 权限加载流程

#### 步骤 1: 系统启动时加载资源权限

在 `MallSecurityConfig.java:35-48` 中定义了动态权限数据源：

```java
@Bean
public DynamicSecurityService dynamicSecurityService() {
    return new DynamicSecurityService() {
        @Override
        public Map<String, ConfigAttribute> loadDataSource() {
            Map<String, ConfigAttribute> map = new ConcurrentHashMap<>();
            List<UmsResource> resourceList = resourceService.listAll();
            for (UmsResource resource : resourceList) {
                map.put(resource.getUrl(), 
                    new org.springframework.security.access.SecurityConfig(
                        resource.getId() + ":" + resource.getName()));
            }
            return map;
        }
    };
}
```

**关键点：**
- 调用 `resourceService.listAll()` 从数据库查询所有资源
- 权限映射关系：`URL -> "resourceId:resourceName"`
- 例如：`/brand/** -> "1:商品品牌管理"`

#### 步骤 2: 用户登录时加载用户拥有的权限

在 `AdminUserDetails.java:29-34` 中：

```java
@Override
public Collection<? extends GrantedAuthority> getAuthorities() {
    return resourceList.stream()
            .map(resource -> new SimpleGrantedAuthority(
                resource.getId() + ":" + resource.getName()))
            .collect(Collectors.toList());
}
```

**关键点：**
- 用户的权限列表也是 `resourceId:resourceName` 格式
- 从用户角色关联的资源表中查询

#### 步骤 3: 请求时进行权限匹配

在 `DynamicSecurityMetadataSource.java:34-52` 中：

```java
@Override
public Collection<ConfigAttribute> getAttributes(Object o) throws IllegalArgumentException {
    if (configAttributeMap == null) this.loadDataSource();
    List<ConfigAttribute> configAttributes = new ArrayList<>();
    String url = ((FilterInvocation) o).getRequestUrl();
    String path = URLUtil.getPath(url);
    PathMatcher pathMatcher = new AntPathMatcher();
    Iterator<String> iterator = configAttributeMap.keySet().iterator();
    while (iterator.hasNext()) {
        String pattern = iterator.next();
        if (pathMatcher.match(pattern, path)) {
            configAttributes.add(configAttributeMap.get(pattern));
        }
    }
    return configAttributes;  // 未匹配到返回空集合
}
```

**关键点：**
- 使用 Ant 路径匹配（`/brand/**` 匹配 `/brand/list`、`/brand/create` 等）
- **未配置权限的路径返回空集合**

#### 步骤 4: 权限决策

在 `DynamicAccessDecisionManager.java:20-39` 中：

```java
@Override
public void decide(Authentication authentication, Object object,
                   Collection<ConfigAttribute> configAttributes) 
    throws AccessDeniedException, InsufficientAuthenticationException {
    
    // 当接口未被配置资源时直接放行
    if (CollUtil.isEmpty(configAttributes)) {
        return;
    }
    
    Iterator<ConfigAttribute> iterator = configAttributes.iterator();
    while (iterator.hasNext()) {
        ConfigAttribute configAttribute = iterator.next();
        String needAuthority = configAttribute.getAttribute();
        for (GrantedAuthority grantedAuthority : authentication.getAuthorities()) {
            if (needAuthority.trim().equals(grantedAuthority.getAuthority())) {
                return;
            }
        }
    }
    throw new AccessDeniedException("抱歉，您没有访问权限");
}
```

**关键点：**
- **空权限集合 = 直接放行！**
- 匹配到权限配置后，比较用户权限字符串是否完全相等

### 2.3 数据库表结构

资源表 `ums_resource`：

| 字段 | 类型 | 说明 |
|------|------|------|
| id | bigint | 资源 ID（主键） |
| name | varchar(200) | 资源名称 |
| url | varchar(200) | 资源 URL（Ant 模式） |
| description | varchar(500) | 描述 |
| category_id | bigint | 资源分类 ID |

---

## 3. 权限字符串的来源是否统一

### 3.1 权限字符串格式

权限字符串格式为：`resourceId:resourceName`

例如：
- `"1:商品品牌管理"`
- `"25:后台用户管理"`
- `"5:商品管理"`

### 3.2 两个生成点

**生成点 1: 资源权限配置（访问所需权限）**

位置：`MallSecurityConfig.java:43`

```java
map.put(resource.getUrl(), 
    new SecurityConfig(resource.getId() + ":" + resource.getName()));
```

来源：从数据库 `ums_resource` 表加载

**生成点 2: 用户权限（用户拥有的权限）**

位置：`AdminUserDetails.java:32`

```java
.map(resource -> new SimpleGrantedAuthority(
    resource.getId() + ":" + resource.getName()))
```

来源：从用户-角色-资源关联表加载

### 3.3 来源分析

| 生成点 | 数据来源 | 格式 |
|--------|----------|------|
| 资源权限配置 | `ums_resource` 表 | `id:name` |
| 用户权限 | `ums_admin_role_relation` → `ums_role_resource_relation` → `ums_resource` | `id:name` |

**结论：权限字符串的来源是统一的。**

两个生成点最终都从 `ums_resource` 表获取数据：
- 资源权限配置：直接从 `ums_resource` 读取所有资源
- 用户权限：通过角色关联，最终也指向 `ums_resource` 表

---

## 4. 数据库资源配置和代码注解不一致时以谁为准

### 4.1 基本结论

**不存在"数据库配置与代码注解不一致"的问题。**

原因：
1. **项目没有使用代码注解**：没有 `@PreAuthorize`、`@Secured` 等注解
2. **权限完全依赖数据库**：所有权限规则都从 `ums_resource` 表加载
3. **权限决策完全基于动态加载**：`DynamicAccessDecisionManager` 只比较从数据库加载的权限字符串

### 4.2 如果未来添加代码注解会怎样？

假设未来在某个方法上添加了：

```java
@PreAuthorize("hasAuthority('1:商品品牌管理')")
@RequestMapping("/brand/list")
public CommonResult getList() { ... }
```

**此时会有两套权限检查机制：**

1. **动态权限过滤器**（先执行）：根据数据库配置进行检查
2. **方法级注解**（后执行）：如果方法上有注解，Spring Security 的方法拦截器会再次检查

**优先级：**
- 如果动态权限过滤器放行，注解检查仍会执行
- 两者是**AND**关系，必须都通过

---

## 5. 可能导致误放行或误拦截的具体场景

### 5.1 场景一：新增接口但未配置数据库资源 → **误放行**（高风险）

#### 场景描述

**前提条件：**
1. 数据库 `ums_resource` 表中没有配置 `/newApi/**` 的资源
2. 开发者新增了一个 Controller：

```java
@Controller
@RequestMapping("/newApi")
public class NewApiController {
    
    @RequestMapping("/sensitiveData")
    @ResponseBody
    public CommonResult getSensitiveData() {
        // 返回敏感数据
        return CommonResult.success(sensitiveData);
    }
}
```

#### 权限检查流程

1. 用户访问 `/newApi/sensitiveData`
2. `DynamicSecurityMetadataSource` 尝试匹配 URL
3. 数据库中没有 `/newApi/**` 或 `/newApi/sensitiveData` 的配置
4. 返回**空集合** `configAttributes`
5. `DynamicAccessDecisionManager` 检查：
   ```java
   if (CollUtil.isEmpty(configAttributes)) {
       return;  // 直接放行！
   }
   ```
6. **请求被放行！**

#### 影响

- **任何已登录用户**都能访问该接口
- 即使该用户没有任何角色也能访问
- 敏感数据泄露风险

### 5.2 场景二：数据库中资源名称被修改 → **误拦截**

#### 场景描述

**前提条件：**
1. 数据库中有资源：`id=1, name='商品品牌管理', url='/brand/**'`
2. 用户 A 被分配了该资源，登录时权限为 `"1:商品品牌管理"`
3. 管理员通过后台修改了资源名称为 `"商品品牌管理-新版"`

#### 权限检查流程

1. 系统启动时加载资源权限：`/brand/** -> "1:商品品牌管理-新版"`
2. 用户 A 登录，从缓存或数据库获取权限：`"1:商品品牌管理"`（旧值）
3. 用户 A 访问 `/brand/list`
4. 权限决策比较：
   ```
   需要的权限: "1:商品品牌管理-新版"
   用户拥有的权限: "1:商品品牌管理"
   比较结果: 不相等！
   ```
5. **抛出 AccessDeniedException！**

#### 影响

- 用户 A 明明有权限，却被拒绝访问
- 所有拥有该资源的用户都会被误拦截
- 直到用户重新登录（刷新权限缓存）才能恢复

### 5.3 场景三：URL 模式匹配冲突 → **误放行或误拦截**

#### 场景描述

**数据库配置：**
- 资源 1: `/brand/list` → `"1:商品品牌列表"`
- 资源 2: `/brand/**` → `"2:商品品牌管理"`

#### 权限检查流程

1. 系统启动时，两个资源都会被加载
2. 用户访问 `/brand/list`
3. `DynamicSecurityMetadataSource` 遍历所有模式：
   - 匹配 `/brand/list` → 添加权限 `"1:商品品牌列表"`
   - 匹配 `/brand/**` → 添加权限 `"2:商品品牌管理"`
4. 返回 `["1:商品品牌列表", "2:商品品牌管理"]`
5. `DynamicAccessDecisionManager` 检查：
   - 只要用户拥有**任意一个**权限就放行
   - 如果用户只拥有 `"2:商品品牌管理"`，也能访问

#### 影响

- **误放行**：更宽泛的模式可能覆盖更精确的权限配置
- **权限粒度控制失效**：设计上想让某些用户只能看列表不能编辑，但实际上只要有管理权限就能访问列表

### 5.4 场景四：资源 ID 变更 → **权限系统完全混乱**

#### 场景描述

**前提条件：**
1. 原数据库：`id=1, name='商品品牌管理', url='/brand/**'`
2. 由于某种原因（如数据迁移、表重建），资源 ID 变为 `id=100`
3. 但用户-角色-资源关联表中存储的还是旧的 `resource_id=1`

#### 权限检查流程

1. 系统启动时加载资源：`/brand/** -> "100:商品品牌管理"`
2. 用户登录时查询权限：关联表指向 `resource_id=1`，但该资源可能不存在或已改变
3. 用户权限字符串可能为空或错误
4. 用户访问 `/brand/list`：
   - 需要权限：`"100:商品品牌管理"`
   - 用户权限：`""` 或其他
   - **不匹配！**

#### 影响

- 所有用户失去该资源的访问权限
- 需要重新分配所有角色的资源权限
- 如果是核心功能，系统基本瘫痪

---

## 6. 总结与建议

### 6.1 核心发现

| 问题 | 结论 |
|------|------|
| 是否使用 @PreAuthorize 等注解 | **否**，完全基于动态权限 |
| 权限字符串来源是否统一 | **是**，都来自 `ums_resource` 表 |
| 数据库与注解不一致的问题 | **不存在**，没有注解 |
| 最大风险点 | **未配置资源的接口直接放行** |

### 6.2 风险等级评估

| 场景 | 风险等级 | 影响范围 | 触发条件 |
|------|----------|----------|----------|
| 新增接口未配置资源 | 🔴 **高** | 所有新增接口 | 开发时遗漏数据库配置 |
| 资源名称修改导致缓存不一致 | 🟡 **中** | 该资源的所有用户 | 修改资源名称 + 用户未重新登录 |
| URL 模式匹配冲突 | 🟡 **中** | 权限粒度控制失效 | 配置了重叠的 URL 模式 |
| 资源 ID 变更 | 🔴 **高** | 权限系统全面混乱 | 数据迁移或表重建 |

### 6.3 改进建议

1. **添加默认拒绝策略**：修改 `DynamicAccessDecisionManager`，未配置权限的接口默认拒绝
2. **权限字符串只使用 ID**：改为只使用 `resource.getId()`，避免名称修改影响
3. **添加资源配置校验**：启动时扫描所有 Controller，检查是否都在数据库中配置
4. **添加缓存刷新机制**：修改资源后，自动清除相关用户的权限缓存
5. **考虑双重保护**：关键接口添加 `@PreAuthorize` 注解作为兜底
