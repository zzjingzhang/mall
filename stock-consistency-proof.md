# Mall 项目"下单扣库存"一致性分析报告

## 1. 涉及库存字段变更的具体 Mapper/SQL

### 1.1 库存操作核心文件
- **Mapper XML**: `mall-portal/src/main/resources/dao/PortalOrderDao.xml`
- **DAO 接口**: `mall-portal/src/main/java/com/macro/mall/portal/dao/PortalOrderDao.java`
- **Service 实现**: `mall-portal/src/main/java/com/macro/mall/portal/service/impl/OmsPortalOrderServiceImpl.java`

### 1.2 关键 SQL 语句

#### 1.2.1 库存锁定（下单时调用）
**文件**: `PortalOrderDao.xml:91-97`
```xml
<update id="lockStockBySkuId">
    UPDATE pms_sku_stock
    SET lock_stock = lock_stock + #{quantity}
    WHERE
    id = #{productSkuId}
    AND lock_stock + #{quantity} <= stock
</update>
```

**说明**: 
- 下单时先锁定库存，增加 `lock_stock` 字段
- 条件判断：`lock_stock + #{quantity} <= stock`，确保锁定库存不超过可用库存

#### 1.2.2 真实库存扣减（支付成功时调用）
**文件**: `PortalOrderDao.xml:98-106`
```xml
<update id="reduceSkuStock">
    UPDATE pms_sku_stock
    SET lock_stock = lock_stock - #{quantity},
        stock = stock - #{quantity}
    WHERE
        id = #{productSkuId}
      AND stock - #{quantity} >= 0
      AND lock_stock - #{quantity} >= 0
</update>
```

**说明**:
- 支付成功后扣减真实库存
- 同时减少 `lock_stock`（释放锁定）和 `stock`（真实库存）
- 双重条件判断：
  - `stock - #{quantity} >= 0`：确保真实库存不会变负
  - `lock_stock - #{quantity} >= 0`：确保锁定库存不会变负

#### 1.2.3 批量库存扣减（备用方法，未被调用）
**文件**: `PortalOrderDao.xml:50-68`
```xml
<update id="updateSkuStock">
    UPDATE pms_sku_stock
    SET
        stock = CASE id
        <foreach collection="itemList" item="item">
          WHEN #{item.productSkuId} THEN stock - #{item.productQuantity}
        </foreach>
        END,
        lock_stock = CASE id
        <foreach collection="itemList" item="item">
          WHEN #{item.productSkuId} THEN lock_stock - #{item.productQuantity}
        </foreach>
        END
    WHERE
        id IN
    <foreach collection="itemList" item="item" separator="," open="(" close=")">
        #{item.productSkuId}
    </foreach>
</update>
```

**说明**:
- 批量扣减库存方法
- **注意：没有任何条件判断**，直接扣减
- 经代码搜索，该方法在当前项目中**未被任何 Service 调用**

#### 1.2.4 库存释放（订单取消时调用）
**文件**: `PortalOrderDao.xml:77-90`（批量释放）
```xml
<update id="releaseSkuStockLock">
    UPDATE pms_sku_stock
    SET
    lock_stock = CASE id
    <foreach collection="itemList" item="item">
        WHEN #{item.productSkuId} THEN lock_stock - #{item.productQuantity}
    </foreach>
    END
    WHERE
    id IN
    <foreach collection="itemList" item="item" separator="," open="(" close=")">
        #{item.productSkuId}
    </foreach>
</update>
```

**文件**: `PortalOrderDao.xml:107-113`（单条释放）
```xml
<update id="releaseStockBySkuId">
    UPDATE pms_sku_stock
    SET lock_stock = lock_stock - #{quantity}
    WHERE
        id = #{productSkuId}
      AND lock_stock - #{quantity} >= 0
</update>
```

**说明**:
- 订单取消时释放锁定库存
- 只减少 `lock_stock`，不恢复 `stock`
- 单条释放带条件判断：`lock_stock - #{quantity} >= 0`
- 批量释放不带条件判断

---

## 2. 扣减库存是否带条件判断

### 2.1 库存锁定（下单阶段）
- **方法**: `lockStockBySkuId`
- **是否带条件**: ✅ 是
- **条件**: `lock_stock + #{quantity} <= stock`
- **作用**: 确保锁定库存总量不超过可用库存

### 2.2 真实库存扣减（支付阶段）
- **方法**: `reduceSkuStock`
- **是否带条件**: ✅ 是
- **条件**: 
  - `stock - #{quantity} >= 0`
  - `lock_stock - #{quantity} >= 0`
- **作用**: 确保真实库存和锁定库存都不会变成负数

### 2.3 备用批量方法（未使用）
- **方法**: `updateSkuStock`
- **是否带条件**: ❌ 否
- **风险**: 如果被调用，可能导致负库存

---

## 3. 并发扣减时数据库层是否能防止负库存

### 3.1 下单时的库存锁定

**Service 层代码** (`OmsPortalOrderServiceImpl.java:750-759`):
```java
private void lockStock(List<CartPromotionItem> cartPromotionItemList) {
    for (CartPromotionItem cartPromotionItem : cartPromotionItemList) {
        PmsSkuStock skuStock = skuStockMapper.selectByPrimaryKey(cartPromotionItem.getProductSkuId());
        skuStock.setLockStock(skuStock.getLockStock() + cartPromotionItem.getQuantity());
        int count = portalOrderDao.lockStockBySkuId(cartPromotionItem.getProductSkuId(),cartPromotionItem.getQuantity());
        if(count==0){
            Asserts.fail("库存不足，无法下单");
        }
    }
}
```

**数据库层分析**:
- SQL: `UPDATE pms_sku_stock SET lock_stock = lock_stock + #{quantity} WHERE id = #{productSkuId} AND lock_stock + #{quantity} <= stock`
- 这是一个**原子性的更新操作**
- 数据库会对该行加行锁（InnoDB 引擎）
- 如果条件 `lock_stock + #{quantity} <= stock` 不满足，更新返回 0
- Service 层检查返回值，为 0 时抛出异常

**结论**: ✅ 数据库层能防止超卖

### 3.2 支付时的库存扣减

**Service 层代码** (`OmsPortalOrderServiceImpl.java:255-283`):
```java
public Integer paySuccess(Long orderId, Integer payType) {
    // 修改订单支付状态...
    // 恢复所有下单商品的锁定库存，扣减真实库存
    OmsOrderDetail orderDetail = portalOrderDao.getDetail(orderId);
    int totalCount = 0;
    for (OmsOrderItem orderItem : orderDetail.getOrderItemList()) {
        int count = portalOrderDao.reduceSkuStock(orderItem.getProductSkuId(),orderItem.getProductQuantity());
        if(count==0){
            Asserts.fail("库存不足，无法扣减！");
        }
        totalCount+=count;
    }
    return totalCount;
}
```

**数据库层分析**:
- SQL: `UPDATE pms_sku_stock SET lock_stock = lock_stock - #{quantity}, stock = stock - #{quantity} WHERE id = #{productSkuId} AND stock - #{quantity} >= 0 AND lock_stock - #{quantity} >= 0`
- 这是一个**原子性的更新操作**
- 数据库会对该行加行锁
- 双重条件判断确保：
  - 真实库存不会变负
  - 锁定库存不会变负
- 如果条件不满足，更新返回 0
- Service 层检查返回值，为 0 时抛出异常

**结论**: ✅ 数据库层能防止负库存

### 3.3 潜在风险分析

**风险 1：批量库存锁定**
- 当前实现是**逐条锁定**库存（`for` 循环中调用 `lockStockBySkuId`）
- 如果订单包含多个商品，不是原子操作
- 但由于每条都有条件判断，且数据库行锁，整体仍然安全

**风险 2：`updateSkuStock` 方法**
- 该方法没有条件判断，直接扣减
- 如果未来被调用，可能导致负库存
- 但当前代码中**未被调用**，暂时不构成风险

---

## 4. 订单取消时库存如何恢复

### 4.1 订单取消流程

订单取消有两种场景：
1. **用户主动取消**
2. **超时未支付自动取消**

### 4.2 用户主动取消订单

**Service 层代码** (`OmsPortalOrderServiceImpl.java:315-348`):
```java
public void cancelOrder(Long orderId) {
    // 查询未付款的取消订单
    OmsOrderExample example = new OmsOrderExample();
    example.createCriteria().andIdEqualTo(orderId).andStatusEqualTo(0).andDeleteStatusEqualTo(0);
    List<OmsOrder> cancelOrderList = orderMapper.selectByExample(example);
    if (CollectionUtils.isEmpty(cancelOrderList)) {
        return;
    }
    OmsOrder cancelOrder = cancelOrderList.get(0);
    if (cancelOrder != null) {
        // 修改订单状态为取消
        cancelOrder.setStatus(4);
        orderMapper.updateByPrimaryKeySelective(cancelOrder);
        OmsOrderItemExample orderItemExample = new OmsOrderItemExample();
        orderItemExample.createCriteria().andOrderIdEqualTo(orderId);
        List<OmsOrderItem> orderItemList = orderItemMapper.selectByExample(orderItemExample);
        // 解除订单商品库存锁定
        if (!CollectionUtils.isEmpty(orderItemList)) {
            for (OmsOrderItem orderItem : orderItemList) {
                int count = portalOrderDao.releaseStockBySkuId(orderItem.getProductSkuId(),orderItem.getProductQuantity());
                if(count==0){
                    Asserts.fail("库存不足，无法释放！");
                }
            }
        }
        // 修改优惠券使用状态...
        // 返还使用积分...
    }
}
```

**库存恢复机制**:
- 调用 `releaseStockBySkuId` 方法
- SQL: `UPDATE pms_sku_stock SET lock_stock = lock_stock - #{quantity} WHERE id = #{productSkuId} AND lock_stock - #{quantity} >= 0`
- **只释放锁定库存（减少 `lock_stock`）**
- 不恢复真实库存（`stock` 字段不变）
- 带条件判断，防止锁定库存变负

### 4.3 超时订单自动取消

**Service 层代码** (`OmsPortalOrderServiceImpl.java:286-312`):
```java
public Integer cancelTimeOutOrder() {
    Integer count=0;
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
    portalOrderDao.updateOrderStatus(ids, 4);
    for (OmsOrderDetail timeOutOrder : timeOutOrders) {
        // 解除订单商品库存锁定
        portalOrderDao.releaseSkuStockLock(timeOutOrder.getOrderItemList());
        // 修改优惠券使用状态...
        // 返还使用积分...
    }
    return timeOutOrders.size();
}
```

**库存恢复机制**:
- 调用 `releaseSkuStockLock` 方法
- 批量释放锁定库存
- **只释放锁定库存（减少 `lock_stock`）**
- 不恢复真实库存（`stock` 字段不变）
- **注意：批量释放不带条件判断**

### 4.4 库存恢复设计分析

**为什么不恢复 `stock` 字段？**

整个库存管理采用了**"锁定库存 + 真实库存"**的两阶段设计：

```
可用库存 = stock - lock_stock
```

| 阶段 | 操作 | stock 变化 | lock_stock 变化 |
|------|------|-----------|----------------|
| 下单 | 锁定库存 | 不变 | +quantity |
| 支付 | 扣减库存 | -quantity | -quantity |
| 取消 | 释放库存 | 不变 | -quantity |

**设计优点**:
- 下单时只锁定，不扣减真实库存
- 避免支付失败时需要恢复真实库存的复杂操作
- 可用库存始终可以通过 `stock - lock_stock` 计算

**潜在问题**:
- 超时取消时使用的批量释放方法 `releaseSkuStockLock` 没有条件判断
- 如果锁定库存已经被其他操作释放，可能导致锁定库存变负
- 但由于订单状态检查和消息队列机制，这种情况概率较低

---

## 5. 最终结论

### 5.1 一致性判断

**核心结论：强一致（但存在潜在风险）**

#### 强一致的依据：

1. **数据库层原子性更新**
   - 库存锁定和扣减都使用了 `UPDATE ... SET ... WHERE ...` 的原子操作
   - InnoDB 引擎的行锁机制确保并发安全

2. **条件判断防止负库存**
   - 库存锁定：`lock_stock + #{quantity} <= stock`
   - 库存扣减：`stock - #{quantity} >= 0` 和 `lock_stock - #{quantity} >= 0`
   - 通过 SQL 条件在数据库层面防止超卖

3. **返回值校验**
   - Service 层检查更新返回的受影响行数
   - 返回 0 时抛出异常，阻止后续操作

4. **两阶段库存管理**
   - 下单阶段：锁定库存（`lock_stock`）
   - 支付阶段：扣减真实库存（`stock`）
   - 取消阶段：释放锁定库存（`lock_stock`）
   - 设计合理，避免复杂的库存回滚

### 5.2 存在的潜在风险

#### 风险 1：批量释放库存无条件判断
- **位置**: `releaseSkuStockLock` 方法（超时取消使用）
- **问题**: SQL 没有 `lock_stock - #{quantity} >= 0` 条件判断
- **影响**: 极端情况下可能导致 `lock_stock` 变为负数
- **严重程度**: 低（由于订单状态检查和消息队列机制，实际发生概率低）

#### 风险 2：库存锁定非原子性
- **位置**: `lockStock` 方法
- **问题**: 多商品订单逐条锁定，不是一个原子事务
- **影响**: 理论上可能出现部分商品锁定成功、部分失败的情况
- **严重程度**: 低（由于每条都有条件判断，且 Service 层会检查返回值，失败时会回滚）

#### 风险 3：未使用的危险方法
- **位置**: `updateSkuStock` 方法
- **问题**: 直接扣减库存，无任何条件判断
- **影响**: 如果未来被调用，可能导致负库存
- **严重程度**: 当前无风险（未被调用），但代码维护时需注意

### 5.3 综合评估

| 维度 | 评估 |
|------|------|
| 并发安全性 | ✅ 高（数据库行锁 + 条件判断） |
| 防止负库存 | ✅ 是（核心路径有条件判断） |
| 一致性模型 | ✅ 强一致（数据库事务保证） |
| 代码健壮性 | ⚠️ 中等（存在潜在风险点） |

**最终结论**: **强一致**

项目的"下单扣库存"机制采用了**数据库层面的强一致设计**，通过：
1. 原子性的 `UPDATE` 语句
2. 行级锁（InnoDB）
3. SQL 条件判断
4. Service 层返回值校验

确保了在并发场景下不会出现超卖和负库存问题。虽然存在一些潜在风险点，但不影响核心的强一致特性。
