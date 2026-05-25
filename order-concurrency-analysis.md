# mall-portal 提交订单流程并发安全性分析

## 1. 核心流程概述

提交订单的主要入口位于：

- **Controller**: `OmsPortalOrderController.generateOrder()` 
  - 文件位置: `mall-portal/src/main/java/com/macro/mall/portal/controller/OmsPortalOrderController.java:40-46`

- **Service**: `OmsPortalOrderServiceImpl.generateOrder()`
  - 文件位置: `mall-portal/src/main/java/com/macro/mall/portal/service/impl/OmsPortalOrderServiceImpl.java:96-252`

---

## 2. 库存扣减分析

### 2.1 库存扣减发生的位置

库存操作分为两个阶段：

#### 阶段一：下单时锁定库存
- **方法**: `OmsPortalOrderServiceImpl.lockStock()`
  - 文件位置: `mall-portal/src/main/java/com/macro/mall/portal/service/impl/OmsPortalOrderServiceImpl.java:750-759`
  - **调用时机**: `generateOrder()` 方法第 167 行

- **DAO 方法**: `PortalOrderDao.lockStockBySkuId()`
  - 文件位置: `mall-portal/src/main/java/com/macro/mall/portal/dao/PortalOrderDao.java:43`
  - **SQL**: `mall-portal/src/main/resources/dao/PortalOrderDao.xml:91-97`

```sql
UPDATE pms_sku_stock
SET lock_stock = lock_stock + #{quantity}
WHERE
id = #{productSkuId}
AND lock_stock + #{quantity} <= stock
```

#### 阶段二：支付成功时扣减真实库存
- **方法**: `OmsPortalOrderServiceImpl.paySuccess()`
  - 文件位置: `mall-portal/src/main/java/com/macro/mall/portal/service/impl/OmsPortalOrderServiceImpl.java:255-283`
  - **调用链**: 第 273-277 行，调用 `portalOrderDao.reduceSkuStock()`

- **DAO 方法**: `PortalOrderDao.reduceSkuStock()`
  - 文件位置: `mall-portal/src/main/java/com/macro/mall/portal/dao/PortalOrderDao.java:48`
  - **SQL**: `mall-portal/src/main/resources/dao/PortalOrderDao.xml:98-106`

```sql
UPDATE pms_sku_stock
SET lock_stock = lock_stock - #{quantity},
    stock = stock - #{quantity}
WHERE
    id = #{productSkuId}
  AND stock - #{quantity} >= 0
  AND lock_stock - #{quantity} >= 0
```

### 2.2 超卖风险分析

#### 风险点：库存检查与锁定之间存在时间差

在 `generateOrder()` 方法中：

1. **第 125-128 行**: 检查库存是否充足
```java
if (!hasStock(cartPromotionItemList)) {
    Asserts.fail("库存不足，无法下单");
}
```

2. **第 167 行**: 锁定库存
```java
lockStock(cartPromotionItemList);
```

**问题分析**:
- `hasStock()` 方法（第 764-774 行）只是读取 `cartPromotionItem.getRealStock()`，这是从购物车信息中获取的，而 `RealStock = stock - lock_stock`
- 从检查库存到真正锁定库存之间存在时间窗口（约 40 行代码的执行时间）
- 在此期间，其他并发请求可能已经锁定了部分库存

#### 缓解措施：SQL 层面的条件检查

虽然代码层面有时间差，但 `lockStockBySkuId()` 的 SQL 使用了条件更新：

```sql
AND lock_stock + #{quantity} <= stock
```

并且 `lockStock()` 方法会检查返回值：
```java
int count = portalOrderDao.lockStockBySkuId(cartPromotionItem.getProductSkuId(),cartPromotionItem.getQuantity());
if(count==0){
    Asserts.fail("库存不足，无法下单");
}
```

#### 超卖风险判断：**中等风险**

- **优点**: SQL 层面有条件检查，即使并发也不会出现超卖
- **问题**: 前置检查 `hasStock()` 无意义，真正的校验在 SQL 层面
- **潜在问题**: 如果一个订单包含多个商品，锁定是逐个进行的（在 `lockStock()` 的 for 循环中），可能出现部分成功、部分失败的情况，导致不一致

---

## 3. 价格计算逻辑分析

### 3.1 促销价格计算

**核心服务**: `OmsPromotionServiceImpl.calcCartPromotion()`
- 文件位置: `mall-portal/src/main/java/com/macro/mall/portal/service/impl/OmsPromotionServiceImpl.java:29-110`

支持的促销类型（在 `pms_product.promotion_type` 字段）：

1. **单品促销 (type=1)**
   - 代码位置: 第 41-57 行
   - 逻辑: 使用 `skuStock.getPromotionPrice()` 作为促销价
   - 优惠金额 = `originalPrice - promotionPrice`

2. **打折优惠 (type=3)**
   - 代码位置: 第 58-80 行
   - 逻辑: 根据购买数量匹配 `pms_product_ladder` 表的折扣规则
   - 优惠金额 = `originalPrice - (discount * originalPrice)`

3. **满减优惠 (type=4)**
   - 代码位置: 第 81-103 行
   - 逻辑: 根据总金额匹配 `pms_product_full_reduction` 表的满减规则
   - 优惠金额分摊到每个商品

4. **无优惠**
   - 代码位置: 第 104-107 行

### 3.2 优惠券计算

**核心方法**: `OmsPortalOrderServiceImpl.handleCouponAmount()`
- 文件位置: `mall-portal/src/main/java/com/macro/mall/portal/service/impl/OmsPortalOrderServiceImpl.java:653-667`

优惠券使用类型（`sms_coupon.use_type`）：

1. **全场通用 (type=0)**: 第 655-657 行
2. **指定分类 (type=1)**: 第 658-661 行，通过 `getCouponOrderItemByRelation()` 筛选
3. **指定商品 (type=2)**: 第 662-666 行，通过 `getCouponOrderItemByRelation()` 筛选

**分摊逻辑**: `calcPerCouponAmount()` 方法（第 674-681 行）
- 公式: `(商品价格/可用商品总价) * 优惠券面额`

### 3.3 积分抵扣计算

**核心方法**: `OmsPortalOrderServiceImpl.getUseIntegrationAmount()`
- 文件位置: `mall-portal/src/main/java/com/macro/mall/portal/service/impl/OmsPortalOrderServiceImpl.java:621-645`

检查条件：
1. 用户积分是否足够（第 624 行）
2. 是否可与优惠券共用（`UmsIntegrationConsumeSetting.coupon_status`，第 630-633 行）
3. 是否达到最低使用积分门槛（第 635-637 行）
4. 是否超过订单抵用最高百分比（第 639-643 行）

### 3.4 会员价格

**分析结论**: **不确定 - 未实现**

- 搜索结果显示，数据库中有 `pms_member_price` 表
- 但在 `OmsPromotionServiceImpl` 的 `calcCartPromotion()` 方法中，**没有处理会员价格的逻辑**
- `getPromotionProductList()` 的 SQL 查询中也没有查询 `pms_member_price` 表
- 代码中 `promotion_type` 只处理了 1（单品促销）、3（打折）、4（满减）三种类型

缺少的代码证据：
- 没有从 `pms_member_price` 表查询会员价格的 SQL
- 没有根据当前用户的 `member_level_id` 匹配会员价格的逻辑

### 3.5 实付金额计算

**方法**: `handleRealAmount()`（第 534-543 行）
```java
BigDecimal realAmount = orderItem.getProductPrice()
        .subtract(orderItem.getPromotionAmount())
        .subtract(orderItem.getCouponAmount())
        .subtract(orderItem.getIntegrationAmount());
```

**订单应付金额**: `calcPayAmount()`（第 564-572 行）
```java
BigDecimal payAmount = order.getTotalAmount()
        .add(order.getFreightAmount())
        .subtract(order.getPromotionAmount())
        .subtract(order.getCouponAmount())
        .subtract(order.getIntegrationAmount());
```

---

## 4. 事务保护分析

### 4.1 关键发现

**结论**: **无事务保护**

在 `OmsPortalOrderServiceImpl` 的 `generateOrder()` 方法上**没有** `@Transactional` 注解。

搜索确认：
```
Grep 结果: No matches found (在 mall-portal/service/impl 目录下搜索 @Transactional)
```

### 4.2 风险点分析

`generateOrder()` 方法执行了以下数据库操作（按顺序）：

| 序号 | 操作 | 代码行号 | 可能的问题 |
|------|------|----------|------------|
| 1 | 锁定库存 | 167 | 如果后续操作失败，库存已锁定但订单未创建 |
| 2 | 插入订单主表 | 226 | |
| 3 | 插入订单商品表 | 231 | |
| 4 | 更新优惠券状态 | 234 | |
| 5 | 扣减会员积分 | 242 | |
| 6 | 删除购物车商品 | 245 | |
| 7 | 发送延迟消息 | 247 | |

**典型的并发/失败场景**：

1. **场景 A**: 库存锁定成功，但订单创建失败
   - 结果: 库存被锁定，无法释放，导致商品"卖出"但无订单
   - 依赖: 只能靠延迟消息 `cancelOrder()` 或定时任务 `cancelTimeOutOrder()` 释放

2. **场景 B**: 订单创建成功，但购物车删除失败
   - 结果: 用户可以再次用相同的购物车商品下单

3. **场景 C**: 订单创建成功，但优惠券/积分扣减失败
   - 结果: 用户实际支付了更少的金额（未使用优惠券/积分），但订单已创建

4. **场景 D**: 订单创建成功，但延迟消息发送失败
   - 结果: 如果用户不支付，库存无法自动释放

---

## 5. Redis、数据库、消息队列状态一致性分析

### 5.1 Redis 使用位置

Redis 仅用于**生成订单号**：

**方法**: `generateOrderSn()`（第 462-477 行）
```java
String key = REDIS_DATABASE+":"+ REDIS_KEY_ORDER_ID + date;
Long increment = redisService.incr(key, 1);
```

### 5.2 状态一致性分析

#### 分析 1: 订单号生成
- Redis 用于自增生成唯一 ID
- 数据库不依赖这个值，订单号是生成后再写入数据库
- **风险**: 低。即使 Redis 宕机，订单号生成失败，整个下单流程会失败，不会造成不一致

#### 分析 2: 库存状态（锁定 vs 真实扣减）

库存状态流转：
```
下单锁定 (lock_stock++) 
    ↓ 支付成功
真实扣减 (lock_stock--, stock--)
    ↓ 超时/取消
释放锁定 (lock_stock--)
```

**潜在不一致场景**:

1. **订单创建成功 → 延迟消息发送失败**
   - 如果用户不支付，库存永远被锁定
   - 代码位置: `generateOrder()` 第 247 行发送消息，但没有异常处理

2. **支付回调处理**
   - `paySuccess()` 方法（第 255-283 行）
   - 先更新订单状态，再扣减库存
   - 如果订单状态更新成功但库存扣减失败，订单状态是"待发货"但库存未扣减
   - **风险**: 可能导致超卖（多次支付回调？）

3. **取消订单的补偿逻辑**
   - `cancelOrder()` 方法（第 315-348 行）
   - 先修改订单状态，再释放库存
   - 如果库存释放失败，订单已取消但库存未释放

### 5.3 消息队列可靠性

**消息发送**: `CancelOrderSender.sendMessage()`（第 23-34 行）
```java
amqpTemplate.convertAndSend(..., orderId, new MessagePostProcessor() {
    @Override
    public Message postProcessMessage(Message message) throws AmqpException {
        message.getMessageProperties().setExpiration(String.valueOf(delayTimes));
        return message;
    }
});
```

**消息接收**: `CancelOrderReceiver.handle()`（第 22-25 行）
```java
public void handle(Long orderId){
    portalOrderService.cancelOrder(orderId);
}
```

**风险分析**:
- 发送消息时没有异常处理机制
- 没有消息确认（ack）机制
- `cancelOrder()` 方法执行失败不会重试

### 5.4 一致性判断：**高风险**

主要问题：
1. 无分布式事务保证
2. 消息发送无确认机制
3. 各操作之间无回滚保障
4. 补偿逻辑（取消订单）本身也可能失败

---

## 6. 重复提交订单的处理

### 6.1 关键发现

**结论**: **没有任何重复提交防护措施**

搜索结果：
```
Grep 结果: No matches found (搜索 repeat, duplicate, 重复, 幂等)
```

### 6.2 代码证据

`OmsPortalOrderController.generateOrder()`（第 40-46 行）：
```java
@RequestMapping(value = "/generateOrder", method = RequestMethod.POST)
@ResponseBody
public CommonResult generateOrder(@RequestBody OrderParam orderParam) {
    Map<String, Object> result = portalOrderService.generateOrder(orderParam);
    return CommonResult.success(result, "下单成功");
}
```

**问题分析**:

1. **无令牌机制**: 没有请求唯一标识（如 token）
2. **无幂等键**: 没有基于业务主键的幂等判断
3. **无 Redis 锁**: 没有防止短时间内重复请求的机制
4. **前端可重复点击**: 接口本身是幂等的，但如果用户快速点击两次，可以创建两个相同的订单

### 6.3 重复提交风险判断：**高风险**

典型场景：
- 用户网络延迟，多次点击"提交订单"按钮
- 恶意用户通过脚本批量发送请求
- 没有订单号去重逻辑（订单号每次都是新生成的）

---

## 7. 总结与风险等级

| 检查项 | 风险等级 | 关键问题 |
|--------|----------|----------|
| 库存扣减超卖 | ⚠️ 中等 | 有 SQL 条件检查，但前置校验无效，多商品锁定无原子性 |
| 事务保护 | 🔴 高 | 完全无 `@Transactional` 注解，多步操作无回滚 |
| 状态一致性 | 🔴 高 | Redis/DB/MQ 无分布式事务，消息无确认机制 |
| 重复提交 | 🔴 高 | 完全无防护，无 token、无幂等、无锁 |
| 价格计算 | 🟡 低-中等 | 促销/优惠券/积分逻辑基本完整，但**会员价格未实现** |

### 7.1 具体代码位置参考

#### 高风险代码位置：

1. **无事务的 `generateOrder()`**: 
   - `mall-portal/src/main/java/com/macro/mall/portal/service/impl/OmsPortalOrderServiceImpl.java:96-252`

2. **无重复提交防护的 Controller**:
   - `mall-portal/src/main/java/com/macro/mall/portal/controller/OmsPortalOrderController.java:40-46`

3. **无确认的消息发送**:
   - `mall-portal/src/main/java/com/macro/mall/portal/component/CancelOrderSender.java:23-34`

4. **无异常处理的消息接收**:
   - `mall-portal/src/main/java/com/macro/mall/portal/component/CancelOrderReceiver.java:22-25`

#### 需要确认/补充的代码：

1. **会员价格实现**: 检查是否在其他模块（如 mall-admin）有实现，但 mall-portal 未调用
2. **是否有全局事务配置**: 检查是否有 AOP 切面实现的事务管理
3. **前端是否有防重复点击**: 这需要查看前端代码（不在当前项目中）

---

*分析完成时间: 2026-05-09*
