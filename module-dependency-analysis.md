# Mall 项目模块依赖关系分析报告

## 一、模块依赖图

### 1.1 模块层级结构

```
                    mall (父模块, pom)
                          │
        ┌─────────────────┼─────────────────┐
        │                 │                 │
   mall-common      mall-security      mall-mbg
   (公共工具)         (安全认证)        (数据层/MBG)
        │                 │                 │
        └────────┬────────┘                 │
                 │                           │
            mall-demo                    ┌───┼───┐
            (演示模块)                    │   │   │
                                    mall-admin  │
                                    (管理后台)  │
                                           mall-portal
                                           (前台门户)
```

### 1.2 详细依赖关系（基于 pom.xml）

| 模块 | 依赖的内部模块 | 说明 |
|------|---------------|------|
| **mall-common** | 无 | 最底层公共模块，提供通用工具、API返回封装、Redis工具等 |
| **mall-mbg** | mall-common | MyBatis Generator 模块，包含所有数据库表的 Mapper 和实体类 |
| **mall-security** | mall-common | 安全认证模块，包含 Spring Security + JWT 相关配置 |
| **mall-demo** | mall-mbg | 演示/示例模块 |
| **mall-admin** | mall-mbg, mall-security | 后台管理服务 |
| **mall-search** | mall-mbg | 搜索服务（排除了 redis 依赖） |
| **mall-portal** | mall-mbg, mall-security | 前台门户服务 |

### 1.3 完整依赖链

```
mall-admin
  └── mall-mbg ──→ mall-common
  └── mall-security ──→ mall-common

mall-portal
  └── mall-mbg ──→ mall-common
  └── mall-security ──→ mall-common

mall-search
  └── mall-mbg ──→ mall-common

mall-demo
  └── mall-mbg ──→ mall-common
```

---

## 二、核心模块被依赖情况分析

### 2.1 mall-common 被谁依赖

**被依赖模块**:
- `mall-mbg`（直接依赖）
- `mall-security`（直接依赖）

**间接被依赖**（通过 mbg/security 传递）:
- mall-admin
- mall-portal
- mall-search
- mall-demo

**mall-common 包含的功能**（从源码可见）:
- `CommonResult`、`CommonPage`：统一返回结果封装
- `IErrorCode`、`ResultCode`：错误码定义
- `ApiException`、`Asserts`、`GlobalExceptionHandler`：全局异常处理
- `BaseRedisConfig`、`RedisService`：Redis 配置与服务
- `BaseSwaggerConfig`、`SwaggerProperties`：Swagger 基础配置
- `WebLog`、`WebLogAspect`：日志切面
- `RequestUtil`：工具类

**定位**: 整个项目的基础公共模块，所有业务模块最终都依赖它。

---

### 2.2 mall-mbg 被谁依赖

**被依赖模块**:
- `mall-admin`（直接依赖）
- `mall-portal`（直接依赖）
- `mall-search`（直接依赖，排除了 redis）
- `mall-demo`（直接依赖）

**mall-mbg 包含的功能**:
- 所有数据库表的 Mapper 接口（自动生成）
- 所有数据库表对应的实体类（Model/POJO）
- MyBatis Generator 配置和代码生成器

**包含的表（从 Mapper 名称推断）**:
- **商品相关**：PmsBrand、PmsProduct、PmsProductCategory、PmsSkuStock 等
- **订单相关**：OmsOrder、OmsOrderItem、OmsCartItem、OmsOrderReturnApply 等
- **营销相关**：SmsCoupon、SmsFlashPromotion、SmsHomeAdvertise 等
- **会员相关**：UmsAdmin、UmsRole、UmsMenu、UmsResource 等
- **内容相关**：CmsSubject、CmsPrefrenceArea 等

**定位**: 数据层共享模块，所有数据库访问入口和实体都在这里。

---

### 2.3 mall-security 被谁依赖

**被依赖模块**:
- `mall-admin`（直接依赖）
- `mall-portal`（直接依赖）

**mall-security 包含的功能**:
- Spring Security 配置（权限控制）
- JWT Token 生成与验证
- 用户登录认证逻辑

**定位**: 安全认证共享模块，被需要认证授权的服务依赖。

---

## 三、不合理依赖检查

### 3.1 是否存在反向依赖？

**结论**: ❌ **不存在反向依赖**

依赖流向分析：
```
上层（业务层）: mall-admin, mall-portal, mall-search, mall-demo
       │
       ▼
中层（功能层）: mall-mbg, mall-security
       │
       ▼
底层（公共层）: mall-common
```

所有依赖都是从**上层 → 下层**，符合依赖倒置原则（上层依赖下层，低层不依赖高层）。

### 3.2 是否存在循环依赖？

**结论**: ❌ **不存在循环依赖**

检查每个模块的依赖关系：

| 模块 | 依赖 | 是否被其他模块依赖 | 循环风险 |
|------|------|-------------------|---------|
| mall-common | 无 | mbg, security | ⚪ 无 |
| mall-mbg | common | admin, portal, search, demo | ⚪ 无 |
| mall-security | common | admin, portal | ⚪ 无 |
| mall-admin | mbg, security | 无 | ⚪ 无 |
| mall-portal | mbg, security | 无 | ⚪ 无 |
| mall-search | mbg | 无 | ⚪ 无 |
| mall-demo | mbg | 无 | ⚪ 无 |

形成的是一个**有向无环图 (DAG)**，没有循环。

### 3.3 业务模块之间是否直接依赖？

**结论**: ❌ **业务模块之间没有直接依赖**

业务模块定义：
- mall-admin（后台管理）
- mall-portal（前台门户）
- mall-search（搜索服务）
- mall-demo（演示）

检查这四个业务模块的 pom.xml：
- mall-admin: 只依赖 mbg 和 security
- mall-portal: 只依赖 mbg 和 security
- mall-search: 只依赖 mbg
- mall-demo: 只依赖 mbg

**业务模块之间没有互相依赖**，都只依赖共享的基础设施模块（mbg、security、common）。

---

## 四、对微服务拆分的影响分析

### 4.1 当前架构特点

**当前是单体架构**，通过 Maven 多模块进行逻辑分层：

```
┌─────────────────────────────────────────────────┐
│              单体应用（单进程）                   │
├─────────────────────────────────────────────────┤
│  mall-admin  │  mall-portal  │  mall-search     │  ← 业务模块（jar包，同进程）
├─────────────────────────────────────────────────┤
│  mall-mbg     │  mall-security                  │  ← 共享模块
├─────────────────────────────────────────────────┤
│              mall-common                         │  ← 公共模块
└─────────────────────────────────────────────────┘
```

### 4.2 对微服务拆分的有利方面 ✅

1. **业务模块无直接耦合**：mall-admin、mall-portal、mall-search 之间没有直接依赖，理论上可以独立拆分
2. **公共逻辑已抽取**：mall-common 抽取得比较好
3. **无循环依赖**：依赖层次清晰

### 4.3 对微服务拆分的不利方面 ⚠️

**最大问题：共享 mall-mbg 模块导致的紧耦合**

```
当前问题：
    ┌──────────┐    ┌──────────┐    ┌──────────┐
    │订单服务   │    │商品服务   │    │搜索服务   │
    │(想拆分)   │    │(想拆分)   │    │(想拆分)   │
    └────┬─────┘    └────┬─────┘    └────┬─────┘
         │               │               │
         └───────────────┼───────────────┘
                         │
                   ┌─────▼──────┐
                   │  mall-mbg   │  ← 共享数据层
                   │(所有表映射) │
                   └────────────┘
```

**具体问题**：

1. **数据库紧耦合**：所有业务模块共享同一个 mall-mbg，意味着：
   - 所有模块访问**同一个数据库**（没有按业务拆分数据库）
   - 订单模块可以直接访问商品表，商品模块可以直接访问订单表
   - 数据库表之间可能存在**跨业务的 JOIN 查询**

2. **实体类共享问题**：OmsOrder 和 PmsProduct 在同一个 mall-mbg 模块中，拆分时：
   - 订单服务会带着商品相关的 Mapper/Entity
   - 商品服务会带着订单相关的 Mapper/Entity
   - 造成**代码冗余**或**依赖残留**

3. **安全模块共享**：mall-security 也是共享的：
   - 微服务架构下通常是独立的认证服务（如 OAuth2 + Gateway）
   - 共享 security 模块意味着认证逻辑在各服务中重复

---

## 五、拆分订单服务和商品服务的最大阻碍

### 5.1 阻碍分析

**最大阻碍：共享数据库 + mall-mbg 导致的数据层耦合**

```
当前状态：
┌─────────────────────────────────────────────────────┐
│              单体应用                                │
│  ┌───────────────────────────────────────────────┐  │
│  │              mall-mbg                         │  │
│  │  OmsOrderMapper  PmsProductMapper  UmsAdmin...│  │ ← 所有表混在一起
│  └───────────────────┬───────────────────────────┘  │
│                      │                              │
│              ┌───────▼───────┐                      │
│              │   单一数据库   │                      │
│              │ (所有业务表)   │                      │
│              └───────────────┘                      │
└─────────────────────────────────────────────────────┘

理想微服务状态：
┌──────────────┐      ┌──────────────┐
│  订单服务     │      │  商品服务     │
│  (独立进程)   │      │  (独立进程)   │
└──────┬───────┘      └──────┬───────┘
       │                     │
┌──────▼───────┐      ┌──────▼───────┐
│ 订单数据库   │      │ 商品数据库   │
│ oms_* 表    │      │ pms_* 表    │
└──────────────┘      └──────────────┘
```

### 5.2 具体拆分困难点

#### 困难点 1：数据库未拆分

从 mall-mbg 的 Mapper 看，所有表在同一个模块，意味着是**单库单 Schema**：
- `oms_*`（订单表）
- `pms_*`（商品表）
- `sms_*`（营销表）
- `ums_*`（用户表）
- `cms_*`（内容表）

都在同一个数据库中，没有做垂直拆分。

#### 困难点 2：业务代码中可能存在跨表 JOIN

从 mall-admin 中的 Dao 层可以看到一些**自定义查询**：
- `OmsOrderDao`、`PmsProductDao` 等自定义查询可能涉及多表关联
- 例如：订单详情查询可能 JOIN 商品表、用户表
- 如果数据库拆分，这些 JOIN 需要改为**服务间调用**

#### 困难点 3：mall-mbg 模块是整体的

mall-mbg 没有按业务领域拆分：
- 如果订单服务拆分出去，它仍然依赖整个 mall-mbg
- 这意味着订单服务包含了商品表、营销表等无关的 Mapper
- 要么接受冗余，要么需要重构 mall-mbg

#### 困难点 4：事务一致性问题

单体架构中跨业务操作是本地事务：
```java
@Transactional
public void createOrder() {
    // 1. 创建订单（oms_order）
    // 2. 扣减库存（pms_sku_stock）
    // 3. 扣减优惠券（sms_coupon）
    // 全部在一个事务中
}
```

拆分成微服务后：
- 订单服务和商品服务是**独立进程、独立数据库**
- 需要使用**分布式事务**（Seata、TCC、Saga、最终一致性）
- 这是一个重大的架构变更

### 5.3 总结：核心阻碍排序

| 优先级 | 阻碍 | 影响程度 |
|--------|------|---------|
| 1 | **单一共享数据库** | 🔴 高 - 微服务的前提是数据库拆分 |
| 2 | **mall-mbg 未按领域拆分** | 🔴 高 - 数据层耦合严重 |
| 3 | **可能存在的跨业务 JOIN 查询** | 🟠 中 - 需要改造为服务调用 |
| 4 | **本地事务变为分布式事务** | 🟠 中 - 需要引入事务协调机制 |
| 5 | **mall-security 共享认证** | 🟡 低 - 可抽为独立认证服务 |

---

## 六、总结

### 6.1 依赖关系总结

- **模块结构清晰**：是标准的三层结构（common → mbg/security → 业务模块）
- **无反向依赖、无循环依赖**：从依赖设计角度看是合理的
- **业务模块间无直接依赖**：这点对微服务拆分是有利的

### 6.2 微服务拆分建议

如果要从单体演进为微服务，建议按以下顺序：

1. **第一步**：数据库垂直拆分（按业务领域分库）
2. **第二步**：重构 mall-mbg，按领域拆分（如 mall-mbg-oms、mall-mbg-pms）
3. **第三步**：业务服务独立部署（mall-admin/portal/search 独立进程）
4. **第四步**：引入 API Gateway 和独立认证中心
5. **第五步**：引入分布式事务和服务治理

### 6.3 关于"不合理依赖"的最终结论

| 检查项 | 结果 |
|--------|------|
| 反向依赖 | ❌ 不存在 |
| 循环依赖 | ❌ 不存在 |
| 业务模块直接依赖 | ❌ 不存在 |

**但**：当前架构虽然"依赖设计合理"，但本质是**单体架构的模块化**，与**微服务架构**的要求仍有差距。最大的差距不在于 Maven 依赖层面，而在于**数据层（数据库 + mall-mbg）的共享耦合**。
