# Mall 项目中间件实际使用分析报告

## 概述

本报告详细分析 mall 项目中 MongoDB、RabbitMQ、Elasticsearch、Redis 四个中间件的实际使用情况，从以下五个维度进行分析：
1. 配置文件
2. 依赖声明
3. Bean 或 Template 注入
4. 真实业务调用点
5. 依赖或配置存在但业务链路不一定触发的情况

---

## 1. Redis 分析

### 1.1 配置文件

**配置位置：**
- `mall-portal/src/main/resources/application-dev.yml:23-28`
  ```yaml
  spring:
    redis:
      host: localhost # Redis服务器地址
      database: 0 # Redis数据库索引（默认为0）
      port: 6379 # Redis服务器连接端口
      password: # Redis服务器连接密码（默认为空）
      timeout: 300ms # 连接超时时间（毫秒）
  ```

- `mall-admin/src/main/resources/application-dev.yml:15-20`
  ```yaml
  spring:
    redis:
      host: localhost # Redis服务器地址
      database: 0 # Redis数据库索引（默认为0）
      port: 6379 # Redis服务器连接端口
      password: # Redis服务器连接密码（默认为空）
      timeout: 300ms # 连接超时时间（毫秒）
  ```

**自定义配置：**
- `mall-portal/src/main/resources/application.yml:41-50`
  ```yaml
  redis:
    database: mall
    key:
      authCode: 'ums:authCode'
      orderId: 'oms:orderId'
      member: 'ums:member'
    expire:
      authCode: 90 # 验证码超期时间
      common: 86400 # 24小时
  ```

- `mall-admin/src/main/resources/application.yml:25-31`
  ```yaml
  redis:
    database: mall
    key:
      admin: 'ums:admin'
      resourceList: 'ums:resourceList'
    expire:
      common: 86400 # 24小时
  ```

### 1.2 依赖声明

**pom.xml 中的依赖：**

- `mall-common/pom.xml:32`
  ```xml
  <artifactId>spring-boot-starter-data-redis</artifactId>
  ```

- `mall-security/pom.xml:34`
  ```xml
  <artifactId>spring-boot-starter-data-redis</artifactId>
  ```

- `mall-portal/pom.xml:37`
  ```xml
  <artifactId>spring-boot-starter-data-redis</artifactId>
  ```

- `mall-search/pom.xml:27`
  ```xml
  <artifactId>spring-boot-starter-data-redis</artifactId>
  ```

**注意：** mall-search 模块虽然声明了 Redis 依赖，但在 pom.xml 中对 mall-mbg 依赖排除了 Redis（第25-29行），且 mall-search 的配置文件中没有 Redis 配置。

### 1.3 Bean 或 Template 注入

**Redis 配置类：** `mall-common/src/main/java/com/macro/mall/common/config/BaseRedisConfig.java`

配置了以下 Bean：
1. **RedisTemplate** (第28-38行)
   ```java
   @Bean
   public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory redisConnectionFactory,RedisSerializer<Object> redisSerializer)
   ```

2. **RedisSerializer** (第40-50行)
   ```java
   @Bean
   public RedisSerializer<Object> redisSerializer()
   ```
   使用 Jackson2JsonRedisSerializer 进行 JSON 序列化。

3. **RedisCacheManager** (第52-59行)
   ```java
   @Bean
   public RedisCacheManager redisCacheManager(RedisConnectionFactory redisConnectionFactory)
   ```
   设置缓存有效期为 1 天。

4. **RedisService** (第62-65行)
   ```java
   @Bean
   public RedisService redisService() {
       return new RedisServiceImpl();
   }
   ```

### 1.4 真实业务调用点

#### 1.4.1 mall-portal 模块

**1. 会员信息缓存 - UmsMemberCacheServiceImpl**
位置：`mall-portal/src/main/java/com/macro/mall/portal/service/impl/UmsMemberCacheServiceImpl.java`

**使用的 Redis 操作：**
- `redisService.set(key, member, REDIS_EXPIRE)` - 缓存会员信息 (第51行)
- `redisService.get(key)` - 获取缓存的会员信息 (第45行)
- `redisService.del(key)` - 删除缓存的会员信息 (第38行)
- `redisService.set(key, authCode, REDIS_EXPIRE_AUTH_CODE)` - 缓存验证码 (第58行)
- `redisService.get(key)` - 获取缓存的验证码 (第65行)

**业务场景：**
- 会员登录时缓存会员信息
- 发送验证码时缓存验证码
- 会员信息更新时清除缓存

#### 1.4.2 mall-admin 模块

**2. 管理员信息缓存 - UmsAdminCacheServiceImpl**
位置：`mall-admin/src/main/java/com/macro/mall/service/impl/UmsAdminCacheServiceImpl.java`

**使用的 Redis 操作：**
- `redisService.set(key, admin, REDIS_EXPIRE)` - 缓存管理员信息 (第101行)
- `redisService.get(key)` - 获取缓存的管理员信息 (第95行)
- `redisService.del(key)` - 删除单个缓存 (第48行)
- `redisService.del(keys)` - 批量删除缓存 (第66行, 78行, 88行)
- `redisService.set(key, resourceList, REDIS_EXPIRE)` - 缓存资源列表 (第113行)
- `redisService.get(key)` - 获取缓存的资源列表 (第107行)

**业务场景：**
- 管理员登录时缓存管理员信息和权限资源
- 管理员角色或权限变更时清除相关缓存

#### 1.4.3 mall-portal 模块

**3. 订单ID生成 - OmsPortalOrderServiceImpl**
位置：`mall-portal/src/main/java/com/macro/mall/portal/service/impl/OmsPortalOrderServiceImpl.java`

**使用的 Redis 操作：**
- `redisService.incr(key, 1)` - 生成递增的订单ID (第466行)

**业务场景：**
- 生成订单编号时使用 Redis 的自增特性
- 订单编号格式：8位日期 + 2位平台号码 + 2位支付方式 + 6位以上自增ID

### 1.5 依赖或配置存在但业务链路不一定触发的情况

**mall-search 模块：**
- 声明了 Redis 依赖，但配置文件中没有 Redis 配置
- 代码中没有实际使用 Redis 的地方
- 可能是为了与 mall-mbg 模块保持依赖一致性，但实际并未使用

---

## 2. Elasticsearch 分析

### 2.1 配置文件

**配置位置：** `mall-search/src/main/resources/application-dev.yml:15-20`
```yaml
spring:
  data:
    elasticsearch:
      repositories:
        enabled: true
  elasticsearch:
    uris: localhost:9200
```

### 2.2 依赖声明

**pom.xml 中的依赖：**
- `mall-search/pom.xml:33`
  ```xml
  <artifactId>spring-boot-starter-data-elasticsearch</artifactId>
  ```

### 2.3 Bean 或 Template 注入

**1. 文档实体 - EsProduct**
位置：`mall-search/src/main/java/com/macro/mall/search/domain/EsProduct.java`

使用注解定义：
- `@Document(indexName = "pms")` - 定义索引名称 (第21行)
- `@Setting(shards = 1, replicas = 0)` - 定义分片和副本设置 (第22行)
- `@Field(type = FieldType.Keyword)` - 定义字段类型 (第27行, 30行, 33行)
- `@Field(analyzer = "ik_max_word", type = FieldType.Text)` - 定义中文分词字段 (第36行, 38行, 40行)

**2. Repository 接口 - EsProductRepository**
位置：`mall-search/src/main/java/com/macro/mall/search/repository/EsProductRepository.java`

继承 `ElasticsearchRepository<EsProduct, Long>`，Spring Data 自动生成实现。

**3. ElasticsearchRestTemplate**
位置：`mall-search/src/main/java/com/macro/mall/search/service/impl/EsProductServiceImpl.java:60`
```java
@Autowired
private ElasticsearchRestTemplate elasticsearchRestTemplate;
```

### 2.4 真实业务调用点

**Service 实现 - EsProductServiceImpl**
位置：`mall-search/src/main/java/com/macro/mall/search/service/impl/EsProductServiceImpl.java`

**使用的 Repository 操作：**
1. `productRepository.saveAll(esProductList)` - 批量导入商品到 ES (第64行)
2. `productRepository.deleteById(id)` - 根据 ID 删除商品 (第76行)
3. `productRepository.save(esProduct)` - 保存单个商品 (第85行)
4. `productRepository.deleteAll(esProductList)` - 批量删除商品 (第99行)
5. `productRepository.findByNameOrSubTitleOrKeywords()` - 简单搜索 (第106行)

**使用的 ElasticsearchRestTemplate 操作：**
1. `elasticsearchRestTemplate.search(searchQuery, EsProduct.class)` - 复杂搜索 (第164行, 208行, 242行)
2. 使用 NativeSearchQueryBuilder 构建复杂查询，包括：
   - 多字段匹配搜索 (第130-142行)
   - 权重打分 (第131-136行)
   - 过滤条件 (第116-125行)
   - 排序 (第145-160行)
   - 聚合搜索 (第227-240行)

**Controller 暴露的 API - EsProductController**
位置：`mall-search/src/main/java/com/macro/mall/search/controller/EsProductController.java`

暴露的接口：
1. `POST /esProduct/importAll` - 导入所有数据库商品到 ES
2. `GET /esProduct/delete/{id}` - 根据 ID 删除商品
3. `POST /esProduct/delete/batch` - 批量删除商品
4. `POST /esProduct/create/{id}` - 根据 ID 创建商品
5. `GET /esProduct/search/simple` - 简单搜索
6. `GET /esProduct/search` - 综合搜索、筛选、排序
7. `GET /esProduct/recommend/{id}` - 根据商品 ID 推荐商品
8. `GET /esProduct/search/relate` - 获取搜索的相关品牌、分类及筛选属性

### 2.5 依赖或配置存在但业务链路不一定触发的情况

**无** - Elasticsearch 只在 mall-search 模块中使用，配置、依赖、业务代码完全匹配。

---

## 3. RabbitMQ 分析

### 3.1 配置文件

**配置位置：** `mall-portal/src/main/resources/application-dev.yml:29-34`
```yaml
spring:
  rabbitmq:
    host: localhost
    port: 5672
    virtual-host: /mall
    username: mall
    password: mall
```

**自定义配置：** `mall-portal/src/main/resources/application.yml:56-60`
```yaml
rabbitmq:
  queue:
    name:
      cancelOrder: cancelOrderQueue
```

### 3.2 依赖声明

**pom.xml 中的依赖：**
- `mall-portal/pom.xml:42`
  ```xml
  <artifactId>spring-boot-starter-amqp</artifactId>
  ```

### 3.3 Bean 或 Template 注入

**RabbitMQ 配置类 - RabbitMqConfig**
位置：`mall-portal/src/main/java/com/macro/mall/portal/config/RabbitMqConfig.java`

配置了以下 Bean：

**1. 交换机定义：**
- `orderDirect()` - 订单实际消费队列所绑定的交换机 (第18-24行)
- `orderTtlDirect()` - 订单延迟队列所绑定的交换机 (第29-35行)

**2. 队列定义：**
- `orderQueue()` - 订单实际消费队列 (第40-43行)
- `orderTtlQueue()` - 订单延迟队列（死信队列）(第48-55行)
  - 配置了死信交换机和死信路由键
  - 消息到期后自动转发到实际消费队列

**3. 绑定定义：**
- `orderBinding()` - 将订单队列绑定到交换机 (第60-66行)
- `orderTtlBinding()` - 将订单延迟队列绑定到交换机 (第71-77行)

**队列枚举 - QueueEnum**
位置：`mall-portal/src/main/java/com/macro/mall/portal/domain/QueueEnum.java`

定义了两个队列：
1. `QUEUE_ORDER_CANCEL` - 消息通知队列
   - exchange: `mall.order.direct`
   - name: `mall.order.cancel`
   - routeKey: `mall.order.cancel`

2. `QUEUE_TTL_ORDER_CANCEL` - 消息通知 TTL 队列
   - exchange: `mall.order.direct.ttl`
   - name: `mall.order.cancel.ttl`
   - routeKey: `mall.order.cancel.ttl`

### 3.4 真实业务调用点

#### 3.4.1 消息发送者 - CancelOrderSender
位置：`mall-portal/src/main/java/com/macro/mall/portal/component/CancelOrderSender.java`

**使用的 AmqpTemplate 操作：**
```java
@Autowired
private AmqpTemplate amqpTemplate;

public void sendMessage(Long orderId, final long delayTimes) {
    amqpTemplate.convertAndSend(
        QueueEnum.QUEUE_TTL_ORDER_CANCEL.getExchange(), 
        QueueEnum.QUEUE_TTL_ORDER_CANCEL.getRouteKey(), 
        orderId, 
        new MessagePostProcessor() {
            @Override
            public Message postProcessMessage(Message message) throws AmqpException {
                message.getMessageProperties().setExpiration(String.valueOf(delayTimes));
                return message;
            }
        }
    );
}
```

**业务场景：** 发送延迟消息到 TTL 队列，消息到期后自动转发到实际消费队列。

#### 3.4.2 消息接收者 - CancelOrderReceiver
位置：`mall-portal/src/main/java/com/macro/mall/portal/component/CancelOrderReceiver.java`

**使用的 @RabbitListener：**
```java
@Component
@RabbitListener(queues = "mall.order.cancel")
public class CancelOrderReceiver {
    @RabbitHandler
    public void handle(Long orderId) {
        portalOrderService.cancelOrder(orderId);
        LOGGER.info("process orderId:{}", orderId);
    }
}
```

**业务场景：** 监听 `mall.order.cancel` 队列，接收到订单 ID 后调用取消订单服务。

#### 3.4.3 业务调用入口 - OmsPortalOrderServiceImpl
位置：`mall-portal/src/main/java/com/macro/mall/portal/service/impl/OmsPortalOrderServiceImpl.java`

**调用链路：**
1. `generateOrder()` 方法 (第96行) - 创建订单
2. 在订单创建成功后调用 `sendDelayMessageCancelOrder(order.getId())` (第247行)
3. `sendDelayMessageCancelOrder()` 方法 (第351-357行)：
   - 获取订单超时时间配置
   - 计算延迟时间（毫秒）
   - 调用 `cancelOrderSender.sendMessage(orderId, delayTimes)`

**业务流程：**
1. 用户下单 → 创建订单（状态：待支付）
2. 发送延迟消息（延迟时间 = 订单超时时间）
3. 如果用户在超时时间内支付成功，消息在到达接收者时订单状态已改变，取消操作不会执行
4. 如果用户超时未支付，消息到达接收者时执行取消订单操作

### 3.5 依赖或配置存在但业务链路不一定触发的情况

**无** - RabbitMQ 只在 mall-portal 模块中使用，配置、依赖、业务代码完全匹配，且有明确的业务链路触发。

---

## 4. MongoDB 分析

### 4.1 配置文件

**配置位置：** `mall-portal/src/main/resources/application-dev.yml:18-22`
```yaml
spring:
  data:
    mongodb:
      host: localhost
      port: 27017
      database: mall-port
```

**自定义配置：** `mall-portal/src/main/resources/application.yml:52-54`
```yaml
mongo:
  insert:
    sqlEnable: true # 用于控制是否通过数据库数据来插入mongo
```

### 4.2 依赖声明

**pom.xml 中的依赖：**
- `mall-portal/pom.xml:32`
  ```xml
  <artifactId>spring-boot-starter-data-mongodb</artifactId>
  ```

### 4.3 Bean 或 Template 注入

**文档实体类（使用 @Document 注解）：**

**1. 会员浏览历史 - MemberReadHistory**
位置：`mall-portal/src/main/java/com/macro/mall/portal/domain/MemberReadHistory.java`

字段包括：
- `@Id` - 主键 ID
- `@Indexed memberId` - 会员 ID（索引）
- `memberNickname` - 会员昵称
- `memberIcon` - 会员头像
- `@Indexed productId` - 商品 ID（索引）
- `productName` - 商品名称
- `productPic` - 商品图片
- `productSubTitle` - 商品副标题
- `productPrice` - 商品价格
- `createTime` - 创建时间

**2. 会员商品收藏 - MemberProductCollection**
位置：`mall-portal/src/main/java/com/macro/mall/portal/domain/MemberProductCollection.java`

字段与 MemberReadHistory 类似，用于存储用户收藏的商品信息。

**3. 会员品牌关注 - MemberBrandAttention**
位置：`mall-portal/src/main/java/com/macro/mall/portal/domain/MemberBrandAttention.java`

字段包括：
- `@Id` - 主键 ID
- `@Indexed memberId` - 会员 ID（索引）
- `memberNickname` - 会员昵称
- `memberIcon` - 会员头像
- `@Indexed brandId` - 品牌 ID（索引）
- `brandName` - 品牌名称
- `brandLogo` - 品牌 Logo
- `brandCity` - 品牌城市
- `createTime` - 创建时间

**Repository 接口（继承 MongoRepository）：**

**1. MemberReadHistoryRepository**
位置：`mall-portal/src/main/java/com/macro/mall/portal/repository/MemberReadHistoryRepository.java`

方法：
- `Page<MemberReadHistory> findByMemberIdOrderByCreateTimeDesc(Long memberId, Pageable pageable)` - 根据会员 ID 分页查找记录
- `void deleteAllByMemberId(Long memberId)` - 根据会员 ID 删除记录

**2. MemberProductCollectionRepository**
位置：`mall-portal/src/main/java/com/macro/mall/portal/repository/MemberProductCollectionRepository.java`

方法：
- `MemberProductCollection findByMemberIdAndProductId(Long memberId, Long productId)` - 根据会员 ID 和商品 ID 查找
- `int deleteByMemberIdAndProductId(Long memberId, Long productId)` - 根据会员 ID 和商品 ID 删除
- `Page<MemberProductCollection> findByMemberId(Long memberId, Pageable pageable)` - 根据会员 ID 分页查询
- `void deleteAllByMemberId(Long memberId)` - 根据会员 ID 删除所有记录

**3. MemberBrandAttentionRepository**
位置：`mall-portal/src/main/java/com/macro/mall/portal/repository/MemberBrandAttentionRepository.java`

方法与 MemberProductCollectionRepository 类似，操作品牌关注数据。

### 4.4 真实业务调用点

#### 4.4.1 会员浏览历史 - MemberReadHistoryServiceImpl

位置：`mall-portal/src/main/java/com/macro/mall/portal/service/impl/MemberReadHistoryServiceImpl.java`

**使用的 Repository 操作：**
1. `memberReadHistoryRepository.save(memberReadHistory)` - 创建浏览记录 (第57行)
2. `memberReadHistoryRepository.deleteAll(deleteList)` - 批量删除 (第69行)
3. `memberReadHistoryRepository.findByMemberIdOrderByCreateTimeDesc()` - 分页查询 (第77行)
4. `memberReadHistoryRepository.deleteAllByMemberId()` - 清空记录 (第83行)

**Controller - MemberReadHistoryController**
位置：`mall-portal/src/main/java/com/macro/mall/portal/controller/MemberReadHistoryController.java`

暴露的接口：
1. `POST /member/readHistory/create` - 创建浏览记录
2. `POST /member/readHistory/delete` - 删除浏览记录
3. `POST /member/readHistory/clear` - 清空浏览记录
4. `GET /member/readHistory/list` - 分页获取浏览记录

#### 4.4.2 会员商品收藏 - MemberCollectionServiceImpl

位置：`mall-portal/src/main/java/com/macro/mall/portal/service/impl/MemberCollectionServiceImpl.java`

**使用的 Repository 操作：**
1. `productCollectionRepository.findByMemberIdAndProductId()` - 检查是否已收藏 (第42行)
2. `productCollectionRepository.save(productCollection)` - 添加收藏 (第54行)
3. `productCollectionRepository.deleteByMemberIdAndProductId()` - 删除收藏 (第63行)
4. `productCollectionRepository.findByMemberId()` - 分页查询收藏列表 (第70行)
5. `productCollectionRepository.findByMemberIdAndProductId()` - 查询收藏详情 (第76行)
6. `productCollectionRepository.deleteAllByMemberId()` - 清空收藏 (第82行)

**Controller - MemberProductCollectionController**
位置：`mall-portal/src/main/java/com/macro/mall/portal/controller/MemberProductCollectionController.java`

暴露的接口：
1. `POST /member/productCollection/add` - 添加商品收藏
2. `POST /member/productCollection/delete` - 删除商品收藏
3. `GET /member/productCollection/list` - 显示当前用户商品收藏列表
4. `GET /member/productCollection/detail` - 显示商品收藏详情
5. `POST /member/productCollection/clear` - 清空当前用户商品收藏列表

#### 4.4.3 会员品牌关注 - MemberAttentionServiceImpl

位置：`mall-portal/src/main/java/com/macro/mall/portal/service/impl/MemberAttentionServiceImpl.java`

**使用的 Repository 操作：**
与 MemberCollectionServiceImpl 类似，操作品牌关注数据。

### 4.5 依赖或配置存在但业务链路不一定触发的情况

**配置开关 `mongo.insert.sqlEnable`：**
- 位置：`mall-portal/src/main/resources/application.yml:52-54`
- 默认值：`true`

**影响：**
当 `mongo.insert.sqlEnable = true` 时：
- 插入 MongoDB 前会先查询 MySQL 数据库获取完整的商品/品牌信息
- 确保 MongoDB 中存储的数据是完整的

当 `mongo.insert.sqlEnable = false` 时：
- 直接使用传入的参数插入 MongoDB
- 可能导致数据不完整

**业务链路触发条件：**
- 用户浏览商品 → 触发浏览历史记录
- 用户收藏商品 → 触发商品收藏记录
- 用户关注品牌 → 触发品牌关注记录

这些操作都是用户主动触发的，不是系统自动触发的。

---

## 5. 综合分析总结

### 5.1 各中间件使用情况汇总表

| 中间件 | 配置文件 | 依赖声明 | Bean/Template 注入 | 真实业务调用 | 使用模块 | 使用程度 |
|--------|----------|----------|-------------------|--------------|----------|----------|
| Redis | ✅ mall-portal, mall-admin | ✅ mall-common, mall-security, mall-portal, mall-search | ✅ BaseRedisConfig | ✅ 用户缓存、验证码、订单ID生成 | mall-portal, mall-admin | 核心依赖，广泛使用 |
| Elasticsearch | ✅ mall-search | ✅ mall-search | ✅ EsProduct, EsProductRepository, ElasticsearchRestTemplate | ✅ 商品搜索、导入、推荐 | mall-search | 搜索服务核心 |
| RabbitMQ | ✅ mall-portal | ✅ mall-portal | ✅ RabbitMqConfig, QueueEnum, AmqpTemplate | ✅ 订单超时取消 | mall-portal | 订单业务核心 |
| MongoDB | ✅ mall-portal | ✅ mall-portal | ✅ 3个 Document 类, 3个 Repository | ✅ 浏览历史、商品收藏、品牌关注 | mall-portal | 用户互动功能 |

### 5.2 真正在业务链路中使用的中间件

**1. Redis：核心依赖，广泛使用**
- 在 mall-portal 和 mall-admin 两个模块中使用
- 用于用户信息缓存、验证码缓存、订单 ID 生成
- 配置完整，业务调用明确

**2. Elasticsearch：搜索服务核心**
- 仅在 mall-search 模块中使用
- 用于商品的全文搜索、导入、推荐
- 配置完整，业务调用明确

**3. RabbitMQ：订单业务核心**
- 仅在 mall-portal 模块中使用
- 用于订单超时自动取消
- 使用延迟队列（死信队列）实现
- 配置完整，业务调用明确

**4. MongoDB：用户互动功能**
- 仅在 mall-portal 模块中使用
- 用于存储会员浏览历史、商品收藏、品牌关注
- 配置完整，业务调用明确
- 有配置开关 `mongo.insert.sqlEnable` 控制数据来源

### 5.3 依赖或配置存在但业务链路不一定触发的情况

**1. Redis (mall-search 模块)**
- 声明了 Redis 依赖
- 但没有 Redis 配置
- 代码中也没有实际使用 Redis 的地方
- **结论：** 只是为了与 mall-mbg 模块保持依赖一致性，实际并未使用

**2. MongoDB (配置开关影响)**
- 虽然配置和代码都存在
- 但业务操作需要用户主动触发（浏览商品、收藏商品、关注品牌）
- 如果用户不进行这些操作，MongoDB 不会被实际使用
- **结论：** 代码和配置完整，但需要用户触发才能真正使用

### 5.4 模块间中间件使用分布

| 模块 | Redis | Elasticsearch | RabbitMQ | MongoDB |
|------|-------|---------------|----------|---------|
| mall-common | ✅ 配置和 Service | ❌ | ❌ | ❌ |
| mall-security | ✅ 依赖 | ❌ | ❌ | ❌ |
| mall-admin | ✅ 配置和业务调用 | ❌ | ❌ | ❌ |
| mall-portal | ✅ 配置和业务调用 | ❌ | ✅ 配置和业务调用 | ✅ 配置和业务调用 |
| mall-search | ⚠️ 依赖但未使用 | ✅ 配置和业务调用 | ❌ | ❌ |
| mall-demo | ❌ | ❌ | ❌ | ❌ |
| mall-mbg | ❌ | ❌ | ❌ | ❌ |

**说明：**
- ✅ = 实际使用
- ⚠️ = 有依赖但未使用
- ❌ = 未使用

---

## 6. 结论

**mall 项目中四个中间件都有实际使用：**

1. **Redis** - 核心缓存组件，在 mall-portal 和 mall-admin 中广泛使用，用于用户信息缓存、验证码、订单 ID 生成等场景。

2. **Elasticsearch** - 专门的搜索服务，在 mall-search 模块中使用，用于商品的全文搜索、筛选、排序和推荐。

3. **RabbitMQ** - 消息队列组件，在 mall-portal 中使用，通过延迟队列实现订单超时自动取消功能。

4. **MongoDB** - 文档数据库，在 mall-portal 中使用，用于存储会员浏览历史、商品收藏、品牌关注等用户互动数据。

**需要注意的点：**
- mall-search 模块声明了 Redis 依赖但实际并未使用，可能是历史遗留或为了保持依赖一致性。
- MongoDB 的使用有配置开关 `mongo.insert.sqlEnable`，可以控制是否从 MySQL 查询完整数据。
- 所有中间件的配置、依赖、业务代码都是完整的，没有"只有配置没有代码"或"只有代码没有配置"的情况。
