# mall-admin 商品创建/更新流程分析

## 1. Controller 入口

### 1.1 创建商品

- **文件路径**: [PmsProductController.java:34-41](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/mall/mall-admin/src/main/java/com/macro/mall/controller/PmsProductController.java#L34-L41)
- **方法**: `create(@RequestBody PmsProductParam productParam)`
- **请求路径**: `POST /product/create`
- **调用链**: Controller → `productService.create(productParam)`

### 1.2 更新商品

- **文件路径**: [PmsProductController.java:54-61](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/mall/mall-admin/src/main/java/com/macro/mall/controller/PmsProductController.java#L54-L61)
- **方法**: `update(@PathVariable Long id, @RequestBody PmsProductParam productParam)`
- **请求路径**: `POST /product/update/{id}`
- **调用链**: Controller → `productService.update(id, productParam)`

---

## 2. Service 实现与事务边界

### 2.1 事务注解位置确认

**`@Transactional` 注解在接口方法上，而非实现类上**：

- **接口文件**: [PmsProductService.java](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/mall/mall-admin/src/main/java/com/macro/mall/service/PmsProductService.java)
- **`create()` 注解位置**: [PmsProductService.java:21](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/mall/mall-admin/src/main/java/com/macro/mall/service/PmsProductService.java#L21-L22)
  ```java
  @Transactional(isolation = Isolation.DEFAULT, propagation = Propagation.REQUIRED)
  int create(PmsProductParam productParam);
  ```
- **`update()` 注解位置**: [PmsProductService.java:32](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/mall/mall-admin/src/main/java/com/macro/mall/service/PmsProductService.java#L32-L33)
  ```java
  @Transactional
  int update(Long id, PmsProductParam productParam);
  ```

**实现类无 `@Transactional` 注解**：

- **实现类文件**: [PmsProductServiceImpl.java](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/mall/mall-admin/src/main/java/com/macro/mall/service/impl/PmsProductServiceImpl.java)
- 类和方法上均未添加 `@Transactional` 注解

### 2.2 事务边界分析

> **重要前提**：事务覆盖整个 `create()` 和 `update()` 方法的执行，**仅在 Spring 事务代理生效的前提下成立**。

事务生效的必要条件：

1. `@EnableTransactionManagement` 已启用
   - **代码证据**: [MyBatisConfig.java:12](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/mall/mall-admin/src/main/java/com/macro/mall/config/MyBatisConfig.java#L12) 存在 `@EnableTransactionManagement`
2. `PmsProductService` 被 Spring 管理且通过代理对象调用
3. 方法是 `public` 的（已满足）
4. 调用方不是同一个类内部的方法调用（避免 this 调用绕过代理）

满足以上条件时，事务才会覆盖：

- `create()` 方法：主商品写入、SKU 编码处理、各子表批量插入等全部操作
- `update()` 方法：主表更新、部分子数据先删后插、SKU 差量增删改等全部操作

---

## 3. 数据拆分与写入分析

### 3.1 数据结构 DTO

- **文件路径**: [PmsProductParam.java](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/mall/mall-admin/src/main/java/com/macro/mall/dto/PmsProductParam.java)

`PmsProductParam` 继承自 `PmsProduct`，并包含以下子数据列表：

| 字段名                             | 类型                                    | 说明               |
| ---------------------------------- | --------------------------------------- | ------------------ |
| `productLadderList`                | `List<PmsProductLadder>`                | 阶梯价格           |
| `productFullReductionList`         | `List<PmsProductFullReduction>`         | 满减价格           |
| `memberPriceList`                  | `List<PmsMemberPrice>`                  | 会员价格           |
| `skuStockList`                     | `List<PmsSkuStock>`                     | SKU 库存信息       |
| `productAttributeValueList`        | `List<PmsProductAttributeValue>`        | 商品参数及规格属性 |
| `subjectProductRelationList`       | `List<CmsSubjectProductRelation>`       | 专题关联           |
| `prefrenceAreaProductRelationList` | `List<CmsPrefrenceAreaProductRelation>` | 优选专区关联       |

### 3.2 关于"图片"数据的说明

**明确结论**：该商品创建/更新流程**没有独立图片表写入**。所有图片数据均直接存储在主表或 SKU 表的字段中，具体分析如下：

#### 3.2.1 商品主表中的图片字段

- **商品主图 (`pic`)**：存储在 `pms_product` 表的 `pic` 字段（`VARCHAR(255)`）
  - **代码证据**：[PmsProduct.java:21](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/mall/mall-mbg/src/main/java/com/macro/mall/model/PmsProduct.java#L21) 定义了 `private String pic;`
  - **写入方式**：`PmsProductParam` 继承自 `PmsProduct`，通过 [PmsProductServiceImpl.java:74](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/mall/mall-admin/src/main/java/com/macro/mall/service/impl/PmsProductServiceImpl.java#L74) 的 `productMapper.insertSelective(product)` 或 [PmsProductServiceImpl.java:126](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/mall/mall-admin/src/main/java/com/macro/mall/service/impl/PmsProductServiceImpl.java#L126) 的 `productMapper.updateByPrimaryKeySelective(product)` 直接写入

- **画册图片 (`albumPics`)**：存储在 `pms_product` 表的 `album_pics` 字段（`VARCHAR(255)`），以逗号分隔的 URL 字符串存储
  - **代码证据**：[PmsProduct.java:90](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/mall/mall-mbg/src/main/java/com/macro/mall/model/PmsProduct.java#L90) 定义了 `private String albumPics;`
  - **写入方式**：同商品主图，通过 `PmsProduct` 实体直接写入

- **详情 HTML (`detailHtml`)**：存储在 `pms_product` 表的 `detail_html` 字段（`TEXT`）
  - **代码证据**：[PmsProduct.java:120](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/mall/mall-mbg/src/main/java/com/macro/mall/model/PmsProduct.java#L120) 定义了 `private String detailHtml;`
  - **写入方式**：同商品主图，通过 `PmsProduct` 实体直接写入

#### 3.2.2 SKU 表中的图片字段

- **SKU 图片 (`pic`)**：存储在 `pms_sku_stock` 表的 `pic` 字段（`VARCHAR(255)`）
  - **代码证据**：`PmsSkuStock` 模型中包含 `pic` 字段，通过 [PmsProductServiceImpl.java:86](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/mall/mall-admin/src/main/java/com/macro/mall/service/impl/PmsProductServiceImpl.java#L86) 的 `relateAndInsertList(skuStockDao, ...)` 批量写入
  - **更新时处理**：通过 [PmsProductServiceImpl.java:143](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/mall/mall-admin/src/main/java/com/macro/mall/service/impl/PmsProductServiceImpl.java#L143) 的 `handleUpdateSkuStockList()` 增量更新

#### 3.2.3 未使用的图片表

`document/sql/mall.sql` 中虽然存在 `pms_album` 和 `pms_album_pic` 表，但：

- **代码证据**：在 [PmsProductServiceImpl.java](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/mall/mall-admin/src/main/java/com/macro/mall/service/impl/PmsProductServiceImpl.java) 的 `create()` 和 `update()` 方法中，**未注入** `PmsAlbumMapper` 或 `PmsAlbumPicMapper`
- **结论**：这两个表不在当前商品创建/更新流程中使用

#### 3.2.4 对象存储与数据库事务边界

**重要说明**：图片文件的上传与商品数据库写入是两个独立的流程，不在同一个事务内：

1. **图片文件上传**：通过独立的 Controller 接口完成
   - MinIO 上传：[MinioController.java:45-85](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/mall/mall-admin/src/main/java/com/macro/mall/controller/MinioController.java#L45-L85) 的 `/minio/upload` 接口，直接调用 `minioClient.putObject()` 写入对象存储
   - OSS 上传：[OssController.java](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/mall/mall-admin/src/main/java/com/macro/mall/controller/OssController.java) 的 `/aliyun/oss/policy` 生成签名，由客户端直传阿里云 OSS
   - **特点**：文件上传成功即持久化，不参与 Spring 数据库事务

2. **商品创建/更新**：仅保存图片 URL 字符串到数据库
   - 商品主图/相册图/详情 HTML：通过 [PmsProductServiceImpl.java:74](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/mall/mall-admin/src/main/java/com/macro/mall/service/impl/PmsProductServiceImpl.java#L74) 写入 `pms_product` 表
   - SKU 图片：通过 [PmsProductServiceImpl.java:86](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/mall/mall-admin/src/main/java/com/macro/mall/service/impl/PmsProductServiceImpl.java#L86) 写入 `pms_sku_stock` 表

3. **事务边界**：
   - **数据库字段**：`pic`、`album_pics`、`detail_html`、`pms_sku_stock.pic` 等字段会随 Spring 事务回滚
   - **对象存储文件**：已上传到 MinIO/OSS 的图片文件**不在该事务内**，事务回滚时**不会自动删除**这些文件，可能产生"孤儿"文件
   - **代码证据**：[PmsProductServiceImpl.java](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/mall/mall-admin/src/main/java/com/macro/mall/service/impl/PmsProductServiceImpl.java) 的 `create()` 和 `update()` 方法中未注入 `MinioClient` 或 OSS 客户端，也没有任何删除对象存储文件的回滚逻辑

---

## 4. 各类子数据对应的 Mapper / DAO

### 4.1 Service 实现文件

- **文件路径**: [PmsProductServiceImpl.java](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/mall/mall-admin/src/main/java/com/macro/mall/service/impl/PmsProductServiceImpl.java)

### 4.2 数据写入映射表

| 数据类型     | 数据库表                              | 写入 DAO                             | 批量插入方法                                          | 对应 XML                                 |
| ------------ | ------------------------------------- | ------------------------------------ | ----------------------------------------------------- | ---------------------------------------- |
| **商品主表** | `pms_product`                         | `PmsProductMapper`                   | `insertSelective()` / `updateByPrimaryKeySelective()` | MyBatis Generator 生成                   |
| **会员价**   | `pms_member_price`                    | `PmsMemberPriceDao`                  | `insertList()`                                        | `PmsMemberPriceDao.xml`                  |
| **阶梯价**   | `pms_product_ladder`                  | `PmsProductLadderDao`                | `insertList()`                                        | `PmsProductLadderDao.xml`                |
| **满减**     | `pms_product_full_reduction`          | `PmsProductFullReductionDao`         | `insertList()`                                        | `PmsProductFullReductionDao.xml`         |
| **SKU/库存** | `pms_sku_stock`                       | `PmsSkuStockDao`                     | `insertList()`                                        | `PmsSkuStockDao.xml`                     |
| **属性值**   | `pms_product_attribute_value`         | `PmsProductAttributeValueDao`        | `insertList()`                                        | `PmsProductAttributeValueDao.xml`        |
| **专题关联** | `cms_subject_product_relation`        | `CmsSubjectProductRelationDao`       | `insertList()`                                        | `CmsSubjectProductRelationDao.xml`       |
| **优选关联** | `cms_prefrence_area_product_relation` | `CmsPrefrenceAreaProductRelationDao` | `insertList()`                                        | `CmsPrefrenceAreaProductRelationDao.xml` |

### 4.3 删除操作使用的 Mapper

| 数据类型 | 删除 Mapper                             | 删除方法                                                                   |
| -------- | --------------------------------------- | -------------------------------------------------------------------------- |
| 会员价   | `PmsMemberPriceMapper`                  | `deleteByExample()`                                                        |
| 阶梯价   | `PmsProductLadderMapper`                | `deleteByExample()`                                                        |
| 满减     | `PmsProductFullReductionMapper`         | `deleteByExample()`                                                        |
| SKU      | `PmsSkuStockMapper`                     | `deleteByExample()` (旧数据) / 循环 `updateByPrimaryKeySelective()` (更新) |
| 属性值   | `PmsProductAttributeValueMapper`        | `deleteByExample()`                                                        |
| 专题关联 | `CmsSubjectProductRelationMapper`       | `deleteByExample()`                                                        |
| 优选关联 | `CmsPrefrenceAreaProductRelationMapper` | `deleteByExample()`                                                        |

---

## 5. 创建商品流程 (create)

### 5.1 代码位置

- **方法**: [PmsProductServiceImpl.java:69-95](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/mall/mall-admin/src/main/java/com/macro/mall/service/impl/PmsProductServiceImpl.java#L69-L95)

### 5.2 执行顺序

```
1. productMapper.insertSelective(product)           // 插入商品主表，获取 productId
2. relateAndInsertList(memberPriceDao, ...)         // 批量插入会员价
3. relateAndInsertList(productLadderDao, ...)       // 批量插入阶梯价
4. relateAndInsertList(productFullReductionDao, ...) // 批量插入满减
5. handleSkuStockCode(skuStockList, productId)      // 生成 SKU 编码
6. relateAndInsertList(skuStockDao, ...)            // 批量插入 SKU
7. relateAndInsertList(productAttributeValueDao, ...) // 批量插入属性值
8. relateAndInsertList(subjectProductRelationDao, ...) // 插入专题关联
9. relateAndInsertList(prefrenceAreaProductRelationDao, ...) // 插入优选关联
```

### 5.3 `relateAndInsertList` 方法

- **文件位置**: [PmsProductServiceImpl.java:310-325](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/mall/mall-admin/src/main/java/com/macro/mall/service/impl/PmsProductServiceImpl.java#L310-L325)

该方法通过反射机制：

1. 调用每个对象的 `setId(null)` 清空 ID
2. 调用每个对象的 `setProductId(productId)` 设置关联
3. 调用 DAO 的 `insertList()` 方法批量插入

**异常处理关键代码**:

```java
try {
    // ... 反射调用 DAO.insertList()
} catch (Exception e) {
    LOGGER.warn("创建商品出错:{}", e.getMessage());
    throw new RuntimeException(e.getMessage());  // 包装为 RuntimeException 抛出
}
```

---

## 6. 更新商品流程 (update)

### 6.1 代码位置

- **方法**: [PmsProductServiceImpl.java:121-161](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/mall/mall-admin/src/main/java/com/macro/mall/service/impl/PmsProductServiceImpl.java#L121-L161)

### 6.2 执行顺序

```
1. productMapper.updateByPrimaryKeySelective(product)  // 更新商品主表

2. // ===== 会员价 =====
   memberPriceMapper.deleteByExample(...)              // 先删除旧数据
   relateAndInsertList(memberPriceDao, ...)            // 再插入新数据

3. // ===== 阶梯价 =====
   productLadderMapper.deleteByExample(...)            // 先删除旧数据
   relateAndInsertList(productLadderDao, ...)          // 再插入新数据

4. // ===== 满减 =====
   productFullReductionMapper.deleteByExample(...)     // 先删除旧数据
   relateAndInsertList(productFullReductionDao, ...)   // 再插入新数据

5. // ===== SKU (特殊处理) =====
   handleUpdateSkuStockList(id, productParam)          // 增量更新

6. // ===== 属性值 =====
   productAttributeValueMapper.deleteByExample(...)    // 先删除旧数据
   relateAndInsertList(productAttributeValueDao, ...)  // 再插入新数据

7. // ===== 专题关联 =====
   subjectProductRelationMapper.deleteByExample(...)   // 先删除旧数据
   relateAndInsertList(subjectProductRelationDao, ...) // 再插入新数据

8. // ===== 优选关联 =====
   prefrenceAreaProductRelationMapper.deleteByExample(...) // 先删除旧数据
   relateAndInsertList(prefrenceAreaProductRelationDao, ...) // 再插入新数据
```

### 6.3 SKU 更新特殊处理

- **方法**: [handleUpdateSkuStockList()](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/mall/mall-admin/src/main/java/com/macro/mall/service/impl/PmsProductServiceImpl.java#L163-L204)

SKU 采用增量更新策略：

1. 查询现有 SKU 列表 (`oriStuList`)
2. 区分：
   - **新增 SKU**: `id == null` → 批量插入
   - **更新 SKU**: `id != null` → 逐条 `updateByPrimaryKeySelective()`
   - **删除 SKU**: 原列表中存在但新列表中不存在 → 按 ID 删除

---

## 7. 一致性风险分析

### 7.1 更新操作的"先删后插"模式风险

**代码证据**：以下数据均采用"先删除旧数据，再插入新数据"的模式：

- 会员价: [PmsProductServiceImpl.java:128-131](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/mall/mall-admin/src/main/java/com/macro/mall/service/impl/PmsProductServiceImpl.java#L128-L131)
- 阶梯价: [PmsProductServiceImpl.java:133-136](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/mall/mall-admin/src/main/java/com/macro/mall/service/impl/PmsProductServiceImpl.java#L133-L136)
- 满减: [PmsProductServiceImpl.java:138-141](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/mall/mall-admin/src/main/java/com/macro/mall/service/impl/PmsProductServiceImpl.java#L138-L141)
- 属性值: [PmsProductServiceImpl.java:145-148](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/mall/mall-admin/src/main/java/com/macro/mall/service/impl/PmsProductServiceImpl.java#L145-L148)
- 专题关联: [PmsProductServiceImpl.java:150-153](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/mall/mall-admin/src/main/java/com/macro/mall/service/impl/PmsProductServiceImpl.java#L150-L153)
- 优选关联: [PmsProductServiceImpl.java:155-158](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/mall/mall-admin/src/main/java/com/macro/mall/service/impl/PmsProductServiceImpl.java#L155-L158)

**风险说明**：

- **事务正常生效时**：即使 `deleteByExample()` 执行成功后，后续 `insertList()` 失败，**整个事务会整体回滚**，已删除的旧数据会恢复，不会出现数据丢失
- **永久丢失风险**仅发生在以下前提：
  1. **事务未生效**：如代理未创建、`@EnableTransactionManagement` 未配置、内部 this 调用绕过代理
  2. **事务被绕过**：如直接在 Service 外部调用 Mapper/DAO 方法
  3. **数据库表不支持事务**：如使用 MyISAM 引擎（当前代码库使用 InnoDB，见 `mall.sql` 中 `ENGINE = InnoDB`）
  4. **异常被吞掉未抛出**：导致事务无法感知到失败

### 7.2 SKU 更新的多步操作分析

**代码位置**: [PmsProductServiceImpl.java:187-202](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/mall/mall-admin/src/main/java/com/macro/mall/service/impl/PmsProductServiceImpl.java#L187-L202)

操作顺序：

```
1. relateAndInsertList(skuStockDao, insertSkuList, id)   // 新增 SKU
2. skuStockMapper.deleteByExample(removeExample)          // 删除 SKU
3. 循环: skuStockMapper.updateByPrimaryKeySelective(...)  // 更新 SKU (逐条)
```

**分析**:

- **事务正常生效时**：全部操作原子性，失败则整体回滚，**不存在一致性风险**
- **逐条循环更新的性能/效率问题**：虽然在事务内是原子的，但每条更新都是独立的 SQL 语句（非批量更新），当 SKU 数量较多时性能较低；若循环中途失败（如 NPE 等非数据库异常），已执行的更新会随事务一起回滚

**仅在以下情况才可能出现持久化的部分成功**：

1. 事务代理未生效（如 this 调用绕过代理）
2. 事务被绕过（直接外部调用 Mapper）
3. 数据库不支持事务（如 MyISAM 引擎）
4. 异常被吞掉未抛出

#### 7.2.1 SKU ID 归属校验缺失风险

**代码证据**：[PmsProductServiceImpl.java:178-202](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/mall/mall-admin/src/main/java/com/macro/mall/service/impl/PmsProductServiceImpl.java#L178-L202)

```java
// 获取需要更新的sku信息 (第 180 行)
List<PmsSkuStock> updateSkuList = currSkuList.stream().filter(item->item.getId()!=null).collect(Collectors.toList());
// ...
// 修改sku (第 198-202 行)
if(CollUtil.isNotEmpty(updateSkuList)){
    for (PmsSkuStock pmsSkuStock : updateSkuList) {
        skuStockMapper.updateByPrimaryKeySelective(pmsSkuStock);
    }
}
```

**问题分析**：
- 代码从前端传入的 `currSkuList` 中筛选出 `id != null` 的 SKU 直接放入 `updateSkuList`
- 后续循环调用 `updateByPrimaryKeySelective(pmsSkuStock)` 时，**仅按主键 ID 更新**，没有校验该 SKU ID 是否属于当前 `productId`
- **缺少的代码**：没有"按 `productId + skuId` 联合校验或联合更新"的逻辑，例如：
  - 没有先检查 `updateSkuList` 中的 ID 是否都在查询出的 `oriStuList`（当前商品的原始 SKU 列表）中
  - 没有使用带 `productId` 条件的更新（如 `updateByExampleSelective` 配合 `andIdEqualTo(skuId).andProductIdEqualTo(productId)`）

**风险**：
- 如果请求传入**其他商品的 SKU ID**，会导致**误更新其他商品的 SKU 数据**
- 即使在事务内，这个误更新也会被提交（因为从数据库角度看，ID 是合法的）
- 这是一个独立的数据一致性/数据误改风险，与事务是否生效无关

#### 7.2.2 SKU 逐条 SQL 更新的性能问题

**代码证据**：[PmsProductServiceImpl.java:198-202](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/mall/mall-admin/src/main/java/com/macro/mall/service/impl/PmsProductServiceImpl.java#L198-L202)

```java
for (PmsSkuStock pmsSkuStock : updateSkuList) {
    skuStockMapper.updateByPrimaryKeySelective(pmsSkuStock);
}
```

**分析**：
- SKU 更新时对 `updateSkuList` 中的每条记录独立调用 `updateByPrimaryKeySelective()`，而非批量更新
- 这主要是**性能/效率问题**，当 SKU 数量较多时会产生较多数据库往返
- 在事务正常生效时，逐条更新是原子性的，中途失败会随事务整体回滚，**不存在一致性风险**

### 7.3 商品主表与子表的顺序风险

**创建流程** ([PmsProductServiceImpl.java:74-92](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/mall/mall-admin/src/main/java/com/macro/mall/service/impl/PmsProductServiceImpl.java#L74-L92)):

```
1. productMapper.insertSelective(product)  // 主表
2. relateAndInsertList(memberPriceDao, ...) // 子表1
3. relateAndInsertList(productLadderDao, ...) // 子表2
...
```

**风险**:

- **事务正常生效时**：主表插入成功后，任一子表插入失败，**整体回滚**，不会出现"孤儿"商品
- **仅事务失效时**：才会出现主表已插入、子表未插入的不一致状态

### 7.4 异常转换与回滚机制分析

#### MyBatis 异常转换机制

MyBatis 本身不会抛出受检的 `SQLException`。MyBatis 的 `SqlSession` 会将所有 SQL 异常转换为**非受检异常**：

- `PersistenceException` (MyBatis 自身的运行时异常)
- 或 Spring 整合后转换为 `DataAccessException` 体系的异常

因此，**直接从 MyBatis Mapper/DAO 抛出的异常都是 `RuntimeException` 子类**，会触发 Spring 事务回滚。

#### `relateAndInsertList()` 的异常包装

- **代码位置**: [PmsProductServiceImpl.java:310-325](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/mall/mall-admin/src/main/java/com/macro/mall/service/impl/PmsProductServiceImpl.java#L310-L325)

```java
try {
    // ... 反射调用
} catch (Exception e) {
    LOGGER.warn("创建商品出错:{}", e.getMessage());
    throw new RuntimeException(e.getMessage());  // 明确包装为 RuntimeException
}
```

无论 catch 到的是受检异常还是非受检异常，最终都会**包装为 `RuntimeException` 抛出**，符合 Spring 默认回滚规则。

#### 关于"受检异常不回滚"的澄清

**结论**：在当前代码实现中，**"受检异常导致不回滚"不是实际风险**，理由如下：

1. **MyBatis 层**：所有 SQL 异常已被转换为 `RuntimeException` 子类
2. **DAO 层**：`relateAndInsertList()` 会 catch 所有异常并包装为 `RuntimeException` 抛出
3. **Service 层**：其他直接调用 Mapper 的 `deleteByExample()`、`updateByPrimaryKeySelective()` 等方法，抛出的也是 MyBatis 转换后的运行时异常

因此，**除非业务代码显式抛出受检异常且未被 `relateAndInsertList()` 包装**，否则不会出现"受检异常不回滚"的情况。当前代码中未发现此类风险点。

---

## 8. 异常回滚分析

### 8.1 理论上的事务回滚（代理生效前提）

在 Spring 事务代理生效的前提下，由于 `create()` 和 `update()` 方法都标记了 `@Transactional`：

- 方法内抛出 `RuntimeException` 或 `Error` 时，**所有数据库操作都会回滚**
- 包括：主表插入/更新、所有子表的删除和插入操作

### 8.2 实际不回滚的场景

#### 场景 1: 事务代理未生效

**触发条件**：

- 调用方通过 `this.create()` 或 `this.update()` 直接调用（绕过代理）
- `PmsProductService` 未被 Spring 容器管理
- `@EnableTransactionManagement` 未配置

**结果**：

- 每个数据库操作独立提交，无事务保护
- 中途失败时，已执行的操作**永久生效**，不会回滚

#### 场景 2: 显式捕获并吞掉异常

**触发条件**：

- 在事务方法内部 try-catch 了异常但未重新抛出
- 或调用了 `relateAndInsertList()` 之外的代码路径且吞掉了异常

**当前代码检查**：

- `relateAndInsertList()` 明确抛出了 `RuntimeException`，无异常吞掉
- 其他 Mapper 调用未被 try-catch 包裹

**结论**：当前代码未发现此类风险。

#### 场景 3: 数据库表不支持事务

**触发条件**：

- 数据库表使用 MyISAM 等非事务引擎

**代码证据**:

- 所有表在 `mall.sql` 中均指定 `ENGINE = InnoDB`，支持事务
- 因此这个场景**不构成实际风险**

### 8.3 中途异常时的回滚分析（代理生效前提下）

#### 创建商品中途异常

| 执行步骤                   | 异常点位置  | 回滚后状态                   |
| -------------------------- | ----------- | ---------------------------- |
| 1. 插入 pms_product        | 步骤 1 失败 | 全部回滚，无数据             |
| 2. 插入 member_price       | 步骤 2 失败 | pms_product 回滚，全部无数据 |
| 3. 插入 product_ladder     | 步骤 3 失败 | pms_product 回滚，全部无数据 |
| 4. 插入 full_reduction     | 步骤 4 失败 | pms_product 回滚，全部无数据 |
| 5. 插入 sku_stock          | 步骤 5 失败 | pms_product 回滚，全部无数据 |
| 6. 插入 attribute_value    | 步骤 6 失败 | pms_product 回滚，全部无数据 |
| 7. 插入 subject_relation   | 步骤 7 失败 | pms_product 回滚，全部无数据 |
| 8. 插入 prefrence_relation | 步骤 8 失败 | pms_product 回滚，全部无数据 |

**结论**: 事务代理生效时，创建商品中途异常**全部回滚**。

#### 更新商品中途异常

| 执行步骤             | 异常点位置            | 回滚后状态                  |
| -------------------- | --------------------- | --------------------------- |
| 1. 更新 pms_product  | 步骤 1 失败           | 全部回滚，保持原状态        |
| 2. 删除 member_price | 步骤 2 删除成功后失败 | 全部回滚，member_price 恢复 |
| 3. 插入 member_price | 步骤 3 失败           | 全部回滚，保持原状态        |
| 4. 删除 ladder       | 步骤 4 删除成功后失败 | 全部回滚，ladder 恢复       |
| ... (后续步骤)       | ...                   | 全部回滚                    |

**结论**: 事务代理生效时，更新商品中途异常**全部回滚**，包括已删除的旧数据。

### 8.4 事务代理未生效时的最坏情况

**如果事务代理未生效（如 this 调用绕过代理）**：

#### 创建商品

- 主表 `pms_product` 已插入，但后续子表插入失败
- **结果**: 存在"孤儿"商品记录，没有子数据

#### 更新商品 (最严重)

- 主表已更新
- 某类子数据的 `deleteByExample()` 已执行（旧数据已删除）
- 但 `insertList()` 失败（新数据未插入）
- **结果**: 该类子数据**永久丢失**

最危险的场景：

```
1. 更新 pms_product ✓
2. 删除 member_price ✓
3. 删除 product_ladder ✓
4. 删除 product_full_reduction ✓
5. 处理 SKU 时失败 ✗
```

**结果**:

- 商品主表已更新
- member_price、ladder、full_reduction 的旧数据已被删除且无法恢复
- 新数据未插入
- 这三类数据**永久丢失**

---

## 9. 总结

### 9.1 数据流总结

```
PmsProductParam (DTO)
├── PmsProduct → pms_product (PmsProductMapper)
├── memberPriceList → pms_member_price (PmsMemberPriceDao.insertList)
├── productLadderList → pms_product_ladder (PmsProductLadderDao.insertList)
├── productFullReductionList → pms_product_full_reduction (PmsProductFullReductionDao.insertList)
├── skuStockList → pms_sku_stock (PmsSkuStockDao.insertList / 增量更新)
├── productAttributeValueList → pms_product_attribute_value (PmsProductAttributeValueDao.insertList)
├── subjectProductRelationList → cms_subject_product_relation (CmsSubjectProductRelationDao.insertList)
└── prefrenceAreaProductRelationList → cms_prefrence_area_product_relation (CmsPrefrenceAreaProductRelationDao.insertList)
```

### 9.2 关键发现

1. **事务注解位置**: `@Transactional` 标注在接口 `PmsProductService` 的方法上，**而非实现类**上
2. **事务生效前提**: 必须通过 Spring 代理对象调用，`this` 内部调用会绕过事务
3. **更新模式**: 多数子数据采用"先删后插"模式，**但在事务生效时不会丢失数据**
4. **SKU 特殊处理**: 采用增量更新（新增/删除/更新分开处理），但更新时逐条操作
5. **图片数据**: 无独立图片表写入，商品主图/相册图/详情 HTML 写入 `pms_product`，SKU 图片写入 `pms_sku_stock.pic`
6. **异常处理**: `relateAndInsertList()` 会将所有异常包装为 `RuntimeException` 抛出，确保触发回滚
7. **MyBatis 异常转换**: MyBatis 本身不抛出受检异常，所有 SQL 异常均为运行时异常

### 9.3 配置说明（非已证实风险）

| 配置项                                  | 说明                                                                                                                                                                                           |
| --------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 全局事务 `rollbackFor` 显式配置         | 未在代码中找到 `@Transactional(rollbackFor = ...)` 全局配置；但 MyBatis/Spring 异常转换与 `relateAndInsertList()` 的 `RuntimeException` 包装已覆盖主要异常路径，当前代码路径下不构成已证实风险 |
| `DataSourceTransactionManager` 显式配置 | 未找到显式 Bean 定义，依赖 Spring Boot 自动配置；Spring Boot 具备自动配置事务管理器的能力，最终是否生效需运行时确认，但不构成已证实风险                                                        |

### 9.4 风险等级（修正后）

| 风险点                          | 风险等级 | 说明                                                                 |
| ------------------------------- | -------- | -------------------------------------------------------------------- |
| 事务代理未生效（如 this 调用）  | **高**   | 绕过代理时无事务保护，先删后插会导致数据永久丢失                     |
| SKU ID 归属校验缺失             | **高**   | 仅按主键更新 SKU，未校验 productId，可能误更新其他商品的 SKU         |
| 对象存储文件不回滚              | **中**   | 数据库中的图片 URL/HTML 字段可随事务回滚，但 MinIO/OSS 已上传文件不会自动回滚，可能产生孤儿文件 |
| SKU 逐条 SQL 更新                | **低**   | 主要是性能/效率问题，事务内原子性，无一致性风险                               |
| "受检异常不回滚"                | **无**   | MyBatis 异常转换 + `relateAndInsertList()` 包装，不构成实际风险       |
| 先删后插在事务生效时的数据丢失 | **无**   | 事务生效时整体回滚，不会丢失数据                                     |
