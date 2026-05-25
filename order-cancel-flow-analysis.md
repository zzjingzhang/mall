# Mall 项目订单取消流程分析

## 一、两种取消订单的方式概述

Mall 项目中存在多种订单取消方式：
1. **用户主动取消订单**：用户在前端商城主动发起取消
2. **超时自动取消订单**：系统自动取消超时未支付的订单，包含两种实现方式：
   - RabbitMQ 延迟消息机制（主要方式）
   - 定时任务扫描机制（已注释掉的备选方案）

---

## 二、用户主动取消订单分析

### 1. 入口位置

**控制器层**：
- 文件位置：`mall-portal/src/main/java/com/macro/mall/portal/controller/OmsPortalOrderController.java:92-98`
- 方法名：`cancelUserOrder(Long orderId)`
- 请求路径：`POST /order/cancelUserOrder`
- 调用方式：前端用户发起请求

### 2. 是否通过 RabbitMQ 延迟消息实现

**否**，用户主动取消订单是**同步直接调用**，不经过消息队列。

### 3. 取消时的资源恢复

**核心实现**：`mall-portal/src/main/java/com/macro/mall/portal/service/impl/OmsPortalOrderServiceImpl.java:315-348`

#### 3.1 订单状态处理
```java
// 只查询状态为 0（待付款）且未删除的订单
OmsOrderExample example = new OmsOrderExample();
example.createCriteria().andIdEqualTo(orderId).andStatusEqualTo(0).andDeleteStatusEqualTo(0);
List<OmsOrder> cancelOrderList = orderMapper.selectByExample(example);
if (CollectionUtils.isEmpty(cancelOrderList)) {
    return; // 订单不存在或状态不符合，直接返回
}
// 修改订单状态为 4（已关闭）
cancelOrder.setStatus(4);
orderMapper.updateByPrimaryKeySelective(cancelOrder);
```

#### 3.2 库存恢复
```java
// 解除订单商品库存锁定
for (OmsOrderItem orderItem : orderItemList) {
    int count = portalOrderDao.releaseStockBySkuId(orderItem.getProductSkuId(), orderItem.getProductQuantity());
    if(count==0){
        Asserts.fail("库存不足，无法释放！");
    }
}
```
- 调用 `portalOrderDao.releaseStockBySkuId()` 释放锁定库存
- 逐个商品释放，失败则抛出异常

#### 3.3 优惠券恢复
```java
// 修改优惠券使用状态
updateCouponStatus(cancelOrder.getCouponId(), cancelOrder.getMemberId(), 0);
```
- 调用 `updateCouponStatus()` 方法
- 将优惠券状态从 1（已使用）改回 0（未使用）

#### 3.4 积分恢复
```java
// 返还使用积分
if (cancelOrder.getUseIntegration() != null) {
    UmsMember member = memberService.getById(cancelOrder.getMemberId());
    memberService.updateIntegration(cancelOrder.getMemberId(), member.getIntegration() + cancelOrder.getUseIntegration());
}
```
- 查询当前用户积分
- 将扣除的积分加回用户账户

### 4. 是否存在重复取消导致状态二次变更的问题

**不存在**，代码中通过前置条件判断保证了只处理符合条件的订单：
```java
example.createCriteria()
    .andIdEqualTo(orderId)
    .andStatusEqualTo(0)  // 只处理状态为 0（待付款）的订单
    .andDeleteStatusEqualTo(0);  // 只处理未删除的订单
```
- 如果订单已经被取消（状态变为 4）或已支付（状态变为 1），查询结果为空，直接 return
- 因此不会出现重复取消导致状态二次变更的问题

### 5. 消息重复消费时是否幂等

由于用户主动取消不涉及消息队列，所以**不涉及消息重复消费问题**。

---

## 三、超时自动取消订单分析（RabbitMQ 延迟消息方式）

### 1. 入口位置

#### 1.1 延迟消息发送入口
**下单时自动发送**：
- 文件位置：`mall-portal/src/main/java/com/macro/mall/portal/service/impl/OmsPortalOrderServiceImpl.java:247`
- 方法名：`sendDelayMessageCancelOrder(order.getId())`
- 触发时机：用户下单成功后

**主动触发接口**：
- 文件位置：`mall-portal/src/main/java/com/macro/mall/portal/controller/OmsPortalOrderController.java:64-70`
- 方法名：`cancelOrder(Long orderId)`
- 请求路径：`POST /order/cancelOrder`
- 说明：用于测试或手动触发单个订单的取消消息

#### 1.2 延迟消息消费入口
- 文件位置：`mall-portal/src/main/java/com/macro/mall/portal/component/CancelOrderReceiver.java:22-25`
- 方法名：`handle(Long orderId)`
- 监听队列：`mall.order.cancel`

### 2. 是否通过 RabbitMQ 延迟消息实现

**是**，使用 **RabbitMQ 死信队列（Dead Letter Queue）实现延迟消息**。

#### RabbitMQ 配置说明

**队列枚举**：`mall-portal/src/main/java/com/macro/mall/portal/domain/QueueEnum.java`
- `QUEUE_TTL_ORDER_CANCEL`：延迟队列（死信队列）
  - 交换机：`mall.order.direct.ttl`
  - 队列名：`mall.order.cancel.ttl`
  - 路由键：`mall.order.cancel.ttl`
- `QUEUE_ORDER_CANCEL`：实际消费队列
  - 交换机：`mall.order.direct`
  - 队列名：`mall.order.cancel`
  - 路由键：`mall.order.cancel`

**配置类**：`mall-portal/src/main/java/com/macro/mall/portal/config/RabbitMqConfig.java`
- 延迟队列配置了死信交换机和死信路由键
- 消息在延迟队列中等待 TTL 过期后，自动转发到实际消费队列

#### 消息发送流程
**发送者**：`mall-portal/src/main/java/com/macro/mall/portal/component/CancelOrderSender.java`
```java
public void sendMessage(Long orderId, final long delayTimes) {
    // 给延迟队列发送消息
    amqpTemplate.convertAndSend(
        QueueEnum.QUEUE_TTL_ORDER_CANCEL.getExchange(),
        QueueEnum.QUEUE_TTL_ORDER_CANCEL.getRouteKey(),
        orderId,
        new MessagePostProcessor() {
            @Override
            public Message postProcessMessage(Message message) throws AmqpException {
                // 给消息设置延迟毫秒值
                message.getMessageProperties().setExpiration(String.valueOf(delayTimes));
                return message;
            }
        }
    );
}
```

#### 延迟时间计算
**延迟时间来源**：`mall-portal/src/main/java/com/macro/mall/portal/service/impl/OmsPortalOrderServiceImpl.java:351-357`
```java
@Override
public void sendDelayMessageCancelOrder(Long orderId) {
    // 获取订单超时时间
    OmsOrderSetting orderSetting = orderSettingMapper.selectByPrimaryKey(1L);
    long delayTimes = orderSetting.getNormalOrderOvertime() * 60 * 1000;
    // 发送延迟消息
    cancelOrderSender.sendMessage(orderId, delayTimes);
}
```
- 从 `OmsOrderSetting` 表获取 `normalOrderOvertime` 字段（单位：分钟）
- 转换为毫秒作为消息 TTL

### 3. 取消时的资源恢复

超时自动取消的资源恢复逻辑**与用户主动取消完全相同**：
- 调用相同的 `cancelOrder(Long orderId)` 方法
- 位置：`mall-portal/src/main/java/com/macro/mall/portal/component/CancelOrderReceiver.java:22-25`
```java
@RabbitHandler
public void handle(Long orderId) {
    portalOrderService.cancelOrder(orderId);
    LOGGER.info("process orderId:{}", orderId);
}
```

资源恢复细节同上：
- **订单状态**：从 0（待付款）改为 4（已关闭）
- **库存**：通过 `portalOrderDao.releaseStockBySkuId()` 释放锁定库存
- **优惠券**：通过 `updateCouponStatus()` 改回未使用状态
- **积分**：加回用户账户

### 4. 是否存在重复取消导致状态二次变更的问题

**不存在**，原因与用户主动取消相同：
- 在 `cancelOrder()` 方法中查询时，只查询状态为 0（待付款）的订单
- 如果订单已经支付（状态 1）或已经取消（状态 4），则查询结果为空，直接 return
- 因此不会出现重复取消的问题

### 5. 消息重复消费时是否幂等

**具有幂等性**，但不是通过消息去重实现，而是通过**业务状态判断**实现：

#### 幂等性实现机制
1. **查询时状态判断**：只处理状态为 0（待付款）的订单
2. **订单状态一旦变更**：支付后变为 1，取消后变为 4，都不会再被处理
3. **多次调用 cancelOrder()**：对同一个订单调用多次，只有第一次有效

#### 潜在问题
虽然具有业务幂等性，但**存在以下潜在问题**：
1. **日志冗余**：重复消费会产生多次日志记录
2. **数据库查询**：重复消费会产生不必要的数据库查询
3. **缺乏消息确认机制**：代码中没有显式的消息确认逻辑

---

## 四、超时自动取消订单分析（定时任务方式）

### 概述
定时任务方式是备选方案，当前已被注释掉（`@Component` 注解被注释）。

### 1. 入口位置

**定时任务类**：`mall-portal/src/main/java/com/macro/mall/portal/component/OrderTimeOutCancelTask.java`
```java
//@Component  // 已被注释
public class OrderTimeOutCancelTask {
    @Scheduled(cron = "0 0/10 * ? * ?")  // 每10分钟执行一次
    private void cancelTimeOutOrder() {
        Integer count = portalOrderService.cancelTimeOutOrder();
        LOGGER.info("取消订单，并根据sku编号释放锁定库存，取消订单数量：{}", count);
    }
}
```

**主动触发接口**：
- 文件位置：`mall-portal/src/main/java/com/macro/mall/portal/controller/OmsPortalOrderController.java:56-62`
- 方法名：`cancelTimeOutOrder()`
- 请求路径：`POST /order/cancelTimeOutOrder`

### 2. 核心实现

**方法**：`mall-portal/src/main/java/com/macro/mall/portal/service/impl/OmsPortalOrderServiceImpl.java:285-312`

```java
@Override
public Integer cancelTimeOutOrder() {
    Integer count = 0;
    OmsOrderSetting orderSetting = orderSettingMapper.selectByPrimaryKey(1L);
    // 查询超时、未支付的订单及订单详情
    List<OmsOrderDetail> timeOutOrders = portalOrderDao.getTimeOutOrders(orderSetting.getNormalOrderOvertime());
    if (CollectionUtils.isEmpty(timeOutOrders)) {
        return count;
    }
    // 修改订单状态为交易取消
    List<Long> ids = new ArrayList<>();
    for (OmsOrderDetail timeOutOrder : timeOutOrders) {
        ids.add(timeOutOrder.getId());
    }
    portalOrderDao.updateOrderStatus(ids, 4);  // 批量更新状态为 4（已关闭）
    
    for (OmsOrderDetail timeOutOrder : timeOutOrders) {
        // 解除订单商品库存锁定
        portalOrderDao.releaseSkuStockLock(timeOutOrder.getOrderItemList());
        // 修改优惠券使用状态
        updateCouponStatus(timeOutOrder.getCouponId(), timeOutOrder.getMemberId(), 0);
        // 返还使用积分
        if (timeOutOrder.getUseIntegration() != null) {
            UmsMember member = memberService.getById(timeOutOrder.getMemberId());
            memberService.updateIntegration(timeOutOrder.getMemberId(), member.getIntegration() + timeOutOrder.getUseIntegration());
        }
    }
    return timeOutOrders.size();
}
```

### 3. 与 RabbitMQ 方式的差异

| 维度 | 定时任务方式 | RabbitMQ 延迟消息方式 |
|------|------------|---------------------|
| **取消时机** | 每 10 分钟扫描一次，存在延迟误差 | 精确到秒，超时立即取消 |
| **资源消耗** | 每次扫描全表，大订单量时性能差 | 只处理超时订单，资源占用低 |
| **实时性** | 差，最多有 10 分钟延迟 | 好，几乎实时 |
| **实现复杂度** | 简单 | 相对复杂，需要配置死信队列 |
| **当前状态** | 已注释，未启用 | 已启用，是主要方式 |

---

## 五、完整调用链

### 5.1 用户主动取消订单调用链

```
用户发起请求
    ↓
OmsPortalOrderController.cancelUserOrder(orderId)
    ↓  直接调用
OmsPortalOrderServiceImpl.cancelOrder(orderId)
    ├── ↓ 查询订单（只查状态=0且未删除）
    │   OmsOrderMapper.selectByExample(example)
    ├── ↓ 更新订单状态
    │   OmsOrderMapper.updateByPrimaryKeySelective(order)
    ├── ↓ 查询订单商品
    │   OmsOrderItemMapper.selectByExample(orderItemExample)
    ├── ↓ 释放库存（逐个商品）
    │   PortalOrderDao.releaseStockBySkuId(skuId, quantity)
    ├── ↓ 恢复优惠券
    │   updateCouponStatus(couponId, memberId, 0)
    │   └── SmsCouponHistoryMapper.updateByPrimaryKeySelective(history)
    └── ↓ 返还积分
        UmsMemberService.getById(memberId)
        UmsMemberService.updateIntegration(memberId, newIntegration)
```

### 5.2 超时自动取消订单调用链（RabbitMQ 方式）

#### 消息发送阶段（下单时）
```
用户下单
    ↓
OmsPortalOrderServiceImpl.generateOrder(orderParam)
    ├── ↓ 锁定库存
    ├── ↓ 扣减优惠券
    ├── ↓ 扣减积分
    ├── ↓ 插入订单和订单项
    └── ↓ 发送延迟取消消息
        OmsPortalOrderServiceImpl.sendDelayMessageCancelOrder(orderId)
            ├── ↓ 获取超时时间
            │   OmsOrderSettingMapper.selectByPrimaryKey(1L)
            └── ↓ 发送延迟消息
                CancelOrderSender.sendMessage(orderId, delayTimes)
                    └── amqpTemplate.convertAndSend(...) 发送到 mall.order.cancel.ttl 队列
```

#### 消息等待阶段
```
消息在 mall.order.cancel.ttl 队列中等待
    ↓  等待 TTL 超时
消息过期，自动转发到死信交换机
    ↓  根据死信路由键路由
消息转发到 mall.order.cancel 队列
```

#### 消息消费阶段
```
CancelOrderReceiver 监听 mall.order.cancel 队列
    ↓
CancelOrderReceiver.handle(orderId)
    ↓
OmsPortalOrderServiceImpl.cancelOrder(orderId)  [与用户主动取消相同]
    ├── ↓ 查询订单（只查状态=0且未删除）
    │   OmsOrderMapper.selectByExample(example)
    ├── ↓ 更新订单状态
    │   OmsOrderMapper.updateByPrimaryKeySelective(order)
    ├── ↓ 查询订单商品
    │   OmsOrderItemMapper.selectByExample(orderItemExample)
    ├── ↓ 释放库存（逐个商品）
    │   PortalOrderDao.releaseStockBySkuId(skuId, quantity)
    ├── ↓ 恢复优惠券
    │   updateCouponStatus(couponId, memberId, 0)
    └── ↓ 返还积分
        UmsMemberService.updateIntegration(memberId, newIntegration)
```

#### 支付成功后取消消息处理（重要）
```
用户支付成功
    ↓
OmsPortalOrderServiceImpl.paySuccess(orderId, payType)
    ├── ↓ 更新订单状态为 1（待发货）
    │   OmsOrderMapper.updateByExampleSelective(order, orderExample)
    │   └── 只更新状态为 0 的订单（使用 CAS 方式）
    └── ↓ 扣减真实库存
        PortalOrderDao.reduceSkuStock(skuId, quantity)

延迟消息到达时：
    ↓
CancelOrderReceiver.handle(orderId)
    ↓
OmsPortalOrderServiceImpl.cancelOrder(orderId)
    └── ↓ 查询时只找 status=0 的订单
        此时订单 status=1，查询结果为空，直接 return
        ↓
        取消逻辑不执行，避免了误取消
```

---

## 六、两种取消方式对比总结

| 对比维度 | 用户主动取消订单 | 超时自动取消订单 |
|---------|---------------|----------------|
| **入口** | `/order/cancelUserOrder` 接口 | 1. 下单时自动发送延迟消息<br>2. `/order/cancelOrder` 测试接口<br>3. 死信队列消费者 |
| **消息队列** | 否，直接调用 | 是，使用 RabbitMQ 死信队列 |
| **取消时机** | 用户主动发起，实时 | 订单超时后自动取消 |
| **核心方法** | `cancelOrder(orderId)` | `cancelOrder(orderId)`（同一个方法） |
| **库存恢复** | `releaseStockBySkuId()` 逐个释放 | 与主动取消相同 |
| **优惠券恢复** | `updateCouponStatus()` 改回未使用 | 与主动取消相同 |
| **积分恢复** | 加回用户账户 | 与主动取消相同 |
| **订单状态** | 0 → 4 | 0 → 4 |
| **重复取消保护** | 有，只处理 status=0 的订单 | 有，相同机制 |
| **幂等性** | 不涉及消息队列 | 有，基于状态判断 |

---

## 七、代码设计亮点与潜在问题

### 7.1 设计亮点

1. **核心取消逻辑复用**：
   - 用户主动取消和超时取消都调用同一个 `cancelOrder()` 方法
   - 代码复用，维护方便

2. **状态前置检查**：
   - 只处理 `status=0`（待付款）的订单
   - 天然防止重复取消和误取消

3. **支付成功时的 CAS 更新**：
   ```java
   orderExample.createCriteria()
       .andIdEqualTo(order.getId())
       .andDeleteStatusEqualTo(0)
       .andStatusEqualTo(0);  // 只更新状态为 0 的订单
   int updateCount = orderMapper.updateByExampleSelective(order, orderExample);
   ```
   - 利用数据库的条件更新实现 CAS（Compare And Swap）
   - 防止并发支付问题

4. **RabbitMQ 死信队列实现延迟消息**：
   - 无需引入额外的延迟队列插件
   - 可靠性高，消息持久化

### 7.2 潜在问题与改进建议

1. **RabbitMQ 消息幂等性**：
   - **现状**：依赖业务状态判断实现幂等
   - **问题**：重复消费会产生不必要的数据库查询
   - **建议**：可以增加 Redis 去重，记录已处理的 orderId，设置过期时间

2. **取消订单时的事务问题**：
   - **现状**：`cancelOrder()` 方法没有 `@Transactional` 注解
   - **问题**：如果库存释放成功但积分返还失败，会导致数据不一致
   - **建议**：添加 `@Transactional(rollbackFor = Exception.class)` 注解

3. **批量库存释放效率**：
   - **现状**：`cancelOrder()` 中逐个商品调用 `releaseStockBySkuId()`
   - **问题**：多个商品时产生多次数据库交互
   - **建议**：参考 `cancelTimeOutOrder()` 中的 `releaseSkuStockLock()` 批量处理方式

4. **管理后台关闭订单不完善**：
   - **位置**：`mall-admin/src/main/java/com/macro/mall/service/impl/OmsOrderServiceImpl.java:61-78`
   - **现状**：只更新订单状态为 4，没有恢复库存、优惠券、积分
   - **问题**：管理后台关闭订单会导致资源锁定无法释放
   - **建议**：复用前台的取消逻辑，或实现完整的资源恢复

5. **定时任务与延迟消息同时启用的风险**：
   - **现状**：两种方式代码都存在，定时任务已注释
   - **风险**：如果同时启用，同一订单可能被取消两次
   - **建议**：明确选择一种方式，删除另一种的代码或配置

6. **消息确认机制**：
   - **现状**：`CancelOrderReceiver` 没有显式的消息确认逻辑
   - **问题**：依赖 Spring AMQP 的默认确认机制
   - **建议**：根据业务需求考虑是否需要手动确认（ack/nack）

---

## 八、订单状态流转图

```
下单成功
    ↓
┌─────────────┐
│  status=0   │  待付款
│  (未支付)   │
└─────────────┘
    ↓                    ↓
    ↓ 支付成功            ↓ 用户取消 / 超时取消
    ↓                    ↓
┌─────────────┐      ┌─────────────┐
│  status=1   │      │  status=4   │
│  待发货      │      │  已关闭      │
│  (已支付)   │      │  (取消订单)  │
└─────────────┘      └─────────────┘
    ↓
    ↓ 发货
    ↓
┌─────────────┐
│  status=2   │
│  已发货      │
└─────────────┘
    ↓
    ↓ 确认收货
    ↓
┌─────────────┐
│  status=3   │
│  已完成      │
└─────────────┘
```

---

## 九、关键文件位置索引

| 文件路径 | 说明 |
|---------|------|
| `mall-portal/src/main/java/com/macro/mall/portal/controller/OmsPortalOrderController.java` | 前台订单控制器，包含所有取消入口 |
| `mall-portal/src/main/java/com/macro/mall/portal/service/impl/OmsPortalOrderServiceImpl.java` | 订单服务实现，核心取消逻辑 |
| `mall-portal/src/main/java/com/macro/mall/portal/component/CancelOrderSender.java` | 延迟消息发送者 |
| `mall-portal/src/main/java/com/macro/mall/portal/component/CancelOrderReceiver.java` | 延迟消息消费者 |
| `mall-portal/src/main/java/com/macro/mall/portal/component/OrderTimeOutCancelTask.java` | 定时任务（已注释） |
| `mall-portal/src/main/java/com/macro/mall/portal/config/RabbitMqConfig.java` | RabbitMQ 配置 |
| `mall-portal/src/main/java/com/macro/mall/portal/domain/QueueEnum.java` | 队列枚举定义 |
| `mall-admin/src/main/java/com/macro/mall/service/impl/OmsOrderServiceImpl.java` | 后台订单服务（关闭订单不完善） |
