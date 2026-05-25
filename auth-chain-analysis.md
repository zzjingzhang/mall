# Mall 项目后台用户完整认证链路分析

## 一、项目模块概述

本项目认证体系主要分布在两个核心模块：

| 模块 | 路径 | 职责 |
|------|------|------|
| mall-security | `/mall-security` | 通用安全组件，包括 JWT 工具、过滤器、异常处理、安全配置等 |
| mall-admin | `/mall-admin` | 后台管理业务实现，包括用户服务、登录接口、权限动态配置等 |

---

## 二、登录流程：用户名密码认证实现

### 2.1 完整调用链路

```
UmsAdminController.login()
    └──→ UmsAdminServiceImpl.login()
            ├──→ loadUserByUsername()  ← UserDetailsService 实现
            │       ├──→ getAdminByUsername()  ← 查询用户信息（含 Redis 缓存）
            │       └──→ getResourceList()  ← 获取用户资源权限
            ├──→ passwordEncoder.matches()  ← BCrypt 密码验证
            ├──→ userDetails.isEnabled()  ← 账号状态检查
            ├──→ 创建 UsernamePasswordAuthenticationToken
            ├──→ SecurityContextHolder.getContext().setAuthentication()
            └──→ jwtTokenUtil.generateToken()  ← 生成 JWT Token
```

### 2.2 核心类及路径

#### 1) 登录接口 Controller
- **类路径**：`com.macro.mall.controller.UmsAdminController`
- **文件位置**：`mall-admin/src/main/java/com/macro/mall/controller/UmsAdminController.java`
- **核心方法**：`login()` (第 58-70 行)

```java
@ApiOperation(value = "登录以后返回token")
@RequestMapping(value = "/login", method = RequestMethod.POST)
@ResponseBody
public CommonResult login(@Validated @RequestBody UmsAdminLoginParam umsAdminLoginParam) {
    String token = adminService.login(umsAdminLoginParam.getUsername(), umsAdminLoginParam.getPassword());
    if (token == null) {
        return CommonResult.validateFailed("用户名或密码错误");
    }
    Map<String, String> tokenMap = new HashMap<>();
    tokenMap.put("token", token);
    tokenMap.put("tokenHead", tokenHead);
    return CommonResult.success(tokenMap);
}
```

#### 2) 用户详情服务 UserDetailsService
- **Bean 定义位置**：`com.macro.mall.config.MallSecurityConfig` (第 29-33 行)

```java
@Bean
public UserDetailsService userDetailsService() {
    // 获取登录用户信息
    return username -> adminService.loadUserByUsername(username);
}
```

- **实际实现方法**：`UmsAdminServiceImpl.loadUserByUsername()` (第 265-273 行)

```java
@Override
public UserDetails loadUserByUsername(String username){
    // 获取用户信息
    UmsAdmin admin = getAdminByUsername(username);
    if (admin != null) {
        List<UmsResource> resourceList = getResourceList(admin.getId());
        return new AdminUserDetails(admin,resourceList);
    }
    throw new UsernameNotFoundException("用户名或密码错误");
}
```

#### 3) 业务逻辑实现 Service
- **类路径**：`com.macro.mall.service.impl.UmsAdminServiceImpl`
- **文件位置**：`mall-admin/src/main/java/com/macro/mall/service/impl/UmsAdminServiceImpl.java`
- **核心方法**：`login()` (第 99-119 行)

```java
@Override
public String login(String username, String password) {
    String token = null;
    // 密码需要客户端加密后传递
    try {
        UserDetails userDetails = loadUserByUsername(username);
        if(!passwordEncoder.matches(password,userDetails.getPassword())){
            Asserts.fail("密码不正确");
        }
        if(!userDetails.isEnabled()){
            Asserts.fail("帐号已被禁用");
        }
        UsernamePasswordAuthenticationToken authentication = 
            new UsernamePasswordAuthenticationToken(userDetails, null, userDetails.getAuthorities());
        SecurityContextHolder.getContext().setAuthentication(authentication);
        token = jwtTokenUtil.generateToken(userDetails);
        insertLoginLog(username);
    } catch (AuthenticationException e) {
        LOGGER.warn("登录异常:{}", e.getMessage());
    }
    return token;
}
```

**关键特点**：
- 该项目**没有使用** Spring Security 默认的 `AuthenticationManager.authenticate()` 进行认证
- 而是在 Service 层手动调用 `PasswordEncoder.matches()` 进行密码比对
- 手动创建 `UsernamePasswordAuthenticationToken` 并设置到 `SecurityContextHolder`

#### 4) 密码编码器
- **Bean 定义**：`com.macro.mall.security.config.CommonSecurityConfig` (第 19-22 行)

```java
@Bean
public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder();
}
```

#### 5) UserDetails 封装类
- **类路径**：`com.macro.mall.bo.AdminUserDetails`
- **文件位置**：`mall-admin/src/main/java/com/macro/mall/bo/AdminUserDetails.java`

该类实现了 Spring Security 的 `UserDetails` 接口，封装了：
- `umsAdmin`：后台用户实体
- `resourceList`：用户拥有的资源列表（转换为 GrantedAuthority）

---

## 三、JWT 令牌生命周期管理

### 3.1 Token 生成

#### 生成类与方法
- **工具类**：`com.macro.mall.security.util.JwtTokenUtil`
- **文件位置**：`mall-security/src/main/java/com/macro/mall/security/util/JwtTokenUtil.java`
- **Bean 定义**：`CommonSecurityConfig.jwtTokenUtil()` (第 30-32 行)

```java
@Bean
public JwtTokenUtil jwtTokenUtil() {
    return new JwtTokenUtil();
}
```

#### 核心生成方法

**方法 1**：`generateToken(UserDetails userDetails)` (第 117-122 行)

```java
public String generateToken(UserDetails userDetails) {
    Map<String, Object> claims = new HashMap<>();
    claims.put(CLAIM_KEY_USERNAME, userDetails.getUsername());
    claims.put(CLAIM_KEY_CREATED, new Date());
    return generateToken(claims);
}
```

**方法 2**：`generateToken(Map<String, Object> claims)` (第 42-48 行) - 私有方法

```java
private String generateToken(Map<String, Object> claims) {
    return Jwts.builder()
            .setClaims(claims)
            .setExpiration(generateExpirationDate())
            .signWith(SignatureAlgorithm.HS512, secret)
            .compact();
}
```

#### JWT Payload 结构
- `sub` (CLAIM_KEY_USERNAME)：用户名
- `created` (CLAIM_KEY_CREATED)：创建时间
- `exp`：过期时间（由 expiration 配置计算）

### 3.2 JWT 配置参数

配置文件：`mall-admin/src/main/resources/application.yml` (第 19-23 行)

```yaml
jwt:
  tokenHeader: Authorization   # JWT存储的请求头
  secret: mall-admin-secret    # JWT加解密使用的密钥
  expiration: 604800           # 过期时间(60*60*24*7) = 7天
  tokenHead: 'Bearer '         # Token前缀
```

### 3.3 Token 的服务端存储机制

**该项目 JWT Token 采用无状态设计，服务端不存储 Token**。

验证方式：
1. 每次请求从 Header 中提取 Token
2. 使用密钥 `secret` 进行签名验证
3. 验证过期时间 `exp`
4. 从 Payload 中解析用户名，再从数据库/缓存查询用户信息

### 3.4 Token 刷新机制

**方法**：`JwtTokenUtil.refreshHeadToken()` (第 129-153 行)

```java
public String refreshHeadToken(String oldToken) {
    if(StrUtil.isEmpty(oldToken)){
        return null;
    }
    String token = oldToken.substring(tokenHead.length());
    if(StrUtil.isEmpty(token)){
        return null;
    }
    // token校验不通过
    Claims claims = getClaimsFromToken(token);
    if(claims==null){
        return null;
    }
    // 如果token已经过期，不支持刷新
    if(isTokenExpired(token)){
        return null;
    }
    // 如果token在30分钟之内刚刷新过，返回原token
    if(tokenRefreshJustBefore(token,30*60)){
        return token;
    }else{
        claims.put(CLAIM_KEY_CREATED, new Date());
        return generateToken(claims);
    }
}
```

刷新策略：
- Token 过期后不支持刷新
- 30 分钟内重复刷新返回原 Token
- 超过 30 分钟则更新 `created` 时间并重新生成

### 3.5 返回客户端的完整代码路径

**Controller 层**：`UmsAdminController.login()` (第 66-69 行)

```java
Map<String, String> tokenMap = new HashMap<>();
tokenMap.put("token", token);
tokenMap.put("tokenHead", tokenHead);
return CommonResult.success(tokenMap);
```

返回格式：
```json
{
  "code": 200,
  "message": "操作成功",
  "data": {
    "token": "eyJhbGciOiJIUzUxMiJ9...",
    "tokenHead": "Bearer "
  }
}
```

---

## 四、后续请求处理：Token 提取机制

### 4.1 核心过滤器

**类路径**：`com.macro.mall.security.component.JwtAuthenticationTokenFilter`
**文件位置**：`mall-security/src/main/java/com/macro/mall/security/component/JwtAuthenticationTokenFilter.java`
**继承关系**：继承 `OncePerRequestFilter`（保证每个请求只执行一次）

### 4.2 Token 读取位置与方法

**核心方法**：`doFilterInternal()` (第 37-56 行)

```java
@Override
protected void doFilterInternal(HttpServletRequest request,
                                HttpServletResponse response,
                                FilterChain chain) throws ServletException, IOException {
    String authHeader = request.getHeader(this.tokenHeader);  // 从 Authorization Header 读取
    if (authHeader != null && authHeader.startsWith(this.tokenHead)) {
        String authToken = authHeader.substring(this.tokenHead.length()); // 去掉 "Bearer " 前缀
        String username = jwtTokenUtil.getUserNameFromToken(authToken);   // 解析用户名
        LOGGER.info("checking username:{}", username);
        if (username != null && SecurityContextHolder.getContext().getAuthentication() == null) {
            UserDetails userDetails = this.userDetailsService.loadUserByUsername(username);
            if (jwtTokenUtil.validateToken(authToken, userDetails)) {
                UsernamePasswordAuthenticationToken authentication = 
                    new UsernamePasswordAuthenticationToken(userDetails, null, userDetails.getAuthorities());
                authentication.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
                LOGGER.info("authenticated user:{}", username);
                SecurityContextHolder.getContext().setAuthentication(authentication);
            }
        }
    }
    chain.doFilter(request, response);
}
```

### 4.3 Token 提取流程

1. **读取位置**：HTTP 请求头 `Authorization`
2. **配置 Key**：`jwt.tokenHeader` = `Authorization`
3. **前缀检查**：必须以 `jwt.tokenHead` = `Bearer ` 开头
4. **Token 解析**：
   - `JwtTokenUtil.getUserNameFromToken()` → 调用 `getClaimsFromToken()` 解析 JWT
   - 使用 `Jwts.parser().setSigningKey(secret).parseClaimsJws(token)` 进行验证和解码

### 4.4 过滤器注册位置

**配置类**：`com.macro.mall.security.config.SecurityConfig` (第 67 行)

```java
.addFilterBefore(jwtAuthenticationTokenFilter, UsernamePasswordAuthenticationFilter.class);
```

注册在 `UsernamePasswordAuthenticationFilter` **之前**，确保先进行 JWT 认证。

---

## 五、SecurityContext 的构建与设置

### 5.1 构建时机与位置

SecurityContext 的设置发生在两个场景：

#### 场景 1：登录时（登录成功后）

**位置**：`UmsAdminServiceImpl.login()` (第 111 行)

```java
UsernamePasswordAuthenticationToken authentication = 
    new UsernamePasswordAuthenticationToken(userDetails, null, userDetails.getAuthorities());
SecurityContextHolder.getContext().setAuthentication(authentication);
```

#### 场景 2：后续请求（JWT 过滤器中）

**位置**：`JwtAuthenticationTokenFilter.doFilterInternal()` (第 51 行)

```java
if (jwtTokenUtil.validateToken(authToken, userDetails)) {
    UsernamePasswordAuthenticationToken authentication = 
        new UsernamePasswordAuthenticationToken(userDetails, null, userDetails.getAuthorities());
    authentication.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
    SecurityContextHolder.getContext().setAuthentication(authentication);
}
```

### 5.2 完整构建链路（后续请求场景）

```
请求到达
    ↓
JwtAuthenticationTokenFilter.doFilterInternal()
    ├──→ request.getHeader("Authorization")  ← 提取 Token
    ├──→ jwtTokenUtil.getUserNameFromToken() ← 解析用户名
    ├──→ userDetailsService.loadUserByUsername() ← 加载用户详情
    │       └──→ UmsAdminServiceImpl.loadUserByUsername()
    │               ├──→ getAdminByUsername() ← 查询用户（Redis 缓存优先）
    │               └──→ getResourceList() ← 查询资源权限
    ├──→ jwtTokenUtil.validateToken() ← 验证 Token 有效性
    │       └──→ 检查用户名匹配 + 检查是否过期
    ├──→ new UsernamePasswordAuthenticationToken(userDetails, null, authorities)
    ├──→ authentication.setDetails(WebAuthenticationDetails)
    └──→ SecurityContextHolder.getContext().setAuthentication(authentication)
            ↓
    SecurityContext 已设置到 ThreadLocal
```

### 5.3 SecurityContext 存储机制

- **存储类**：`SecurityContextHolder`
- **默认策略**：`MODE_THREADLOCAL`（线程本地存储）
- **上下文对象**：包含 `Authentication`（认证信息）
  - `Principal`：`UserDetails` 实现类 `AdminUserDetails`
  - `Credentials`：`null`（JWT 认证不需要密码）
  - `Authorities`：用户资源权限列表（格式：`resourceId:resourceName`）

---

## 六、异常处理机制

### 6.1 异常处理类总览

| 异常场景 | 处理类 | 处理接口 | HTTP 状态 |
|---------|--------|---------|-----------|
| 权限不足（已登录但无权限） | `RestfulAccessDeniedHandler` | `AccessDeniedHandler` | 403 Forbidden |
| Token 过期 / 未登录 / Token 无效 | `RestAuthenticationEntryPoint` | `AuthenticationEntryPoint` | 401 Unauthorized |

### 6.2 权限不足异常处理

#### 处理类
- **类路径**：`com.macro.mall.security.component.RestfulAccessDeniedHandler`
- **文件位置**：`mall-security/src/main/java/com/macro/mall/security/component/RestfulAccessDeniedHandler.java`

```java
public class RestfulAccessDeniedHandler implements AccessDeniedHandler{
    @Override
    public void handle(HttpServletRequest request,
                       HttpServletResponse response,
                       AccessDeniedException e) throws IOException, ServletException {
        response.setHeader("Access-Control-Allow-Origin", "*");
        response.setHeader("Cache-Control","no-cache");
        response.setCharacterEncoding("UTF-8");
        response.setContentType("application/json");
        response.getWriter().println(JSONUtil.parse(CommonResult.forbidden(e.getMessage())));
        response.getWriter().flush();
    }
}
```

#### 触发时机
当动态权限决策管理器 `DynamicAccessDecisionManager.decide()` 抛出 `AccessDeniedException` 时触发。

#### 注册位置
`SecurityConfig.filterChain()` (第 63 行)

```java
.exceptionHandling()
.accessDeniedHandler(restfulAccessDeniedHandler)
```

### 6.3 Token 过期 / 未登录异常处理

#### 处理类
- **类路径**：`com.macro.mall.security.component.RestAuthenticationEntryPoint`
- **文件位置**：`mall-security/src/main/java/com/macro/mall/security/component/RestAuthenticationEntryPoint.java`

```java
public class RestAuthenticationEntryPoint implements AuthenticationEntryPoint {
    @Override
    public void commence(HttpServletRequest request, HttpServletResponse response, 
                        AuthenticationException authException) throws IOException, ServletException {
        response.setHeader("Access-Control-Allow-Origin", "*");
        response.setHeader("Cache-Control","no-cache");
        response.setCharacterEncoding("UTF-8");
        response.setContentType("application/json");
        response.getWriter().println(JSONUtil.parse(CommonResult.unauthorized(authException.getMessage())));
        response.getWriter().flush();
    }
}
```

#### 触发场景
1. **未登录**：请求受保护资源且 `SecurityContext` 中无 `Authentication`
2. **Token 过期**：JWT 解析时 `exp` 已过期，`isTokenExpired()` 返回 true
3. **Token 无效**：JWT 签名验证失败或格式错误

#### 注册位置
`SecurityConfig.filterChain()` (第 64 行)

```java
.authenticationEntryPoint(restAuthenticationEntryPoint)
```

### 6.4 动态权限校验中的异常抛出

**类**：`DynamicAccessDecisionManager.decide()` (第 21-39 行)

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
        // 将访问所需资源或用户拥有资源进行比对
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

---

## 七、白名单配置分析

### 7.1 白名单配置类

- **类路径**：`com.macro.mall.security.config.IgnoreUrlsConfig`
- **文件位置**：`mall-security/src/main/java/com/macro/mall/security/config/IgnoreUrlsConfig.java`

```java
@Getter
@Setter
@ConfigurationProperties(prefix = "secure.ignored")
public class IgnoreUrlsConfig {
    private List<String> urls = new ArrayList<>();
}
```

### 7.2 白名单具体路径

**配置文件**：`mall-admin/src/main/resources/application.yml` (第 33-51 行)

```yaml
secure:
  ignored:
    urls: # 安全路径白名单
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

### 7.3 白名单生效位置

白名单在两处生效：

#### 位置 1：SecurityFilterChain 配置

**类**：`SecurityConfig.filterChain()` (第 43-45 行)

```java
// 不需要保护的资源路径允许访问
for (String url : ignoreUrlsConfig.getUrls()) {
    registry.antMatchers(url).permitAll();
}
```

#### 位置 2：动态权限过滤器

**类**：`DynamicSecurityFilter.doFilter()` (第 47-52 行)

```java
// 白名单请求直接放行
PathMatcher pathMatcher = new AntPathMatcher();
for (String path : ignoreUrlsConfig.getUrls()) {
    if(pathMatcher.match(path,request.getRequestURI())){
        fi.getChain().doFilter(fi.getRequest(), fi.getResponse());
        return;
    }
}
```

### 7.4 白名单路径分类

| 类别 | 路径示例 | 用途 |
|------|---------|------|
| Swagger 文档 | `/swagger-ui/`, `/swagger-resources/**`, `/**/v2/api-docs` | API 文档访问 |
| 静态资源 | `/**/*.html`, `/**/*.js`, `/**/*.css`, `/**/*.png` | 前端静态文件 |
| 监控端点 | `/actuator/**`, `/druid/**` | 健康检查、监控 |
| 认证接口 | `/admin/login`, `/admin/register`, `/admin/info`, `/admin/logout` | 登录注册登出 |
| 文件上传 | `/minio/upload` | 文件上传接口 |

---

## 八、完整认证链路时序图

```
┌─────────────┐     ┌──────────────────┐     ┌─────────────────────┐     ┌──────────────┐
│   Client    │     │ UmsAdminController│     │ UmsAdminServiceImpl │     │ JwtTokenUtil │
└──────┬──────┘     └────────┬─────────┘     └──────────┬──────────┘     └──────┬───────┘
       │ POST /admin/login    │                         │                      │
       │─────────────────────>│                         │                      │
       │                     │ login(username,password) │                      │
       │                     │────────────────────────>│                      │
       │                     │                         │ loadUserByUsername() │
       │                     │                         │─────────────────────>│ (内部调用)
       │                     │                         │   查询用户+资源权限  │
       │                     │                         │<─────────────────────│
       │                     │                         │                      │
       │                     │                         │ passwordEncoder.     │
       │                     │                         │ matches(password)   │
       │                     │                         │   验证通过 ──┐       │
       │                     │                         │              │       │
       │                     │                         │ setAuthentication  │
       │                     │                         │  to SecurityContext│
       │                     │                         │                      │
       │                     │                         │ generateToken()      │
       │                     │                         │────────────────────────────────────>│
       │                     │                         │                      │ 生成 JWT
       │                     │                         │<────────────────────────────────────│
       │                     │                         │   返回 token         │
       │                     │<────────────────────────│                      │
       │ {token, tokenHead}  │                         │                      │
       │<─────────────────────│                         │                      │
       │                     │                         │                      │
       │                     │                         │                      │
       │ 后续请求（带 Authorization Header）             │                      │
       │─────────────────────────────────────────────────────────────────────>│
       │                     │                         │                      │
       │                     │     ┌───────────────────────────────────────┐   │
       │                     │     │ JwtAuthenticationTokenFilter          │   │
       │                     │     │ 1. 从 Header 提取 Token               │   │
       │                     │     │ 2. 解析用户名                         │   │
       │                     │     │ 3. loadUserByUsername()              │   │
       │                     │     │ 4. validateToken()                    │   │
       │                     │     │ 5. setAuthentication to Context      │   │
       │                     │     └───────────────────────────────────────┘   │
       │                     │                         │                      │
       │                     │     ┌───────────────────────────────────────┐   │
       │                     │     │ DynamicSecurityFilter (动态权限校验)   │   │
       │                     │     │ 1. 检查白名单                         │   │
       │                     │     │ 2. DynamicSecurityMetadataSource      │   │
       │                     │     │    获取访问所需权限                   │   │
       │                     │     │ 3. DynamicAccessDecisionManager       │   │
       │                     │     │    decide() 比对权限                 │   │
       │                     │     └───────────────────────────────────────┘   │
       │                     │                         │                      │
       │ 业务响应            │                         │                      │
       │<─────────────────────────────────────────────────────────────────────│
```

---

## 九、核心类汇总表

| 组件类型 | 类名 | 完整路径 | 核心职责 |
|---------|------|---------|---------|
| 登录接口 | UmsAdminController | `com.macro.mall.controller.UmsAdminController` | 接收登录请求，返回 Token |
| 用户服务 | UmsAdminServiceImpl | `com.macro.mall.service.impl.UmsAdminServiceImpl` | 登录逻辑、用户查询、权限加载 |
| UserDetails | AdminUserDetails | `com.macro.mall.bo.AdminUserDetails` | Spring Security 用户信息封装 |
| JWT 工具 | JwtTokenUtil | `com.macro.mall.security.util.JwtTokenUtil` | Token 生成、解析、验证、刷新 |
| JWT 过滤器 | JwtAuthenticationTokenFilter | `com.macro.mall.security.component.JwtAuthenticationTokenFilter` | 请求 Token 提取与认证 |
| 安全配置 | SecurityConfig | `com.macro.mall.security.config.SecurityConfig` | SecurityFilterChain 配置 |
| 通用安全配置 | CommonSecurityConfig | `com.macro.mall.security.config.CommonSecurityConfig` | 通用 Bean 定义 |
| 模块安全配置 | MallSecurityConfig | `com.macro.mall.config.MallSecurityConfig` | mall-admin 模块定制配置 |
| 白名单配置 | IgnoreUrlsConfig | `com.macro.mall.security.config.IgnoreUrlsConfig` | 放行路径配置 |
| 权限不足处理 | RestfulAccessDeniedHandler | `com.macro.mall.security.component.RestfulAccessDeniedHandler` | 403 异常处理 |
| 未登录处理 | RestAuthenticationEntryPoint | `com.macro.mall.security.component.RestAuthenticationEntryPoint` | 401 异常处理 |
| 动态权限过滤器 | DynamicSecurityFilter | `com.macro.mall.security.component.DynamicSecurityFilter` | 基于路径的动态权限过滤 |
| 动态权限数据源 | DynamicSecurityMetadataSource | `com.macro.mall.security.component.DynamicSecurityMetadataSource` | 加载资源-权限映射 |
| 动态权限决策 | DynamicAccessDecisionManager | `com.macro.mall.security.component.DynamicAccessDecisionManager` | 权限比对决策 |
