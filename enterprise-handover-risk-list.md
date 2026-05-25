# Mall 项目企业接手二次开发风险清单

本文档从企业接手二次开发的角度，分析 mall 项目最容易踩坑的 10 个风险点。重点关注权限、事务、订单、库存、搜索同步、缓存、消息队列和代码生成八个核心领域。

---

## 风险点 1：订单提交无事务保护（高风险）

### 具体代码位置
- `mall-portal/src/main/java/com/macro/mall/portal/service/impl/OmsPortalOrderServiceImpl.java:96-252`
  - `generateOrder()` 方法，订单提交核心逻辑

### 为什么容易误解
1. **Spring Boot 默认配置陷阱**：开发者可能误以为 Spring Boot 自动管理事务，但 `generateOrder()` 方法本身没有 `@Transactional` 注解
2. **接口层注解不一致**：虽然接口 `OmsPortalOrderService` 上可能有 `@Transactional`，但实现类方法可能未正确继承
3. **多步骤操作看似安全**：代码执行流程看似连贯，但实际上每一步数据库操作都在独立事务中

### 错误修改会造成什么后果
1. **库存锁定但订单未创建**：如果库存锁定成功后订单插入失败，库存将被永久锁定
2. **优惠券扣减但订单失败**：用户优惠券被标记为已使用，但订单未创建
3. **积分扣减但订单失败**：用户积分被扣除，但订单未创建
4. **购物车已删除但订单失败**：用户购物车被清空，但无法继续下单
5. **部分成功部分失败**：出现数据不一致，需要人工修复

### 正确修改方式或排查入口
```java
// 正确方式：添加事务注解
@Transactional(rollbackFor = Exception.class)
public Map<String, Object> generateOrder(OrderParam orderParam) {
    // 所有数据库操作在同一事务内
}
```

**排查入口**：
- 在 `generateOrder()` 方法入口和各关键步骤添加日志
- 检查 `@Transactional` 注解是否正确配置
- 确认事务传播行为和隔离级别
- 测试异常场景：故意在某一步抛出异常，检查数据是否回滚

---

## 风险点 2：动态权限缓存不刷新（高风险）

### 具体代码位置
- `mall-security/src/main/java/com/macro/mall/security/component/DynamicSecurityMetadataSource.java:18-63`
  - 动态权限数据源，权限规则缓存
- `mall-admin/src/main/java/com/macro/mall/service/impl/UmsRoleServiceImpl.java:88-118`
  - 角色菜单/资源分配方法 `allocMenu()` 和 `allocResource()`

### 为什么容易误解
1. **缓存加载时机**：`DynamicSecurityMetadataSource` 使用 `@PostConstruct` 在启动时加载权限规则到静态变量 `configAttributeMap`
2. **静态变量缓存**：权限规则存储在 `private static Map<String, ConfigAttribute> configAttributeMap` 中
3. **缺少触发刷新机制**：`clearDataSource()` 方法存在，但在角色/资源变更时没有被调用

### 错误修改会造成什么后果
1. **新权限不生效**：分配新菜单或资源权限后，用户无法访问，直到重启服务
2. **权限变更不及时**：修改角色权限后，旧权限仍然有效，存在安全隐患
3. **删除权限后仍可访问**：移除权限后，用户仍能访问该资源
4. **排查困难**：开发者可能怀疑代码逻辑问题，而不会想到是缓存未刷新

### 正确修改方式或排查入口
```java
// 在角色权限变更后调用刷新
@Override
public int allocResource(Long roleId, List<Long> resourceIds) {
    // 原有逻辑...
    adminCacheService.delResourceListByRole(roleId);
    // 添加：刷新动态权限缓存
    dynamicSecurityMetadataSource.clearDataSource();
    return resourceIds.size();
}
```

**排查入口**：
- 检查 `DynamicSecurityMetadataSource.configAttributeMap` 是否在权限变更后被更新
- 确认 `clearDataSource()` 方法是否在角色/资源变更时被调用
- 查看是否有统一的权限变更事件通知机制
- 建议：使用 Redis 存储权限规则，避免静态变量缓存

---

## 风险点 3：消息队列取消订单消息无确认机制（高风险）

### 具体代码位置
- `mall-portal/src/main/java/com/macro/mall/portal/component/CancelOrderSender.java:17-34`
  - 延迟消息发送者
- `mall-portal/src/main/java/com/macro/mall/portal/component/CancelOrderReceiver.java:15-25`
  - 延迟消息消费者
- `mall-portal/src/main/java/com/macro/mall/portal/service/impl/OmsPortalOrderServiceImpl.java:315-348`
  - `cancelOrder()` 方法

### 为什么容易误解
1. **依赖默认自动确认**：Spring AMQP 默认自动确认消息，但代码中没有显式配置
2. **异常处理缺失**：`CancelOrderReceiver.handle()` 方法没有 try-catch，异常时消息可能丢失
3. **幂等性依赖业务状态**：虽然 `cancelOrder()` 有状态检查，但如果消息处理失败，订单可能永远无法取消

### 错误修改会造成什么后果
1. **库存永久锁定**：如果取消消息处理失败，库存锁定无法释放
2. **优惠券无法恢复**：用户优惠券无法回收，影响复购
3. **积分无法返还**：用户积分无法返还，造成用户投诉
4. **消息丢失**：MQ 服务重启或网络波动可能导致消息丢失
5. **重复消费无去重**：虽然业务层有状态检查，但日志冗余，排查困难

### 正确修改方式或排查入口
```java
// 正确方式：手动确认 + 异常处理
@RabbitHandler
public void handle(Long orderId, Channel channel, @Header(AmqpHeaders.DELIVERY_TAG) long deliveryTag) {
    try {
        portalOrderService.cancelOrder(orderId);
        channel.basicAck(deliveryTag, false);
        LOGGER.info("process orderId:{}", orderId);
    } catch (Exception e) {
        LOGGER.error("cancel order failed, orderId:{}", orderId, e);
        // 失败时拒绝消息，可以选择是否重新入队
        channel.basicNack(deliveryTag, false, false);
        // 建议：记录到死信队列或数据库，后续人工处理
    }
}
```

**排查入口**：
- 检查 RabbitMQ 队列的消息确认模式配置
- 查看 `CancelOrderReceiver` 是否有异常处理和手动确认
- 测试 MQ 服务重启场景，验证消息是否丢失
- 建议：添加 Redis 去重，记录已处理的 orderId

---

## 风险点 4：订单提交无重复提交防护（高风险）

### 具体代码位置
- `mall-portal/src/main/java/com/macro/mall/portal/controller/OmsPortalOrderController.java:40-46`
  - `generateOrder()` 控制器入口
- `mall-portal/src/main/java/com/macro/mall/portal/service/impl/OmsPortalOrderServiceImpl.java:96-252`
  - 订单生成逻辑

### 为什么容易误解
1. **前端防重复点击≠后端幂等**：开发者可能依赖前端按钮禁用，但网络延迟时用户仍可刷新重发
2. **订单号每次新生成**：`generateOrderSn()` 使用 Redis 自增，每次都是新的订单号，无法通过订单号去重
3. **缺少请求唯一标识**：接口参数中没有请求 ID 或 Token

### 错误修改会造成什么后果
1. **重复下单**：用户网络延迟时多次点击，创建多个相同订单
2. **重复扣减库存**：多个订单都锁定库存，可能导致超卖或库存紧张
3. **重复扣减优惠券**：用户优惠券被多次使用
4. **重复扣减积分**：用户积分被多次扣除
5. **用户投诉**：用户需要手动取消多余订单，体验差

### 正确修改方式或排查入口
```java
// 正确方式：使用 Token 机制防止重复提交
// 1. 预下单时获取 Token
@RequestMapping(value = "/getOrderToken", method = RequestMethod.GET)
public CommonResult getOrderToken() {
    String token = UUID.randomUUID().toString();
    redisService.set("order:token:" + memberId, token, 5 * 60);
    return CommonResult.success(token);
}

// 2. 下单时校验 Token
@RequestMapping(value = "/generateOrder", method = RequestMethod.POST)
public CommonResult generateOrder(@RequestBody OrderParam orderParam,
                                   @RequestHeader("orderToken") String orderToken) {
    // 校验并删除 Token（原子操作）
    String key = "order:token:" + memberId;
    String cachedToken = (String) redisService.get(key);
    if (!orderToken.equals(cachedToken)) {
        return CommonResult.failed("请不要重复提交");
    }
    redisService.del(key);
    // 继续下单逻辑...
}
```

**排查入口**：
- 检查是否有防重复提交的拦截器或 AOP 切面
- 搜索是否有 `@RepeatSubmit` 或类似注解
- 测试快速点击提交按钮，观察是否生成多个订单
- 建议：添加接口限流，防止恶意刷订单

---

## 风险点 5：Elasticsearch 与数据库同步不一致（高风险）

### 具体代码位置
- `mall-search/src/main/java/com/macro/mall/search/service/impl/EsProductServiceImpl.java:52-289`
  - ES 商品服务实现
- `mall-admin/src/main/java/com/macro/mall/service/impl/PmsProductServiceImpl.java:69-161`
  - 商品创建和更新方法

### 为什么容易误解
1. **同步时机不明确**：商品创建/更新后，ES 索引更新是手动调用还是自动触发？
2. **两个独立服务**：mall-admin 和 mall-search 是独立模块，没有事件驱动机制
3. **无失败重试**：如果 ES 服务不可用，索引更新失败后没有补偿机制
4. **部分字段同步**：需要确认哪些字段变化需要同步到 ES

### 错误修改会造成什么后果
1. **搜索不到新商品**：商品已上架，但搜索结果中找不到
2. **搜索到已下架商品**：商品已下架或删除，但 ES 索引未更新
3. **价格/库存不一致**：商品价格或库存变化后，搜索结果显示旧数据
4. **影响转化率**：用户搜索体验差，影响购买决策
5. **排查困难**：问题出现后，需要对比数据库和 ES 数据

### 正确修改方式或排查入口
```java
// 正确方式：商品变更后同步更新 ES（或使用消息队列异步同步）
@Override
@Transactional(rollbackFor = Exception.class)
public int create(PmsProductParam productParam) {
    // 原有创建逻辑...
    int count = 1;
    
    // 同步到 ES（建议使用消息队列解耦）
    try {
        esProductService.create(product.getId());
    } catch (Exception e) {
        LOGGER.error("sync product to ES failed, productId:{}", product.getId(), e);
        // 记录到同步失败表，后续重试
        syncFailLogService.save("ES", "PRODUCT", product.getId(), "CREATE");
    }
    
    return count;
}
```

**排查入口**：
- 检查 `PmsProductServiceImpl` 的 `create()` 和 `update()` 方法后是否调用 ES 同步
- 查看是否有商品变更事件的消息发布机制
- 对比数据库 `pms_product` 表和 ES 索引中的商品数量
- 建议：使用 Canal 监听 MySQL binlog，自动同步到 ES

---

## 风险点 6：代码生成器覆盖自定义代码（中高风险）

### 具体代码位置
- `mall-mbg/src/main/java/com/macro/mall/Generator.java:16-37`
  - MBG 代码生成器入口
- `mall-mbg/src/main/resources/generatorConfig.xml`
  - 生成器配置文件
- `mall-mbg/src/main/java/com/macro/mall/mapper/` 目录
  - 自动生成的 Mapper 文件

### 为什么容易误解
1. **overwrite = true**：生成器配置中 `overwrite = true`，会覆盖同名文件
2. **目录结构混淆**：自动生成的 Mapper 和自定义 Dao 可能在同一目录
3. **命名规则相似**：`XxxMapper` 和 `XxxDao` 容易混淆
4. **二次开发者不了解历史**：接手项目后可能不知道哪些文件是自动生成的

### 错误修改会造成什么后果
1. **自定义逻辑丢失**：在自动生成的 Mapper 中添加的自定义方法被覆盖
2. **XML 映射文件覆盖**：在 `XxxMapper.xml` 中添加的 SQL 被覆盖
3. **Model 类修改丢失**：在 Model 类中添加的字段或注解被覆盖
4. **生产事故**：如果在生产环境前重新生成代码，可能导致功能回退

### 正确修改方式或排查入口
```xml
<!-- generatorConfig.xml 正确配置建议 -->
<!-- 1. 明确区分自动生成和自定义目录 -->
<javaModelGenerator targetPackage="com.macro.mall.model" 
                    targetProject="mall-mbg/src/main/java">
    <!-- 自动生成的 Model 放在独立模块 -->
</javaModelGenerator>

<sqlMapGenerator targetPackage="com.macro.mall.mapper" 
                 targetProject="mall-mbg/src/main/resources">
    <!-- 自动生成的 Mapper XML 放在独立模块 -->
</sqlMapGenerator>

<!-- 2. 自定义 Dao 使用不同命名和目录 -->
<!-- 自定义 Dao 放在 mall-admin 或 mall-portal 的 dao 包下 -->
```

**排查入口**：
- 检查 `generatorConfig.xml` 中的 `targetProject` 和 `targetPackage`
- 对比 `mall-mbg` 和 `mall-admin` 中的 Mapper/Dao 目录
- 搜索 `@mbg.generated` 注释，这是 MBG 生成的标记
- 建议：
  1. 自动生成的代码放在 `mall-mbg` 模块，永远不要手动修改
  2. 自定义逻辑放在 `mall-admin` 或 `mall-portal` 的 `dao` 包下，命名为 `XxxDao`
  3. 运行生成器前备份代码，或使用 Git 管理

---

## 风险点 7：管理员角色分配无事务保护（高风险）

### 具体代码位置
- `mall-admin/src/main/java/com/macro/mall/service/impl/UmsAdminServiceImpl.java:199-218`
  - `updateRole()` 方法，分配管理员角色
- `mall-admin/src/main/java/com/macro/mall/service/impl/UmsRoleServiceImpl.java:88-118`
  - `allocMenu()` 和 `allocResource()` 方法

### 为什么容易误解
1. **先删后插模式**：方法都是先删除原有关系，再插入新关系
2. **缺少 `@Transactional`**：`UmsAdminServiceImpl.updateRole()` 没有事务注解
3. **开发者可能认为"简单操作不需要事务"**：误以为单表操作不需要事务保护

### 错误修改会造成什么后果
1. **角色关系全部丢失**：删除原有关系成功，但插入新关系失败，管理员失去所有角色
2. **菜单权限丢失**：角色菜单关系删除成功但插入失败，角色无法访问任何菜单
3. **资源权限丢失**：角色资源关系删除成功但插入失败，动态权限校验失败
4. **无法登录**：管理员失去角色后，可能无法登录或访问系统
5. **生产事故**：如果修改超级管理员角色失败，可能需要直接操作数据库恢复

### 正确修改方式或排查入口
```java
// 正确方式：添加事务保护
@Transactional(rollbackFor = Exception.class)
@Override
public int updateRole(Long adminId, List<Long> roleIds) {
    int count = roleIds == null ? 0 : roleIds.size();
    // 先删除原来的关系
    UmsAdminRoleRelationExample adminRoleRelationExample = new UmsAdminRoleRelationExample();
    adminRoleRelationExample.createCriteria().andAdminIdEqualTo(adminId);
    adminRoleRelationMapper.deleteByExample(adminRoleRelationExample);
    // 建立新关系
    if (!CollectionUtils.isEmpty(roleIds)) {
        List<UmsAdminRoleRelation> list = new ArrayList<>();
        for (Long roleId : roleIds) {
            UmsAdminRoleRelation relation = new UmsAdminRoleRelation();
            relation.setAdminId(adminId);
            relation.setRoleId(roleId);
            list.add(relation);
        }
        adminRoleRelationDao.insertList(list);
    }
    getCacheService().delResourceList(adminId);
    return count;
}
```

**排查入口**：
- 检查 `UmsAdminService` 接口和实现类是否有 `@Transactional`
- 查看 `updateRole()`、`allocMenu()`、`allocResource()` 方法的事务配置
- 测试异常场景：在删除后、插入前抛出异常，检查数据是否回滚
- 建议：所有"先删后插"模式的方法都必须有事务保护

---

## 风险点 8：事务方法缺少 rollbackFor 配置（中风险）

### 具体代码位置
- `mall-portal/src/main/java/com/macro/mall/portal/service/OmsPortalOrderService.java`
  - 接口上的 `@Transactional` 注解
- `mall-admin/src/main/java/com/macro/mall/service/PmsProductService.java`
  - 接口上的 `@Transactional` 注解
- 所有 Service 接口层的事务注解

### 为什么容易误解
1. **默认行为认知偏差**：开发者可能认为 `@Transactional` 默认回滚所有异常
2. **Spring 默认行为**：`@Transactional` 默认只回滚 `RuntimeException` 和 `Error`
3. **受检异常被忽略**：如果方法抛出 `IOException`、`SQLException` 等受检异常，事务不会回滚
4. **接口层注解不规范**：注解在接口上，实现类可能覆盖或修改行为

### 错误修改会造成什么后果
1. **受检异常不回滚**：数据库部分操作成功，部分失败，数据不一致
2. **业务逻辑错误掩盖**：异常被捕获但事务未回滚，留下脏数据
3. **排查困难**：问题复现需要构造特定的受检异常场景
4. **资金风险**：订单、支付等关键业务可能出现资金损失

### 正确修改方式或排查入口
```java
// 正确方式：显式指定 rollbackFor
@Transactional(rollbackFor = Exception.class)
public interface OmsPortalOrderService {
    // 方法定义...
}

// 或者在实现类上添加
@Service
@Transactional(rollbackFor = Exception.class)
public class OmsPortalOrderServiceImpl implements OmsPortalOrderService {
    // 方法实现...
}
```

**排查入口**：
- 全局搜索 `@Transactional`，检查是否有 `rollbackFor` 属性
- 检查是否有方法抛出受检异常但没有配置 `rollbackFor`
- 建议项目规范：所有 `@Transactional` 都必须包含 `rollbackFor = Exception.class`

---

## 风险点 9：库存锁定与真实扣减两阶段不一致（中高风险）

### 具体代码位置
- `mall-portal/src/main/java/com/macro/mall/portal/service/impl/OmsPortalOrderServiceImpl.java:750-759`
  - `lockStock()` 方法，锁定库存
- `mall-portal/src/main/java/com/macro/mall/portal/service/impl/OmsPortalOrderServiceImpl.java:255-283`
  - `paySuccess()` 方法，扣减真实库存
- `mall-portal/src/main/resources/dao/PortalOrderDao.xml:91-106`
  - 库存操作 SQL

### 为什么容易误解
1. **两阶段操作复杂**：下单时锁定库存（`lock_stock++`），支付成功后才扣减真实库存（`stock--`, `lock_stock--`）
2. **锁定有条件**：`lockStockBySkuId()` 的 SQL 有条件 `lock_stock + #{quantity} <= stock`
3. **扣减也有条件**：`reduceSkuStock()` 的 SQL 有条件 `stock - #{quantity} >= 0 AND lock_stock - #{quantity} >= 0`
4. **无分布式事务**：两个阶段在不同请求中，没有事务保证

### 错误修改会造成什么后果
1. **超卖风险**：如果支付成功时库存扣减条件检查有漏洞，可能出现超卖
2. **库存锁定不释放**：订单取消失败或消息丢失，库存锁定无法释放
3. **重复扣减**：支付回调被多次调用，可能导致库存重复扣减
4. **库存不一致**：数据库中的 `stock`、`lock_stock` 字段与实际库存不符

### 正确修改方式或排查入口
```sql
-- 确保库存操作的 SQL 条件正确
-- 锁定库存（下单时）
UPDATE pms_sku_stock
SET lock_stock = lock_stock + #{quantity}
WHERE id = #{productSkuId}
  AND lock_stock + #{quantity} <= stock  -- 确保锁定后不超过真实库存

-- 扣减真实库存（支付成功时）
UPDATE pms_sku_stock
SET lock_stock = lock_stock - #{quantity},
    stock = stock - #{quantity}
WHERE id = #{productSkuId}
  AND stock - #{quantity} >= 0  -- 真实库存足够
  AND lock_stock - #{quantity} >= 0  -- 锁定库存足够
```

**排查入口**：
- 检查 `PortalOrderDao.xml` 中的库存操作 SQL
- 确认 `paySuccess()` 方法的幂等性（只处理状态为 0 的订单）
- 测试取消订单流程，确认库存是否正确释放
- 建议：
  1. 添加库存操作日志表，记录每次库存变更
  2. 定期对账，对比订单量和库存变化
  3. 考虑使用 Redis 分布式锁防止并发扣减

---

## 风险点 10：缓存与数据库双写不一致（中风险）

### 具体代码位置
- `mall-portal/src/main/java/com/macro/mall/portal/service/impl/UmsMemberCacheServiceImpl.java:17-67`
  - 会员缓存服务
- `mall-admin/src/main/java/com/macro/mall/service/impl/UmsAdminCacheServiceImpl.java`
  - 管理员缓存服务
- `mall-admin/src/main/java/com/macro/mall/service/impl/UmsAdminServiceImpl.java:171-188`
  - 管理员更新方法，包含缓存清理

### 为什么容易误解
1. **缓存更新策略不明确**：是先更数据库再删缓存，还是先删缓存再更数据库？
2. **清理时机不当**：缓存清理在数据库操作之前或之后？
3. **异常时的处理**：如果数据库更新成功但缓存清理失败，会怎样？
4. **并发场景**：读请求和写请求并发时，可能出现缓存脏数据

### 错误修改会造成什么后果
1. **读到旧数据**：用户信息已更新，但缓存中仍是旧数据
2. **权限不一致**：管理员权限已变更，但缓存中仍是旧权限
3. **安全风险**：用户已被禁用，但缓存中仍认为是启用状态
4. **排查困难**：缓存问题难以复现，具有随机性

### 正确修改方式或排查入口
```java
// 正确的缓存更新策略：先更新数据库，再删除缓存
@Override
@Transactional(rollbackFor = Exception.class)
public int update(Long id, UmsAdmin admin) {
    admin.setId(id);
    // ... 密码处理逻辑 ...
    
    // 1. 先更新数据库
    int count = adminMapper.updateByPrimaryKeySelective(admin);
    
    // 2. 再删除缓存（注意：即使删除失败，也不应该影响数据库更新）
    try {
        getCacheService().delAdmin(id);
    } catch (Exception e) {
        LOGGER.error("delete admin cache failed, adminId:{}", id, e);
        // 记录到缓存清理失败表，后续重试
    }
    
    return count;
}
```

**排查入口**：
- 检查所有缓存操作的顺序：数据库更新和缓存清理的先后
- 确认缓存清理是否在 try-catch 中，避免影响主流程
- 查看是否有缓存过期时间设置，作为最终一致性的兜底
- 建议：
  1. 采用"Cache-Aside Pattern"：先更新数据库，再删缓存
  2. 给缓存设置合理的过期时间（如 30 分钟）
  3. 考虑使用 MQ 异步清理缓存，确保最终一致性

---

## 风险等级汇总

| 风险点 | 风险等级 | 涉及领域 |
|--------|----------|----------|
| 1. 订单提交无事务保护 | 🔴 高 | 订单、事务 |
| 2. 动态权限缓存不刷新 | 🔴 高 | 权限、缓存 |
| 3. MQ 取消订单消息无确认 | 🔴 高 | 消息队列、订单 |
| 4. 订单无重复提交防护 | 🔴 高 | 订单、并发 |
| 5. ES 与数据库同步不一致 | 🔴 高 | 搜索同步 |
| 6. 代码生成器覆盖自定义代码 | 🟠 中高 | 代码生成 |
| 7. 角色分配无事务保护 | 🔴 高 | 权限、事务 |
| 8. 事务缺少 rollbackFor | 🟡 中 | 事务 |
| 9. 库存两阶段不一致 | 🟠 中高 | 库存、订单 |
| 10. 缓存与数据库双写不一致 | 🟡 中 | 缓存 |

---

## 建议优先级

### 第一优先级（立即处理）
1. 订单提交添加事务保护
2. 角色分配添加事务保护
3. 订单提交添加重复提交防护
4. MQ 消息添加确认机制

### 第二优先级（本周处理）
5. 动态权限缓存刷新机制
6. ES 与数据库同步机制
7. 所有事务添加 rollbackFor

### 第三优先级（规划处理）
8. 代码生成器使用规范制定
9. 库存对账机制
10. 缓存一致性优化

---

*文档生成时间：2026-05-09*
*分析范围：mall-admin、mall-portal、mall-search、mall-security、mall-mbg 模块*