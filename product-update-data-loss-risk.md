# mall 项目商品更新关联数据风险分析

## 1. 概述

本文档分析 mall 项目中"更新商品"功能对关联数据的处理机制，追踪 `update` 方法的完整实现流程，明确更新时对各类关联表数据的删除、插入和保留策略，并回答关键风险问题。

**核心代码位置**: `mall-admin/src/main/java/com/macro/mall/service/impl/PmsProductServiceImpl.java:121-161`

---

## 2. update 方法完整实现追踪

### 2.1 方法签名与事务配置

**文件**: `mall-admin/src/main/java/com/macro/mall/service/PmsProductService.java:32`

```java
@Transactional
int update(Long id, PmsProductParam productParam);
```

**事务管理启用**: `mall-admin/src/main/java/com/macro/mall/config/MyBatisConfig.java:12`

```java
@EnableTransactionManagement
```

### 2.2 完整执行流程

**文件**: `mall-admin/src/main/java/com/macro/mall/service/impl/PmsProductServiceImpl.java:121-161`

执行顺序如下：

```
1. 更新商品基础信息 (pms_product)
2. 处理会员价格 (pms_member_price)
3. 处理阶梯价格 (pms_product_ladder)
4. 处理满减价格 (pms_product_full_reduction)
5. 处理 SKU 库存 (pms_sku_stock) ── 特殊处理
6. 处理商品属性值 (pms_product_attribute_value)
7. 处理专题关联 (cms_subject_product_relation)
8. 处理优选专区关联 (cms_prefrence_area_product_relation)
```

---

## 3. 各类关联表的更新策略

### 3.1 PmsProductParam 包含的子列表

**文件**: `mall-admin/src/main/java/com/macro/mall/dto/PmsProductParam.java`

| 字段名 | 对应数据库表 | 更新策略 |
|--------|-------------|----------|
| `memberPriceList` | `pms_member_price` | 先删后插 |
| `productLadderList` | `pms_product_ladder` | 先删后插 |
| `productFullReductionList` | `pms_product_full_reduction` | 先删后插 |
| `skuStockList` | `pms_sku_stock` | 增量更新（特殊） |
| `productAttributeValueList` | `pms_product_attribute_value` | 先删后插 |
| `subjectProductRelationList` | `cms_subject_product_relation` | 先删后插 |
| `prefrenceAreaProductRelationList` | `cms_prefrence_area_product_relation` | 先删后插 |

---

## 4. 更新时删除、重新插入、保留的数据明细

### 4.1 采用"先删后插"策略的关联表（共 6 类）

对于以下 6 类关联数据，更新策略完全一致：

**通用代码模式** (`PmsProductServiceImpl.java:128-131` 为例):

```java
// 步骤1: 先删除该商品的所有旧数据
PmsMemberPriceExample pmsMemberPriceExample = new PmsMemberPriceExample();
pmsMemberPriceExample.createCriteria().andProductIdEqualTo(id);
memberPriceMapper.deleteByExample(pmsMemberPriceExample);

// 步骤2: 再插入新数据（如果有）
relateAndInsertList(memberPriceDao, productParam.getMemberPriceList(), id);
```

**应用的 6 类数据**:

| 序号 | 数据类型 | 数据库表 | 删除代码位置 | 插入代码位置 |
|------|----------|----------|-------------|-------------|
| 1 | 会员价格 | `pms_member_price` | 第 128-130 行 | 第 131 行 |
| 2 | 阶梯价格 | `pms_product_ladder` | 第 133-135 行 | 第 136 行 |
| 3 | 满减价格 | `pms_product_full_reduction` | 第 138-140 行 | 第 141 行 |
| 4 | 商品属性值 | `pms_product_attribute_value` | 第 145-147 行 | 第 148 行 |
| 5 | 专题关联 | `cms_subject_product_relation` | 第 150-152 行 | 第 153 行 |
| 6 | 优选专区关联 | `cms_prefrence_area_product_relation` | 第 155-157 行 | 第 158 行 |

**这 6 类数据的处理结果**:

| 操作类型 | 说明 |
|----------|------|
| **删除** | 无条件删除该商品在该表中的**所有旧记录** |
| **重新插入** | 仅当请求的子列表**不为空**时，才插入新数据 |
| **保留** | **无任何保留**，所有旧数据都会被删除 |

---

### 4.2 采用"增量更新"策略的关联表：SKU（特殊处理）

**文件**: `mall-admin/src/main/java/com/macro/mall/service/impl/PmsProductServiceImpl.java:163-204`

**方法**: `handleUpdateSkuStockList(Long id, PmsProductParam productParam)`

**处理逻辑**:

```java
// 1. 获取请求中的 SKU 列表
List<PmsSkuStock> currSkuList = productParam.getSkuStockList();

// 2. 如果请求中没有 SKU 列表 → 直接删除所有
if (CollUtil.isEmpty(currSkuList)) {
    PmsSkuStockExample skuStockExample = new PmsSkuStockExample();
    skuStockExample.createCriteria().andProductIdEqualTo(id);
    skuStockMapper.deleteByExample(skuStockExample);
    return;
}

// 3. 获取数据库中现有的 SKU
List<PmsSkuStock> oriStuList = skuStockMapper.selectByExample(skuStockExample);

// 4. 区分三类操作
List<PmsSkuStock> insertSkuList = currSkuList.stream()
    .filter(item -> item.getId() == null)
    .collect(Collectors.toList());  // 新增

List<PmsSkuStock> updateSkuList = currSkuList.stream()
    .filter(item -> item.getId() != null)
    .collect(Collectors.toList());  // 更新
List<Long> updateSkuIds = updateSkuList.stream()
    .map(PmsSkuStock::getId)
    .collect(Collectors.toList());

List<PmsSkuStock> removeSkuList = oriStuList.stream()
    .filter(item -> !updateSkuIds.contains(item.getId()))
    .collect(Collectors.toList());  // 删除

// 5. 执行操作
if (CollUtil.isNotEmpty(insertSkuList)) {
    relateAndInsertList(skuStockDao, insertSkuList, id);  // 批量新增
}
if (CollUtil.isNotEmpty(removeSkuList)) {
    skuStockMapper.deleteByExample(removeExample);        // 按 ID 删除
}
if (CollUtil.isNotEmpty(updateSkuList)) {
    for (PmsSkuStock pmsSkuStock : updateSkuList) {
        skuStockMapper.updateByPrimaryKeySelective(pmsSkuStock);  // 逐条更新
    }
}
```

**SKU 的处理结果**:

| 操作类型 | 条件 | 说明 |
|----------|------|------|
| **删除** | 情况 1: 请求中无 `skuStockList` | 无条件删除**所有**旧 SKU |
| **删除** | 情况 2: 请求中有 `skuStockList` | 删除**新列表中未包含 ID** 的旧 SKU |
| **重新插入** | 请求中的 SKU `id == null` | 作为新 SKU 批量插入 |
| **保留并更新** | 请求中的 SKU `id != null` | 更新该 ID 对应的 SKU 记录（保留主键） |
| **保留** | 无 | 请求中未引用 ID 的旧 SKU 都会被删除 |

---

### 4.3 relateAndInsertList 方法分析

**文件**: `mall-admin/src/main/java/com/macro/mall/service/impl/PmsProductServiceImpl.java:310-325`

```java
private void relateAndInsertList(Object dao, List dataList, Long productId) {
    try {
        if (CollectionUtils.isEmpty(dataList)) return;  // ← 关键：空列表直接返回
        for (Object item : dataList) {
            Method setId = item.getClass().getMethod("setId", Long.class);
            setId.invoke(item, (Long) null);
            Method setProductId = item.getClass().getMethod("setProductId", Long.class);
            setProductId.invoke(item, productId);
        }
        Method insertList = dao.getClass().getMethod("insertList", List.class);
        insertList.invoke(dao, dataList);
    } catch (Exception e) {
        LOGGER.warn("创建商品出错:{}", e.getMessage());
        throw new RuntimeException(e.getMessage());
    }
}
```

**关键点**: `if (CollectionUtils.isEmpty(dataList)) return;`

这意味着：如果请求的子列表为 `null` 或空数组，**只执行删除操作，不执行插入操作**。

---

## 5. 关键问题回答

### 问题 1：如果请求缺少某个子列表，是跳过更新还是删除旧数据？

**回答：会删除旧数据，但不插入新数据。**

**详细解释**:

对于采用"先删后插"策略的 6 类关联数据：

```
执行顺序：
1. deleteByExample(...)  → 删除该商品的所有旧记录（无条件执行）
2. relateAndInsertList(...) → 仅当列表不为空时才插入
   └── if (CollectionUtils.isEmpty(dataList)) return; ← 空列表直接返回
```

**场景分析**:

| 请求场景 | 旧数据处理 | 新数据处理 | 最终结果 |
|----------|-----------|-----------|----------|
| 子列表有数据（非空数组） | 全部删除 | 全部插入 | 用新数据替换旧数据 |
| 子列表为空数组 `[]` | 全部删除 | 不插入 | **该类关联数据被清空** |
| 子列表为 `null`（缺字段） | 全部删除 | 不插入 | **该类关联数据被清空** |
| 字段在 JSON 中不存在 | 全部删除 | 不插入 | **该类关联数据被清空** |

**代码证据**:

- 删除无条件执行：`PmsProductServiceImpl.java:128-130`（memberPrice 为例）
- 插入有条件跳过：`PmsProductServiceImpl.java:312`

**特别注意 SKU**:

SKU 也有类似行为，但判断位置不同：

```java
// PmsProductServiceImpl.java:167-172
if (CollUtil.isEmpty(currSkuList)) {
    PmsSkuStockExample skuStockExample = new PmsSkuStockExample();
    skuStockExample.createCriteria().andProductIdEqualTo(id);
    skuStockMapper.deleteByExample(skuStockExample);
    return;  // 直接删除所有 SKU 后返回
}
```

**结论**: **请求缺少任何子列表都会导致该类关联数据被全部删除**。

---

### 问题 2：如果某个子表插入失败，前面删除的数据是否回滚？

**回答：理论上会回滚，但存在条件限制和配置不确定性。**

**详细解释**:

#### 2.1 理论上的事务回滚

由于 `update()` 方法标记了 `@Transactional`，在 Spring 事务管理正常工作的前提下：

- 方法内抛出 `RuntimeException` 或 `Error` 时，所有数据库操作都会回滚
- 包括：商品基础信息更新、所有子表的删除和插入操作

**代码证据**:
- 事务注解：`PmsProductService.java:32`
- 事务启用：`MyBatisConfig.java:12`

#### 2.2 实际风险分析

**风险场景示例**（事务正确配置时）:

```
执行顺序：
1. 更新 pms_product  ✓
2. 删除 member_price  ✓
3. 删除 ladder        ✓
4. 删除 full_reduction ✓
5. 处理 SKU 时失败     ✗ → 抛出 RuntimeException
```

**事务正确时**: 步骤 1-4 全部回滚，恢复到更新前状态。

**事务未生效时**: 步骤 1-4 已提交，member_price、ladder、full_reduction 的旧数据**永久丢失**。

#### 2.3 可能导致不回滚的情况

**情况 1: 非 RuntimeException 异常**

Spring `@Transactional` 默认只对 `RuntimeException` 和 `Error` 回滚。

**代码证据缺失**: 未看到 `@Transactional(rollbackFor = Exception.class)` 配置

当前配置：
```java
@Transactional  // PmsProductService.java:32
// 没有 rollbackFor 属性
```

**问题**: 如果底层抛出受检异常（如 `SQLException`），事务**不会回滚**。

**情况 2: 事务未生效**

虽然有 `@EnableTransactionManagement`，但以下条件需同时满足：
- 数据源配置正确
- 事务管理器 Bean 存在
- 方法是 `public` 且被 Spring AOP 代理

**代码证据缺失**: 未看到 `DataSourceTransactionManager` 或 `PlatformTransactionManager` 的显式配置

**情况 3: 异常被吞掉**

查看 `relateAndInsertList` 方法：

```java
// PmsProductServiceImpl.java:321-324
catch (Exception e) {
    LOGGER.warn("创建商品出错:{}", e.getMessage());
    throw new RuntimeException(e.getMessage());  // 抛出了 RuntimeException
}
```

**好消息**: 该方法最终抛出了 `RuntimeException`，会触发事务回滚。

**结论**: **事务正确配置时会回滚，否则可能导致数据永久丢失。**

---

### 问题 3：是否存在商品基础信息已更新但 SKU/属性未更新的情况？

**回答：理论上不存在（事务正确时），但存在执行顺序风险。**

**详细解释**:

#### 3.1 执行顺序

```java
// PmsProductServiceImpl.java:121-161
1. productMapper.updateByPrimaryKeySelective(product);  // 商品基础信息（第1步）
2. // ... 删除/插入会员价、阶梯价、满减（第2-4步）
3. handleUpdateSkuStockList(id, productParam);           // SKU（第5步）
4. // ... 删除/插入属性值、专题、优选（第6-8步）
```

商品基础信息更新在**第 1 步**，SKU 在**第 5 步**，属性值在**第 6 步**。

#### 3.2 事务正确时

如果第 1 步之后任何步骤失败：
- 抛出 `RuntimeException`
- Spring 事务回滚
- 第 1 步的商品基础信息更新也会回滚
- **不会出现**"基础信息更新了但子数据没更新"的情况

#### 3.3 事务未生效时

如果事务未生效（或异常不触发回滚）：

```
执行到第 1 步：商品基础信息已更新  ✓
执行到第 5 步：SKU 处理失败       ✗
```

**结果**:
- 商品基础信息：已更新（新值）
- 会员价、阶梯价、满减：旧数据已删除，新数据可能未插入
- SKU：未处理（保持旧值或部分处理）
- 属性值、专题、优选：未处理（保持旧值）

**这是一种数据不一致状态**。

#### 3.4 SKU 内部的风险

`handleUpdateSkuStockList` 方法内部有多个独立操作：

```java
// PmsProductServiceImpl.java:187-202
1. relateAndInsertList(skuStockDao, insertSkuList, id);    // 新增 SKU
2. skuStockMapper.deleteByExample(removeExample);           // 删除 SKU
3. for (PmsSkuStock pmsSkuStock : updateSkuList) {          // 逐条更新 SKU
       skuStockMapper.updateByPrimaryKeySelective(pmsSkuStock);
   }
```

如果事务未生效：
- 新增成功、删除失败 → 出现重复 SKU
- 删除成功、更新失败 → 部分 SKU 丢失
- 循环更新中某条失败 → 部分 SKU 已更新、部分未更新

**结论**: **事务正确时不存在不一致，事务失效时存在严重的不一致风险。**

---

### 问题 4：哪些结论需要代码证据支持？

**回答：所有结论都需要代码证据支持。** 以下是每个结论对应的代码证据：

#### 4.1 关于"请求缺少子列表会删除旧数据"的证据

| 结论 | 代码证据 | 文件位置 |
|------|----------|----------|
| 旧数据无条件删除 | `memberPriceMapper.deleteByExample(pmsMemberPriceExample);` | `PmsProductServiceImpl.java:130` |
| 空列表不插入 | `if (CollectionUtils.isEmpty(dataList)) return;` | `PmsProductServiceImpl.java:312` |
| SKU 空列表删除所有 | `if (CollUtil.isEmpty(currSkuList)) { ... deleteByExample ... }` | `PmsProductServiceImpl.java:167-172` |

#### 4.2 关于"事务回滚"的证据

| 结论 | 代码证据 | 文件位置 |
|------|----------|----------|
| 方法在事务内 | `@Transactional` | `PmsProductService.java:32` |
| 事务管理已启用 | `@EnableTransactionManagement` | `MyBatisConfig.java:12` |
| 异常会被抛出 | `throw new RuntimeException(e.getMessage());` | `PmsProductServiceImpl.java:323` |
| 缺少 rollbackFor 配置 | `@Transactional`（无属性） | `PmsProductService.java:32` |
| 缺少显式事务管理器配置 | 未找到 `PlatformTransactionManager` Bean 定义 | 需全局搜索确认 |

#### 4.3 关于"执行顺序"的证据

| 结论 | 代码证据 | 文件位置 |
|------|----------|----------|
| 商品基础信息先更新 | `productMapper.updateByPrimaryKeySelective(product);` | `PmsProductServiceImpl.java:126` |
| 子表处理在后面 | 会员价第 128-131 行、SKU 第 143 行、属性值第 145-148 行 | `PmsProductServiceImpl.java:128-158` |
| SKU 多步操作 | 新增第 187-189 行、删除第 191-195 行、更新第 198-201 行 | `PmsProductServiceImpl.java:187-202` |

#### 4.4 关于"SKU 增量更新"的证据

| 结论 | 代码证据 | 文件位置 |
|------|----------|----------|
| 查询现有 SKU | `skuStockMapper.selectByExample(skuStockExample);` | `PmsProductServiceImpl.java:176` |
| 区分新增/更新/删除 | Stream 过滤 `item.getId() == null` / `!= null` | `PmsProductServiceImpl.java:178-183` |
| 新增 SKU 批量插入 | `relateAndInsertList(skuStockDao, insertSkuList, id);` | `PmsProductServiceImpl.java:188` |
| 删除 SKU 按 ID | `deleteExample.createCriteria().andIdIn(removeSkuIds);` | `PmsProductServiceImpl.java:193-194` |
| 更新 SKU 逐条操作 | `for 循环 + updateByPrimaryKeySelective` | `PmsProductServiceImpl.java:198-201` |

---

## 6. 风险总结

### 6.1 高风险点

| 风险 | 风险等级 | 说明 |
|------|----------|------|
| 6 类关联数据"先删后插" | **高** | 若事务失效，旧数据删除后无法恢复 |
| 请求缺少子列表即删除全部 | **高** | 前端漏传字段会导致关联数据被清空 |
| 执行顺序：主表先更新 | **中** | 事务失效时主表与子表不一致 |

### 6.2 中风险点

| 风险 | 风险等级 | 说明 |
|------|----------|------|
| SKU 逐条循环更新 | **中** | 非原子操作，可能部分成功 |
| 缺少 `rollbackFor` 显式配置 | **中** | 受检异常可能不触发回滚 |

### 6.3 低风险点

| 风险 | 风险等级 | 说明 |
|------|----------|------|
| 依赖 Spring 自动事务配置 | **低** | 通常能正常工作，但缺乏显式确认 |

---

## 7. 建议改进方向

1. **添加 `@Transactional(rollbackFor = Exception.class)`** 确保所有异常触发回滚
2. **采用"软删除"或"先插后删"策略** 避免数据丢失风险
3. **子列表缺省字段时跳过更新** 而非删除全部数据
4. **SKU 更新改为批量操作** 减少逐条更新的风险
5. **添加显式事务管理器配置** 提高可维护性和可预测性
