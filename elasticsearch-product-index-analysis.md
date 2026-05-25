# mall-search 模块 Elasticsearch 商品索引生命周期分析

## 1. 商品数据从数据库同步到 ES 的入口

### 1.1 手动同步接口

商品数据从 MySQL 同步到 Elasticsearch 的入口位于 `EsProductController` 控制器，通过以下接口手动触发：

- **全量导入**：`POST /esProduct/importAll`
  - 调用 `esProductService.importAll()` 方法
  - 从数据库导入所有符合条件的商品到 ES

- **单个商品创建**：`POST /esProduct/create/{id}`
  - 调用 `esProductService.create(id)` 方法
  - 根据商品 ID 从数据库查询并导入到 ES

- **单个商品删除**：`GET /esProduct/delete/{id}`
  - 调用 `esProductService.delete(id)` 方法
  - 从 ES 中删除指定商品

- **批量删除**：`POST /esProduct/delete/batch`
  - 调用 `esProductService.delete(ids)` 方法
  - 从 ES 中批量删除商品

### 1.2 服务层实现

服务层 `EsProductServiceImpl` 中：

1. **`importAll()` 方法**：
   - 调用 `productDao.getAllEsProductList(null)` 从 MySQL 查询所有商品
   - 通过 `productRepository.saveAll(esProductList)` 批量保存到 ES

2. **`create(id)` 方法**：
   - 调用 `productDao.getAllEsProductList(id)` 查询单个商品
   - 通过 `productRepository.save(esProduct)` 保存到 ES

### 1.3 数据来源

数据通过 `EsProductDao.xml` 中的 SQL 查询获取，查询条件：
- `delete_status = 0`（未删除）
- `publish_status = 1`（已上架）

关联表：
- `pms_product`（主表）
- `pms_product_attribute_value`（商品属性值）
- `pms_product_attribute`（商品属性）

---

## 2. 索引文档结构由哪个对象定义

索引文档结构由 `EsProduct` 类定义，位于 `com.macro.mall.search.domain.EsProduct`。

### 2.1 索引配置

```java
@Document(indexName = "pms")
@Setting(shards = 1, replicas = 0)
public class EsProduct implements Serializable {
    // ...
}
```

- **索引名称**：`pms`
- **分片数**：1
- **副本数**：0

### 2.2 字段结构

| 字段名 | 类型 | 说明 | 索引类型 |
|--------|------|------|----------|
| id | Long | 商品ID | @Id |
| productSn | String | 商品编号 | Keyword |
| brandId | Long | 品牌ID | 默认 |
| brandName | String | 品牌名称 | Keyword |
| productCategoryId | Long | 分类ID | 默认 |
| productCategoryName | String | 分类名称 | Keyword |
| pic | String | 商品图片 | 默认 |
| name | String | 商品名称 | Text (ik_max_word 分词) |
| subTitle | String | 副标题 | Text (ik_max_word 分词) |
| keywords | String | 关键词 | Text (ik_max_word 分词) |
| price | BigDecimal | 价格 | 默认 |
| sale | Integer | 销量 | 默认 |
| newStatus | Integer | 新品状态 | 默认 |
| recommandStatus | Integer | 推荐状态 | 默认 |
| stock | Integer | 库存 | 默认 |
| promotionType | Integer | 促销类型 | 默认 |
| sort | Integer | 排序 | 默认 |
| attrValueList | List<EsProductAttributeValue> | 属性列表 | Nested |

### 2.3 嵌套对象

`attrValueList` 是嵌套类型，包含 `EsProductAttributeValue` 对象：

```java
@Data
public class EsProductAttributeValue implements Serializable {
    private Long id;
    private Long productAttributeId;
    @Field(type = FieldType.Keyword)
    private String value;           // 属性值
    private Integer type;           // 0->规格；1->参数
    @Field(type = FieldType.Keyword)
    private String name;            // 属性名称
}
```

---

## 3. 搜索请求如何构造查询条件

搜索请求通过 `EsProductServiceImpl` 中的两个主要方法处理：

### 3.1 简单搜索

**方法签名**：
```java
Page<EsProduct> search(String keyword, Integer pageNum, Integer pageSize)
```

**实现**：
- 调用 `productRepository.findByNameOrSubTitleOrKeywords(keyword, keyword, keyword, pageable)`
- 使用 Spring Data Elasticsearch 的方法名查询
- 同时搜索 `name`、`subTitle`、`keywords` 三个字段

### 3.2 综合搜索（支持筛选和排序）

**方法签名**：
```java
Page<EsProduct> search(String keyword, Long brandId, Long productCategoryId, 
                       Integer pageNum, Integer pageSize, Integer sort)
```

**查询构建流程**：

1. **分页设置**：
   - 使用 `PageRequest.of(pageNum, pageSize)`

2. **过滤条件**（使用 `BoolQueryBuilder`）：
   - 如果 `brandId` 不为空：添加 `termQuery("brandId", brandId)`
   - 如果 `productCategoryId` 不为空：添加 `termQuery("productCategoryId", productCategoryId)`

3. **搜索条件**：
   - 无关键词：使用 `matchAllQuery()`
   - 有关键词：使用 `functionScoreQuery` 加权搜索
     - `name` 字段：权重 10
     - `subTitle` 字段：权重 5
     - `keywords` 字段：权重 2
     - 最低得分：2
     - 计分模式：SUM

4. **排序**：
   - sort=0：按相关度（默认）
   - sort=1：按新品（id 倒序）
   - sort=2：按销量（sale 倒序）
   - sort=3：按价格从低到高（price 正序）
   - sort=4：按价格从高到低（price 倒序）

### 3.3 查询执行

使用 `ElasticsearchRestTemplate.search(searchQuery, EsProduct.class)` 执行查询。

---

## 4. 关键词、品牌、分类、属性筛选分别如何转换为 ES 查询

### 4.1 关键词搜索

**转换方式**：Function Score Query + Match Query

```java
List<FunctionScoreQueryBuilder.FilterFunctionBuilder> filterFunctionBuilders = new ArrayList<>();
filterFunctionBuilders.add(new FunctionScoreQueryBuilder.FilterFunctionBuilder(
    QueryBuilders.matchQuery("name", keyword),
    ScoreFunctionBuilders.weightFactorFunction(10)));
filterFunctionBuilders.add(new FunctionScoreQueryBuilder.FilterFunctionBuilder(
    QueryBuilders.matchQuery("subTitle", keyword),
    ScoreFunctionBuilders.weightFactorFunction(5)));
filterFunctionBuilders.add(new FunctionScoreQueryBuilder.FilterFunctionBuilder(
    QueryBuilders.matchQuery("keywords", keyword),
    ScoreFunctionBuilders.weightFactorFunction(2)));

FunctionScoreQueryBuilder functionScoreQueryBuilder = QueryBuilders.functionScoreQuery(builders)
        .scoreMode(FunctionScoreQuery.ScoreMode.SUM)
        .setMinScore(2);
```

**特点**：
- 对 `name`、`subTitle`、`keywords` 三个字段分别使用 `matchQuery`
- 不同字段设置不同权重（name 最高）
- 使用 `ik_max_word` 分词器（在 EsProduct 注解中定义）
- 最低得分为 2，过滤掉不相关结果

### 4.2 品牌筛选

**转换方式**：Term Query

```java
if (brandId != null) {
    boolQueryBuilder.must(QueryBuilders.termQuery("brandId", brandId));
}
```

**特点**：
- 使用 `termQuery` 精确匹配
- 过滤条件放在 `BoolQueryBuilder` 中

### 4.3 分类筛选

**转换方式**：Term Query

```java
if (productCategoryId != null) {
    boolQueryBuilder.must(QueryBuilders.termQuery("productCategoryId", productCategoryId));
}
```

**特点**：
- 使用 `termQuery` 精确匹配
- 过滤条件放在 `BoolQueryBuilder` 中

### 4.4 属性筛选（聚合分析）

属性筛选主要在 `searchRelatedInfo` 方法中实现，用于获取搜索结果的相关属性信息。

**转换方式**：Nested Aggregation + Filter Aggregation + Terms Aggregation

```java
AbstractAggregationBuilder aggregationBuilder = AggregationBuilders.nested("allAttrValues", "attrValueList")
        .subAggregation(AggregationBuilders.filter("productAttrs", 
            QueryBuilders.termQuery("attrValueList.type", 1))
                .subAggregation(AggregationBuilders.terms("attrIds")
                        .field("attrValueList.productAttributeId")
                        .subAggregation(AggregationBuilders.terms("attrValues")
                                .field("attrValueList.value"))
                        .subAggregation(AggregationBuilders.terms("attrNames")
                                .field("attrValueList.name"))));
```

**特点**：
- 使用 `nested` 聚合处理嵌套类型的 `attrValueList`
- 过滤 `type=1` 的属性（参数类型，排除规格类型）
- 按 `productAttributeId` 分组
- 每个分组下聚合 `value` 和 `name`

---

## 5. MySQL 商品数据变更后 ES 是否自动同步

### 5.1 分析结论

**当前项目中没有实现 MySQL 商品数据变更后的自动同步机制**。

### 5.2 证据

1. **mall-admin 商品服务** (`PmsProductServiceImpl`)：
   - `create()` 方法：仅向 MySQL 插入数据，无 ES 同步代码
   - `update()` 方法：仅更新 MySQL，无 ES 同步代码
   - `updatePublishStatus()`、`updateDeleteStatus()` 等方法：仅操作 MySQL

2. **mall-search 模块**：
   - 仅提供手动同步接口（`/importAll`、`/create/{id}`、`/delete`）
   - 没有消息监听器、定时任务或其他自动同步机制

3. **消息队列** (`RabbitMqConfig`)：
   - mall-portal 中配置了 RabbitMQ，但仅用于订单超时取消
   - 没有配置商品数据同步的队列

### 5.3 数据流

当前数据流：
```
MySQL (商品数据) 
  ↓ 手动调用接口
EsProductController.importAll() / create()
  ↓
EsProductDao.getAllEsProductList()
  ↓
EsProductRepository.save() / saveAll()
  ↓
Elasticsearch
```

---

## 6. 如果 ES 数据和 MySQL 不一致，项目中有没有修复机制

### 6.1 现有修复机制

**项目中提供了手动修复机制**，通过以下接口实现：

1. **全量重新导入**：
   - `POST /esProduct/importAll`
   - 从 MySQL 重新导入所有符合条件的商品
   - 会覆盖 ES 中的现有数据

2. **单个商品重新导入**：
   - `POST /esProduct/create/{id}`
   - 重新从 MySQL 查询并导入指定商品
   - 用于修复单个商品的数据不一致

3. **删除 ES 中数据**：
   - `GET /esProduct/delete/{id}`
   - `POST /esProduct/delete/batch`
   - 可先删除再重新导入

### 6.2 修复流程

当 ES 与 MySQL 数据不一致时，修复流程：

**全量不一致**：
1. 调用 `POST /esProduct/importAll`
2. 系统会从 MySQL 查询所有 `delete_status=0` 且 `publish_status=1` 的商品
3. 通过 `productRepository.saveAll()` 批量更新 ES

**单个商品不一致**：
1. 调用 `POST /esProduct/create/{id}`
2. 重新从 MySQL 查询该商品并更新 ES

### 6.3 局限性

1. **需要手动触发**：没有自动检测和修复机制
2. **全量导入可能耗时**：商品量大时需要较长时间
3. **没有增量同步**：无法只同步变更的商品
4. **没有版本控制**：无法追踪数据变更历史

### 6.4 建议的改进方案

如果需要实现自动同步，可以考虑以下方案：

1. **基于消息队列的异步同步**：
   - mall-admin 商品变更时发送消息到 RabbitMQ
   - mall-search 消费消息并更新 ES

2. **基于 Canal 的数据库变更捕获**：
   - 监听 MySQL binlog
   - 自动同步数据变更

3. **定时任务同步**：
   - 定期对比 ES 和 MySQL 的数据
   - 同步不一致的数据

4. **双写机制**：
   - 在 mall-admin 的商品服务中，操作 MySQL 后立即同步 ES
   - 适用于对实时性要求高的场景

---

## 总结

| 问题 | 答案 |
|------|------|
| 数据同步入口 | `EsProductController` 的手动接口 |
| 索引结构定义 | `EsProduct` 类（@Document 注解） |
| 查询条件构造 | `NativeSearchQueryBuilder` + `FunctionScoreQuery` |
| 关键词转换 | `matchQuery` + 权重计算 |
| 品牌/分类转换 | `termQuery` |
| 属性转换 | `nested` 聚合 + `filter` 聚合 |
| 自动同步 | ❌ 未实现 |
| 修复机制 | ✅ 手动全量导入 + 单个商品导入 |
