# mall-admin 与 mall-portal 认证鉴权机制对比分析

## 1. 是否复用同一套 mall-security 逻辑

**结论：是，两个模块都复用了 mall-security 模块的核心框架逻辑。**

### 代码证据

#### 1.1 pom.xml 依赖关系

两个模块都在 pom.xml 中引入了 mall-security 依赖：

**mall-admin/pom.xml:25-27**
```xml
<dependency>
    <groupId>com.macro.mall</groupId>
    <artifactId>mall-security</artifactId>
</dependency>
```

**mall-portal/pom.xml:25-28**
```xml
<dependency>
    <groupId>com.macro.mall</groupId>
    <artifactId>mall-security</artifactId>
</dependency>
```

#### 1.2 复用的 mall-security 核心组件

两个模块都复用了以下 mall-security 组件：

1. **JwtTokenUtil** (`mall-security/util/JwtTokenUtil.java`) - JWT 工具类
2. **JwtAuthenticationTokenFilter** (`mall-security/component/JwtAuthenticationTokenFilter.java`) - JWT 认证过滤器
3. **SecurityConfig** (`mall-security/config/SecurityConfig.java`) - Spring Security 配置
4. **CommonSecurityConfig** (`mall-security/config/CommonSecurityConfig.java`) - 通用安全配置
5. **IgnoreUrlsConfig** (`mall-security/config/IgnoreUrlsConfig.java`) - 白名单配置
6. **RestfulAccessDeniedHandler** - 访问拒绝处理器
7. **RestAuthenticationEntryPoint** - 认证入口点

## 2. 登录接口、UserDetailsService、权限加载逻辑是否不同

**结论：UserDetailsService 接口实现、权限加载逻辑不同；登录接口路径和参数也不同。**

### 2.1 登录接口对比

| 特性 | mall-admin | mall-portal |
|------|-----------|-------------|
| Controller 路径 | `/admin` | `/sso` |
| 登录路径 | `/admin/login` | `/sso/login` |
| 注册路径 | `/admin/register` | `/sso/register` |
| 请求方式 | POST | POST |
| 参数类型 | JSON Body (`UmsAdminLoginParam`) | Form 参数 |
| 验证码 | 无 | 需要（注册和修改密码） |

**代码证据：**

**mall-admin/controller/UmsAdminController.java:58-70**
```java
@ApiOperation(value = "登录以后返回token")
@RequestMapping(value = "/login", method = RequestMethod.POST)
@ResponseBody
public CommonResult login(@Validated @RequestBody UmsAdminLoginParam umsAdminLoginParam) {
    String token = adminService.login(umsAdminLoginParam.getUsername(), umsAdminLoginParam.getPassword());
    // ...
}
```

**mall-portal/controller/UmsMemberController.java:49-62**
```java
@ApiOperation("会员登录")
@RequestMapping(value = "/login", method = RequestMethod.POST)
@ResponseBody
public CommonResult login(@RequestParam String username,
                          @RequestParam String password) {
    String token = memberService.login(username, password);
    // ...
}
```

### 2.2 UserDetailsService 实现对比

| 特性 | mall-admin | mall-portal |
|------|-----------|-------------|
| 实现类 | `UmsAdminServiceImpl` | `UmsMemberServiceImpl` |
| 用户模型 | `UmsAdmin` (后台管理员) | `UmsMember` (前台会员) |
| UserDetails 封装 | `AdminUserDetails` | `MemberDetails` |
| 数据源 | `ums_admin` 表 | `ums_member` 表 |

**代码证据：**

**mall-admin/config/MallSecurityConfig.java:29-33**
```java
@Bean
public UserDetailsService userDetailsService() {
    return username -> adminService.loadUserByUsername(username);
}
```

**mall-portal/config/MallSecurityConfig.java:19-23**
```java
@Bean
public UserDetailsService userDetailsService() {
    return username -> memberService.loadUserByUsername(username);
}
```

**mall-admin/service/impl/UmsAdminServiceImpl.java:264-273**
```java
@Override
public UserDetails loadUserByUsername(String username){
    UmsAdmin admin = getAdminByUsername(username);
    if (admin != null) {
        List<UmsResource> resourceList = getResourceList(admin.getId());
        return new AdminUserDetails(admin, resourceList);
    }
    throw new UsernameNotFoundException("用户名或密码错误");
}
```

**mall-portal/service/impl/UmsMemberServiceImpl.java:155-162**
```java
@Override
public UserDetails loadUserByUsername(String username) {
    UmsMember member = getByUsername(username);
    if(member!=null){
        return new MemberDetails(member);
    }
    throw new UsernameNotFoundException("用户名或密码错误");
}
```

### 2.3 权限加载逻辑对比

**结论：权限加载逻辑完全不同。mall-admin 使用基于资源的动态权限系统，mall-portal 使用固定的简单权限。**

#### mall-admin 权限加载逻辑

1. **AdminUserDetails** (`mall-admin/bo/AdminUserDetails.java:28-34`)
   ```java
   @Override
   public Collection<? extends GrantedAuthority> getAuthorities() {
       return resourceList.stream()
               .map(resource -> new SimpleGrantedAuthority(resource.getId() + ":" + resource.getName()))
               .collect(Collectors.toList());
   }
   ```

2. **动态权限服务** (`mall-admin/config/MallSecurityConfig.java:35-48`)
   ```java
   @Bean
   public DynamicSecurityService dynamicSecurityService() {
       return new DynamicSecurityService() {
           @Override
           public Map<String, ConfigAttribute> loadDataSource() {
               Map<String, ConfigAttribute> map = new ConcurrentHashMap<>();
               List<UmsResource> resourceList = resourceService.listAll();
               for (UmsResource resource : resourceList) {
                   map.put(resource.getUrl(), new org.springframework.security.access.SecurityConfig(resource.getId() + ":" + resource.getName()));
               }
               return map;
           }
       };
   }
   ```

3. **权限来源**：通过 `ums_admin_role_relation` → `ums_role_resource_relation` → `ums_resource` 关联查询用户可访问的资源

#### mall-portal 权限加载逻辑

**MemberDetails** (`mall-portal/domain/MemberDetails.java:22-26`)
```java
@Override
public Collection<? extends GrantedAuthority> getAuthorities() {
    return Arrays.asList(new SimpleGrantedAuthority("TEST"));
}
```

**对比总结：**

| 特性 | mall-admin | mall-portal |
|------|-----------|-------------|
| 权限类型 | 基于资源的动态权限 | 固定的简单权限 |
| 权限内容 | `资源ID:资源名称`（如 `1:用户管理`） | 固定值 `"TEST"` |
| 权限来源 | 数据库资源表 + 角色关联 | 硬编码 |
| 动态权限过滤 | 启用（`DynamicSecurityFilter`） | 未启用 |
| 权限校验 | 基于 URL 路径匹配 | 仅认证，无精细授权 |

## 3. token 生成和解析是否共用

**结论：token 生成和解的工具类共用，但配置（密钥）不同，因此 token 不能跨模块使用。**

### 3.1 共用的 JwtTokenUtil

两个模块都使用 mall-security 中的 `JwtTokenUtil` 类：

**mall-security/util/JwtTokenUtil.java:114-122**
```java
public String generateToken(UserDetails userDetails) {
    Map<String, Object> claims = new HashMap<>();
    claims.put(CLAIM_KEY_USERNAME, userDetails.getUsername());
    claims.put(CLAIM_KEY_CREATED, new Date());
    return generateToken(claims);
}
```

**mall-security/util/JwtTokenUtil.java:73-85**
```java
public String getUserNameFromToken(String token) {
    String username;
    try {
        Claims claims = getClaimsFromToken(token);
        username = claims.getSubject();
    } catch (Exception e) {
        username = null;
    }
    return username;
}
```

### 3.2 不同的 JWT 配置（关键差异）

| 配置项 | mall-admin | mall-portal |
|--------|-----------|-------------|
| 密钥 (secret) | `mall-admin-secret` | `mall-portal-secret` |
| 过期时间 | 604800 秒 (7天) | 604800 秒 (7天) |
| Token Header | `Authorization` | `Authorization` |
| Token Prefix | `Bearer ` | `Bearer ` |

**代码证据：**

**mall-admin/application.yml:19-23**
```yaml
jwt:
  tokenHeader: Authorization
  secret: mall-admin-secret
  expiration: 604800
  tokenHead: 'Bearer '
```

**mall-portal/application.yml:15-19**
```yaml
jwt:
  tokenHeader: Authorization
  secret: mall-portal-secret
  expiration: 604800
  tokenHead: 'Bearer '
```

### 3.3 token 解析流程

**JwtAuthenticationTokenFilter** (`mall-security/component/JwtAuthenticationTokenFilter.java:36-56`)

```java
@Override
protected void doFilterInternal(HttpServletRequest request,
                                HttpServletResponse response,
                                FilterChain chain) throws ServletException, IOException {
    String authHeader = request.getHeader(this.tokenHeader);
    if (authHeader != null && authHeader.startsWith(this.tokenHead)) {
        String authToken = authHeader.substring(this.tokenHead.length());
        String username = jwtTokenUtil.getUserNameFromToken(authToken);
        if (username != null && SecurityContextHolder.getContext().getAuthentication() == null) {
            UserDetails userDetails = this.userDetailsService.loadUserByUsername(username);
            if (jwtTokenUtil.validateToken(authToken, userDetails)) {
                // 设置认证信息
            }
        }
    }
    chain.doFilter(request, response);
}
```

**关键点：**
1. token 解析时使用当前模块配置的 `secret` 进行签名验证
2. 解析出 username 后，调用当前模块的 `UserDetailsService.loadUserByUsername()` 从数据库加载用户

## 4. 白名单配置是否一致

**结论：白名单配置机制一致，但具体路径配置不同。**

### 4.1 共用的白名单配置类

两个模块都使用 mall-security 中的 `IgnoreUrlsConfig`：

**mall-security/config/IgnoreUrlsConfig.java:14-19**
```java
@Getter
@Setter
@ConfigurationProperties(prefix = "secure.ignored")
public class IgnoreUrlsConfig {
    private List<String> urls = new ArrayList<>();
}
```

### 4.2 不同的白名单路径配置

| 模块 | 白名单路径 | 说明 |
|------|----------|------|
| mall-admin | `/swagger-ui/**`, `/swagger-resources/**`, `/**/v2/api-docs` | Swagger 文档 |
| | `/**/*.html`, `/**/*.js`, `/**/*.css`, `/**/*.png`, `/**/*.map`, `/favicon.ico` | 静态资源 |
| | `/actuator/**`, `/druid/**` | 监控和数据库管理 |
| | `/admin/login`, `/admin/register`, `/admin/info`, `/admin/logout` | 认证相关 |
| | `/minio/upload` | 文件上传 |
| mall-portal | `/swagger-ui/**`, `/swagger-resources/**`, `/**/v2/api-docs` | Swagger 文档 |
| | `/**/*.html`, `/**/*.js`, `/**/*.css`, `/**/*.png`, `/**/*.map`, `/favicon.ico` | 静态资源 |
| | `/actuator/**`, `/druid/**` | 监控和数据库管理 |
| | `/sso/**` | 会员认证相关（登录、注册、验证码等） |
| | `/home/**`, `/product/**`, `/brand/**` | 前台公开资源 |
| | `/alipay/**` | 支付宝支付回调 |

**代码证据：**

**mall-admin/application.yml:33-51**
```yaml
secure:
  ignored:
    urls:
      - /swagger-ui/
      - /swagger-resources/**
      - /**/v2/api-docs
      - /**/*.html
      - /**/*.js
      - /**/*.css
      - /**/*.png
      - /**/*.map
      - /favicon.ico
      - /actuator/**
      - /druid/**
      - /admin/login
      - /admin/register
      - /admin/info
      - /admin/logout
      - /minio/upload
```

**mall-portal/application.yml:21-39**
```yaml
secure:
  ignored:
    urls:
      - /swagger-ui/
      - /swagger-resources/**
      - /**/v2/api-docs
      - /**/*.html
      - /**/*.js
      - /**/*.css
      - /**/*.png
      - /**/*.map
      - /favicon.ico
      - /druid/**
      - /actuator/**
      - /sso/**
      - /home/**
      - /product/**
      - /brand/**
      - /alipay/**
```

### 4.3 白名单处理流程

白名单在两个地方生效：

1. **SecurityConfig** (`mall-security/config/SecurityConfig.java:42-45`)
   ```java
   for (String url : ignoreUrlsConfig.getUrls()) {
       registry.antMatchers(url).permitAll();
   }
   ```

2. **DynamicSecurityFilter** (仅 mall-admin) (`mall-security/component/DynamicSecurityFilter.java:46-53`)
   ```java
   PathMatcher pathMatcher = new AntPathMatcher();
   for (String path : ignoreUrlsConfig.getUrls()) {
       if(pathMatcher.match(path, request.getRequestURI())){
           fi.getChain().doFilter(fi.getRequest(), fi.getResponse());
           return;
       }
   }
   ```

## 5. 是否存在前台 token 调用后台接口或后台 token 调用前台接口的可能性

**结论：理论上存在可能性，但需要满足多个条件，实际风险较低。**

### 5.1 技术可行性分析

#### 场景 1：前台用户 token 调用后台接口

**能否成功？取决于以下条件：**

| 条件 | 是否满足 | 说明 |
|------|---------|------|
| 1. JWT 签名验证通过 | ❌ 否 | 两个模块使用不同的 secret：`mall-admin-secret` vs `mall-portal-secret` |
| 2. 用户名在后台用户表存在 | ⚠️ 可能 | 如果前台用户名和后台用户名相同 |
| 3. 拥有访问后台接口的权限 | ❌ 否 | 前台会员只有 `"TEST"` 权限，后台需要资源权限 |

**详细流程：**

1. **JWT 签名验证失败**（最关键）
   - 前台 token 使用 `mall-portal-secret` 签名
   - 后台模块解析时使用 `mall-admin-secret` 验证
   - `JwtTokenUtil.getClaimsFromToken()` 会抛出异常，返回 null
   - `JwtAuthenticationTokenFilter` 不会设置认证信息

2. **即使绕过签名验证（理论假设）**
   - `UserDetailsService.loadUserByUsername()` 会从 `ums_admin` 表查询
   - 前台用户名可能不存在于 `ums_admin` 表
   - 即使存在，`AdminUserDetails.getAuthorities()` 返回的是后台资源权限
   - 动态权限过滤器 `DynamicSecurityFilter` 会检查用户是否有访问该 URL 的资源权限
   - 前台用户没有绑定后台资源，权限校验会失败

#### 场景 2：后台用户 token 调用前台接口

**能否成功？**

| 条件 | 是否满足 | 说明 |
|------|---------|------|
| 1. JWT 签名验证通过 | ❌ 否 | 两个模块使用不同的 secret |
| 2. 用户名在前台会员表存在 | ⚠️ 可能 | 如果后台用户名和前台会员名相同 |
| 3. 拥有访问前台接口的权限 | ✅ 是 | 前台接口认证后即可访问，不做精细权限校验 |

**详细流程：**

1. **JWT 签名验证失败**（最关键）
   - 后台 token 使用 `mall-admin-secret` 签名
   - 前台模块解析时使用 `mall-portal-secret` 验证
   - 同样会失败

2. **即使绕过签名验证（理论假设）**
   - `UserDetailsService.loadUserByUsername()` 会从 `ums_member` 表查询
   - 后台用户名可能不存在于 `ums_member` 表
   - 前台接口大部分在白名单中（`/home/**`, `/product/**` 等），不需要认证
   - 需要认证的接口（如购物车、订单），只要用户存在就能访问

### 5.2 风险评估

**实际风险：低**

| 风险类型 | 可能性 | 说明 |
|---------|-------|------|
| 前台 token 访问后台 | 极低 | JWT 签名密钥不同，无法通过验证 |
| 后台 token 访问前台 | 极低 | JWT 签名密钥不同，无法通过验证 |
| 用户名冲突导致越权 | 低 | 需要同时满足：1) 绕过 JWT 验证 2) 用户名在另一张表存在 |

### 5.3 安全边界总结

| 边界 | 机制 | 有效性 |
|------|------|--------|
| 跨模块 token 使用 | 不同的 JWT secret | ✅ 有效 |
| 跨表用户查询 | 独立的 UserDetailsService | ✅ 有效 |
| 后台精细权限控制 | 动态资源权限 + DynamicSecurityFilter | ✅ 有效 |
| 前台简单权限 | 固定 TEST 权限（无精细控制） | ⚠️ 仅认证 |

## 总结对比表

| 对比维度 | mall-admin | mall-portal | 结论 |
|---------|-----------|-------------|------|
| 1. mall-security 复用 | ✅ 复用全部核心组件 | ✅ 复用全部核心组件 | **复用同一套框架逻辑** |
| 2. 登录接口 | `/admin/login` (JSON) | `/sso/login` (Form) | **不同** |
| 2. UserDetailsService | `UmsAdminServiceImpl` (查 ums_admin) | `UmsMemberServiceImpl` (查 ums_member) | **不同** |
| 2. 权限加载 | 动态资源权限（角色-资源关联） | 固定 TEST 权限 | **完全不同** |
| 3. token 生成/解析 | 共用 JwtTokenUtil | 共用 JwtTokenUtil | **工具类共用** |
| 3. JWT 配置 | secret: mall-admin-secret | secret: mall-portal-secret | **配置不同，token 不通用** |
| 4. 白名单机制 | 共用 IgnoreUrlsConfig | 共用 IgnoreUrlsConfig | **机制一致** |
| 4. 白名单路径 | 包含 `/admin/**`, `/minio/upload` | 包含 `/sso/**`, `/home/**`, `/product/**`, `/alipay/**` | **具体路径不同** |
| 5. 跨模块 token 使用 | 受限于 secret 不同 | 受限于 secret 不同 | **存在理论可能，但实际风险低** |
