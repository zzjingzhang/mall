# Mall 项目 Redis 使用场景与风险分析

## 1. 项目 Redis 使用概述

### 1.1 技术栈
- **核心框架**: Spring Data Redis
- **模板类**: `RedisTemplate<String, Object>`
- **序列化方式**: Jackson2JsonRedisSerializer（带类型信息）
- **自定义封装**: `RedisService` 接口 + `RedisServiceImpl` 实现
- **缓存管理**: 自定义 CacheService 层，配合 AOP 切面处理 Redis 异常

### 1.2 全局配置
- **缓存默认 TTL**: 1天（24小时）
- **Key 前缀**: `mall:`（通过 `redis.database` 配置）

---

## 2. Redis 操作入口汇总

### 2.1 基础操作层

| 类名 | 文件路径 | 职责说明 |
|------|----------|----------|
| `RedisService` | `mall-common/src/main/java/com/macro/mall/common/service/RedisService.java` | Redis 操作接口定义，封装 String/Hash/Set/List 等数据结构操作 |
| `RedisServiceImpl` | `mall-common/src/main/java/com/macro/mall/common/service/impl/RedisServiceImpl.java` | 基于 `RedisTemplate` 的实现类 |
| `BaseRedisConfig` | `mall-common/src/main/java/com/macro/mall/common/config/BaseRedisConfig.java` | Redis 基础配置，配置 `RedisTemplate`、序列化器、`RedisCacheManager` |
| `RedisConfig` | `mall-security/src/main/java/com/macro/mall/security/config/RedisConfig.java` | 继承 `BaseRedisConfig`，启用 `@EnableCaching` |
| `RedisCacheAspect` | `mall-security/src/main/java/com/macro/mall/security/aspect/RedisCacheAspect.java` | 缓存异常切面，捕获 CacheService 层的 Redis 异常，非 `@CacheException` 标记的方法只记录日志不抛出 |

### 2.2 业务使用入口

#### 2.2.1 mall-admin 模块

| 类名 | 文件路径 | 主要职责 |
|------|----------|----------|
| `UmsAdminCacheService` | `mall-admin/src/main/java/com/macro/mall/service/UmsAdminCacheService.java` | 后台管理员缓存服务接口 |
| `UmsAdminCacheServiceImpl` | `mall-admin/src/main/java/com/macro/mall/service/impl/UmsAdminCacheServiceImpl.java` | 后台管理员缓存实现 |

#### 2.2.2 mall-portal 模块

| 类名 | 文件路径 | 主要职责 |
|------|----------|----------|
| `UmsMemberCacheService` | `mall-portal/src/main/java/com/macro/mall/portal/service/UmsMemberCacheService.java` | 前台会员缓存服务接口 |
| `UmsMemberCacheServiceImpl` | `mall-portal/src/main/java/com/macro/mall/portal/service/impl/UmsMemberCacheServiceImpl.java` | 前台会员缓存实现（含验证码） |
| `UmsMemberServiceImpl` | `mall-portal/src/main/java/com/macro/mall/portal/service/impl/UmsMemberServiceImpl.java` | 会员业务逻辑，使用缓存 |
| `OmsPortalOrderServiceImpl` | `mall-portal/src/main/java/com/macro/mall/portal/service/impl/OmsPortalOrderServiceImpl.java` | 订单业务，使用 Redis 生成订单号自增序列 |

---

## 3. Key 命名规则

### 3.1 统一命名模式

```
{redis.database}:{redis.key.xxx}:{业务标识}
```

- **`redis.database`**: 固定值 `mall`
- **`redis.key.xxx`**: 业务类型前缀（配置文件定义）
- **`业务标识`**: 用户ID、用户名、手机号、日期等

### 3.2 各业务 Key 详情

| 业务模块 | Key 前缀 | 完整 Key 示例 | 说明 |
|----------|----------|---------------|------|
| 后台管理员信息 | `mall:ums:admin` | `mall:ums:admin:{username}` | 按用户名缓存管理员对象 |
| 管理员资源列表 | `mall:ums:resourceList` | `mall:ums:resourceList:{adminId}` | 按管理员ID缓存权限资源列表 |
| 前台会员信息 | `mall:ums:member` | `mall:ums:member:{username}` | 按用户名缓存会员对象 |
| 验证码 | `mall:ums:authCode` | `mall:ums:authCode:{telephone}` | 按手机号缓存验证码 |
| 订单ID序列 | `mall:oms:orderId` | `mall:oms:orderId:{yyyyMMdd}` | 按日期存储订单号自增计数器 |

### 3.3 配置文件定义

**mall-admin/application.yml**:
```yaml
redis:
  database: mall
  key:
    admin: 'ums:admin'
    resourceList: 'ums:resourceList'
  expire:
    common: 86400  # 24小时
```

**mall-portal/application.yml**:
```yaml
redis:
  database: mall
  key:
    authCode: 'ums:authCode'
    orderId: 'oms:orderId'
    member: 'ums:member'
  expire:
    authCode: 90     # 90秒
    common: 86400    # 24小时
```

---

## 4. 使用场景分类

### 4.1 缓存（Cache）

| 场景 | Key 模式 | TTL | 存储内容 | 读取位置 |
|------|----------|-----|----------|----------|
| 后台管理员信息缓存 | `mall:ums:admin:{username}` | 86400s | `UmsAdmin` 对象 | `UmsAdminCacheServiceImpl.getAdmin()` |
| 管理员权限资源列表缓存 | `mall:ums:resourceList:{adminId}` | 86400s | `List<UmsResource>` | `UmsAdminCacheServiceImpl.getResourceList()` |
| 前台会员信息缓存 | `mall:ums:member:{username}` | 86400s | `UmsMember` 对象 | `UmsMemberCacheServiceImpl.getMember()` |

**缓存读写策略**:
- **Cache-Aside Pattern**: 先查缓存，未命中则查数据库并回填缓存
- **失效策略**: 主动删除（Cache-Invalidation），数据更新后删除缓存

### 4.2 验证码/登录态（验证码类）

| 场景 | Key 模式 | TTL | 存储内容 | 读取位置 |
|------|----------|-----|----------|----------|
| 手机号验证码 | `mall:ums:authCode:{telephone}` | 90s | 6位数字字符串 | `UmsMemberCacheServiceImpl.getAuthCode()` |

**注意**: 登录态采用 **JWT Token** 方案，**不存储在 Redis**，Token 本身携带信息。

### 4.3 分布式锁（业务临时计数器）

| 场景 | Key 模式 | TTL | 实现方式 | 代码位置 |
|------|----------|-----|----------|----------|
| 订单号生成自增序列 | `mall:oms:orderId:{yyyyMMdd}` | 未显式设置（永久） | `INCR` 命令原子自增 | `OmsPortalOrderServiceImpl.generateOrderSn()` |

**说明**: 
- 使用 Redis `INCR` 命令实现分布式自增序列
- 按日期分 Key，每日从 1 开始计数
- **未设置过期时间**，历史日期的 Key 会永久残留

### 4.4 业务临时数据

本项目**无**其他典型业务临时数据（如购物车、限时活动等）存储在 Redis 中。

---

## 5. 风险分析

### 5.1 缓存穿透风险

**定义**: 查询不存在的数据，缓存未命中导致每次都查数据库。

#### 5.1.1 会员信息查询 - 缓存穿透风险 ⚠️

**风险描述**:
`UmsMemberServiceImpl.getByUsername()` 方法中，当查询一个**不存在的用户名**时：
1. 缓存查询返回 `null`
2. 查询数据库返回 `null`
3. **未将空值写入缓存**
4. 下次相同查询仍然穿透到数据库

**代码位置**: 
`mall-portal/src/main/java/com/macro/mall/portal/service/impl/UmsMemberServiceImpl.java:58-70`

```java
public UmsMember getByUsername(String username) {
    UmsMember member = memberCacheService.getMember(username);
    if(member!=null) return member;  // 缓存命中返回
    UmsMemberExample example = new UmsMemberExample();
    example.createCriteria().andUsernameEqualTo(username);
    List<UmsMember> memberList = memberMapper.selectByExample(example);
    if (!CollectionUtils.isEmpty(memberList)) {
        member = memberList.get(0);
        memberCacheService.setMember(member);  // 只有非空才写入缓存
        return member;
    }
    return null;  // 空结果未缓存，导致穿透
}
```

**风险等级**: 中

**影响**: 若有人恶意用不存在的用户名频繁请求，可绕过缓存直接打数据库。

#### 5.1.2 管理员信息查询 - 缓存穿透风险 ⚠️

**风险描述**:
类似会员查询，管理员信息查询同样存在空值未缓存问题。虽然管理员接口通常有鉴权保护，但理论上仍存在风险。

**代码位置**:
`mall-admin/src/main/java/com/macro/mall/service/impl/UmsAdminCacheServiceImpl.java`

（上层调用逻辑决定，`setAdmin` 只在存在时被调用）

**风险等级**: 低（受权限保护）

---

### 5.2 缓存雪崩风险

**定义**: 大量缓存 Key 在同一时间失效，导致请求同时打到数据库。

#### 5.2.1 统一 TTL 导致的雪崩风险 ⚠️

**风险描述**:
所有缓存数据使用**统一的 TTL（24小时）**，且**未加随机偏移**：
- 若系统上线时批量加载缓存，24小时后可能集体失效
- 管理员/会员信息缓存 `REDIS_EXPIRE = 86400` 秒

**相关代码位置**:
1. `mall-admin/src/main/java/com/macro/mall/service/impl/UmsAdminCacheServiceImpl.java:101,113`
2. `mall-portal/src/main/java/com/macro/mall/portal/service/impl/UmsMemberCacheServiceImpl.java:51`

**风险等级**: 中

**影响**: 在业务高峰期若出现缓存集中失效，可能造成数据库压力突增。

---

### 5.3 脏数据风险

**定义**: 缓存数据与数据库真实数据不一致，且错误数据被读取。

#### 5.3.1 管理员资源列表 - 脏数据风险 ⚠️

**风险描述**:
管理员权限资源列表缓存基于 `adminId` 缓存，当**角色-资源关系**发生变化时：
- `delResourceListByRole()` 和 `delResourceListByRoleIds()` 会正确删除相关管理员缓存
- `delResourceListByResource()` 会删除与该资源相关的管理员缓存

**潜在问题**:
缓存有效期 24 小时，若在以下场景可能出现脏数据窗口：
1. 管理员角色变更 → 先更新数据库 → 后删除缓存（双写期间）
2. 资源分配变更 → 同理
3. 若删除缓存操作失败（Redis 异常），将依赖 AOP 吞掉异常，**脏数据保留至 TTL 过期**

**代码位置**:
`mall-security/src/main/java/com/macro/mall/security/aspect/RedisCacheAspect.java:31-48`

```java
@Around("cacheAspect()")
public Object doAround(ProceedingJoinPoint joinPoint) throws Throwable {
    // ...
    try {
        result = joinPoint.proceed();
    } catch (Throwable throwable) {
        if (method.isAnnotationPresent(CacheException.class)) {
            throw throwable;  // 只有 @CacheException 标记才抛出
        } else {
            LOGGER.error(throwable.getMessage());  // 其他只记录日志
        }
    }
    return result;
}
```

`UmsAdminCacheServiceImpl` 中**所有删除缓存方法都没有 `@CacheException` 注解**，意味着：
- 数据库更新成功
- Redis 删除失败（网络问题/Redis 宕机）
- 异常被吞掉，业务流程继续
- 缓存中残留旧数据 → **脏数据**

**代码位置**:
`mall-admin/src/main/java/com/macro/mall/service/impl/UmsAdminCacheServiceImpl.java:44-90`
（`delAdmin`, `delResourceList`, `delResourceListByRole` 等方法均无 `@CacheException`）

**风险等级**: 高

**影响**: 权限数据不一致可能导致安全问题（应收回的权限仍可访问，或新权限无法生效）。

#### 5.3.2 会员信息 - 脏数据风险 ⚠️

**风险描述**:
与管理员缓存类似，会员信息缓存更新时：
- `updatePassword()` → 数据库更新 → `delMember()` 删除缓存
- `updateIntegration()` → 数据库更新 → `delMember()` 删除缓存

**问题**:
`UmsMemberCacheServiceImpl` 中：
- `delMember()` 方法**无 `@CacheException`** 注解
- `setMember()` 和 `getMember()` 方法**也无 `@CacheException`**

若缓存删除失败，会员旧信息将在缓存中保留 24 小时。

**代码位置**:
`mall-portal/src/main/java/com/macro/mall/portal/service/impl/UmsMemberCacheServiceImpl.java:33-52`

**风险等级**: 中

**影响**: 会员积分、密码等信息可能不一致。

---

### 5.4 内存泄漏风险

**定义**: Redis 数据无限增长，未清理。

#### 5.4.1 订单号序列 Key 永久残留 ⚠️

**风险描述**:
`OmsPortalOrderServiceImpl.generateOrderSn()` 使用 `INCR` 生成订单号：

```java
private String generateOrderSn(OmsOrder order) {
    // ...
    String key = REDIS_DATABASE+":"+ REDIS_KEY_ORDER_ID + date;  // mall:oms:orderId20260509
    Long increment = redisService.incr(key, 1);  // 自增
    // ...
}
```

**问题**:
- `INCR` 操作的 Key **未设置过期时间**
- 每天产生一个新 Key（`mall:oms:orderId20260509`, `mall:oms:orderId20260510`, ...）
- 这些 Key 会**永久保留**在 Redis 中

**代码位置**:
`mall-portal/src/main/java/com/macro/mall/portal/service/impl/OmsPortalOrderServiceImpl.java:462-477`

**风险等级**: 低（单个 Key 体积小，但逐年累积）

**影响**: Redis 内存缓慢增长，需人工定期清理。

---

## 6. Redis 与数据库一致性问题

### 6.1 一致性策略分析

项目采用 **"先更新数据库，后删除缓存" (Cache-Invalidation)** 模式：

```
更新操作:
1. 数据库 UPDATE
2. Redis DEL

读取操作:
1. Redis GET → 命中返回
2. 未命中 → DB SELECT → Redis SET
```

### 6.2 存在一致性问题的场景

| 数据类型 | 一致性策略 | 问题 | 风险等级 |
|----------|-----------|------|----------|
| 管理员信息 (UmsAdmin) | 数据库更新后 `delAdmin()` | 缓存删除可能失败（无 `@CacheException`） | 中 |
| 管理员资源列表 (UmsResource) | 角色/资源变更后 `delResourceList*()` | 缓存删除可能失败，且有多级依赖 | 高 |
| 会员信息 (UmsMember) | 密码/积分变更后 `delMember()` | 缓存删除可能失败 | 中 |

### 6.3 一致性问题详述

#### 6.3.1 管理员权限多级缓存一致性 ⚠️

**问题场景**:
管理员权限涉及多层关系：
```
UmsAdmin ←→ UmsAdminRoleRelation ←→ UmsRole ←→ UmsRoleResourceRelation ←→ UmsResource
```

缓存的是 `List<UmsResource>`（最终资源列表）。

**变更触发点**:
1. 管理员角色变更 (`delResourceListByRoleIds`)
2. 角色资源变更 (`delResourceListByRole`)
3. 资源本身变更 (`delResourceListByResource`)

**风险**:
- 多层级缓存失效依赖多级 `del*` 方法调用
- 任何一层删除失败都会导致权限不一致
- 特别是 `UmsAdminRoleRelationDao.getAdminIdList()` 查询异常时，可能漏删

**代码位置**:
`mall-admin/src/main/java/com/macro/mall/service/impl/UmsAdminCacheServiceImpl.java:83-90`
```java
public void delResourceListByResource(Long resourceId) {
    List<Long> adminIdList = adminRoleRelationDao.getAdminIdList(resourceId);
    if (CollUtil.isNotEmpty(adminIdList)) {
        // 基于查询结果删除，如果查询失败或数据量大时可能有问题
        List<String> keys = adminIdList.stream()
            .map(adminId -> keyPrefix + adminId)
            .collect(Collectors.toList());
        redisService.del(keys);  // 批量删除
    }
}
```

**风险等级**: 高

#### 6.3.2 验证码无需数据库同步

**说明**:
验证码 (`mall:ums:authCode:{telephone}`) 仅存在于 Redis，无需数据库同步，不存在一致性问题。

#### 6.3.3 订单号序列无需数据库同步

**说明**:
订单号序列 (`mall:oms:orderId:{date}`) 是纯 Redis 计数器，不对应数据库特定行，无一致性问题。

---

## 7. 风险代码位置汇总

### 7.1 缓存穿透风险

| 风险点 | 文件 | 行号 |
|--------|------|------|
| 会员空值未缓存 | `mall-portal/.../UmsMemberServiceImpl.java` | 58-70 |

### 7.2 缓存雪崩风险

| 风险点 | 文件 | 行号 |
|--------|------|------|
| 管理员信息统一 TTL | `mall-admin/.../UmsAdminCacheServiceImpl.java` | 101 |
| 管理员资源列表统一 TTL | `mall-admin/.../UmsAdminCacheServiceImpl.java` | 113 |
| 会员信息统一 TTL | `mall-portal/.../UmsMemberCacheServiceImpl.java` | 51 |

### 7.3 脏数据/一致性风险

| 风险点 | 文件 | 行号 |
|--------|------|------|
| 缓存删除失败被 AOP 吞掉 | `mall-security/.../RedisCacheAspect.java` | 31-48 |
| delAdmin() 无异常保护 | `mall-admin/.../UmsAdminCacheServiceImpl.java` | 44-50 |
| delResourceList() 无异常保护 | `mall-admin/.../UmsAdminCacheServiceImpl.java` | 53-56 |
| delResourceListByRole() 无异常保护 | `mall-admin/.../UmsAdminCacheServiceImpl.java` | 59-68 |
| delResourceListByRoleIds() 无异常保护 | `mall-admin/.../UmsAdminCacheServiceImpl.java` | 71-80 |
| delResourceListByResource() 无异常保护 | `mall-admin/.../UmsAdminCacheServiceImpl.java` | 83-90 |
| delMember() 无异常保护 | `mall-portal/.../UmsMemberCacheServiceImpl.java` | 33-40 |
| 会员信息空值未缓存（穿透+并发） | `mall-portal/.../UmsMemberServiceImpl.java` | 58-70 |

### 7.4 内存泄漏风险

| 风险点 | 文件 | 行号 |
|--------|------|------|
| 订单号序列 Key 无 TTL | `mall-portal/.../OmsPortalOrderServiceImpl.java` | 465-466 |

---

## 8. 总结

| 风险类型 | 数量 | 最高风险等级 |
|----------|------|-------------|
| 缓存穿透 | 2 | 中 |
| 缓存雪崩 | 1（统一 TTL） | 中 |
| 脏数据/一致性 | 9（含 AOP 异常处理） | 高 |
| 内存泄漏 | 1 | 低 |

**最严重风险**:
1. **缓存删除失败导致脏数据**（权限相关）- 高风险
2. **管理员权限多级缓存一致性** - 高风险
3. **缓存穿透（空值未缓存）** - 中风险

**建议优化方向**:
1. 关键缓存删除操作添加 `@CacheException` 或改为失败重试
2. 空值缓存（设置较短 TTL）防止穿透
3. TTL 添加随机偏移防止雪崩
4. 订单号 Key 设置合理过期时间（如 30 天）
5. 考虑引入分布式锁防止缓存击穿（并发热点 Key 重建）
