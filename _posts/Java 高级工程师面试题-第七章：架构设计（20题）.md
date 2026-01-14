# 第七章：架构设计（20题）

## 7.1 高性能（7题）

### 181. 如何设计高并发系统？

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        高并发系统设计全景图                               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  用户请求                                                                │
│     │                                                                   │
│     ▼                                                                   │
│  ┌─────────┐                                                           │
│  │  CDN    │ ◄─── 静态资源加速、边缘计算                                │
│  └────┬────┘                                                           │
│       │                                                                 │
│       ▼                                                                 │
│  ┌─────────┐                                                           │
│  │  WAF    │ ◄─── 安全防护、CC攻击防御                                  │
│  └────┬────┘                                                           │
│       │                                                                 │
│       ▼                                                                 │
│  ┌─────────────────────────────────────┐                               │
│  │      负载均衡层 (LVS/Nginx)          │                               │
│  │   DNS轮询 → LVS → Nginx集群          │                               │
│  └──────────────────┬──────────────────┘                               │
│                     │                                                   │
│       ┌─────────────┼─────────────┐                                    │
│       │             │             │                                    │
│       ▼             ▼             ▼                                    │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐                                │
│  │ 网关 1  │  │ 网关 2  │  │ 网关 3  │  ◄─── 限流、熔断、路由           │
│  └────┬────┘  └────┬────┘  └────┬────┘                                │
│       │             │             │                                    │
│       └─────────────┼─────────────┘                                    │
│                     │                                                   │
│  ┌──────────────────┴──────────────────┐                               │
│  │          服务集群 (水平扩展)          │                               │
│  │  ┌───────┐ ┌───────┐ ┌───────┐      │                               │
│  │  │服务A-1│ │服务A-2│ │服务A-N│      │                               │
│  │  └───────┘ └───────┘ └───────┘      │                               │
│  └──────────────────┬──────────────────┘                               │
│                     │                                                   │
│       ┌─────────────┼─────────────┐                                    │
│       │             │             │                                    │
│       ▼             ▼             ▼                                    │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐                                │
│  │  缓存层  │  │ 消息队列 │  │搜索引擎 │                                │
│  │  Redis  │  │  Kafka  │  │   ES    │                                │
│  └────┬────┘  └────┬────┘  └────┬────┘                                │
│       │             │             │                                    │
│       └─────────────┼─────────────┘                                    │
│                     │                                                   │
│  ┌──────────────────┴──────────────────┐                               │
│  │          数据层 (分库分表)            │                               │
│  │  ┌────────┐  ┌────────┐  ┌────────┐ │                               │
│  │  │Master 1│  │Master 2│  │Master N│ │                               │
│  │  │ Slave  │  │ Slave  │  │ Slave  │ │                               │
│  │  └────────┘  └────────┘  └────────┘ │                               │
│  └─────────────────────────────────────┘                               │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

#### 核心设计原则

```java
/**
 * 高并发系统设计核心组件示例
 */
@Configuration
public class HighConcurrencyConfig {

    // ==================== 1. 分层架构 ====================

    /**
     * 请求处理流程
     *
     * 1. 接入层：负载均衡、SSL卸载、请求路由
     * 2. 网关层：认证授权、限流熔断、协议转换
     * 3. 服务层：业务逻辑、服务编排、数据聚合
     * 4. 缓存层：热点数据、会话数据、计算结果
     * 5. 数据层：持久化存储、数据分片、主从复制
     */

    // ==================== 2. 无状态设计 ====================

    /**
     * 无状态服务设计
     * - Session外置到Redis
     * - 使用JWT Token
     * - 配置中心统一管理
     */
    @Bean
    public RedisTemplate<String, Object> sessionRedisTemplate(
            RedisConnectionFactory factory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(factory);
        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(new GenericJackson2JsonRedisSerializer());
        return template;
    }

    // ==================== 3. 水平扩展设计 ====================

    /**
     * 服务注册与发现配置
     */
    @Bean
    @LoadBalanced
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
}

/**
 * 高并发服务实现示例
 */
@Service
@Slf4j
public class HighConcurrencyOrderService {

    @Autowired
    private StringRedisTemplate redisTemplate;

    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate;

    @Autowired
    private OrderMapper orderMapper;

    /**
     * 高并发下单 - 完整流程
     */
    public OrderResult createOrder(OrderRequest request) {
        String userId = request.getUserId();
        String productId = request.getProductId();

        // 1. 幂等性检查
        String orderToken = request.getOrderToken();
        Boolean success = redisTemplate.opsForValue()
                .setIfAbsent("order:token:" + orderToken, "1",
                             Duration.ofMinutes(30));
        if (Boolean.FALSE.equals(success)) {
            return OrderResult.duplicate("重复提交");
        }

        try {
            // 2. 库存预扣减 (Redis Lua脚本保证原子性)
            Long stock = deductStock(productId, request.getQuantity());
            if (stock < 0) {
                return OrderResult.fail("库存不足");
            }

            // 3. 生成订单号 (分布式ID)
            String orderId = generateOrderId();

            // 4. 创建订单消息
            OrderMessage message = OrderMessage.builder()
                    .orderId(orderId)
                    .userId(userId)
                    .productId(productId)
                    .quantity(request.getQuantity())
                    .amount(request.getAmount())
                    .createTime(System.currentTimeMillis())
                    .build();

            // 5. 发送到消息队列异步处理
            kafkaTemplate.send("order-topic", orderId,
                              JSON.toJSONString(message));

            // 6. 返回订单创建中状态
            return OrderResult.processing(orderId, "订单创建中");

        } catch (Exception e) {
            // 7. 异常处理，回滚库存
            rollbackStock(productId, request.getQuantity());
            redisTemplate.delete("order:token:" + orderToken);
            throw new OrderException("创建订单失败", e);
        }
    }

    /**
     * Redis Lua脚本扣减库存
     */
    private Long deductStock(String productId, int quantity) {
        String script =
            "local stock = redis.call('get', KEYS[1]) " +
            "if stock and tonumber(stock) >= tonumber(ARGV[1]) then " +
            "    return redis.call('decrby', KEYS[1], ARGV[1]) " +
            "else " +
            "    return -1 " +
            "end";

        RedisScript<Long> redisScript = new DefaultRedisScript<>(script, Long.class);
        return redisTemplate.execute(redisScript,
                Collections.singletonList("stock:" + productId),
                String.valueOf(quantity));
    }

    /**
     * 库存回滚
     */
    private void rollbackStock(String productId, int quantity) {
        redisTemplate.opsForValue().increment("stock:" + productId, quantity);
    }

    /**
     * 分布式ID生成 (雪花算法)
     */
    private String generateOrderId() {
        // 使用雪花算法或其他分布式ID方案
        return String.valueOf(SnowflakeIdWorker.nextId());
    }
}

/**
 * 订单消息消费者
 */
@Component
@Slf4j
public class OrderMessageConsumer {

    @Autowired
    private OrderMapper orderMapper;

    @Autowired
    private InventoryService inventoryService;

    @KafkaListener(topics = "order-topic", groupId = "order-group")
    public void consumeOrder(ConsumerRecord<String, String> record) {
        String orderId = record.key();
        String message = record.value();

        try {
            OrderMessage orderMessage = JSON.parseObject(message, OrderMessage.class);

            // 幂等性处理
            Order existOrder = orderMapper.selectById(orderId);
            if (existOrder != null) {
                log.info("订单已存在，跳过处理: {}", orderId);
                return;
            }

            // 创建订单
            Order order = buildOrder(orderMessage);
            orderMapper.insert(order);

            // 扣减数据库库存
            inventoryService.deductStock(orderMessage.getProductId(),
                                         orderMessage.getQuantity());

            // 更新订单状态
            orderMapper.updateStatus(orderId, OrderStatus.CREATED);

            log.info("订单创建成功: {}", orderId);

        } catch (Exception e) {
            log.error("订单处理失败: {}", orderId, e);
            // 发送到死信队列或重试
            throw e;
        }
    }
}
```

#### 高并发设计核心策略

```
┌─────────────────────────────────────────────────────────────────┐
│                    高并发设计六大策略                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 1. 缓存策略                                              │   │
│  │    ┌─────────┐  ┌─────────┐  ┌─────────┐               │   │
│  │    │浏览器缓存│→ │ CDN缓存 │→ │本地缓存 │→ 分布式缓存    │   │
│  │    └─────────┘  └─────────┘  └─────────┘               │   │
│  │    • 多级缓存架构                                        │   │
│  │    • 热点数据预加载                                      │   │
│  │    • 缓存穿透/击穿/雪崩防护                              │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 2. 异步策略                                              │   │
│  │    同步请求 ──→ 消息队列 ──→ 异步处理                    │   │
│  │    • 削峰填谷                                            │   │
│  │    • 解耦服务                                            │   │
│  │    • 最终一致性                                          │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 3. 分片策略                                              │   │
│  │    ┌─────────┐  ┌─────────┐  ┌─────────┐               │   │
│  │    │ 数据分片 │  │ 流量分片 │  │ 任务分片 │               │   │
│  │    └─────────┘  └─────────┘  └─────────┘               │   │
│  │    • 水平分库分表                                        │   │
│  │    • 一致性Hash                                          │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 4. 池化策略                                              │   │
│  │    连接池 + 线程池 + 对象池                               │   │
│  │    • 资源复用                                            │   │
│  │    • 减少创建销毁开销                                    │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 5. 并行策略                                              │   │
│  │    串行调用 ──→ 并行调用                                 │   │
│  │    • CompletableFuture                                   │   │
│  │    • 并行流处理                                          │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 6. 预处理策略                                            │   │
│  │    • 数据预热                                            │   │
│  │    • 资源预分配                                          │   │
│  │    • 结果预计算                                          │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

### 182. 如何提升系统QPS？

```
┌─────────────────────────────────────────────────────────────────┐
│                     QPS提升全方位优化                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│              QPS = 并发数 / 平均响应时间                         │
│                                                                 │
│  提升QPS两个方向：                                               │
│  1. 提高并发处理能力                                            │
│  2. 降低平均响应时间                                            │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                  优化层次金字塔                           │   │
│  │                                                         │   │
│  │                    ┌─────────┐                          │   │
│  │                    │  架构   │ ← 分布式、微服务           │   │
│  │                  ┌─┴─────────┴─┐                        │   │
│  │                  │   缓存层    │ ← 多级缓存               │   │
│  │                ┌─┴─────────────┴─┐                      │   │
│  │                │    数据库层     │ ← 读写分离、分库分表     │   │
│  │              ┌─┴─────────────────┴─┐                    │   │
│  │              │      代码层         │ ← 算法、并发优化       │   │
│  │            ┌─┴─────────────────────┴─┐                  │   │
│  │            │        JVM层            │ ← GC调优           │   │
│  │          ┌─┴─────────────────────────┴─┐                │   │
│  │          │          系统层             │ ← 网络、IO优化     │   │
│  │          └─────────────────────────────┘                │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### 各层优化实践

```java
/**
 * QPS优化实践示例
 */
@Service
@Slf4j
public class QpsOptimizationService {

    // ==================== 1. 缓存优化 ====================

    @Autowired
    private StringRedisTemplate redisTemplate;

    // 本地缓存 - 使用Caffeine
    private final Cache<String, Product> localCache = Caffeine.newBuilder()
            .maximumSize(10000)
            .expireAfterWrite(Duration.ofMinutes(5))
            .recordStats()
            .build();

    /**
     * 多级缓存查询
     * L1: 本地缓存 (响应时间 < 1ms)
     * L2: Redis缓存 (响应时间 1-5ms)
     * L3: 数据库 (响应时间 10-100ms)
     */
    public Product getProductWithMultiLevelCache(String productId) {
        // L1 - 本地缓存
        Product product = localCache.getIfPresent(productId);
        if (product != null) {
            return product;
        }

        // L2 - Redis缓存
        String cacheKey = "product:" + productId;
        String json = redisTemplate.opsForValue().get(cacheKey);
        if (StringUtils.isNotBlank(json)) {
            product = JSON.parseObject(json, Product.class);
            localCache.put(productId, product);
            return product;
        }

        // L3 - 数据库
        product = productMapper.selectById(productId);
        if (product != null) {
            redisTemplate.opsForValue().set(cacheKey,
                    JSON.toJSONString(product), Duration.ofHours(1));
            localCache.put(productId, product);
        }

        return product;
    }

    // ==================== 2. 并行调用优化 ====================

    @Autowired
    private ThreadPoolTaskExecutor asyncExecutor;

    /**
     * 串行改并行 - 聚合多个服务调用
     * 优化前: T = T1 + T2 + T3 = 300ms
     * 优化后: T = max(T1, T2, T3) = 100ms
     */
    public OrderDetailVO getOrderDetail(String orderId) {
        // 并行获取各种信息
        CompletableFuture<Order> orderFuture = CompletableFuture
                .supplyAsync(() -> orderService.getOrder(orderId), asyncExecutor);

        CompletableFuture<User> userFuture = orderFuture
                .thenComposeAsync(order -> CompletableFuture
                        .supplyAsync(() -> userService.getUser(order.getUserId()),
                                    asyncExecutor));

        CompletableFuture<List<OrderItem>> itemsFuture = CompletableFuture
                .supplyAsync(() -> orderItemService.getItems(orderId), asyncExecutor);

        CompletableFuture<Logistics> logisticsFuture = CompletableFuture
                .supplyAsync(() -> logisticsService.getLogistics(orderId), asyncExecutor);

        // 等待所有结果并组装
        try {
            return CompletableFuture.allOf(orderFuture, userFuture,
                                           itemsFuture, logisticsFuture)
                    .thenApply(v -> {
                        OrderDetailVO vo = new OrderDetailVO();
                        vo.setOrder(orderFuture.join());
                        vo.setUser(userFuture.join());
                        vo.setItems(itemsFuture.join());
                        vo.setLogistics(logisticsFuture.join());
                        return vo;
                    })
                    .get(3, TimeUnit.SECONDS);
        } catch (Exception e) {
            throw new ServiceException("获取订单详情超时", e);
        }
    }

    // ==================== 3. 批量操作优化 ====================

    /**
     * 批量查询优化
     * 优化前: N次数据库查询
     * 优化后: 1次批量查询
     */
    public List<Product> batchGetProducts(List<String> productIds) {
        if (CollectionUtils.isEmpty(productIds)) {
            return Collections.emptyList();
        }

        // 1. 先从缓存批量获取
        List<String> cacheKeys = productIds.stream()
                .map(id -> "product:" + id)
                .collect(Collectors.toList());

        List<String> cachedValues = redisTemplate.opsForValue()
                .multiGet(cacheKeys);

        // 2. 找出缓存中没有的ID
        Map<String, Product> resultMap = new HashMap<>();
        List<String> missedIds = new ArrayList<>();

        for (int i = 0; i < productIds.size(); i++) {
            String productId = productIds.get(i);
            String cached = cachedValues.get(i);
            if (StringUtils.isNotBlank(cached)) {
                resultMap.put(productId, JSON.parseObject(cached, Product.class));
            } else {
                missedIds.add(productId);
            }
        }

        // 3. 批量查询数据库
        if (!missedIds.isEmpty()) {
            List<Product> dbProducts = productMapper.selectBatchIds(missedIds);

            // 4. 批量写入缓存
            Map<String, String> cacheMap = new HashMap<>();
            for (Product product : dbProducts) {
                resultMap.put(product.getId(), product);
                cacheMap.put("product:" + product.getId(),
                            JSON.toJSONString(product));
            }

            if (!cacheMap.isEmpty()) {
                redisTemplate.opsForValue().multiSet(cacheMap);
                // 设置过期时间
                cacheMap.keySet().forEach(key ->
                        redisTemplate.expire(key, Duration.ofHours(1)));
            }
        }

        // 5. 保持原有顺序返回
        return productIds.stream()
                .map(resultMap::get)
                .filter(Objects::nonNull)
                .collect(Collectors.toList());
    }

    // ==================== 4. 连接复用优化 ====================

    /**
     * HTTP连接池配置
     */
    @Bean
    public CloseableHttpClient httpClient() {
        PoolingHttpClientConnectionManager connectionManager =
                new PoolingHttpClientConnectionManager();
        connectionManager.setMaxTotal(500);           // 最大连接数
        connectionManager.setDefaultMaxPerRoute(100); // 每个路由最大连接

        RequestConfig requestConfig = RequestConfig.custom()
                .setConnectTimeout(3000)
                .setSocketTimeout(5000)
                .setConnectionRequestTimeout(1000)
                .build();

        return HttpClients.custom()
                .setConnectionManager(connectionManager)
                .setDefaultRequestConfig(requestConfig)
                .setKeepAliveStrategy((response, context) -> 30 * 1000) // 30秒
                .build();
    }
}

/**
 * 数据库优化配置
 */
@Configuration
public class DatabaseOptimizationConfig {

    /**
     * 数据库连接池优化 (HikariCP)
     */
    @Bean
    @ConfigurationProperties("spring.datasource.hikari")
    public HikariDataSource dataSource() {
        HikariDataSource dataSource = new HikariDataSource();
        // 核心配置
        dataSource.setMaximumPoolSize(50);        // 最大连接数
        dataSource.setMinimumIdle(10);            // 最小空闲连接
        dataSource.setConnectionTimeout(30000);   // 连接超时
        dataSource.setIdleTimeout(600000);        // 空闲超时
        dataSource.setMaxLifetime(1800000);       // 连接最大生命周期

        // 性能优化配置
        dataSource.addDataSourceProperty("cachePrepStmts", "true");
        dataSource.addDataSourceProperty("prepStmtCacheSize", "250");
        dataSource.addDataSourceProperty("prepStmtCacheSqlLimit", "2048");
        dataSource.addDataSourceProperty("useServerPrepStmts", "true");

        return dataSource;
    }
}
```

#### QPS优化效果对比

```
┌─────────────────────────────────────────────────────────────────┐
│                    QPS优化效果对比                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  优化措施                    优化前      优化后      提升倍数    │
│  ─────────────────────────────────────────────────────────────  │
│  │                                                              │
│  │ 添加本地缓存              1000 QPS   5000 QPS     5x         │
│  │ 添加Redis缓存             1000 QPS   3000 QPS     3x         │
│  │ 串行改并行调用            500 QPS    2000 QPS     4x         │
│  │ 批量操作优化              200 QPS    1000 QPS     5x         │
│  │ 连接池优化                800 QPS    1500 QPS     1.8x       │
│  │ 数据库索引优化            300 QPS    2000 QPS     6.7x       │
│  │ 读写分离                  1000 QPS   3000 QPS     3x         │
│  │ JVM GC调优                1500 QPS   2000 QPS     1.3x       │
│  │                                                              │
│  ─────────────────────────────────────────────────────────────  │
│                                                                 │
│  综合优化效果: 100 QPS → 10000+ QPS (100倍提升)                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

### 183. 读写分离如何实现？

```
┌─────────────────────────────────────────────────────────────────┐
│                      读写分离架构                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                        应用程序                                  │
│                           │                                     │
│                           ▼                                     │
│              ┌─────────────────────────┐                        │
│              │      数据源路由          │                        │
│              │   (AbstractRoutingDS)   │                        │
│              └───────────┬─────────────┘                        │
│                          │                                      │
│           ┌──────────────┼──────────────┐                       │
│           │              │              │                       │
│           ▼              ▼              ▼                       │
│      ┌────────┐    ┌────────┐    ┌────────┐                    │
│      │ Master │    │ Slave1 │    │ Slave2 │                    │
│      │  写库  │───▶│  读库  │    │  读库  │                    │
│      └────────┘    └────────┘    └────────┘                    │
│           │              ▲              ▲                       │
│           │              │              │                       │
│           └──────────────┴──────────────┘                       │
│                     主从复制                                     │
│                                                                 │
│  路由规则:                                                       │
│  • 写操作 (INSERT/UPDATE/DELETE) → Master                       │
│  • 读操作 (SELECT) → Slave (负载均衡)                           │
│  • 事务中的读操作 → Master (保证一致性)                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### 实现方案

```java
/**
 * 方案一：基于Spring的动态数据源
 */

// 1. 数据源类型枚举
public enum DataSourceType {
    MASTER, SLAVE
}

// 2. 数据源上下文持有者
public class DataSourceContextHolder {
    private static final ThreadLocal<DataSourceType> CONTEXT =
            new ThreadLocal<>();

    public static void setDataSourceType(DataSourceType type) {
        CONTEXT.set(type);
    }

    public static DataSourceType getDataSourceType() {
        return CONTEXT.get();
    }

    public static void clear() {
        CONTEXT.remove();
    }
}

// 3. 动态数据源实现
public class DynamicDataSource extends AbstractRoutingDataSource {

    @Override
    protected Object determineCurrentLookupKey() {
        DataSourceType type = DataSourceContextHolder.getDataSourceType();
        return type != null ? type : DataSourceType.MASTER;
    }
}

// 4. 数据源配置
@Configuration
public class DataSourceConfig {

    @Bean
    @ConfigurationProperties("spring.datasource.master")
    public DataSource masterDataSource() {
        return DataSourceBuilder.create().build();
    }

    @Bean
    @ConfigurationProperties("spring.datasource.slave")
    public DataSource slaveDataSource() {
        return DataSourceBuilder.create().build();
    }

    @Bean
    @Primary
    public DataSource dynamicDataSource(
            @Qualifier("masterDataSource") DataSource master,
            @Qualifier("slaveDataSource") DataSource slave) {

        Map<Object, Object> targetDataSources = new HashMap<>();
        targetDataSources.put(DataSourceType.MASTER, master);
        targetDataSources.put(DataSourceType.SLAVE, slave);

        DynamicDataSource dynamicDataSource = new DynamicDataSource();
        dynamicDataSource.setTargetDataSources(targetDataSources);
        dynamicDataSource.setDefaultTargetDataSource(master);

        return dynamicDataSource;
    }
}

// 5. 自定义注解
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
public @interface ReadOnly {
}

// 6. AOP切面自动切换数据源
@Aspect
@Component
@Order(-1) // 确保在事务之前执行
public class DataSourceAspect {

    @Pointcut("@annotation(com.example.annotation.ReadOnly)")
    public void readOnlyPointcut() {}

    @Pointcut("execution(* com.example.service..*.*(..))")
    public void servicePointcut() {}

    @Around("servicePointcut()")
    public Object around(ProceedingJoinPoint point) throws Throwable {
        MethodSignature signature = (MethodSignature) point.getSignature();
        Method method = signature.getMethod();

        // 检查是否有@ReadOnly注解
        ReadOnly readOnly = method.getAnnotation(ReadOnly.class);
        if (readOnly == null) {
            readOnly = point.getTarget().getClass()
                    .getAnnotation(ReadOnly.class);
        }

        if (readOnly != null) {
            DataSourceContextHolder.setDataSourceType(DataSourceType.SLAVE);
        } else {
            // 根据方法名自动判断
            String methodName = method.getName();
            if (isReadMethod(methodName)) {
                DataSourceContextHolder.setDataSourceType(DataSourceType.SLAVE);
            } else {
                DataSourceContextHolder.setDataSourceType(DataSourceType.MASTER);
            }
        }

        try {
            return point.proceed();
        } finally {
            DataSourceContextHolder.clear();
        }
    }

    private boolean isReadMethod(String methodName) {
        return methodName.startsWith("get") ||
               methodName.startsWith("find") ||
               methodName.startsWith("query") ||
               methodName.startsWith("select") ||
               methodName.startsWith("list") ||
               methodName.startsWith("count");
    }
}

// 7. 使用示例
@Service
public class UserService {

    @Autowired
    private UserMapper userMapper;

    // 自动路由到从库
    @ReadOnly
    public User findById(Long id) {
        return userMapper.selectById(id);
    }

    // 自动路由到主库
    @Transactional
    public void updateUser(User user) {
        userMapper.updateById(user);
    }

    // 事务中的读操作，强制走主库
    @Transactional
    public User findAndUpdate(Long id) {
        // 这里走主库，避免主从延迟导致数据不一致
        User user = userMapper.selectById(id);
        user.setUpdateTime(new Date());
        userMapper.updateById(user);
        return user;
    }
}

/**
 * 方案二：使用ShardingSphere读写分离
 */
// application.yml配置
/*
spring:
  shardingsphere:
    datasource:
      names: master,slave0,slave1
      master:
        type: com.zaxxer.hikari.HikariDataSource
        driver-class-name: com.mysql.cj.jdbc.Driver
        jdbc-url: jdbc:mysql://master:3306/db
        username: root
        password: root
      slave0:
        type: com.zaxxer.hikari.HikariDataSource
        driver-class-name: com.mysql.cj.jdbc.Driver
        jdbc-url: jdbc:mysql://slave0:3306/db
        username: root
        password: root
      slave1:
        type: com.zaxxer.hikari.HikariDataSource
        driver-class-name: com.mysql.cj.jdbc.Driver
        jdbc-url: jdbc:mysql://slave1:3306/db
        username: root
        password: root
    rules:
      readwrite-splitting:
        data-sources:
          readwrite_ds:
            static-strategy:
              write-data-source-name: master
              read-data-source-names: slave0,slave1
            load-balancer-name: round_robin
        load-balancers:
          round_robin:
            type: ROUND_ROBIN
*/

/**
 * 方案三：多从库负载均衡
 */
public class LoadBalancedSlaveDataSource extends AbstractRoutingDataSource {

    private final List<DataSource> slaveDataSources;
    private final AtomicInteger counter = new AtomicInteger(0);

    public LoadBalancedSlaveDataSource(List<DataSource> slaves) {
        this.slaveDataSources = slaves;
    }

    // 轮询算法
    public DataSource getSlaveDataSource() {
        int index = Math.abs(counter.getAndIncrement()) % slaveDataSources.size();
        return slaveDataSources.get(index);
    }

    // 加权轮询
    public DataSource getWeightedSlaveDataSource() {
        // 实现加权轮询逻辑
        return null;
    }
}

/**
 * 处理主从延迟问题
 */
@Service
public class ConsistentReadService {

    @Autowired
    private StringRedisTemplate redisTemplate;

    /**
     * 写后读一致性保证
     * 写操作后设置标记，短时间内强制读主库
     */
    public void updateWithConsistentRead(String userId, User user) {
        // 1. 执行写操作
        userMapper.updateById(user);

        // 2. 设置强制读主库标记，有效期等于主从复制延迟时间
        String key = "force_master:" + userId;
        redisTemplate.opsForValue().set(key, "1", Duration.ofSeconds(3));
    }

    public User getUser(String userId) {
        // 检查是否需要强制读主库
        String key = "force_master:" + userId;
        if (Boolean.TRUE.equals(redisTemplate.hasKey(key))) {
            DataSourceContextHolder.setDataSourceType(DataSourceType.MASTER);
        } else {
            DataSourceContextHolder.setDataSourceType(DataSourceType.SLAVE);
        }

        try {
            return userMapper.selectById(userId);
        } finally {
            DataSourceContextHolder.clear();
        }
    }
}
```

---

### 184. 数据库连接池如何优化？

```
┌─────────────────────────────────────────────────────────────────┐
│                    数据库连接池工作原理                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  应用线程                        连接池                          │
│                                                                 │
│  Thread-1 ─┐                    ┌────────────────────────┐     │
│            │  getConnection()   │  ┌────┐ ┌────┐ ┌────┐ │     │
│  Thread-2 ─┼───────────────────▶│  │ C1 │ │ C2 │ │ C3 │ │     │
│            │                    │  │空闲│ │使用│ │空闲│ │     │
│  Thread-3 ─┤                    │  └────┘ └────┘ └────┘ │     │
│            │  releaseConn()     │                        │     │
│  Thread-4 ─┼◀──────────────────│  ┌────┐ ┌────┐ ┌────┐ │     │
│            │                    │  │ C4 │ │ C5 │ │ C6 │ │     │
│  Thread-5 ─┘                    │  │使用│ │使用│ │空闲│ │     │
│                                 │  └────┘ └────┘ └────┘ │     │
│                                 └────────────────────────┘     │
│                                         │                       │
│                                         │ JDBC                  │
│                                         ▼                       │
│                                 ┌────────────────┐              │
│                                 │   数据库服务器   │              │
│                                 └────────────────┘              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### HikariCP优化配置

```java
/**
 * HikariCP连接池优化配置
 */
@Configuration
public class HikariCPOptimizationConfig {

    @Bean
    public HikariDataSource optimizedDataSource() {
        HikariConfig config = new HikariConfig();

        // ==================== 基础配置 ====================
        config.setJdbcUrl("jdbc:mysql://localhost:3306/db?useSSL=false&serverTimezone=UTC");
        config.setUsername("root");
        config.setPassword("password");
        config.setDriverClassName("com.mysql.cj.jdbc.Driver");

        // ==================== 核心参数优化 ====================

        /**
         * maximumPoolSize - 最大连接数
         *
         * 计算公式: connections = ((core_count * 2) + effective_spindle_count)
         *
         * 对于4核CPU、1个磁盘: 4 * 2 + 1 = 9
         * 一般建议: 10-20 (根据实际压测调整)
         *
         * 过大问题: 上下文切换开销、数据库端连接限制
         * 过小问题: 连接等待、吞吐量下降
         */
        config.setMaximumPoolSize(20);

        /**
         * minimumIdle - 最小空闲连接数
         *
         * 建议: 等于maximumPoolSize (避免连接创建开销)
         * 生产环境通常设置为相同值
         */
        config.setMinimumIdle(10);

        /**
         * connectionTimeout - 获取连接超时时间
         *
         * 默认: 30秒
         * 建议: 根据业务容忍度设置，通常3-10秒
         */
        config.setConnectionTimeout(10000); // 10秒

        /**
         * idleTimeout - 空闲连接超时时间
         *
         * 默认: 600000 (10分钟)
         * 仅当minimumIdle < maximumPoolSize时生效
         * 设为0表示永不超时
         */
        config.setIdleTimeout(600000);

        /**
         * maxLifetime - 连接最大生命周期
         *
         * 默认: 1800000 (30分钟)
         * 必须小于数据库的wait_timeout
         * 建议设置为数据库超时时间的80%
         */
        config.setMaxLifetime(1800000);

        /**
         * keepaliveTime - 连接保活时间
         *
         * HikariCP 4.0+支持
         * 防止连接被数据库/网络设备关闭
         */
        config.setKeepaliveTime(300000); // 5分钟

        /**
         * connectionTestQuery - 连接测试SQL
         *
         * HikariCP推荐不设置，使用JDBC4的isValid()
         * 如需设置: SELECT 1
         */
        // config.setConnectionTestQuery("SELECT 1");

        // ==================== 性能优化参数 ====================

        /**
         * prepStmtCacheSize - PreparedStatement缓存大小
         *
         * 默认: 25
         * 建议: 250-500
         */
        config.addDataSourceProperty("cachePrepStmts", "true");
        config.addDataSourceProperty("prepStmtCacheSize", "250");

        /**
         * prepStmtCacheSqlLimit - 可缓存的SQL最大长度
         *
         * 默认: 256
         * 建议: 2048
         */
        config.addDataSourceProperty("prepStmtCacheSqlLimit", "2048");

        /**
         * useServerPrepStmts - 使用服务端预编译
         *
         * 开启后真正使用服务端预编译，提升性能
         */
        config.addDataSourceProperty("useServerPrepStmts", "true");

        /**
         * rewriteBatchedStatements - 批量语句重写
         *
         * 开启后批量操作性能提升明显
         */
        config.addDataSourceProperty("rewriteBatchedStatements", "true");

        /**
         * useLocalSessionState - 使用本地会话状态
         */
        config.addDataSourceProperty("useLocalSessionState", "true");

        /**
         * cacheResultSetMetadata - 缓存结果集元数据
         */
        config.addDataSourceProperty("cacheResultSetMetadata", "true");

        /**
         * cacheServerConfiguration - 缓存服务器配置
         */
        config.addDataSourceProperty("cacheServerConfiguration", "true");

        // ==================== 监控配置 ====================

        /**
         * 连接池名称 (便于监控区分)
         */
        config.setPoolName("HikariCP-Primary");

        /**
         * 注册MBean用于JMX监控
         */
        config.setRegisterMbeans(true);

        /**
         * 泄漏检测阈值
         *
         * 如果连接借出超过此时间未归还，记录警告
         * 0表示禁用
         */
        config.setLeakDetectionThreshold(60000); // 1分钟

        return new HikariDataSource(config);
    }
}

/**
 * 连接池监控
 */
@Component
@Slf4j
public class ConnectionPoolMonitor {

    @Autowired
    private HikariDataSource dataSource;

    @Scheduled(fixedRate = 60000) // 每分钟执行
    public void monitor() {
        HikariPoolMXBean poolMXBean = dataSource.getHikariPoolMXBean();

        log.info("连接池状态 - 池名:{}, 总连接:{}, 活跃连接:{}, 空闲连接:{}, 等待线程:{}",
                dataSource.getPoolName(),
                poolMXBean.getTotalConnections(),
                poolMXBean.getActiveConnections(),
                poolMXBean.getIdleConnections(),
                poolMXBean.getThreadsAwaitingConnection());

        // 告警判断
        if (poolMXBean.getThreadsAwaitingConnection() > 0) {
            log.warn("存在等待连接的线程，考虑增加连接池大小");
        }

        double usageRate = (double) poolMXBean.getActiveConnections() /
                          poolMXBean.getTotalConnections();
        if (usageRate > 0.8) {
            log.warn("连接池使用率超过80%: {}%", usageRate * 100);
        }
    }
}
```

#### 连接池参数对照表

```
┌─────────────────────────────────────────────────────────────────┐
│                    连接池参数最佳实践                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  参数名                    默认值      建议值        说明        │
│  ─────────────────────────────────────────────────────────────  │
│                                                                 │
│  maximumPoolSize          10         10-20      最大连接数      │
│  minimumIdle              10         =maximum   最小空闲连接    │
│  connectionTimeout        30s        10s        获取连接超时    │
│  idleTimeout              600s       600s       空闲连接超时    │
│  maxLifetime              1800s      1800s      连接最大生命    │
│  keepaliveTime            0          300s       连接保活时间    │
│  leakDetectionThreshold   0          60s        泄漏检测阈值    │
│                                                                 │
│  ─────────────────── MySQL优化参数 ───────────────────────────  │
│                                                                 │
│  cachePrepStmts           false      true       缓存预编译语句  │
│  prepStmtCacheSize        25         250        预编译缓存大小  │
│  prepStmtCacheSqlLimit    256        2048       可缓存SQL长度   │
│  useServerPrepStmts       false      true       服务端预编译    │
│  rewriteBatchedStatements false      true       批量语句重写    │
│                                                                 │
│  ─────────────────── 连接池大小计算 ───────────────────────────  │
│                                                                 │
│  公式: pool_size = (core_count * 2) + effective_spindle_count   │
│                                                                 │
│  示例:                                                          │
│  • 4核CPU + SSD: 4 * 2 + 1 = 9 ≈ 10                            │
│  • 8核CPU + SSD: 8 * 2 + 1 = 17 ≈ 20                           │
│  • 实际建议: 根据压测结果微调                                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

### 185. 如何做接口性能优化？

```
┌─────────────────────────────────────────────────────────────────┐
│                    接口性能优化全流程                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  请求 ─────────────────────────────────────────────────▶ 响应   │
│                                                                 │
│  ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐  │
│  │网络传输│─▶│请求解析│─▶│业务处理│─▶│数据访问│─▶│响应返回│  │
│  └────────┘  └────────┘  └────────┘  └────────┘  └────────┘  │
│       │          │           │           │           │        │
│       ▼          ▼           ▼           ▼           ▼        │
│  • 压缩传输   • 参数校验   • 并行处理   • 缓存优化   • 压缩响应│
│  • 连接复用   • 快速失败   • 异步处理   • SQL优化    • 分页返回│
│  • CDN加速   • 限流熔断   • 算法优化   • 批量操作   • 按需返回│
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### 接口优化实践

```java
/**
 * 接口性能优化综合示例
 */
@RestController
@RequestMapping("/api/v1/orders")
@Slf4j
public class OrderController {

    @Autowired
    private OrderService orderService;

    /**
     * 优化点1: 合理的超时设置
     */
    @PostMapping
    @Timeout(value = 3, unit = TimeUnit.SECONDS)
    public Result<OrderVO> createOrder(@Valid @RequestBody OrderRequest request) {
        return Result.success(orderService.createOrder(request));
    }

    /**
     * 优化点2: 分页查询 + 按需返回字段
     */
    @GetMapping
    public Result<PageResult<OrderVO>> listOrders(
            @RequestParam(defaultValue = "1") int page,
            @RequestParam(defaultValue = "20") int size,
            @RequestParam(required = false) String fields) {  // 指定返回字段

        // 限制每页最大数量
        size = Math.min(size, 100);

        PageResult<OrderVO> result = orderService.listOrders(page, size, fields);
        return Result.success(result);
    }
}

/**
 * 服务层性能优化
 */
@Service
@Slf4j
public class OptimizedOrderService {

    @Autowired
    private OrderMapper orderMapper;

    @Autowired
    private StringRedisTemplate redisTemplate;

    @Autowired
    private ThreadPoolTaskExecutor asyncExecutor;

    // ==================== 1. 缓存优化 ====================

    /**
     * 缓存热点数据
     */
    @Cacheable(value = "order", key = "#orderId", unless = "#result == null")
    public Order getOrderById(String orderId) {
        return orderMapper.selectById(orderId);
    }

    /**
     * 缓存穿透防护
     */
    public Order getOrderWithBloomFilter(String orderId) {
        // 1. 布隆过滤器判断
        if (!bloomFilter.mightContain(orderId)) {
            return null; // 快速返回
        }

        // 2. 缓存查询
        String cacheKey = "order:" + orderId;
        String cached = redisTemplate.opsForValue().get(cacheKey);

        if (cached != null) {
            if ("NULL".equals(cached)) {
                return null; // 空值缓存
            }
            return JSON.parseObject(cached, Order.class);
        }

        // 3. 数据库查询
        Order order = orderMapper.selectById(orderId);

        // 4. 写入缓存 (包括空值)
        if (order != null) {
            redisTemplate.opsForValue().set(cacheKey,
                    JSON.toJSONString(order), Duration.ofHours(1));
        } else {
            redisTemplate.opsForValue().set(cacheKey,
                    "NULL", Duration.ofMinutes(5));
        }

        return order;
    }

    // ==================== 2. 并行优化 ====================

    /**
     * 串行改并行
     */
    public OrderDetailVO getOrderDetailParallel(String orderId) {
        long startTime = System.currentTimeMillis();

        // 并行获取各种数据
        CompletableFuture<Order> orderFuture = CompletableFuture
                .supplyAsync(() -> orderMapper.selectById(orderId), asyncExecutor);

        CompletableFuture<List<OrderItem>> itemsFuture = CompletableFuture
                .supplyAsync(() -> orderItemMapper.selectByOrderId(orderId), asyncExecutor);

        CompletableFuture<Logistics> logisticsFuture = CompletableFuture
                .supplyAsync(() -> logisticsMapper.selectByOrderId(orderId), asyncExecutor);

        CompletableFuture<List<PayRecord>> payFuture = CompletableFuture
                .supplyAsync(() -> payRecordMapper.selectByOrderId(orderId), asyncExecutor);

        try {
            // 等待所有结果
            CompletableFuture.allOf(orderFuture, itemsFuture,
                                   logisticsFuture, payFuture)
                    .get(5, TimeUnit.SECONDS);

            // 组装结果
            OrderDetailVO vo = new OrderDetailVO();
            vo.setOrder(orderFuture.join());
            vo.setItems(itemsFuture.join());
            vo.setLogistics(logisticsFuture.join());
            vo.setPayRecords(payFuture.join());

            log.info("并行查询耗时: {}ms", System.currentTimeMillis() - startTime);
            return vo;

        } catch (Exception e) {
            throw new ServiceException("获取订单详情失败", e);
        }
    }

    // ==================== 3. 批量优化 ====================

    /**
     * 批量查询优化 - 避免N+1问题
     */
    public List<OrderVO> listOrdersOptimized(List<String> orderIds) {
        if (CollectionUtils.isEmpty(orderIds)) {
            return Collections.emptyList();
        }

        // 1. 批量查询订单
        List<Order> orders = orderMapper.selectBatchIds(orderIds);

        // 2. 批量查询订单项 (一次查询所有)
        List<OrderItem> allItems = orderItemMapper.selectByOrderIds(orderIds);
        Map<String, List<OrderItem>> itemsMap = allItems.stream()
                .collect(Collectors.groupingBy(OrderItem::getOrderId));

        // 3. 批量查询用户信息
        Set<String> userIds = orders.stream()
                .map(Order::getUserId)
                .collect(Collectors.toSet());
        Map<String, User> userMap = userMapper.selectBatchIds(userIds)
                .stream()
                .collect(Collectors.toMap(User::getId, Function.identity()));

        // 4. 组装结果
        return orders.stream()
                .map(order -> {
                    OrderVO vo = new OrderVO();
                    BeanUtils.copyProperties(order, vo);
                    vo.setItems(itemsMap.getOrDefault(order.getId(),
                                                      Collections.emptyList()));
                    vo.setUser(userMap.get(order.getUserId()));
                    return vo;
                })
                .collect(Collectors.toList());
    }

    // ==================== 4. SQL优化 ====================

    /**
     * SQL优化技巧
     */
    public PageResult<Order> listOrdersWithOptimizedSql(OrderQuery query) {
        // 1. 覆盖索引优化 - 先查ID，再查详情
        List<Long> ids = orderMapper.selectIdsByCondition(query);

        if (ids.isEmpty()) {
            return PageResult.empty();
        }

        // 2. IN查询优化 - 保持排序
        List<Order> orders = orderMapper.selectByIdsKeepOrder(ids);

        // 3. 总数查询优化 - 只在第一页时查询
        Long total = null;
        if (query.getPage() == 1) {
            total = orderMapper.countByCondition(query);
        }

        return PageResult.of(orders, total, query.getPage(), query.getSize());
    }

    // ==================== 5. 异步化优化 ====================

    /**
     * 异步处理非核心逻辑
     */
    @Transactional
    public OrderResult createOrderAsync(OrderRequest request) {
        // 1. 核心逻辑同步处理
        Order order = createOrderCore(request);

        // 2. 非核心逻辑异步处理
        CompletableFuture.runAsync(() -> {
            // 发送通知
            notificationService.sendOrderNotification(order);
        }, asyncExecutor);

        CompletableFuture.runAsync(() -> {
            // 记录日志
            orderLogService.recordOrderLog(order);
        }, asyncExecutor);

        CompletableFuture.runAsync(() -> {
            // 更新统计
            statisticsService.updateOrderStatistics(order);
        }, asyncExecutor);

        return OrderResult.success(order);
    }
}

/**
 * 响应压缩配置
 */
@Configuration
public class ResponseCompressionConfig implements WebMvcConfigurer {

    /**
     * 配置响应压缩
     */
    @Bean
    public FilterRegistrationBean<GzipFilter> gzipFilter() {
        FilterRegistrationBean<GzipFilter> registration =
                new FilterRegistrationBean<>();
        registration.setFilter(new GzipFilter());
        registration.addUrlPatterns("/*");
        return registration;
    }
}

// application.yml 配置
/*
server:
  compression:
    enabled: true
    mime-types: application/json,application/xml,text/html,text/plain
    min-response-size: 1024  # 大于1KB才压缩
*/
```

---

### 186. 异步化设计如何做？

```
┌─────────────────────────────────────────────────────────────────┐
│                      异步化设计模式                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 模式1: 线程池异步                                        │   │
│  │                                                         │   │
│  │  主线程 ───▶ 提交任务 ───▶ 线程池 ───▶ 异步执行           │   │
│  │        │                              │                 │   │
│  │        └──────── 立即返回 ◀────────────┘                 │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 模式2: 消息队列异步                                      │   │
│  │                                                         │   │
│  │  生产者 ───▶ 消息队列 ───▶ 消费者                        │   │
│  │    │            │            │                          │   │
│  │    └─ 解耦 ─────┴─ 削峰 ─────┴─ 最终一致                 │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 模式3: 事件驱动异步                                      │   │
│  │                                                         │   │
│  │  事件源 ───▶ 事件总线 ───▶ 监听器1                       │   │
│  │                   │    ───▶ 监听器2                       │   │
│  │                   └────▶ 监听器N                         │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### 异步化实现

```java
/**
 * 异步化设计完整示例
 */

// ==================== 1. Spring @Async 异步 ====================

@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {

    @Override
    @Bean("asyncExecutor")
    public Executor getAsyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10);
        executor.setMaxPoolSize(50);
        executor.setQueueCapacity(1000);
        executor.setKeepAliveSeconds(60);
        executor.setThreadNamePrefix("Async-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.setWaitForTasksToCompleteOnShutdown(true);
        executor.setAwaitTerminationSeconds(60);
        executor.initialize();
        return executor;
    }

    @Override
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        return (ex, method, params) -> {
            log.error("异步方法异常: method={}, params={}",
                     method.getName(), Arrays.toString(params), ex);
        };
    }
}

@Service
@Slf4j
public class AsyncService {

    /**
     * 简单异步方法 - 无返回值
     */
    @Async("asyncExecutor")
    public void asyncTask(String param) {
        log.info("异步任务开始: {}", param);
        // 执行耗时操作
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        log.info("异步任务完成: {}", param);
    }

    /**
     * 异步方法 - 有返回值
     */
    @Async("asyncExecutor")
    public CompletableFuture<String> asyncTaskWithResult(String param) {
        log.info("异步任务开始: {}", param);
        // 模拟耗时操作
        String result = processData(param);
        return CompletableFuture.completedFuture(result);
    }
}

// ==================== 2. CompletableFuture 编排 ====================

@Service
public class CompletableFutureService {

    @Autowired
    @Qualifier("asyncExecutor")
    private Executor executor;

    /**
     * 并行执行多个异步任务
     */
    public OrderDetailVO getOrderDetailAsync(String orderId) {
        // 并行获取各种数据
        CompletableFuture<Order> orderFuture = CompletableFuture
                .supplyAsync(() -> orderService.getById(orderId), executor);

        CompletableFuture<User> userFuture = orderFuture
                .thenComposeAsync(order -> CompletableFuture
                        .supplyAsync(() -> userService.getById(order.getUserId()), executor));

        CompletableFuture<List<OrderItem>> itemsFuture = CompletableFuture
                .supplyAsync(() -> orderItemService.getByOrderId(orderId), executor);

        CompletableFuture<Logistics> logisticsFuture = CompletableFuture
                .supplyAsync(() -> logisticsService.getByOrderId(orderId), executor);

        // 组合所有结果
        return CompletableFuture.allOf(orderFuture, userFuture, itemsFuture, logisticsFuture)
                .thenApply(v -> {
                    OrderDetailVO vo = new OrderDetailVO();
                    vo.setOrder(orderFuture.join());
                    vo.setUser(userFuture.join());
                    vo.setItems(itemsFuture.join());
                    vo.setLogistics(logisticsFuture.join());
                    return vo;
                })
                .exceptionally(ex -> {
                    log.error("获取订单详情异常", ex);
                    throw new ServiceException("获取订单详情失败");
                })
                .join();
    }

    /**
     * 异步任务编排 - 复杂场景
     */
    public void complexAsyncFlow(OrderRequest request) {
        // 第一阶段: 校验 (并行)
        CompletableFuture<Boolean> stockCheck = CompletableFuture
                .supplyAsync(() -> inventoryService.checkStock(request), executor);
        CompletableFuture<Boolean> userCheck = CompletableFuture
                .supplyAsync(() -> userService.checkUser(request.getUserId()), executor);
        CompletableFuture<Boolean> priceCheck = CompletableFuture
                .supplyAsync(() -> priceService.checkPrice(request), executor);

        // 所有校验通过后继续
        CompletableFuture.allOf(stockCheck, userCheck, priceCheck)
                .thenAcceptAsync(v -> {
                    if (stockCheck.join() && userCheck.join() && priceCheck.join()) {
                        // 第二阶段: 创建订单
                        Order order = orderService.createOrder(request);

                        // 第三阶段: 后续处理 (并行、不阻塞)
                        CompletableFuture.runAsync(() ->
                                inventoryService.lockStock(order), executor);
                        CompletableFuture.runAsync(() ->
                                notifyService.sendNotification(order), executor);
                        CompletableFuture.runAsync(() ->
                                logService.recordLog(order), executor);
                    }
                }, executor)
                .exceptionally(ex -> {
                    log.error("订单流程异常", ex);
                    return null;
                });
    }
}

// ==================== 3. 消息队列异步 ====================

@Service
public class MqAsyncService {

    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate;

    /**
     * 发送异步消息
     */
    public void asyncProcess(Order order) {
        // 发送到消息队列，异步处理
        OrderMessage message = OrderMessage.builder()
                .orderId(order.getId())
                .type(MessageType.ORDER_CREATED)
                .data(JSON.toJSONString(order))
                .createTime(System.currentTimeMillis())
                .build();

        kafkaTemplate.send("order-topic", order.getId(),
                          JSON.toJSONString(message))
                .addCallback(
                    result -> log.info("消息发送成功: {}", order.getId()),
                    ex -> log.error("消息发送失败: {}", order.getId(), ex)
                );
    }
}

@Component
@Slf4j
public class OrderMessageConsumer {

    @KafkaListener(topics = "order-topic", groupId = "order-processor")
    public void consume(ConsumerRecord<String, String> record) {
        try {
            OrderMessage message = JSON.parseObject(record.value(), OrderMessage.class);

            // 根据消息类型处理
            switch (message.getType()) {
                case ORDER_CREATED:
                    handleOrderCreated(message);
                    break;
                case ORDER_PAID:
                    handleOrderPaid(message);
                    break;
                default:
                    log.warn("未知消息类型: {}", message.getType());
            }
        } catch (Exception e) {
            log.error("消息处理失败", e);
            // 发送到死信队列或重试
        }
    }
}

// ==================== 4. 事件驱动异步 ====================

// 定义事件
public class OrderCreatedEvent extends ApplicationEvent {
    private final Order order;

    public OrderCreatedEvent(Object source, Order order) {
        super(source);
        this.order = order;
    }

    public Order getOrder() {
        return order;
    }
}

// 发布事件
@Service
public class OrderService {

    @Autowired
    private ApplicationEventPublisher eventPublisher;

    @Transactional
    public Order createOrder(OrderRequest request) {
        Order order = buildOrder(request);
        orderMapper.insert(order);

        // 发布事件，异步处理后续逻辑
        eventPublisher.publishEvent(new OrderCreatedEvent(this, order));

        return order;
    }
}

// 监听事件
@Component
@Slf4j
public class OrderEventListener {

    /**
     * 异步监听订单创建事件
     */
    @Async
    @EventListener
    public void onOrderCreated(OrderCreatedEvent event) {
        Order order = event.getOrder();
        log.info("处理订单创建事件: {}", order.getId());

        // 发送通知
        notificationService.sendOrderCreatedNotification(order);

        // 更新统计
        statisticsService.updateStatistics(order);
    }

    /**
     * 使用@TransactionalEventListener确保事务提交后执行
     */
    @Async
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void afterOrderCommitted(OrderCreatedEvent event) {
        Order order = event.getOrder();
        log.info("订单事务提交后处理: {}", order.getId());

        // 发送消息到外部系统
        externalService.notifyOrderCreated(order);
    }
}
```

---

### 187. 池化技术有哪些应用？

```
┌─────────────────────────────────────────────────────────────────┐
│                      池化技术应用全景                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                     池化技术核心思想                       │   │
│  │                                                         │   │
│  │  问题: 资源创建/销毁开销大                                │   │
│  │  解决: 预创建 + 复用 + 统一管理                          │   │
│  │                                                         │   │
│  │  ┌─────────┐     借用      ┌─────────┐                  │   │
│  │  │  调用方  │ ───────────▶ │  资源池  │                  │   │
│  │  │         │ ◀─────────── │         │                  │   │
│  │  └─────────┘     归还      └─────────┘                  │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌───────────────────┬───────────────────┬─────────────────┐   │
│  │    连接池          │     线程池         │     对象池      │   │
│  ├───────────────────┼───────────────────┼─────────────────┤   │
│  │ • 数据库连接池     │ • 业务线程池       │ • StringBuilder │   │
│  │ • Redis连接池      │ • IO线程池        │ • 大对象复用    │   │
│  │ • HTTP连接池       │ • 定时任务池       │ • 字节数组池    │   │
│  │ • TCP连接池        │ • ForkJoin池      │ • Buffer池     │   │
│  └───────────────────┴───────────────────┴─────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### 各类池化实现

```java
/**
 * 池化技术应用示例
 */

// ==================== 1. 数据库连接池 ====================

@Configuration
public class DatabasePoolConfig {

    /**
     * HikariCP - 最快的连接池
     */
    @Bean
    public HikariDataSource hikariDataSource() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:mysql://localhost:3306/db");
        config.setUsername("root");
        config.setPassword("password");
        config.setMaximumPoolSize(20);
        config.setMinimumIdle(5);
        config.setConnectionTimeout(30000);
        config.setIdleTimeout(600000);
        config.setMaxLifetime(1800000);
        return new HikariDataSource(config);
    }
}

// ==================== 2. Redis连接池 ====================

@Configuration
public class RedisPoolConfig {

    @Bean
    public LettuceConnectionFactory redisConnectionFactory() {
        // 连接池配置
        GenericObjectPoolConfig<Object> poolConfig = new GenericObjectPoolConfig<>();
        poolConfig.setMaxTotal(100);
        poolConfig.setMaxIdle(50);
        poolConfig.setMinIdle(10);
        poolConfig.setMaxWait(Duration.ofMillis(3000));
        poolConfig.setTestOnBorrow(true);

        LettucePoolingClientConfiguration clientConfig =
                LettucePoolingClientConfiguration.builder()
                        .commandTimeout(Duration.ofMillis(3000))
                        .poolConfig(poolConfig)
                        .build();

        RedisStandaloneConfiguration serverConfig =
                new RedisStandaloneConfiguration("localhost", 6379);

        return new LettuceConnectionFactory(serverConfig, clientConfig);
    }
}

// ==================== 3. HTTP连接池 ====================

@Configuration
public class HttpPoolConfig {

    @Bean
    public CloseableHttpClient httpClient() {
        // 连接池管理器
        PoolingHttpClientConnectionManager connectionManager =
                new PoolingHttpClientConnectionManager();
        connectionManager.setMaxTotal(500);           // 最大连接数
        connectionManager.setDefaultMaxPerRoute(100); // 每个路由最大连接
        connectionManager.setValidateAfterInactivity(30000); // 空闲验证

        // 请求配置
        RequestConfig requestConfig = RequestConfig.custom()
                .setConnectTimeout(3000)              // 连接超时
                .setSocketTimeout(10000)              // 读取超时
                .setConnectionRequestTimeout(3000)   // 从池获取连接超时
                .build();

        return HttpClients.custom()
                .setConnectionManager(connectionManager)
                .setDefaultRequestConfig(requestConfig)
                .setKeepAliveStrategy((response, context) -> 60 * 1000) // 保活60秒
                .evictExpiredConnections()            // 清除过期连接
                .evictIdleConnections(60, TimeUnit.SECONDS) // 清除空闲连接
                .build();
    }

    /**
     * OkHttp连接池
     */
    @Bean
    public OkHttpClient okHttpClient() {
        ConnectionPool connectionPool = new ConnectionPool(
                100,        // 最大空闲连接数
                5,          // 保活时间
                TimeUnit.MINUTES
        );

        return new OkHttpClient.Builder()
                .connectionPool(connectionPool)
                .connectTimeout(3, TimeUnit.SECONDS)
                .readTimeout(10, TimeUnit.SECONDS)
                .writeTimeout(10, TimeUnit.SECONDS)
                .retryOnConnectionFailure(true)
                .build();
    }
}

// ==================== 4. 线程池 ====================

@Configuration
public class ThreadPoolConfig {

    /**
     * CPU密集型任务线程池
     */
    @Bean("cpuExecutor")
    public ThreadPoolTaskExecutor cpuExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        // CPU密集型: 核心线程数 = CPU核心数 + 1
        int cpuCores = Runtime.getRuntime().availableProcessors();
        executor.setCorePoolSize(cpuCores + 1);
        executor.setMaxPoolSize(cpuCores + 1);
        executor.setQueueCapacity(1000);
        executor.setThreadNamePrefix("CPU-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.initialize();
        return executor;
    }

    /**
     * IO密集型任务线程池
     */
    @Bean("ioExecutor")
    public ThreadPoolTaskExecutor ioExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        // IO密集型: 核心线程数 = CPU核心数 * 2
        int cpuCores = Runtime.getRuntime().availableProcessors();
        executor.setCorePoolSize(cpuCores * 2);
        executor.setMaxPoolSize(cpuCores * 4);
        executor.setQueueCapacity(2000);
        executor.setKeepAliveSeconds(60);
        executor.setThreadNamePrefix("IO-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.initialize();
        return executor;
    }

    /**
     * 调度任务线程池
     */
    @Bean
    public ScheduledThreadPoolExecutor scheduledExecutor() {
        ScheduledThreadPoolExecutor executor = new ScheduledThreadPoolExecutor(
                4,
                new ThreadFactoryBuilder()
                        .setNameFormat("Scheduled-%d")
                        .setDaemon(true)
                        .build()
        );
        executor.setRemoveOnCancelPolicy(true);
        return executor;
    }
}

// ==================== 5. 通用对象池 ====================

/**
 * 使用Apache Commons Pool2实现对象池
 */
public class ExpensiveObjectPool {

    private final GenericObjectPool<ExpensiveObject> pool;

    public ExpensiveObjectPool() {
        GenericObjectPoolConfig<ExpensiveObject> config =
                new GenericObjectPoolConfig<>();
        config.setMaxTotal(100);
        config.setMaxIdle(50);
        config.setMinIdle(10);
        config.setMaxWait(Duration.ofMillis(3000));
        config.setTestOnBorrow(true);
        config.setTestWhileIdle(true);

        pool = new GenericObjectPool<>(new ExpensiveObjectFactory(), config);
    }

    public ExpensiveObject borrowObject() throws Exception {
        return pool.borrowObject();
    }

    public void returnObject(ExpensiveObject obj) {
        pool.returnObject(obj);
    }

    public void close() {
        pool.close();
    }
}

class ExpensiveObjectFactory extends BasePooledObjectFactory<ExpensiveObject> {

    @Override
    public ExpensiveObject create() throws Exception {
        // 创建开销大的对象
        return new ExpensiveObject();
    }

    @Override
    public PooledObject<ExpensiveObject> wrap(ExpensiveObject obj) {
        return new DefaultPooledObject<>(obj);
    }

    @Override
    public void destroyObject(PooledObject<ExpensiveObject> p) throws Exception {
        p.getObject().close();
    }

    @Override
    public boolean validateObject(PooledObject<ExpensiveObject> p) {
        return p.getObject().isValid();
    }

    @Override
    public void activateObject(PooledObject<ExpensiveObject> p) throws Exception {
        p.getObject().reset();
    }
}

// 使用示例
@Service
public class PoolUsageService {

    @Autowired
    private ExpensiveObjectPool objectPool;

    public void doSomething() {
        ExpensiveObject obj = null;
        try {
            obj = objectPool.borrowObject();
            // 使用对象
            obj.process();
        } catch (Exception e) {
            throw new RuntimeException(e);
        } finally {
            if (obj != null) {
                objectPool.returnObject(obj);
            }
        }
    }
}

// ==================== 6. 字节数组池 ====================

/**
 * Netty的ByteBuf池化
 */
public class ByteBufferPoolExample {

    // 使用池化的ByteBuf分配器
    private final ByteBufAllocator allocator = PooledByteBufAllocator.DEFAULT;

    public void processData(byte[] data) {
        ByteBuf buffer = allocator.buffer(data.length);
        try {
            buffer.writeBytes(data);
            // 处理数据
        } finally {
            // 释放回池
            buffer.release();
        }
    }
}

/**
 * 简单的字节数组池
 */
public class ByteArrayPool {

    private final int bufferSize;
    private final BlockingQueue<byte[]> pool;

    public ByteArrayPool(int poolSize, int bufferSize) {
        this.bufferSize = bufferSize;
        this.pool = new ArrayBlockingQueue<>(poolSize);

        // 预创建
        for (int i = 0; i < poolSize; i++) {
            pool.offer(new byte[bufferSize]);
        }
    }

    public byte[] borrow() {
        byte[] buffer = pool.poll();
        return buffer != null ? buffer : new byte[bufferSize];
    }

    public void returnBuffer(byte[] buffer) {
        if (buffer != null && buffer.length == bufferSize) {
            Arrays.fill(buffer, (byte) 0); // 清空
            pool.offer(buffer);
        }
    }
}
```

---

## 7.2 高可用（7题）

### 188. 如何设计高可用系统？

```
┌─────────────────────────────────────────────────────────────────┐
│                      高可用设计架构图                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    DNS + 全局负载均衡                     │   │
│  │              (多机房、多地域故障转移)                      │   │
│  └───────────────────────┬─────────────────────────────────┘   │
│                          │                                      │
│          ┌───────────────┼───────────────┐                     │
│          │               │               │                     │
│          ▼               ▼               ▼                     │
│    ┌──────────┐    ┌──────────┐    ┌──────────┐               │
│    │ 机房A    │    │ 机房B    │    │ 机房C    │               │
│    │ (主)     │    │ (热备)   │    │ (冷备)   │               │
│    └────┬─────┘    └────┬─────┘    └────┬─────┘               │
│         │               │               │                      │
│         └───────────────┼───────────────┘                      │
│                         │                                      │
│  ┌──────────────────────┴──────────────────────┐              │
│  │              每个机房内部架构                  │              │
│  │                                              │              │
│  │  ┌─────────────────────────────────────┐   │              │
│  │  │         负载均衡层 (HA)              │   │              │
│  │  │    ┌─────┐         ┌─────┐          │   │              │
│  │  │    │LB主 │◀──VIP──▶│LB备 │          │   │              │
│  │  │    └─────┘         └─────┘          │   │              │
│  │  └─────────────────┬───────────────────┘   │              │
│  │                    │                        │              │
│  │  ┌─────────────────┴───────────────────┐   │              │
│  │  │           网关层 (集群)              │   │              │
│  │  │  ┌────┐ ┌────┐ ┌────┐ ┌────┐       │   │              │
│  │  │  │GW1 │ │GW2 │ │GW3 │ │GWN │       │   │              │
│  │  │  └────┘ └────┘ └────┘ └────┘       │   │              │
│  │  └─────────────────┬───────────────────┘   │              │
│  │                    │                        │              │
│  │  ┌─────────────────┴───────────────────┐   │              │
│  │  │          服务层 (集群+冗余)          │   │              │
│  │  │  ┌─────────┐  ┌─────────┐           │   │              │
│  │  │  │服务A集群│  │服务B集群│           │   │              │
│  │  │  │ (>=3节点) │  │ (>=3节点) │           │   │              │
│  │  │  └─────────┘  └─────────┘           │   │              │
│  │  └─────────────────┬───────────────────┘   │              │
│  │                    │                        │              │
│  │  ┌─────────────────┴───────────────────┐   │              │
│  │  │          数据层 (主从+分片)          │   │              │
│  │  │  ┌────────────┐  ┌────────────┐     │   │              │
│  │  │  │MySQL主从集群│  │Redis哨兵集群│     │   │              │
│  │  │  └────────────┘  └────────────┘     │   │              │
│  │  └─────────────────────────────────────┘   │              │
│  └────────────────────────────────────────────┘              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

高可用指标:
┌─────────────────────────────────────────────────────────────────┐
│  可用性等级      年度停机时间      月度停机时间    设计要求       │
│  ─────────────────────────────────────────────────────────────  │
│  99%           3.65天           7.3小时      基本可用          │
│  99.9%         8.76小时         43.8分钟     较高可用          │
│  99.99%        52.6分钟         4.38分钟     高可用            │
│  99.999%       5.26分钟         26.28秒      极高可用          │
└─────────────────────────────────────────────────────────────────┘
```

#### 高可用实现代码

```java
/**
 * 高可用设计核心组件
 */

// ==================== 1. 服务健康检查 ====================

@RestController
public class HealthController {

    @Autowired
    private DataSource dataSource;

    @Autowired
    private StringRedisTemplate redisTemplate;

    /**
     * Kubernetes存活探针
     */
    @GetMapping("/health/liveness")
    public ResponseEntity<String> liveness() {
        // 检查应用是否存活
        return ResponseEntity.ok("OK");
    }

    /**
     * Kubernetes就绪探针
     */
    @GetMapping("/health/readiness")
    public ResponseEntity<Map<String, Object>> readiness() {
        Map<String, Object> result = new HashMap<>();
        boolean ready = true;

        // 检查数据库连接
        try (Connection conn = dataSource.getConnection()) {
            result.put("database", "UP");
        } catch (Exception e) {
            result.put("database", "DOWN");
            ready = false;
        }

        // 检查Redis连接
        try {
            redisTemplate.opsForValue().get("health:check");
            result.put("redis", "UP");
        } catch (Exception e) {
            result.put("redis", "DOWN");
            ready = false;
        }

        result.put("status", ready ? "UP" : "DOWN");

        return ready ? ResponseEntity.ok(result)
                     : ResponseEntity.status(503).body(result);
    }
}

// ==================== 2. 熔断器实现 ====================

@Service
public class CircuitBreakerService {

    @Autowired
    private CircuitBreakerFactory circuitBreakerFactory;

    /**
     * 使用Resilience4j熔断器
     */
    public String callExternalService(String param) {
        CircuitBreaker circuitBreaker = circuitBreakerFactory
                .create("externalService");

        return circuitBreaker.run(
                () -> externalServiceClient.call(param),
                throwable -> fallback(param, throwable)
        );
    }

    private String fallback(String param, Throwable throwable) {
        log.warn("熔断降级: param={}", param, throwable);
        return "默认响应";
    }
}

// 熔断器配置
@Configuration
public class CircuitBreakerConfig {

    @Bean
    public Customizer<Resilience4JCircuitBreakerFactory> defaultCustomizer() {
        return factory -> factory.configureDefault(id -> new Resilience4JConfigBuilder(id)
                .circuitBreakerConfig(io.github.resilience4j.circuitbreaker.CircuitBreakerConfig
                        .custom()
                        .failureRateThreshold(50)           // 失败率阈值50%
                        .waitDurationInOpenState(Duration.ofSeconds(30)) // 熔断等待30秒
                        .slidingWindowType(SlidingWindowType.COUNT_BASED)
                        .slidingWindowSize(10)              // 滑动窗口大小
                        .minimumNumberOfCalls(5)            // 最小调用次数
                        .build())
                .timeLimiterConfig(TimeLimiterConfig.custom()
                        .timeoutDuration(Duration.ofSeconds(3)) // 超时时间
                        .build())
                .build());
    }
}

// ==================== 3. 重试机制 ====================

@Service
public class RetryService {

    /**
     * Spring Retry重试
     */
    @Retryable(
            value = {RemoteServiceException.class},
            maxAttempts = 3,
            backoff = @Backoff(delay = 1000, multiplier = 2)
    )
    public String callWithRetry(String param) {
        return remoteService.call(param);
    }

    @Recover
    public String recover(RemoteServiceException e, String param) {
        log.error("重试失败，进入降级: param={}", param, e);
        return "降级响应";
    }

    /**
     * 手动实现重试 (带指数退避)
     */
    public <T> T executeWithRetry(Supplier<T> supplier, int maxRetries) {
        int retries = 0;
        Exception lastException = null;

        while (retries < maxRetries) {
            try {
                return supplier.get();
            } catch (Exception e) {
                lastException = e;
                retries++;

                if (retries < maxRetries) {
                    // 指数退避
                    long sleepTime = (long) (1000 * Math.pow(2, retries - 1));
                    sleepTime = Math.min(sleepTime, 30000); // 最大30秒

                    log.warn("第{}次重试失败，{}ms后重试", retries, sleepTime);

                    try {
                        Thread.sleep(sleepTime);
                    } catch (InterruptedException ie) {
                        Thread.currentThread().interrupt();
                        break;
                    }
                }
            }
        }

        throw new RuntimeException("重试次数耗尽", lastException);
    }
}

// ==================== 4. 故障转移 ====================

@Service
public class FailoverService {

    @Autowired
    private List<ServiceInstance> instances;

    private final AtomicInteger currentIndex = new AtomicInteger(0);

    /**
     * 主备切换
     */
    public String callWithFailover(String param) {
        List<ServiceInstance> healthyInstances = getHealthyInstances();

        for (ServiceInstance instance : healthyInstances) {
            try {
                return doCall(instance, param);
            } catch (Exception e) {
                log.warn("实例{}调用失败，尝试下一个", instance.getHost(), e);
                markInstanceUnhealthy(instance);
            }
        }

        throw new NoAvailableInstanceException("没有可用的服务实例");
    }

    /**
     * 获取健康实例
     */
    private List<ServiceInstance> getHealthyInstances() {
        return instances.stream()
                .filter(this::isHealthy)
                .collect(Collectors.toList());
    }
}

// ==================== 5. 优雅停机 ====================

@Component
public class GracefulShutdown implements SmartLifecycle {

    private volatile boolean running = true;

    @Override
    public void start() {
        running = true;
    }

    @Override
    public void stop(Runnable callback) {
        log.info("开始优雅停机...");

        // 1. 从注册中心下线
        deregisterFromRegistry();

        // 2. 等待正在处理的请求完成
        waitForRequestsToComplete();

        // 3. 关闭资源
        closeResources();

        running = false;
        callback.run();

        log.info("优雅停机完成");
    }

    private void deregisterFromRegistry() {
        // 从Nacos/Eureka下线
        log.info("从注册中心下线");
    }

    private void waitForRequestsToComplete() {
        // 等待30秒让请求处理完成
        try {
            Thread.sleep(30000);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }

    private void closeResources() {
        // 关闭数据库连接、消息队列等
        log.info("关闭资源");
    }

    @Override
    public boolean isRunning() {
        return running;
    }
}
```

---

### 189. 服务降级的设计思路？

```
┌─────────────────────────────────────────────────────────────────┐
│                      服务降级设计方案                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    降级触发条件                           │   │
│  │                                                         │   │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐       │   │
│  │  │响应超时 │ │错误率高 │ │资源紧张 │ │人工触发 │       │   │
│  │  └────┬────┘ └────┬────┘ └────┬────┘ └────┬────┘       │   │
│  │       │          │          │          │              │   │
│  │       └──────────┴──────────┴──────────┘              │   │
│  │                       │                                │   │
│  │                       ▼                                │   │
│  │              ┌────────────────┐                        │   │
│  │              │   降级决策器    │                        │   │
│  │              └───────┬────────┘                        │   │
│  │                      │                                 │   │
│  └──────────────────────┼─────────────────────────────────┘   │
│                         │                                      │
│  ┌──────────────────────┴─────────────────────────────────┐   │
│  │                    降级策略                              │   │
│  │                                                         │   │
│  │  ┌───────────────────────────────────────────────────┐ │   │
│  │  │ 1. 返回默认值                                      │ │   │
│  │  │    商品推荐 → 返回热门商品                          │ │   │
│  │  │    用户画像 → 返回默认标签                          │ │   │
│  │  └───────────────────────────────────────────────────┘ │   │
│  │                                                         │   │
│  │  ┌───────────────────────────────────────────────────┐ │   │
│  │  │ 2. 返回缓存数据                                    │ │   │
│  │  │    实时数据 → 返回缓存的历史数据                    │ │   │
│  │  │    动态内容 → 返回静态化内容                        │ │   │
│  │  └───────────────────────────────────────────────────┘ │   │
│  │                                                         │   │
│  │  ┌───────────────────────────────────────────────────┐ │   │
│  │  │ 3. 功能降级                                        │ │   │
│  │  │    精准推荐 → 随机推荐                              │ │   │
│  │  │    实时统计 → 定时统计                              │ │   │
│  │  └───────────────────────────────────────────────────┘ │   │
│  │                                                         │   │
│  │  ┌───────────────────────────────────────────────────┐ │   │
│  │  │ 4. 服务降级                                        │ │   │
│  │  │    非核心服务直接关闭                               │ │   │
│  │  │    日志服务、统计服务暂停                           │ │   │
│  │  └───────────────────────────────────────────────────┘ │   │
│  │                                                         │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### 降级实现

```java
/**
 * 服务降级实现
 */

// ==================== 1. 注解驱动降级 ====================

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Degrade {
    /**
     * 降级处理方法名
     */
    String fallbackMethod() default "";

    /**
     * 降级类
     */
    Class<?> fallbackClass() default void.class;

    /**
     * 降级优先级
     */
    int priority() default 0;
}

@Aspect
@Component
@Slf4j
public class DegradeAspect {

    @Autowired
    private DegradeManager degradeManager;

    @Around("@annotation(degrade)")
    public Object around(ProceedingJoinPoint point, Degrade degrade) throws Throwable {
        String methodKey = getMethodKey(point);

        // 检查是否需要降级
        if (degradeManager.shouldDegrade(methodKey)) {
            log.info("服务降级触发: {}", methodKey);
            return invokeFallback(point, degrade, null);
        }

        try {
            return point.proceed();
        } catch (Exception e) {
            // 异常时触发降级
            if (degradeManager.shouldDegradeOnException(methodKey, e)) {
                return invokeFallback(point, degrade, e);
            }
            throw e;
        }
    }

    private Object invokeFallback(ProceedingJoinPoint point,
                                  Degrade degrade, Exception e) throws Exception {
        String fallbackMethod = degrade.fallbackMethod();

        if (StringUtils.isNotEmpty(fallbackMethod)) {
            // 调用同类中的降级方法
            Method method = point.getTarget().getClass()
                    .getMethod(fallbackMethod, getParameterTypes(point, e));
            return method.invoke(point.getTarget(), getArguments(point, e));
        }

        if (degrade.fallbackClass() != void.class) {
            // 调用降级类中的方法
            Object fallbackInstance = SpringContextUtil.getBean(degrade.fallbackClass());
            Method method = degrade.fallbackClass()
                    .getMethod(point.getSignature().getName(),
                              ((MethodSignature) point.getSignature()).getParameterTypes());
            return method.invoke(fallbackInstance, point.getArgs());
        }

        throw new NoFallbackException("没有配置降级方法");
    }
}

// ==================== 2. 降级管理器 ====================

@Service
@Slf4j
public class DegradeManager {

    @Autowired
    private StringRedisTemplate redisTemplate;

    // 降级开关 (支持动态配置)
    private final ConcurrentHashMap<String, Boolean> degradeSwitch =
            new ConcurrentHashMap<>();

    // 服务调用统计
    private final ConcurrentHashMap<String, ServiceCallStats> statsMap =
            new ConcurrentHashMap<>();

    /**
     * 判断是否需要降级
     */
    public boolean shouldDegrade(String methodKey) {
        // 1. 检查手动降级开关
        if (Boolean.TRUE.equals(degradeSwitch.get(methodKey))) {
            return true;
        }

        // 2. 检查Redis中的全局降级开关
        String globalKey = "degrade:switch:" + methodKey;
        if ("1".equals(redisTemplate.opsForValue().get(globalKey))) {
            return true;
        }

        // 3. 检查自动降级条件
        ServiceCallStats stats = statsMap.get(methodKey);
        if (stats != null) {
            // 错误率超过50%自动降级
            if (stats.getErrorRate() > 0.5) {
                return true;
            }
            // 平均响应时间超过3秒自动降级
            if (stats.getAvgResponseTime() > 3000) {
                return true;
            }
        }

        return false;
    }

    /**
     * 打开降级开关
     */
    public void enableDegrade(String methodKey, Duration duration) {
        String key = "degrade:switch:" + methodKey;
        redisTemplate.opsForValue().set(key, "1", duration);
        log.info("开启降级: {} for {}", methodKey, duration);
    }

    /**
     * 关闭降级开关
     */
    public void disableDegrade(String methodKey) {
        String key = "degrade:switch:" + methodKey;
        redisTemplate.delete(key);
        degradeSwitch.remove(methodKey);
        log.info("关闭降级: {}", methodKey);
    }

    /**
     * 记录调用结果
     */
    public void recordCall(String methodKey, long responseTime, boolean success) {
        statsMap.computeIfAbsent(methodKey, k -> new ServiceCallStats())
                .record(responseTime, success);
    }
}

// ==================== 3. 具体服务降级示例 ====================

@Service
@Slf4j
public class ProductService {

    @Autowired
    private ProductMapper productMapper;

    @Autowired
    private StringRedisTemplate redisTemplate;

    /**
     * 获取商品推荐 - 带降级
     */
    @Degrade(fallbackMethod = "getRecommendProductsFallback")
    public List<Product> getRecommendProducts(String userId) {
        // 调用推荐系统获取个性化推荐
        return recommendClient.getRecommendations(userId);
    }

    /**
     * 降级方法 - 返回热门商品
     */
    public List<Product> getRecommendProductsFallback(String userId, Exception e) {
        log.warn("推荐服务降级，返回热门商品: userId={}", userId, e);

        // 1. 尝试从缓存获取热门商品
        String cacheKey = "hot:products";
        String cached = redisTemplate.opsForValue().get(cacheKey);
        if (StringUtils.isNotBlank(cached)) {
            return JSON.parseArray(cached, Product.class);
        }

        // 2. 从数据库获取热门商品
        List<Product> hotProducts = productMapper.selectHotProducts(10);

        // 3. 写入缓存
        redisTemplate.opsForValue().set(cacheKey,
                JSON.toJSONString(hotProducts), Duration.ofMinutes(10));

        return hotProducts;
    }

    /**
     * 获取商品详情 - 多级降级
     */
    @Degrade(fallbackMethod = "getProductDetailFallback")
    public ProductDetailVO getProductDetail(String productId) {
        ProductDetailVO vo = new ProductDetailVO();

        // 基本信息 (必须)
        Product product = productMapper.selectById(productId);
        vo.setProduct(product);

        // 库存信息 (尝试获取，失败返回默认)
        try {
            vo.setStock(inventoryService.getStock(productId));
        } catch (Exception e) {
            log.warn("获取库存失败，返回默认值", e);
            vo.setStock(-1); // -1表示库存未知
        }

        // 评价信息 (非核心，失败忽略)
        try {
            vo.setReviews(reviewService.getReviews(productId));
        } catch (Exception e) {
            log.warn("获取评价失败，返回空列表", e);
            vo.setReviews(Collections.emptyList());
        }

        return vo;
    }

    public ProductDetailVO getProductDetailFallback(String productId, Exception e) {
        log.warn("商品详情降级: productId={}", productId, e);

        // 返回缓存中的静态化数据
        String cacheKey = "product:detail:static:" + productId;
        String cached = redisTemplate.opsForValue().get(cacheKey);

        if (StringUtils.isNotBlank(cached)) {
            return JSON.parseObject(cached, ProductDetailVO.class);
        }

        // 返回最简信息
        ProductDetailVO vo = new ProductDetailVO();
        Product product = productMapper.selectById(productId);
        vo.setProduct(product);
        return vo;
    }
}

// ==================== 4. 降级开关管理接口 ====================

@RestController
@RequestMapping("/admin/degrade")
public class DegradeController {

    @Autowired
    private DegradeManager degradeManager;

    /**
     * 开启降级
     */
    @PostMapping("/enable")
    public Result<Void> enableDegrade(
            @RequestParam String methodKey,
            @RequestParam(defaultValue = "60") int minutes) {
        degradeManager.enableDegrade(methodKey, Duration.ofMinutes(minutes));
        return Result.success();
    }

    /**
     * 关闭降级
     */
    @PostMapping("/disable")
    public Result<Void> disableDegrade(@RequestParam String methodKey) {
        degradeManager.disableDegrade(methodKey);
        return Result.success();
    }

    /**
     * 查询降级状态
     */
    @GetMapping("/status")
    public Result<Map<String, Boolean>> getDegradeStatus() {
        return Result.success(degradeManager.getAllDegradeStatus());
    }
}
```

---

### 190. 服务限流如何设计？（续）

#### 限流实现

```java
/**
 * 限流实现方案
 */

// ==================== 1. 令牌桶限流 (Guava) ====================

@Service
public class GuavaRateLimiterService {

    // 创建限流器: 每秒10个令牌
    private final RateLimiter rateLimiter = RateLimiter.create(10);

    // 预热限流器: 3秒预热期，从低速逐渐达到10 QPS
    private final RateLimiter warmUpRateLimiter =
            RateLimiter.create(10, 3, TimeUnit.SECONDS);

    /**
     * 阻塞获取令牌
     */
    public void doSomethingWithBlocking() {
        // 获取令牌，如果没有则阻塞等待
        double waitTime = rateLimiter.acquire();
        System.out.println("等待了 " + waitTime + " 秒获取到令牌");
        // 执行业务逻辑
    }

    /**
     * 非阻塞尝试获取令牌
     */
    public boolean doSomethingNonBlocking() {
        // 尝试获取令牌，立即返回结果
        if (rateLimiter.tryAcquire()) {
            // 获取成功，执行业务逻辑
            return true;
        } else {
            // 获取失败，限流
            return false;
        }
    }

    /**
     * 带超时的尝试获取
     */
    public boolean doSomethingWithTimeout() {
        // 最多等待500毫秒
        if (rateLimiter.tryAcquire(500, TimeUnit.MILLISECONDS)) {
            return true;
        }
        return false;
    }
}

// ==================== 2. Redis分布式限流 ====================

@Component
@Slf4j
public class RedisRateLimiter {

    @Autowired
    private StringRedisTemplate redisTemplate;

    /**
     * 滑动窗口限流 - Lua脚本实现
     */
    private static final String SLIDING_WINDOW_SCRIPT =
        "local key = KEYS[1] " +
        "local now = tonumber(ARGV[1]) " +
        "local window = tonumber(ARGV[2]) " +
        "local limit = tonumber(ARGV[3]) " +
        // 移除窗口外的记录
        "redis.call('ZREMRANGEBYSCORE', key, 0, now - window) " +
        // 获取当前窗口内的请求数
        "local count = redis.call('ZCARD', key) " +
        // 判断是否超过限制
        "if count < limit then " +
        "    redis.call('ZADD', key, now, now .. '-' .. math.random()) " +
        "    redis.call('EXPIRE', key, math.ceil(window / 1000)) " +
        "    return 1 " +
        "else " +
        "    return 0 " +
        "end";

    /**
     * 滑动窗口限流
     * @param key 限流key
     * @param windowMs 窗口大小(毫秒)
     * @param limit 限制次数
     */
    public boolean isAllowed(String key, long windowMs, int limit) {
        RedisScript<Long> script = new DefaultRedisScript<>(
                SLIDING_WINDOW_SCRIPT, Long.class);

        Long result = redisTemplate.execute(
                script,
                Collections.singletonList(key),
                String.valueOf(System.currentTimeMillis()),
                String.valueOf(windowMs),
                String.valueOf(limit)
        );

        return result != null && result == 1L;
    }

    /**
     * 令牌桶限流 - Lua脚本实现
     */
    private static final String TOKEN_BUCKET_SCRIPT =
        "local key = KEYS[1] " +
        "local capacity = tonumber(ARGV[1]) " +    // 桶容量
        "local rate = tonumber(ARGV[2]) " +         // 每秒生成令牌数
        "local now = tonumber(ARGV[3]) " +          // 当前时间戳(秒)
        "local requested = tonumber(ARGV[4]) " +    // 请求的令牌数

        // 获取当前令牌数和上次更新时间
        "local tokens = redis.call('HGET', key, 'tokens') " +
        "local lastTime = redis.call('HGET', key, 'lastTime') " +

        // 初始化
        "if tokens == false then " +
        "    tokens = capacity " +
        "    lastTime = now " +
        "else " +
        "    tokens = tonumber(tokens) " +
        "    lastTime = tonumber(lastTime) " +
        "end " +

        // 计算新增的令牌数
        "local deltaTime = math.max(0, now - lastTime) " +
        "local newTokens = math.min(capacity, tokens + deltaTime * rate) " +

        // 判断令牌是否足够
        "if newTokens >= requested then " +
        "    redis.call('HSET', key, 'tokens', newTokens - requested) " +
        "    redis.call('HSET', key, 'lastTime', now) " +
        "    redis.call('EXPIRE', key, 60) " +
        "    return 1 " +
        "else " +
        "    redis.call('HSET', key, 'tokens', newTokens) " +
        "    redis.call('HSET', key, 'lastTime', now) " +
        "    redis.call('EXPIRE', key, 60) " +
        "    return 0 " +
        "end";

    /**
     * 令牌桶限流
     * @param key 限流key
     * @param capacity 桶容量
     * @param rate 每秒生成令牌数
     */
    public boolean acquireToken(String key, int capacity, int rate) {
        return acquireTokens(key, capacity, rate, 1);
    }

    public boolean acquireTokens(String key, int capacity, int rate, int tokens) {
        RedisScript<Long> script = new DefaultRedisScript<>(
                TOKEN_BUCKET_SCRIPT, Long.class);

        Long result = redisTemplate.execute(
                script,
                Collections.singletonList("rate_limit:" + key),
                String.valueOf(capacity),
                String.valueOf(rate),
                String.valueOf(System.currentTimeMillis() / 1000),
                String.valueOf(tokens)
        );

        return result != null && result == 1L;
    }
}

// ==================== 3. 限流注解和切面 ====================

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface RateLimit {
    /**
     * 限流key前缀
     */
    String key() default "";

    /**
     * 时间窗口(秒)
     */
    int window() default 1;

    /**
     * 限制次数
     */
    int limit() default 100;

    /**
     * 限流类型
     */
    LimitType type() default LimitType.DEFAULT;

    /**
     * 限流提示信息
     */
    String message() default "请求过于频繁，请稍后再试";
}

public enum LimitType {
    DEFAULT,    // 默认按方法限流
    IP,         // 按IP限流
    USER,       // 按用户限流
    CUSTOM      // 自定义
}

@Aspect
@Component
@Slf4j
public class RateLimitAspect {

    @Autowired
    private RedisRateLimiter rateLimiter;

    @Autowired
    private HttpServletRequest request;

    @Around("@annotation(rateLimit)")
    public Object around(ProceedingJoinPoint point, RateLimit rateLimit)
            throws Throwable {

        // 构建限流key
        String limitKey = buildLimitKey(point, rateLimit);

        // 判断是否允许通过
        boolean allowed = rateLimiter.isAllowed(
                limitKey,
                rateLimit.window() * 1000L,
                rateLimit.limit()
        );

        if (!allowed) {
            log.warn("触发限流: key={}, limit={}/{}",
                    limitKey, rateLimit.limit(), rateLimit.window());
            throw new RateLimitException(rateLimit.message());
        }

        return point.proceed();
    }

    private String buildLimitKey(ProceedingJoinPoint point, RateLimit rateLimit) {
        StringBuilder key = new StringBuilder("rate_limit:");

        // 添加前缀
        if (StringUtils.isNotBlank(rateLimit.key())) {
            key.append(rateLimit.key());
        } else {
            key.append(point.getSignature().getDeclaringTypeName())
               .append(".")
               .append(point.getSignature().getName());
        }

        // 根据限流类型添加后缀
        switch (rateLimit.type()) {
            case IP:
                key.append(":").append(getClientIP());
                break;
            case USER:
                key.append(":").append(getCurrentUserId());
                break;
            case CUSTOM:
                // 从参数中获取自定义key
                key.append(":").append(getCustomKey(point));
                break;
            default:
                break;
        }

        return key.toString();
    }

    private String getClientIP() {
        String ip = request.getHeader("X-Forwarded-For");
        if (ip == null || ip.isEmpty()) {
            ip = request.getRemoteAddr();
        }
        return ip.split(",")[0].trim();
    }

    private String getCurrentUserId() {
        // 从SecurityContext获取当前用户ID
        return SecurityContextHolder.getContext()
                .getAuthentication().getName();
    }
}

// ==================== 4. 使用示例 ====================

@RestController
@RequestMapping("/api")
public class ApiController {

    /**
     * 全局限流: 每秒最多100次请求
     */
    @GetMapping("/list")
    @RateLimit(limit = 100, window = 1)
    public Result<List<Data>> list() {
        return Result.success(dataService.list());
    }

    /**
     * IP限流: 每个IP每分钟最多10次
     */
    @PostMapping("/submit")
    @RateLimit(limit = 10, window = 60, type = LimitType.IP,
               message = "提交过于频繁")
    public Result<Void> submit(@RequestBody Request request) {
        return Result.success();
    }

    /**
     * 用户限流: 每个用户每秒最多5次
     */
    @GetMapping("/user/data")
    @RateLimit(limit = 5, window = 1, type = LimitType.USER)
    public Result<UserData> getUserData() {
        return Result.success(userService.getData());
    }
}
```

#### 多级限流架构

```
┌─────────────────────────────────────────────────────────────────┐
│                      多级限流架构                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 第1层: 接入层限流 (Nginx/OpenResty)                      │   │
│  │                                                         │   │
│  │ • 基于IP的限流                                          │   │
│  │ • 基于请求特征的限流                                    │   │
│  │ • 防止DDoS攻击                                          │   │
│  └───────────────────────┬─────────────────────────────────┘   │
│                          ▼                                      │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 第2层: 网关层限流 (Gateway/Sentinel)                    │   │
│  │                                                         │   │
│  │ • 基于路由的限流                                        │   │
│  │ • 基于用户/租户的限流                                   │   │
│  │ • 热点参数限流                                          │   │
│  └───────────────────────┬─────────────────────────────────┘   │
│                          ▼                                      │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 第3层: 服务层限流 (业务代码)                             │   │
│  │                                                         │   │
│  │ • 基于方法的限流                                        │   │
│  │ • 基于资源的限流                                        │   │
│  │ • 细粒度业务限流                                        │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

### 191. 超时和重试如何设计？

```
┌─────────────────────────────────────────────────────────────────┐
│                    超时重试设计原则                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    超时设置原则                           │   │
│  │                                                         │   │
│  │  1. 分层超时                                             │   │
│  │     客户端 > 网关 > 服务A > 服务B > 数据库               │   │
│  │       5s  >  4s  >  3s  >  2s  >  1s                    │   │
│  │                                                         │   │
│  │  2. 超时时间 = 99线响应时间 × 系数(1.5~2)                │   │
│  │                                                         │   │
│  │  3. 连接超时 < 读取超时                                  │   │
│  │     connTimeout=1s, readTimeout=3s                      │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    重试设置原则                           │   │
│  │                                                         │   │
│  │  1. 只重试幂等操作 (GET/PUT/DELETE)                      │   │
│  │                                                         │   │
│  │  2. 只重试可重试异常                                     │   │
│  │     ✓ 连接超时、读取超时                                 │   │
│  │     ✓ 503 Service Unavailable                           │   │
│  │     ✗ 400 Bad Request                                   │   │
│  │     ✗ 业务异常                                          │   │
│  │                                                         │   │
│  │  3. 重试次数: 一般2-3次                                  │   │
│  │                                                         │   │
│  │  4. 重试间隔: 指数退避 + 随机抖动                        │   │
│  │     delay = baseDelay × 2^(retryCount) × (1 + random)   │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    重试时序图                             │   │
│  │                                                         │   │
│  │  请求 ──▶ 失败 ──▶ 等1s ──▶ 重试1 ──▶ 失败              │   │
│  │                           ──▶ 等2s ──▶ 重试2 ──▶ 成功   │   │
│  │                                                         │   │
│  │  总耗时 = 原请求时间 + 等待时间 + 重试时间               │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### 超时重试实现

```java
/**
 * 超时重试实现
 */

// ==================== 1. HTTP客户端超时配置 ====================

@Configuration
public class RestTemplateConfig {

    @Bean
    public RestTemplate restTemplate() {
        // 连接工厂配置
        HttpComponentsClientHttpRequestFactory factory =
                new HttpComponentsClientHttpRequestFactory();

        // 超时设置
        factory.setConnectTimeout(3000);       // 连接超时3秒
        factory.setReadTimeout(10000);         // 读取超时10秒
        factory.setConnectionRequestTimeout(3000); // 从连接池获取连接超时

        return new RestTemplate(factory);
    }

    @Bean
    public OkHttpClient okHttpClient() {
        return new OkHttpClient.Builder()
                .connectTimeout(3, TimeUnit.SECONDS)
                .readTimeout(10, TimeUnit.SECONDS)
                .writeTimeout(10, TimeUnit.SECONDS)
                .retryOnConnectionFailure(true)
                .build();
    }
}

// ==================== 2. Feign超时重试配置 ====================

// application.yml
/*
feign:
  client:
    config:
      default:
        connectTimeout: 3000
        readTimeout: 10000
      user-service:  # 针对特定服务
        connectTimeout: 2000
        readTimeout: 5000

# Ribbon重试配置
ribbon:
  ConnectTimeout: 3000
  ReadTimeout: 10000
  MaxAutoRetries: 1              # 同一实例最大重试次数
  MaxAutoRetriesNextServer: 2    # 切换实例重试次数
  OkToRetryOnAllOperations: false # 只重试GET请求
*/

// ==================== 3. 自定义重试器 ====================

@Component
@Slf4j
public class RetryExecutor {

    /**
     * 带指数退避的重试执行器
     */
    public <T> T executeWithRetry(
            Supplier<T> supplier,
            RetryConfig config) throws RetryExhaustedException {

        int retryCount = 0;
        Exception lastException = null;

        while (retryCount <= config.getMaxRetries()) {
            try {
                return supplier.get();
            } catch (Exception e) {
                lastException = e;

                // 判断是否可重试
                if (!isRetryable(e, config)) {
                    throw new RetryExhaustedException("不可重试的异常", e);
                }

                retryCount++;

                if (retryCount <= config.getMaxRetries()) {
                    // 计算等待时间 (指数退避 + 随机抖动)
                    long delay = calculateDelay(retryCount, config);

                    log.warn("第{}次重试失败，{}ms后重试。异常: {}",
                            retryCount, delay, e.getMessage());

                    try {
                        Thread.sleep(delay);
                    } catch (InterruptedException ie) {
                        Thread.currentThread().interrupt();
                        throw new RetryExhaustedException("重试被中断", ie);
                    }
                }
            }
        }

        throw new RetryExhaustedException(
                "重试次数耗尽: " + config.getMaxRetries(), lastException);
    }

    /**
     * 判断是否可重试
     */
    private boolean isRetryable(Exception e, RetryConfig config) {
        // 检查是否在可重试异常列表中
        for (Class<? extends Exception> retryableEx : config.getRetryableExceptions()) {
            if (retryableEx.isInstance(e)) {
                return true;
            }
        }

        // 检查HTTP状态码
        if (e instanceof HttpStatusCodeException) {
            int statusCode = ((HttpStatusCodeException) e).getRawStatusCode();
            return config.getRetryableStatusCodes().contains(statusCode);
        }

        return false;
    }

    /**
     * 计算重试延迟 (指数退避 + 随机抖动)
     */
    private long calculateDelay(int retryCount, RetryConfig config) {
        // 指数退避: baseDelay * 2^(retryCount-1)
        long delay = config.getBaseDelay() * (1L << (retryCount - 1));

        // 最大延迟限制
        delay = Math.min(delay, config.getMaxDelay());

        // 随机抖动 (避免惊群效应): delay * (1 + random(0, jitter))
        if (config.getJitter() > 0) {
            double jitter = ThreadLocalRandom.current()
                    .nextDouble(0, config.getJitter());
            delay = (long) (delay * (1 + jitter));
        }

        return delay;
    }
}

@Data
@Builder
public class RetryConfig {
    @Builder.Default
    private int maxRetries = 3;

    @Builder.Default
    private long baseDelay = 1000; // 基础延迟1秒

    @Builder.Default
    private long maxDelay = 30000; // 最大延迟30秒

    @Builder.Default
    private double jitter = 0.2; // 20%随机抖动

    @Builder.Default
    private Set<Class<? extends Exception>> retryableExceptions =
            Set.of(SocketTimeoutException.class,
                   ConnectTimeoutException.class,
                   IOException.class);

    @Builder.Default
    private Set<Integer> retryableStatusCodes =
            Set.of(502, 503, 504);
}

// ==================== 4. Spring Retry注解方式 ====================

@Configuration
@EnableRetry
public class RetryConfiguration {

    @Bean
    public RetryTemplate retryTemplate() {
        RetryTemplate retryTemplate = new RetryTemplate();

        // 重试策略: 最多重试3次，只重试特定异常
        Map<Class<? extends Throwable>, Boolean> retryableExceptions =
                new HashMap<>();
        retryableExceptions.put(RemoteServiceException.class, true);
        retryableExceptions.put(TimeoutException.class, true);
        retryableExceptions.put(IOException.class, true);

        SimpleRetryPolicy retryPolicy = new SimpleRetryPolicy(3,
                retryableExceptions, true);

        // 退避策略: 指数退避
        ExponentialBackOffPolicy backOffPolicy = new ExponentialBackOffPolicy();
        backOffPolicy.setInitialInterval(1000);  // 初始间隔1秒
        backOffPolicy.setMultiplier(2.0);        // 倍数
        backOffPolicy.setMaxInterval(30000);     // 最大间隔30秒

        retryTemplate.setRetryPolicy(retryPolicy);
        retryTemplate.setBackOffPolicy(backOffPolicy);

        // 重试监听器
        retryTemplate.registerListener(new RetryListenerSupport() {
            @Override
            public <T, E extends Throwable> void onError(
                    RetryContext context, RetryCallback<T, E> callback,
                    Throwable throwable) {
                log.warn("重试第{}次失败: {}",
                        context.getRetryCount(), throwable.getMessage());
            }
        });

        return retryTemplate;
    }
}

@Service
public class RemoteService {

    /**
     * 使用@Retryable注解
     */
    @Retryable(
            value = {RemoteServiceException.class, TimeoutException.class},
            maxAttempts = 3,
            backoff = @Backoff(delay = 1000, multiplier = 2, maxDelay = 10000)
    )
    public String callRemoteService(String param) {
        return httpClient.call(param);
    }

    /**
     * 重试失败后的恢复方法
     */
    @Recover
    public String recover(RemoteServiceException e, String param) {
        log.error("远程服务调用失败，进入降级: param={}", param, e);
        return getDefaultValue(param);
    }
}

// ==================== 5. 幂等性保证 ====================

@Service
public class IdempotentService {

    @Autowired
    private StringRedisTemplate redisTemplate;

    /**
     * 幂等性调用封装
     */
    public <T> T executeIdempotent(
            String idempotentKey,
            Supplier<T> supplier,
            Duration expireTime) {

        String lockKey = "idempotent:" + idempotentKey;
        String resultKey = "idempotent:result:" + idempotentKey;

        // 1. 检查是否已有结果
        String cachedResult = redisTemplate.opsForValue().get(resultKey);
        if (cachedResult != null) {
            return JSON.parseObject(cachedResult, new TypeReference<T>(){});
        }

        // 2. 尝试获取锁
        Boolean locked = redisTemplate.opsForValue()
                .setIfAbsent(lockKey, "1", expireTime);

        if (Boolean.FALSE.equals(locked)) {
            // 正在处理中，等待结果
            return waitForResult(resultKey, expireTime);
        }

        try {
            // 3. 执行业务逻辑
            T result = supplier.get();

            // 4. 缓存结果
            redisTemplate.opsForValue().set(resultKey,
                    JSON.toJSONString(result), expireTime);

            return result;
        } finally {
            redisTemplate.delete(lockKey);
        }
    }
}
```

---

### 192. 灰度发布如何实现？

```
┌─────────────────────────────────────────────────────────────────┐
│                      灰度发布方案对比                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 1. 蓝绿部署                                              │   │
│  │                                                         │   │
│  │    ┌─────────┐         ┌─────────┐                      │   │
│  │    │  蓝环境  │ ◀─100%─ │ 负载均衡 │                      │   │
│  │    │  (旧版) │         │         │                      │   │
│  │    └─────────┘         └─────────┘                      │   │
│  │    ┌─────────┐              │                           │   │
│  │    │  绿环境  │ ◀────0%─────┘                           │   │
│  │    │  (新版) │                                          │   │
│  │    └─────────┘                                          │   │
│  │                                                         │   │
│  │    切换: 100%流量从蓝切到绿                              │   │
│  │    优点: 切换快速，回滚简单                              │   │
│  │    缺点: 需要双倍资源                                    │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 2. 金丝雀发布 (灰度发布)                                 │   │
│  │                                                         │   │
│  │              ┌─────────┐                                │   │
│  │         95%  │  V1.0   │                                │   │
│  │    ┌────────▶│ (旧版)  │                                │   │
│  │    │         └─────────┘                                │   │
│  │  ┌─┴──────┐                                             │   │
│  │  │负载均衡│                                              │   │
│  │  └─┬──────┘  ┌─────────┐                                │   │
│  │    │    5%   │  V2.0   │                                │   │
│  │    └────────▶│ (新版)  │                                │   │
│  │              └─────────┘                                │   │
│  │                                                         │   │
│  │    逐步放量: 5% → 10% → 30% → 50% → 100%                │   │
│  │    优点: 风险可控，可观察                                │   │
│  │    缺点: 发布周期长                                      │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 3. A/B测试                                               │   │
│  │                                                         │   │
│  │    用户特征 ──▶ 分组规则 ──▶ 路由到不同版本              │   │
│  │                                                         │   │
│  │    • 按用户ID分组                                       │   │
│  │    • 按地域分组                                         │   │
│  │    • 按设备类型分组                                     │   │
│  │    • 按用户标签分组                                     │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### 灰度发布实现

```java
/**
 * 灰度发布实现
 */

// ==================== 1. 灰度规则定义 ====================

@Data
public class GrayRule {
    private String id;
    private String serviceName;
    private String targetVersion;
    private boolean enabled;
    private int weight;  // 权重(0-100)
    private List<GrayCondition> conditions;  // 灰度条件
}

@Data
public class GrayCondition {
    private String field;      // 字段名: userId, ip, header.xxx
    private String operator;   // 操作符: equals, in, regex, mod
    private String value;      // 匹配值
}

// ==================== 2. 灰度路由实现 ====================

@Component
@Slf4j
public class GrayRouter {

    @Autowired
    private GrayRuleRepository ruleRepository;

    /**
     * 判断是否路由到灰度版本
     */
    public boolean shouldRouteToGray(String serviceName,
                                     GrayContext context) {
        // 1. 获取灰度规则
        GrayRule rule = ruleRepository.getRule(serviceName);
        if (rule == null || !rule.isEnabled()) {
            return false;
        }

        // 2. 检查条件匹配
        if (matchConditions(rule.getConditions(), context)) {
            return true;
        }

        // 3. 按权重随机
        if (rule.getWeight() > 0) {
            int random = ThreadLocalRandom.current().nextInt(100);
            return random < rule.getWeight();
        }

        return false;
    }

    /**
     * 条件匹配
     */
    private boolean matchConditions(List<GrayCondition> conditions,
                                   GrayContext context) {
        if (conditions == null || conditions.isEmpty()) {
            return false;
        }

        for (GrayCondition condition : conditions) {
            if (!matchCondition(condition, context)) {
                return false;  // AND关系
            }
        }
        return true;
    }

    private boolean matchCondition(GrayCondition condition,
                                   GrayContext context) {
        String actualValue = getFieldValue(condition.getField(), context);
        if (actualValue == null) {
            return false;
        }

        switch (condition.getOperator()) {
            case "equals":
                return condition.getValue().equals(actualValue);
            case "in":
                return Arrays.asList(condition.getValue().split(","))
                        .contains(actualValue);
            case "regex":
                return actualValue.matches(condition.getValue());
            case "mod":
                // userId % 100 < 10 (10%的用户)
                String[] parts = condition.getValue().split(",");
                int divisor = Integer.parseInt(parts[0]);
                int threshold = Integer.parseInt(parts[1]);
                return Math.abs(actualValue.hashCode()) % divisor < threshold;
            default:
                return false;
        }
    }

    private String getFieldValue(String field, GrayContext context) {
        if (field.startsWith("header.")) {
            return context.getHeader(field.substring(7));
        }
        switch (field) {
            case "userId":
                return context.getUserId();
            case "ip":
                return context.getClientIp();
            case "deviceType":
                return context.getDeviceType();
            default:
                return context.getAttribute(field);
        }
    }
}

// ==================== 3. Gateway灰度路由Filter ====================

@Component
@Slf4j
public class GrayLoadBalancerFilter implements GlobalFilter, Ordered {

    @Autowired
    private GrayRouter grayRouter;

    @Autowired
    private DiscoveryClient discoveryClient;

    @Override
    public Mono<Void> filter(ServerWebExchange exchange,
                             GatewayFilterChain chain) {

        URI uri = exchange.getAttribute(GATEWAY_REQUEST_URL_ATTR);
        String serviceName = uri.getHost();

        // 构建灰度上下文
        GrayContext context = buildGrayContext(exchange);

        // 判断是否路由到灰度版本
        boolean routeToGray = grayRouter.shouldRouteToGray(serviceName, context);

        // 获取目标实例
        List<ServiceInstance> instances = discoveryClient
                .getInstances(serviceName);

        ServiceInstance targetInstance;
        if (routeToGray) {
            // 选择灰度版本实例
            targetInstance = selectGrayInstance(instances);
            log.info("路由到灰度版本: {}", targetInstance.getUri());
        } else {
            // 选择正常版本实例
            targetInstance = selectNormalInstance(instances);
        }

        if (targetInstance == null) {
            return Mono.error(new NotFoundException("No available instance"));
        }

        // 重写请求URI
        URI targetUri = buildTargetUri(uri, targetInstance);
        exchange.getAttributes().put(GATEWAY_REQUEST_URL_ATTR, targetUri);

        // 添加标记头
        exchange.getRequest().mutate()
                .header("X-Gray-Version", routeToGray ? "gray" : "normal")
                .build();

        return chain.filter(exchange);
    }

    private ServiceInstance selectGrayInstance(List<ServiceInstance> instances) {
        return instances.stream()
                .filter(instance -> "gray".equals(
                        instance.getMetadata().get("version")))
                .findFirst()
                .orElse(null);
    }

    private ServiceInstance selectNormalInstance(List<ServiceInstance> instances) {
        List<ServiceInstance> normalInstances = instances.stream()
                .filter(instance -> !"gray".equals(
                        instance.getMetadata().get("version")))
                .collect(Collectors.toList());

        if (normalInstances.isEmpty()) {
            return null;
        }

        // 简单轮询选择
        int index = ThreadLocalRandom.current().nextInt(normalInstances.size());
        return normalInstances.get(index);
    }

    @Override
    public int getOrder() {
        return LOWEST_PRECEDENCE - 10;
    }
}

// ==================== 4. Nacos灰度实例注册 ====================

// bootstrap.yml
/*
spring:
  cloud:
    nacos:
      discovery:
        metadata:
          version: gray  # 或 normal
*/

// ==================== 5. 灰度规则管理接口 ====================

@RestController
@RequestMapping("/admin/gray")
public class GrayRuleController {

    @Autowired
    private GrayRuleRepository ruleRepository;

    /**
     * 创建灰度规则
     */
    @PostMapping("/rules")
    public Result<GrayRule> createRule(@RequestBody GrayRule rule) {
        rule.setId(UUID.randomUUID().toString());
        ruleRepository.save(rule);
        return Result.success(rule);
    }

    /**
     * 更新灰度权重
     */
    @PutMapping("/rules/{id}/weight")
    public Result<Void> updateWeight(
            @PathVariable String id,
            @RequestParam int weight) {
        GrayRule rule = ruleRepository.getById(id);
        rule.setWeight(weight);
        ruleRepository.save(rule);
        return Result.success();
    }

    /**
     * 开启/关闭灰度
     */
    @PutMapping("/rules/{id}/toggle")
    public Result<Void> toggle(
            @PathVariable String id,
            @RequestParam boolean enabled) {
        GrayRule rule = ruleRepository.getById(id);
        rule.setEnabled(enabled);
        ruleRepository.save(rule);
        return Result.success();
    }
}
```

---

### 193. 容灾和异地多活如何设计？

```
┌─────────────────────────────────────────────────────────────────┐
│                    异地多活架构设计                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    全局流量调度层                         │   │
│  │              (DNS + GSLB + CDN)                          │   │
│  │                                                         │   │
│  │    用户 ──▶ DNS ──▶ 就近机房                             │   │
│  │                                                         │   │
│  └─────────────────────────────────────────────────────────┘   │
│              │                         │                        │
│              ▼                         ▼                        │
│  ┌───────────────────────┐ ┌───────────────────────┐           │
│  │      北京机房          │ │      上海机房          │           │
│  │                       │ │                       │           │
│  │  ┌─────────────────┐  │ │  ┌─────────────────┐  │           │
│  │  │   接入层        │  │ │  │   接入层        │  │           │
│  │  │ (Nginx + LB)    │  │ │  │ (Nginx + LB)    │  │           │
│  │  └────────┬────────┘  │ │  └────────┬────────┘  │           │
│  │           │           │ │           │           │           │
│  │  ┌────────┴────────┐  │ │  ┌────────┴────────┐  │           │
│  │  │   服务集群       │  │ │  │   服务集群       │  │           │
│  │  │ (全量服务)       │  │ │  │ (全量服务)       │  │           │
│  │  └────────┬────────┘  │ │  └────────┬────────┘  │           │
│  │           │           │ │           │           │           │
│  │  ┌────────┴────────┐  │ │  ┌────────┴────────┐  │           │
│  │  │   数据层        │  │ │  │   数据层        │  │           │
│  │  │ MySQL + Redis   │  │ │  │ MySQL + Redis   │  │           │
│  │  └────────┬────────┘  │ │  └────────┬────────┘  │           │
│  │           │           │ │           │           │           │
│  └───────────┼───────────┘ └───────────┼───────────┘           │
│              │                         │                        │
│              └─────────┬───────────────┘                        │
│                        │                                        │
│              ┌─────────┴─────────┐                              │
│              │   数据同步中心     │                              │
│              │  (双向同步/冲突处理) │                              │
│              └───────────────────┘                              │
│                                                                 │
│  数据分片策略:                                                   │
│  • 用户维度分片: 用户ID hash决定归属机房                         │
│  • 地域分片: 北方用户→北京，南方用户→上海                        │
│  • 单元化: 按业务单元划分，单元内自闭环                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### 异地多活实现

```java
/**
 * 异地多活核心实现
 */

// ==================== 1. 路由规则 ====================

@Component
public class UnitRouter {

    /**
     * 根据用户ID确定归属单元
     */
    public String getUnitId(String userId) {
        // 取用户ID的hash值，对单元数取模
        int hash = Math.abs(userId.hashCode());
        int unitCount = 2; // 两个单元: BJ, SH
        int unitIndex = hash % unitCount;
        return unitIndex == 0 ? "BJ" : "SH";
    }

    /**
     * 判断请求是否在本单元处理
     */
    public boolean isLocalUnit(String userId) {
        String targetUnit = getUnitId(userId);
        String localUnit = getLocalUnit();
        return targetUnit.equals(localUnit);
    }

    /**
     * 获取本机所在单元
     */
    private String getLocalUnit() {
        return System.getenv("UNIT_ID"); // 从环境变量获取
    }
}

// ==================== 2. 跨单元路由 ====================

@Component
@Slf4j
public class CrossUnitService {

    @Autowired
    private UnitRouter unitRouter;

    @Autowired
    private RestTemplate restTemplate;

    private final Map<String, String> unitEndpoints = Map.of(
            "BJ", "http://api.bj.example.com",
            "SH", "http://api.sh.example.com"
    );

    /**
     * 跨单元调用
     */
    public <T> T callCrossUnit(String userId, String path,
                                Class<T> responseType, Object... params) {
        String targetUnit = unitRouter.getUnitId(userId);
        String localUnit = unitRouter.getLocalUnit();

        if (targetUnit.equals(localUnit)) {
            // 本单元直接调用
            return localCall(path, responseType, params);
        } else {
            // 跨单元调用
            String endpoint = unitEndpoints.get(targetUnit);
            String url = endpoint + path;

            log.info("跨单元调用: {} -> {}, url={}",
                    localUnit, targetUnit, url);

            return restTemplate.getForObject(url, responseType, params);
        }
    }
}

// ==================== 3. 数据同步 ====================

/**
 * 基于binlog的数据同步
 */
@Component
@Slf4j
public class DataSyncService {

    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate;

    @Autowired
    private StringRedisTemplate redisTemplate;

    /**
     * 发送数据变更事件 (生产端)
     */
    public void publishDataChange(DataChangeEvent event) {
        String topic = "data-sync-" + event.getTableName();
        String key = event.getPrimaryKey();
        String message = JSON.toJSONString(event);

        // 添加幂等key
        String idempotentKey = "sync:" + event.getTableName() + ":" +
                              event.getPrimaryKey() + ":" + event.getTimestamp();
        event.setIdempotentKey(idempotentKey);

        kafkaTemplate.send(topic, key, message)
                .addCallback(
                    result -> log.debug("同步消息发送成功: {}", key),
                    ex -> log.error("同步消息发送失败: {}", key, ex)
                );
    }

    /**
     * 消费数据变更事件 (消费端)
     */
    @KafkaListener(topicPattern = "data-sync-.*")
    public void consumeDataChange(ConsumerRecord<String, String> record) {
        DataChangeEvent event = JSON.parseObject(
                record.value(), DataChangeEvent.class);

        // 幂等性检查
        String idempotentKey = event.getIdempotentKey();
        Boolean isNew = redisTemplate.opsForValue()
                .setIfAbsent(idempotentKey, "1", Duration.ofDays(1));

        if (Boolean.FALSE.equals(isNew)) {
            log.info("重复事件，跳过: {}", idempotentKey);
            return;
        }

        try {
            // 应用数据变更
            applyDataChange(event);
        } catch (Exception e) {
            log.error("数据同步失败: {}", event, e);
            // 删除幂等key，允许重试
            redisTemplate.delete(idempotentKey);
            throw e;
        }
    }

    /**
     * 冲突检测与处理
     */
    private void applyDataChange(DataChangeEvent event) {
        // 获取本地数据
        Object localData = getLocalData(event.getTableName(),
                                        event.getPrimaryKey());

        if (localData != null) {
            long localTimestamp = getTimestamp(localData);

            // 时间戳冲突检测
            if (event.getTimestamp() <= localTimestamp) {
                log.warn("数据冲突，本地数据更新: local={}, remote={}",
                        localTimestamp, event.getTimestamp());

                // 冲突处理策略: Last Writer Wins
                // 或发送到冲突队列人工处理
                return;
            }
        }

        // 应用远程数据
        saveLocalData(event);
    }
}

@Data
public class DataChangeEvent {
    private String tableName;
    private String primaryKey;
    private String operation;  // INSERT/UPDATE/DELETE
    private Map<String, Object> beforeData;
    private Map<String, Object> afterData;
    private long timestamp;
    private String sourceUnit;
    private String idempotentKey;
}

// ==================== 4. 故障切换 ====================

@Service
@Slf4j
public class FailoverService {

    @Autowired
    private StringRedisTemplate redisTemplate;

    private static final String FAILOVER_KEY = "failover:status";

    /**
     * 检测机房健康状态
     */
    @Scheduled(fixedRate = 5000)
    public void healthCheck() {
        Map<String, Boolean> unitStatus = new HashMap<>();

        for (String unit : Arrays.asList("BJ", "SH")) {
            boolean healthy = checkUnitHealth(unit);
            unitStatus.put(unit, healthy);

            if (!healthy) {
                log.warn("机房{}健康检查失败", unit);
                triggerFailover(unit);
            }
        }

        // 更新状态到Redis
        redisTemplate.opsForHash().putAll(FAILOVER_KEY,
                unitStatus.entrySet().stream()
                        .collect(Collectors.toMap(
                                Map.Entry::getKey,
                                e -> e.getValue().toString())));
    }

    /**
     * 触发故障切换
     */
    private void triggerFailover(String failedUnit) {
        log.error("触发故障切换: {}", failedUnit);

        // 1. 更新DNS，将流量切到其他机房
        updateDns(failedUnit);

        // 2. 通知相关服务
        sendFailoverNotification(failedUnit);

        // 3. 记录故障切换日志
        recordFailoverLog(failedUnit);
    }

    /**
     * 判断是否处于故障切换状态
     */
    public boolean isFailoverActive(String unit) {
        Object status = redisTemplate.opsForHash().get(FAILOVER_KEY, unit);
        return "false".equals(status);
    }
}
```

---

### 194. 监控告警系统如何设计？

```
┌─────────────────────────────────────────────────────────────────┐
│                    监控告警系统架构                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                     采集层                                │   │
│  │                                                         │   │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐       │   │
│  │  │应用指标 │ │系统指标 │ │业务指标 │ │日志数据 │       │   │
│  │  │Micrometer│ │Node Exp │ │ 埋点   │ │Filebeat│       │   │
│  │  └────┬────┘ └────┬────┘ └────┬────┘ └────┬────┘       │   │
│  │       │          │          │          │              │   │
│  └───────┴──────────┴──────────┴──────────┴──────────────┘   │
│                         │                                      │
│                         ▼                                      │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                     存储层                                │   │
│  │                                                         │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │   │
│  │  │ Prometheus   │  │    Kafka     │  │Elasticsearch│  │   │
│  │  │  (时序指标)   │  │  (日志流)    │  │  (日志存储)  │  │   │
│  │  └──────────────┘  └──────────────┘  └──────────────┘  │   │
│  │                                                         │   │
│  └─────────────────────────────────────────────────────────┘   │
│                         │                                      │
│                         ▼                                      │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                     分析层                                │   │
│  │                                                         │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │   │
│  │  │ AlertManager │  │  告警规则    │  │  异常检测    │  │   │
│  │  │  (告警路由)   │  │  (阈值判断)  │  │  (AI算法)   │  │   │
│  │  └──────────────┘  └──────────────┘  └──────────────┘  │   │
│  │                                                         │   │
│  └─────────────────────────────────────────────────────────┘   │
│                         │                                      │
│                         ▼                                      │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                     展示层                                │   │
│  │                                                         │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │   │
│  │  │   Grafana    │  │   告警通知   │  │   值班系统   │  │   │
│  │  │  (可视化)    │  │ 钉钉/企微/短信│  │  (PagerDuty) │  │   │
│  │  └──────────────┘  └──────────────┘  └──────────────┘  │   │
│  │                                                         │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### 监控告警实现

```java
/**
 * 监控告警实现
 */

// ==================== 1. 指标采集 ====================

@Configuration
public class MetricsConfig {

    @Bean
    public MeterRegistryCustomizer<MeterRegistry> metricsCommonTags() {
        return registry -> registry.config().commonTags(
                "application", "order-service",
                "env", System.getenv("ENV")
        );
    }
}

@Component
@Slf4j
public class BusinessMetrics {

    private final MeterRegistry meterRegistry;

    // 订单创建计数器
    private final Counter orderCreatedCounter;

    // 订单金额统计
    private final DistributionSummary orderAmountSummary;

    // 接口响应时间
    private final Timer apiTimer;

    // 当前处理中的订单数
    private final AtomicInteger processingOrders;

    public BusinessMetrics(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;

        // 初始化指标
        this.orderCreatedCounter = Counter.builder("order.created.total")
                .description("创建订单总数")
                .tags("type", "normal")
                .register(meterRegistry);

        this.orderAmountSummary = DistributionSummary.builder("order.amount")
                .description("订单金额分布")
                .baseUnit("yuan")
                .publishPercentiles(0.5, 0.9, 0.99)
                .register(meterRegistry);

        this.apiTimer = Timer.builder("api.response.time")
                .description("API响应时间")
                .publishPercentiles(0.5, 0.9, 0.99)
                .register(meterRegistry);

        this.processingOrders = meterRegistry.gauge(
                "order.processing.count",
                new AtomicInteger(0)
        );
    }

    /**
     * 记录订单创建
     */
    public void recordOrderCreated(Order order) {
        orderCreatedCounter.increment();
        orderAmountSummary.record(order.getAmount().doubleValue());
    }

    /**
     * 记录API响应时间
     */
    public void recordApiTime(String api, long timeMs) {
        Timer.builder("api.response.time")
                .tag("api", api)
                .register(meterRegistry)
                .record(timeMs, TimeUnit.MILLISECONDS);
    }

    /**
     * 记录业务异常
     */
    public void recordBusinessError(String errorType) {
        meterRegistry.counter("business.error.total",
                "type", errorType).increment();
    }
}

// ==================== 2. 告警规则定义 ====================

@Data
public class AlertRule {
    private String id;
    private String name;
    private String metric;          // 指标名
    private String condition;       // 条件表达式
    private double threshold;       // 阈值
    private int duration;           // 持续时间(秒)
    private String severity;        // 严重级别: critical/warning/info
    private List<String> notifyChannels;  // 通知渠道
    private String template;        // 告警模板
    private boolean enabled;
}

// Prometheus告警规则 (prometheus-rules.yml)
/*
groups:
  - name: application
    rules:
      - alert: HighErrorRate
        expr: rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m]) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "高错误率告警"
          description: "服务{{ $labels.service }}错误率超过5%"

      - alert: HighResponseTime
        expr: histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m])) > 2
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "响应时间过长"
          description: "服务{{ $labels.service }} P99响应时间超过2秒"

      - alert: InstanceDown
        expr: up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "实例宕机"
          description: "实例{{ $labels.instance }}已宕机超过1分钟"
*/

// ==================== 3. 告警处理服务 ====================

@Service
@Slf4j
public class AlertService {

    @Autowired
    private AlertRuleRepository ruleRepository;

    @Autowired
    private NotificationService notificationService;

    @Autowired
    private StringRedisTemplate redisTemplate;

    /**
     * 处理告警
     */
    public void handleAlert(AlertEvent event) {
        log.warn("收到告警: {}", event);

        // 1. 告警去重 (相同告警5分钟内不重复发送)
        String dedupeKey = "alert:dedupe:" + event.getAlertId();
        Boolean isNew = redisTemplate.opsForValue()
                .setIfAbsent(dedupeKey, "1", Duration.ofMinutes(5));

        if (Boolean.FALSE.equals(isNew)) {
            log.info("告警去重，跳过: {}", event.getAlertId());
            return;
        }

        // 2. 告警分级
        AlertRule rule = ruleRepository.getById(event.getRuleId());

        // 3. 告警升级检查
        checkAlertEscalation(event, rule);

        // 4. 发送通知
        sendNotification(event, rule);

        // 5. 记录告警
        saveAlertRecord(event);
    }

    /**
     * 告警升级检查
     */
    private void checkAlertEscalation(AlertEvent event, AlertRule rule) {
        String escalationKey = "alert:escalation:" + event.getAlertId();
        Long count = redisTemplate.opsForValue().increment(escalationKey);
        redisTemplate.expire(escalationKey, Duration.ofHours(1));

        // 同一告警触发3次以上，升级处理
        if (count != null && count >= 3) {
            event.setSeverity("critical");
            log.warn("告警升级: {}", event.getAlertId());
        }
    }

    /**
     * 发送通知
     */
    private void sendNotification(AlertEvent event, AlertRule rule) {
        String message = buildAlertMessage(event, rule);

        for (String channel : rule.getNotifyChannels()) {
            try {
                switch (channel) {
                    case "dingtalk":
                        notificationService.sendDingTalk(message,
                                event.getSeverity().equals("critical"));
                        break;
                    case "wechat":
                        notificationService.sendWeChat(message);
                        break;
                    case "sms":
                        notificationService.sendSms(getOnCallPhone(), message);
                        break;
                    case "email":
                        notificationService.sendEmail(getOnCallEmail(),
                                "告警: " + event.getName(), message);
                        break;
                }
            } catch (Exception e) {
                log.error("告警通知发送失败: channel={}", channel, e);
            }
        }
    }

    private String buildAlertMessage(AlertEvent event, AlertRule rule) {
        return String.format(
                "【%s】%s\n" +
                "告警级别: %s\n" +
                "告警指标: %s\n" +
                "当前值: %s\n" +
                "阈值: %s\n" +
                "触发时间: %s\n" +
                "告警详情: %s",
                event.getSeverity().toUpperCase(),
                event.getName(),
                event.getSeverity(),
                event.getMetric(),
                event.getCurrentValue(),
                event.getThreshold(),
                formatTime(event.getTimestamp()),
                event.getDescription()
        );
    }
}

// ==================== 4. 通知服务实现 ====================

@Service
@Slf4j
public class NotificationService {

    @Autowired
    private RestTemplate restTemplate;

    @Value("${notification.dingtalk.webhook}")
    private String dingTalkWebhook;

    /**
     * 发送钉钉通知
     */
    public void sendDingTalk(String message, boolean atAll) {
        Map<String, Object> body = new HashMap<>();
        body.put("msgtype", "text");

        Map<String, Object> text = new HashMap<>();
        text.put("content", message);
        body.put("text", text);

        if (atAll) {
            Map<String, Object> at = new HashMap<>();
            at.put("isAtAll", true);
            body.put("at", at);
        }

        try {
            restTemplate.postForEntity(dingTalkWebhook, body, String.class);
        } catch (Exception e) {
            log.error("钉钉通知发送失败", e);
        }
    }

    /**
     * 发送短信
     */
    public void sendSms(String phone, String message) {
        // 调用短信服务API
        smsClient.send(phone, message);
    }
}
```

---

## 7.3 可扩展（6题）

### 195. 如何设计可扩展的系统？

```
┌─────────────────────────────────────────────────────────────────┐
│                    可扩展系统设计原则                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    分层架构                               │   │
│  │                                                         │   │
│  │    ┌─────────────────────────────────────────────────┐  │   │
│  │    │              表现层 (Controller)                  │  │   │
│  │    │         接口定义、参数校验、响应封装              │  │   │
│  │    └─────────────────────────────────────────────────┘  │   │
│  │                          │                              │   │
│  │    ┌─────────────────────────────────────────────────┐  │   │
│  │    │              应用层 (Service)                     │  │   │
│  │    │         业务编排、事务管理、权限控制              │  │   │
│  │    └─────────────────────────────────────────────────┘  │   │
│  │                          │                              │   │
│  │    ┌─────────────────────────────────────────────────┐  │   │
│  │    │              领域层 (Domain)                      │  │   │
│  │    │         核心业务逻辑、领域模型、领域服务           │  │   │
│  │    └─────────────────────────────────────────────────┘  │   │
│  │                          │                              │   │
│  │    ┌─────────────────────────────────────────────────┐  │   │
│  │    │              基础设施层 (Infrastructure)          │  │   │
│  │    │         数据访问、外部服务、消息队列              │  │   │
│  │    └─────────────────────────────────────────────────┘  │   │
│  │                                                         │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                   设计原则                                │   │
│  │                                                         │   │
│  │  • 开闭原则: 对扩展开放，对修改关闭                      │   │
│  │  • 依赖倒置: 依赖抽象而非具体实现                        │   │
│  │  • 单一职责: 每个模块只负责一件事                        │   │
│  │  • 接口隔离: 接口小而专一                               │   │
│  │  • 高内聚低耦合: 模块内部紧密，模块之间松散              │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### 可扩展设计实现

```java
/**
 * 可扩展系统设计示例
 */

// ==================== 1. 策略模式 - 支付方式扩展 ====================

/**
 * 支付策略接口
 */
public interface PaymentStrategy {
    /**
     * 获取支付方式类型
     */
    String getType();

    /**
     * 执行支付
     */
    PaymentResult pay(PaymentRequest request);

    /**
     * 查询支付结果
     */
    PaymentResult query(String paymentId);

    /**
     * 退款
     */
    RefundResult refund(RefundRequest request);
}

/**
 * 支付宝支付实现
 */
@Component
public class AlipayPaymentStrategy implements PaymentStrategy {

    @Override
    public String getType() {
        return "ALIPAY";
    }

    @Override
    public PaymentResult pay(PaymentRequest request) {
        // 支付宝支付逻辑
        return alipayClient.pay(request);
    }

    @Override
    public PaymentResult query(String paymentId) {
        return alipayClient.query(paymentId);
    }

    @Override
    public RefundResult refund(RefundRequest request) {
        return alipayClient.refund(request);
    }
}

/**
 * 微信支付实现
 */
@Component
public class WechatPaymentStrategy implements PaymentStrategy {

    @Override
    public String getType() {
        return "WECHAT";
    }

    // 微信支付实现...
}

/**
 * 策略工厂 - 自动注册
 */
@Component
public class PaymentStrategyFactory {

    private final Map<String, PaymentStrategy> strategyMap =
            new ConcurrentHashMap<>();

    @Autowired
    public PaymentStrategyFactory(List<PaymentStrategy> strategies) {
        for (PaymentStrategy strategy : strategies) {
            strategyMap.put(strategy.getType(), strategy);
        }
    }

    public PaymentStrategy getStrategy(String type) {
        PaymentStrategy strategy = strategyMap.get(type);
        if (strategy == null) {
            throw new IllegalArgumentException("不支持的支付方式: " + type);
        }
        return strategy;
    }

    /**
     * 动态注册新的支付策略
     */
    public void registerStrategy(PaymentStrategy strategy) {
        strategyMap.put(strategy.getType(), strategy);
    }
}

// ==================== 2. 责任链模式 - 订单处理流程 ====================

/**
 * 订单处理器接口
 */
public interface OrderHandler {
    /**
     * 处理订单
     * @return true继续执行后续处理器，false终止
     */
    boolean handle(OrderContext context);

    /**
     * 获取处理器顺序
     */
    int getOrder();
}

/**
 * 参数校验处理器
 */
@Component
@Order(1)
public class ValidationHandler implements OrderHandler {

    @Override
    public boolean handle(OrderContext context) {
        Order order = context.getOrder();

        // 参数校验
        if (order.getAmount() == null || order.getAmount().compareTo(BigDecimal.ZERO) <= 0) {
            context.setError("订单金额无效");
            return false;
        }

        return true;
    }

    @Override
    public int getOrder() {
        return 1;
    }
}

/**
 * 库存检查处理器
 */
@Component
@Order(2)
public class InventoryHandler implements OrderHandler {

    @Autowired
    private InventoryService inventoryService;

    @Override
    public boolean handle(OrderContext context) {
        Order order = context.getOrder();

        // 检查并锁定库存
        boolean locked = inventoryService.lockStock(
                order.getProductId(), order.getQuantity());

        if (!locked) {
            context.setError("库存不足");
            return false;
        }

        context.setStockLocked(true);
        return true;
    }

    @Override
    public int getOrder() {
        return 2;
    }
}

/**
 * 订单处理链
 */
@Component
public class OrderHandlerChain {

    private final List<OrderHandler> handlers;

    @Autowired
    public OrderHandlerChain(List<OrderHandler> handlers) {
        // 按order排序
        this.handlers = handlers.stream()
                .sorted(Comparator.comparingInt(OrderHandler::getOrder))
                .collect(Collectors.toList());
    }

    public OrderResult process(Order order) {
        OrderContext context = new OrderContext(order);

        for (OrderHandler handler : handlers) {
            try {
                boolean continueProcess = handler.handle(context);
                if (!continueProcess) {
                    // 处理失败，执行回滚
                    rollback(context);
                    return OrderResult.fail(context.getError());
                }
            } catch (Exception e) {
                log.error("订单处理异常", e);
                rollback(context);
                return OrderResult.fail("系统异常");
            }
        }

        return OrderResult.success(context.getOrder());
    }
}

// ==================== 3. 插件化设计 - 动态加载 ====================

/**
 * 插件接口
 */
public interface Plugin {
    String getId();
    String getName();
    void init();
    void execute(PluginContext context);
    void destroy();
}

/**
 * 插件管理器
 */
@Component
@Slf4j
public class PluginManager {

    private final Map<String, Plugin> plugins = new ConcurrentHashMap<>();
    private final Map<String, ClassLoader> pluginClassLoaders =
            new ConcurrentHashMap<>();

    /**
     * 加载插件
     */
    public void loadPlugin(String pluginPath) throws Exception {
        File pluginFile = new File(pluginPath);
        URL[] urls = new URL[]{pluginFile.toURI().toURL()};

        // 创建独立的ClassLoader
        URLClassLoader classLoader = new URLClassLoader(
                urls, getClass().getClassLoader());

        // 读取插件配置
        InputStream is = classLoader.getResourceAsStream("plugin.properties");
        Properties props = new Properties();
        props.load(is);

        String pluginClassName = props.getProperty("plugin.class");

        // 加载插件类
        Class<?> pluginClass = classLoader.loadClass(pluginClassName);
        Plugin plugin = (Plugin) pluginClass.getDeclaredConstructor().newInstance();

        // 初始化插件
        plugin.init();

        // 注册插件
        plugins.put(plugin.getId(), plugin);
        pluginClassLoaders.put(plugin.getId(), classLoader);

        log.info("插件加载成功: {}", plugin.getName());
    }

    /**
     * 卸载插件
     */
    public void unloadPlugin(String pluginId) throws Exception {
        Plugin plugin = plugins.remove(pluginId);
        if (plugin != null) {
            plugin.destroy();

            ClassLoader classLoader = pluginClassLoaders.remove(pluginId);
            if (classLoader instanceof URLClassLoader) {
                ((URLClassLoader) classLoader).close();
            }

            log.info("插件卸载成功: {}", plugin.getName());
        }
    }

    /**
     * 执行插件
     */
    public void executePlugin(String pluginId, PluginContext context) {
        Plugin plugin = plugins.get(pluginId);
        if (plugin != null) {
            plugin.execute(context);
        }
    }
}

// ==================== 4. SPI机制扩展 ====================

/**
 * 序列化器SPI接口
 */
public interface Serializer {
    String getType();
    byte[] serialize(Object obj);
    <T> T deserialize(byte[] data, Class<T> clazz);
}

// META-INF/services/com.example.Serializer
// com.example.JsonSerializer
// com.example.ProtobufSerializer

/**
 * 序列化器加载
 */
@Component
public class SerializerLoader {

    private final Map<String, Serializer> serializerMap =
            new ConcurrentHashMap<>();

    @PostConstruct
    public void init() {
        ServiceLoader<Serializer> loader = ServiceLoader.load(Serializer.class);
        for (Serializer serializer : loader) {
            serializerMap.put(serializer.getType(), serializer);
            log.info("加载序列化器: {}", serializer.getType());
        }
    }

    public Serializer getSerializer(String type) {
        return serializerMap.get(type);
    }
}
```

---

### 196. 微服务拆分的原则？

```
┌─────────────────────────────────────────────────────────────────┐
│                    微服务拆分原则                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 1. 业务领域划分 (DDD)                                    │   │
│  │                                                         │   │
│  │    电商系统拆分示例:                                     │   │
│  │    ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐     │   │
│  │    │用户服务 │ │商品服务 │ │订单服务 │ │支付服务 │     │   │
│  │    └─────────┘ └─────────┘ └─────────┘ └─────────┘     │   │
│  │    ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐     │   │
│  │    │库存服务 │ │物流服务 │ │营销服务 │ │评价服务 │     │   │
│  │    └─────────┘ └─────────┘ └─────────┘ └─────────┘     │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 2. 拆分原则                                              │   │
│  │                                                         │   │
│  │  ✓ 单一职责: 每个服务只做一件事                          │   │
│  │  ✓ 高内聚低耦合: 相关功能聚合，服务间松散耦合            │   │
│  │  ✓ 数据独立: 每个服务有独立的数据存储                    │   │
│  │  ✓ 业务闭环: 能独立完成业务功能                          │   │
│  │  ✓ 团队对齐: 服务边界与团队边界一致                      │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 3. 拆分粒度判断                                          │   │
│  │                                                         │   │
│  │  考虑因素:                                               │   │
│  │  • 团队规模: 2 Pizza Team (6-8人维护1-3个服务)           │   │
│  │  • 代码量: 单服务10万行代码以内                          │   │
│  │  • 复杂度: 新人2周内能理解服务                           │   │
│  │  • 变更频率: 变更频繁的拆开，稳定的合并                  │   │
│  │  • 性能需求: 高并发部分独立                              │   │
│  │                                                         │   │
│  │  警告信号:                                               │   │
│  │  ✗ 服务间循环依赖                                       │   │
│  │  ✗ 频繁的跨服务事务                                     │   │
│  │  ✗ 大量的服务间调用                                     │   │
│  │  ✗ 一个需求需要改多个服务                               │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### 微服务拆分示例

```java
/**
 * 电商系统微服务拆分示例
 */

// ==================== 拆分前: 单体应用 ====================

/**
 * 单体应用中的订单模块
 */
public class OrderService {

    @Autowired
    private UserRepository userRepository;      // 用户数据
    @Autowired
    private ProductRepository productRepository; // 商品数据
    @Autowired
    private InventoryRepository inventoryRepository; // 库存数据
    @Autowired
    private PaymentRepository paymentRepository; // 支付数据

    @Transactional
    public Order createOrder(OrderRequest request) {
        // 1. 校验用户
        User user = userRepository.findById(request.getUserId());

        // 2. 查询商品
        Product product = productRepository.findById(request.getProductId());

        // 3. 扣减库存
        inventoryRepository.deduct(request.getProductId(), request.getQuantity());

        // 4. 创建订单
        Order order = new Order();
        // ...设置订单属性
        orderRepository.save(order);

        // 5. 创建支付单
        Payment payment = new Payment();
        paymentRepository.save(payment);

        return order;
    }
}

// ==================== 拆分后: 微服务架构 ====================

/**
 * 订单服务 (order-service)
 */
@Service
public class OrderServiceImpl implements OrderService {

    @Autowired
    private OrderRepository orderRepository;

    // 通过Feign调用其他服务
    @Autowired
    private UserServiceClient userClient;
    @Autowired
    private ProductServiceClient productClient;
    @Autowired
    private InventoryServiceClient inventoryClient;
    @Autowired
    private PaymentServiceClient paymentClient;

    @Override
    public OrderResult createOrder(OrderRequest request) {
        // 1. 校验用户 (RPC调用)
        UserDTO user = userClient.getUser(request.getUserId());
        if (user == null) {
            return OrderResult.fail("用户不存在");
        }

        // 2. 查询商品 (RPC调用)
        ProductDTO product = productClient.getProduct(request.getProductId());
        if (product == null) {
            return OrderResult.fail("商品不存在");
        }

        // 3. 预扣库存 (RPC调用 + 补偿)
        boolean locked = inventoryClient.lockStock(
                request.getProductId(), request.getQuantity());
        if (!locked) {
            return OrderResult.fail("库存不足");
        }

        try {
            // 4. 创建订单
            Order order = buildOrder(request, user, product);
            orderRepository.save(order);

            // 5. 发送订单创建消息 (异步处理后续流程)
            orderEventPublisher.publish(new OrderCreatedEvent(order));

            return OrderResult.success(order);
        } catch (Exception e) {
            // 回滚库存
            inventoryClient.unlockStock(request.getProductId(),
                                        request.getQuantity());
            throw e;
        }
    }
}

/**
 * 用户服务 (user-service)
 */
@Service
public class UserServiceImpl implements UserService {

    @Autowired
    private UserRepository userRepository;

    @Override
    public UserDTO getUser(Long userId) {
        User user = userRepository.findById(userId).orElse(null);
        return UserConverter.toDTO(user);
    }

    @Override
    public UserDTO createUser(CreateUserRequest request) {
        // 用户创建逻辑
    }
}

/**
 * 库存服务 (inventory-service)
 */
@Service
public class InventoryServiceImpl implements InventoryService {

    @Autowired
    private InventoryRepository inventoryRepository;

    @Autowired
    private StringRedisTemplate redisTemplate;

    @Override
    public boolean lockStock(Long productId, Integer quantity) {
        // 使用Redis预扣，保证高性能
        String key = "stock:lock:" + productId;
        Long result = redisTemplate.execute(DEDUCT_STOCK_SCRIPT,
                Collections.singletonList(key), String.valueOf(quantity));
        return result != null && result >= 0;
    }

    @Override
    public boolean unlockStock(Long productId, Integer quantity) {
        String key = "stock:lock:" + productId;
        redisTemplate.opsForValue().increment(key, quantity);
        return true;
    }

    @Override
    public boolean deductStock(Long productId, Integer quantity) {
        // 真正扣减数据库库存
        return inventoryRepository.deduct(productId, quantity) > 0;
    }
}

/**
 * Feign客户端定义
 */
@FeignClient(name = "user-service", fallback = UserServiceFallback.class)
public interface UserServiceClient {

    @GetMapping("/api/users/{userId}")
    UserDTO getUser(@PathVariable("userId") Long userId);
}

@FeignClient(name = "inventory-service", fallback = InventoryServiceFallback.class)
public interface InventoryServiceClient {

    @PostMapping("/api/inventory/lock")
    boolean lockStock(@RequestParam("productId") Long productId,
                     @RequestParam("quantity") Integer quantity);

    @PostMapping("/api/inventory/unlock")
    boolean unlockStock(@RequestParam("productId") Long productId,
                       @RequestParam("quantity") Integer quantity);
}
```

#### 服务依赖关系

```
┌─────────────────────────────────────────────────────────────────┐
│                    微服务依赖关系图                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                       ┌──────────────┐                          │
│                       │   网关服务    │                          │
│                       │  (Gateway)   │                          │
│                       └──────┬───────┘                          │
│                              │                                  │
│           ┌──────────────────┼──────────────────┐               │
│           │                  │                  │               │
│           ▼                  ▼                  ▼               │
│    ┌────────────┐    ┌────────────┐    ┌────────────┐          │
│    │  用户服务   │    │  订单服务   │    │  商品服务   │          │
│    └──────┬─────┘    └──────┬─────┘    └──────┬─────┘          │
│           │                 │                 │                 │
│           │                 ├─────────────────┤                 │
│           │                 │                 │                 │
│           │                 ▼                 ▼                 │
│           │          ┌────────────┐    ┌────────────┐          │
│           │          │  库存服务   │    │  价格服务   │          │
│           │          └──────┬─────┘    └────────────┘          │
│           │                 │                                   │
│           │                 │                                   │
│           ▼                 ▼                                   │
│    ┌────────────┐    ┌────────────┐    ┌────────────┐          │
│    │  支付服务   │    │  物流服务   │    │  通知服务   │          │
│    └────────────┘    └────────────┘    └────────────┘          │
│                                                                 │
│  调用关系:                                                       │
│  • 同步调用: 实线箭头 (用于需要立即返回结果的场景)               │
│  • 异步调用: 通过消息队列解耦 (虚线，未在图中画出)               │
│                                                                 │
│  数据库:                                                        │
│  • 每个服务独立数据库                                           │
│  • 禁止跨服务直接访问数据库                                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

### 197. DDD领域驱动设计的核心概念？

```
┌─────────────────────────────────────────────────────────────────┐
│                    DDD核心概念                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    战略设计                               │   │
│  │                                                         │   │
│  │  ┌─────────────────┐    ┌─────────────────┐            │   │
│  │  │   限界上下文     │    │   上下文映射     │            │   │
│  │  │ Bounded Context│    │ Context Mapping │            │   │
│  │  └─────────────────┘    └─────────────────┘            │   │
│  │                                                         │   │
│  │  领域划分:                                              │   │
│  │  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐          │   │
│  │  │核心域  │ │支撑域  │ │通用域  │ │外部域  │          │   │
│  │  │ Core  │ │Support│ │Generic│ │External│          │   │
│  │  └────────┘ └────────┘ └────────┘ └────────┘          │   │
│  │                                                         │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    战术设计                               │   │
│  │                                                         │   │
│  │  ┌─────────────────────────────────────────────────┐   │   │
│  │  │ 实体 (Entity)                                    │   │   │
│  │  │ • 有唯一标识                                     │   │   │
│  │  │ • 有生命周期                                     │   │   │
│  │  │ • 可变的                                         │   │   │
│  │  └─────────────────────────────────────────────────┘   │   │
│  │                                                         │   │
│  │  ┌─────────────────────────────────────────────────┐   │   │
│  │  │ 值对象 (Value Object)                            │   │   │
│  │  │ • 无唯一标识                                     │   │   │
│  │  │ • 不可变                                         │   │   │
│  │  │ • 通过属性值判断相等                             │   │   │
│  │  └─────────────────────────────────────────────────┘   │   │
│  │                                                         │   │
│  │  ┌─────────────────────────────────────────────────┐   │   │
│  │  │ 聚合 (Aggregate)                                 │   │   │
│  │  │ • 由聚合根管理                                   │   │   │
│  │  │ • 保证事务一致性边界                             │   │   │
│  │  │ • 外部只能通过聚合根访问                         │   │   │
│  │  └─────────────────────────────────────────────────┘   │   │
│  │                                                         │   │
│  │  ┌─────────────────────────────────────────────────┐   │   │
│  │  │ 领域服务 (Domain Service)                        │   │   │
│  │  │ • 无状态                                         │   │   │
│  │  │ • 跨实体的业务逻辑                               │   │   │
│  │  └─────────────────────────────────────────────────┘   │   │
│  │                                                         │   │
│  │  ┌─────────────────────────────────────────────────┐   │   │
│  │  │ 仓储 (Repository)                                │   │   │
│  │  │ • 聚合的持久化接口                               │   │   │
│  │  │ • 隔离领域层与基础设施层                         │   │   │
│  │  └─────────────────────────────────────────────────┘   │   │
│  │                                                         │   │
│  │  ┌─────────────────────────────────────────────────┐   │   │
│  │  │ 领域事件 (Domain Event)                          │   │   │
│  │  │ • 领域中发生的事实                               │   │   │
│  │  │ • 用于聚合间通信                                 │   │   │
│  │  └─────────────────────────────────────────────────┘   │   │
│  │                                                         │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### DDD代码实现

```java
/**
 * DDD领域驱动设计实现示例
 */

// ==================== 1. 实体 (Entity) ====================

/**
 * 订单实体 - 聚合根
 */
@Entity
@Table(name = "orders")
public class Order extends BaseEntity {

    @Id
    private OrderId id;

    @Embedded
    private CustomerId customerId;

    @Embedded
    private Money totalAmount;

    @Enumerated(EnumType.STRING)
    private OrderStatus status;

    @OneToMany(cascade = CascadeType.ALL, orphanRemoval = true)
    @JoinColumn(name = "order_id")
    private List<OrderItem> items = new ArrayList<>();

    @Embedded
    private Address shippingAddress;

    // 私有构造函数，通过工厂方法创建
    private Order() {}

    /**
     * 创建订单 (工厂方法)
     */
    public static Order create(CustomerId customerId,
                                Address shippingAddress) {
        Order order = new Order();
        order.id = OrderId.generate();
        order.customerId = customerId;
        order.shippingAddress = shippingAddress;
        order.status = OrderStatus.CREATED;
        order.totalAmount = Money.ZERO;

        // 发布领域事件
        order.registerEvent(new OrderCreatedEvent(order.id, customerId));

        return order;
    }

    /**
     * 添加订单项 (业务方法)
     */
    public void addItem(ProductId productId, int quantity, Money unitPrice) {
        // 业务规则校验
        if (status != OrderStatus.CREATED) {
            throw new OrderDomainException("只有创建状态的订单才能添加商品");
        }

        OrderItem item = new OrderItem(productId, quantity, unitPrice);
        items.add(item);

        // 重新计算总价
        recalculateTotalAmount();
    }

    /**
     * 支付订单 (状态变更)
     */
    public void pay(PaymentId paymentId) {
        if (status != OrderStatus.CREATED) {
            throw new OrderDomainException("订单状态不允许支付");
        }

        this.status = OrderStatus.PAID;

        // 发布支付成功事件
        registerEvent(new OrderPaidEvent(this.id, paymentId, this.totalAmount));
    }

    /**
     * 发货
     */
    public void ship(TrackingNumber trackingNumber) {
        if (status != OrderStatus.PAID) {
            throw new OrderDomainException("订单未支付，不能发货");
        }

        this.status = OrderStatus.SHIPPED;
        registerEvent(new OrderShippedEvent(this.id, trackingNumber));
    }

    /**
     * 取消订单
     */
    public void cancel(String reason) {
        if (!canCancel()) {
            throw new OrderDomainException("订单状态不允许取消");
        }

        this.status = OrderStatus.CANCELLED;
        registerEvent(new OrderCancelledEvent(this.id, reason));
    }

    public boolean canCancel() {
        return status == OrderStatus.CREATED || status == OrderStatus.PAID;
    }

    private void recalculateTotalAmount() {
        this.totalAmount = items.stream()
                .map(OrderItem::getSubtotal)
                .reduce(Money.ZERO, Money::add);
    }
}

// ==================== 2. 值对象 (Value Object) ====================

/**
 * 金额值对象 - 不可变
 */
@Embeddable
public class Money {

    public static final Money ZERO = new Money(BigDecimal.ZERO, "CNY");

    private final BigDecimal amount;
    private final String currency;

    protected Money() {
        this.amount = BigDecimal.ZERO;
        this.currency = "CNY";
    }

    public Money(BigDecimal amount, String currency) {
        if (amount == null || amount.compareTo(BigDecimal.ZERO) < 0) {
            throw new IllegalArgumentException("金额不能为负");
        }
        this.amount = amount.setScale(2, RoundingMode.HALF_UP);
        this.currency = currency;
    }

    public Money add(Money other) {
        checkSameCurrency(other);
        return new Money(this.amount.add(other.amount), this.currency);
    }

    public Money multiply(int quantity) {
        return new Money(this.amount.multiply(BigDecimal.valueOf(quantity)),
                        this.currency);
    }

    private void checkSameCurrency(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new IllegalArgumentException("货币类型不一致");
        }
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        Money money = (Money) o;
        return amount.compareTo(money.amount) == 0 &&
               Objects.equals(currency, money.currency);
    }

    @Override
    public int hashCode() {
        return Objects.hash(amount, currency);
    }
}

/**
 * 地址值对象
 */
@Embeddable
public class Address {

    private final String province;
    private final String city;
    private final String district;
    private final String street;
    private final String zipCode;

    public Address(String province, String city, String district,
                   String street, String zipCode) {
        // 验证
        Assert.hasText(province, "省份不能为空");
        Assert.hasText(city, "城市不能为空");
        Assert.hasText(street, "街道地址不能为空");

        this.province = province;
        this.city = city;
        this.district = district;
        this.street = street;
        this.zipCode = zipCode;
    }

    public String getFullAddress() {
        return province + city + district + street;
    }
}

// ==================== 3. 聚合 (Aggregate) ====================

/**
 * 订单项 - 属于订单聚合
 */
@Entity
public class OrderItem {

    @Id
    @GeneratedValue
    private Long id;

    @Embedded
    private ProductId productId;

    private int quantity;

    @Embedded
    @AttributeOverride(name = "amount", column = @Column(name = "unit_price"))
    private Money unitPrice;

    protected OrderItem() {}

    public OrderItem(ProductId productId, int quantity, Money unitPrice) {
        if (quantity <= 0) {
            throw new IllegalArgumentException("数量必须大于0");
        }
        this.productId = productId;
        this.quantity = quantity;
        this.unitPrice = unitPrice;
    }

    public Money getSubtotal() {
        return unitPrice.multiply(quantity);
    }
}

// ==================== 4. 仓储 (Repository) ====================

/**
 * 订单仓储接口 (领域层)
 */
public interface OrderRepository {

    Order findById(OrderId id);

    void save(Order order);

    List<Order> findByCustomerId(CustomerId customerId);

    OrderId nextId();
}

/**
 * 订单仓储实现 (基础设施层)
 */
@Repository
public class JpaOrderRepository implements OrderRepository {

    @Autowired
    private OrderJpaRepository jpaRepository;

    @Autowired
    private ApplicationEventPublisher eventPublisher;

    @Override
    public Order findById(OrderId id) {
        return jpaRepository.findById(id.getValue()).orElse(null);
    }

    @Override
    public void save(Order order) {
        jpaRepository.save(order);

        // 发布领域事件
        order.getDomainEvents().forEach(event ->
                eventPublisher.publishEvent(event));
        order.clearDomainEvents();
    }
}

// ==================== 5. 领域服务 (Domain Service) ====================

/**
 * 订单领域服务 - 处理跨聚合的业务逻辑
 */
@Service
public class OrderDomainService {

    @Autowired
    private OrderRepository orderRepository;

    @Autowired
    private InventoryService inventoryService;

    @Autowired
    private PricingService pricingService;

    /**
     * 创建订单 (涉及多个聚合)
     */
    public Order createOrder(CustomerId customerId,
                             List<OrderItemRequest> items,
                             Address shippingAddress) {

        // 创建订单聚合
        Order order = Order.create(customerId, shippingAddress);

        // 添加订单项
        for (OrderItemRequest item : items) {
            // 获取商品价格 (调用价格服务)
            Money price = pricingService.getPrice(item.getProductId());

            // 检查库存 (调用库存服务)
            if (!inventoryService.checkStock(item.getProductId(),
                                              item.getQuantity())) {
                throw new OrderDomainException("商品库存不足: " + item.getProductId());
            }

            order.addItem(item.getProductId(), item.getQuantity(), price);
        }

        // 保存订单
        orderRepository.save(order);

        return order;
    }
}

// ==================== 6. 领域事件 (Domain Event) ====================

/**
 * 订单创建事件
 */
public class OrderCreatedEvent extends DomainEvent {

    private final OrderId orderId;
    private final CustomerId customerId;

    public OrderCreatedEvent(OrderId orderId, CustomerId customerId) {
        super();
        this.orderId = orderId;
        this.customerId = customerId;
    }
}

/**
 * 事件处理器
 */
@Component
public class OrderEventHandler {

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void handleOrderCreated(OrderCreatedEvent event) {
        // 发送通知
        notificationService.sendOrderCreatedNotification(event.getOrderId());

        // 更新统计
        statisticsService.incrementOrderCount();
    }

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void handleOrderPaid(OrderPaidEvent event) {
        // 扣减库存
        inventoryService.deductStock(event.getOrderId());

        // 发送发货通知
        notificationService.sendPaymentSuccessNotification(event.getOrderId());
    }
}
```

---

### 198. CQRS架构是什么？

```
┌─────────────────────────────────────────────────────────────────┐
│               CQRS (命令查询职责分离) 架构                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    传统架构                               │   │
│  │                                                         │   │
│  │            ┌─────────────────┐                          │   │
│  │   CRUD  ──▶│   Service       │                          │   │
│  │            │  (增删改查混合)  │──▶ 单一数据库             │   │
│  │            └─────────────────┘                          │   │
│  │                                                         │   │
│  └─────────────────────────────────────────────────────────┘   │
│                         │                                      │
│                         ▼                                      │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    CQRS架构                               │   │
│  │                                                         │   │
│  │           写操作                    读操作               │   │
│  │            │                          │                 │   │
│  │            ▼                          ▼                 │   │
│  │   ┌─────────────────┐      ┌─────────────────┐         │   │
│  │   │ Command Handler │      │ Query Handler   │         │   │
│  │   │    (命令处理)    │      │   (查询处理)    │         │   │
│  │   └────────┬────────┘      └────────┬────────┘         │   │
│  │            │                        │                   │   │
│  │            ▼                        ▼                   │   │
│  │   ┌─────────────────┐      ┌─────────────────┐         │   │
│  │   │   写模型        │      │   读模型         │         │   │
│  │   │  (领域模型)     │      │  (查询视图)      │         │   │
│  │   └────────┬────────┘      └────────┬────────┘         │   │
│  │            │                        │                   │   │
│  │            ▼                        ▼                   │   │
│  │   ┌─────────────────┐      ┌─────────────────┐         │   │
│  │   │   写数据库       │─────▶│   读数据库       │         │   │
│  │   │  (MySQL主库)    │ 同步 │(ES/Redis/只读库) │         │   │
│  │   └─────────────────┘      └─────────────────┘         │   │
│  │                                                         │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  优点:                                                          │
│  • 读写分离优化，可独立扩展                                      │
│  • 读模型可针对查询场景优化                                      │
│  • 简化复杂领域模型                                             │
│                                                                 │
│  缺点:                                                          │
│  • 增加系统复杂度                                               │
│  • 数据最终一致性                                               │
│  • 需要处理同步延迟                                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### CQRS实现

```java
/**
 * CQRS架构实现
 */

// ==================== 1. 命令 (Command) ====================

/**
 * 创建订单命令
 */
@Data
@Builder
public class CreateOrderCommand implements Command {
    private final String commandId;
    private String customerId;
    private List<OrderItemDTO> items;
    private AddressDTO shippingAddress;

    public CreateOrderCommand() {
        this.commandId = UUID.randomUUID().toString();
    }
}

/**
 * 取消订单命令
 */
@Data
public class CancelOrderCommand implements Command {
    private final String commandId = UUID.randomUUID().toString();
    private String orderId;
    private String reason;
}

// ==================== 2. 命令处理器 (Command Handler) ====================

/**
 * 命令处理器接口
 */
public interface CommandHandler<C extends Command, R> {
    R handle(C command);
}

/**
 * 创建订单命令处理器
 */
@Component
public class CreateOrderCommandHandler
        implements CommandHandler<CreateOrderCommand, String> {

    @Autowired
    private OrderRepository orderRepository;

    @Autowired
    private OrderDomainService orderDomainService;

    @Autowired
    private EventBus eventBus;

    @Override
    @Transactional
    public String handle(CreateOrderCommand command) {
        // 1. 创建订单聚合
        Order order = orderDomainService.createOrder(
                new CustomerId(command.getCustomerId()),
                toOrderItems(command.getItems()),
                toAddress(command.getShippingAddress())
        );

        // 2. 保存订单
        orderRepository.save(order);

        // 3. 发布事件 (用于同步读模型)
        eventBus.publish(new OrderCreatedEvent(
                order.getId().getValue(),
                order.getCustomerId().getValue(),
                order.getTotalAmount().getAmount(),
                order.getItems().stream()
                        .map(this::toEventItem)
                        .collect(Collectors.toList())
        ));

        return order.getId().getValue();
    }
}

// ==================== 3. 查询 (Query) ====================

/**
 * 订单列表查询
 */
@Data
public class OrderListQuery implements Query {
    private String customerId;
    private String status;
    private int page;
    private int size;
}

/**
 * 订单详情查询
 */
@Data
public class OrderDetailQuery implements Query {
    private String orderId;
}

// ==================== 4. 查询处理器 (Query Handler) ====================

/**
 * 订单查询处理器
 */
@Component
public class OrderQueryHandler {

    @Autowired
    private OrderReadRepository readRepository;  // 读库仓储

    @Autowired
    private ElasticsearchClient esClient;  // ES客户端

    /**
     * 查询订单列表 (从ES查询)
     */
    public PageResult<OrderListVO> handle(OrderListQuery query) {
        SearchRequest request = SearchRequest.of(s -> s
                .index("orders")
                .query(q -> q.bool(b -> {
                    if (query.getCustomerId() != null) {
                        b.must(m -> m.term(t -> t
                                .field("customerId")
                                .value(query.getCustomerId())));
                    }
                    if (query.getStatus() != null) {
                        b.must(m -> m.term(t -> t
                                .field("status")
                                .value(query.getStatus())));
                    }
                    return b;
                }))
                .from(query.getPage() * query.getSize())
                .size(query.getSize())
                .sort(so -> so.field(f -> f
                        .field("createTime")
                        .order(SortOrder.Desc)))
        );

        SearchResponse<OrderDocument> response = esClient.search(
                request, OrderDocument.class);

        List<OrderListVO> list = response.hits().hits().stream()
                .map(hit -> toListVO(hit.source()))
                .collect(Collectors.toList());

        return PageResult.of(list, response.hits().total().value(),
                            query.getPage(), query.getSize());
    }

    /**
     * 查询订单详情 (从读库查询)
     */
    public OrderDetailVO handle(OrderDetailQuery query) {
        OrderReadModel order = readRepository.findById(query.getOrderId());
        if (order == null) {
            throw new NotFoundException("订单不存在");
        }
        return toDetailVO(order);
    }
}

// ==================== 5. 读模型同步 ====================

/**
 * 订单事件监听器 - 同步读模型
 */
@Component
@Slf4j
public class OrderEventListener {

    @Autowired
    private OrderReadRepository readRepository;

    @Autowired
    private ElasticsearchClient esClient;

    /**
     * 监听订单创建事件
     */
    @EventHandler
    public void on(OrderCreatedEvent event) {
        // 1. 构建读模型
        OrderReadModel readModel = OrderReadModel.builder()
                .id(event.getOrderId())
                .customerId(event.getCustomerId())
                .totalAmount(event.getTotalAmount())
                .status("CREATED")
                .createTime(event.getOccurredOn())
                .items(event.getItems())
                .build();

        // 2. 保存到读库
        readRepository.save(readModel);

        // 3. 同步到ES
        syncToElasticsearch(readModel);

        log.info("订单读模型同步完成: {}", event.getOrderId());
    }

    /**
     * 监听订单状态变更事件
     */
    @EventHandler
    public void on(OrderStatusChangedEvent event) {
        // 更新读库
        readRepository.updateStatus(event.getOrderId(), event.getNewStatus());

        // 更新ES
        updateElasticsearch(event.getOrderId(),
                Map.of("status", event.getNewStatus()));
    }

    private void syncToElasticsearch(OrderReadModel readModel) {
        OrderDocument doc = toDocument(readModel);
        esClient.index(i -> i
                .index("orders")
                .id(readModel.getId())
                .document(doc)
        );
    }
}

// ==================== 6. 命令总线 ====================

/**
 * 命令总线
 */
@Component
public class CommandBus {

    private final Map<Class<?>, CommandHandler<?, ?>> handlers =
            new ConcurrentHashMap<>();

    @Autowired
    public CommandBus(List<CommandHandler<?, ?>> handlerList) {
        for (CommandHandler<?, ?> handler : handlerList) {
            Class<?> commandType = getCommandType(handler);
            handlers.put(commandType, handler);
        }
    }

    @SuppressWarnings("unchecked")
    public <C extends Command, R> R dispatch(C command) {
        CommandHandler<C, R> handler =
                (CommandHandler<C, R>) handlers.get(command.getClass());

        if (handler == null) {
            throw new IllegalArgumentException(
                    "No handler for command: " + command.getClass());
        }

        return handler.handle(command);
    }
}

// ==================== 使用示例 ====================

@RestController
@RequestMapping("/api/orders")
public class OrderController {

    @Autowired
    private CommandBus commandBus;

    @Autowired
    private OrderQueryHandler queryHandler;

    /**
     * 创建订单 (走Command)
     */
    @PostMapping
    public Result<String> createOrder(@RequestBody CreateOrderCommand command) {
        String orderId = commandBus.dispatch(command);
        return Result.success(orderId);
    }

    /**
     * 查询订单列表 (走Query)
     */
    @GetMapping
    public Result<PageResult<OrderListVO>> listOrders(OrderListQuery query) {
        return Result.success(queryHandler.handle(query));
    }

    /**
     * 查询订单详情 (走Query)
     */
    @GetMapping("/{orderId}")
    public Result<OrderDetailVO> getOrder(@PathVariable String orderId) {
        OrderDetailQuery query = new OrderDetailQuery();
        query.setOrderId(orderId);
        return Result.success(queryHandler.handle(query));
    }
}
```

---

### 199. 事件驱动架构的设计？

```
┌─────────────────────────────────────────────────────────────────┐
│                    事件驱动架构                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                  事件驱动核心概念                         │   │
│  │                                                         │   │
│  │  ┌─────────┐    ┌─────────┐    ┌─────────┐             │   │
│  │  │事件生产者│───▶│ 事件通道 │───▶│事件消费者│             │   │
│  │  │Producer │    │ Channel │    │Consumer │             │   │
│  │  └─────────┘    └─────────┘    └─────────┘             │   │
│  │                                                         │   │
│  │  事件类型:                                              │   │
│  │  • 领域事件: OrderCreated, PaymentCompleted             │   │
│  │  • 集成事件: 跨服务通信                                 │   │
│  │  • 命令事件: 触发特定动作                               │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                  事件驱动模式                             │   │
│  │                                                         │   │
│  │  模式1: 简单事件通知                                     │   │
│  │  ┌────────┐   事件(仅ID)   ┌────────┐                  │   │
│  │  │ 服务A  │ ─────────────▶ │ 服务B  │ ──▶ 回查获取数据  │   │
│  │  └────────┘               └────────┘                   │   │
│  │                                                         │   │
│  │  模式2: 事件携带状态                                     │   │
│  │  ┌────────┐  事件(含完整数据) ┌────────┐               │   │
│  │  │ 服务A  │ ────────────────▶ │ 服务B  │               │   │
│  │  └────────┘                  └────────┘               │   │
│  │                                                         │   │
│  │  模式3: 事件溯源 (Event Sourcing)                       │   │
│  │  ┌────────┐   ┌────────┐   ┌────────┐   ┌────────┐   │   │
│  │  │Event 1 │──▶│Event 2 │──▶│Event 3 │──▶│当前状态 │   │   │
│  │  └────────┘   └────────┘   └────────┘   └────────┘   │   │
│  │                                                         │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                  架构示意图                               │   │
│  │                                                         │   │
│  │   ┌─────────┐  ┌─────────┐  ┌─────────┐               │   │
│  │   │订单服务 │  │库存服务 │  │支付服务 │               │   │
│  │   └────┬────┘  └────┬────┘  └────┬────┘               │   │
│  │        │           │           │                       │   │
│  │        ▼           ▼           ▼                       │   │
│  │   ═══════════════════════════════════════              │   │
│  │            消息中间件 (Kafka/RabbitMQ)                  │   │
│  │   ═══════════════════════════════════════              │   │
│  │        │           │           │                       │   │
│  │        ▼           ▼           ▼                       │   │
│  │   ┌─────────┐  ┌─────────┐  ┌─────────┐               │   │
│  │   │通知服务 │  │统计服务 │  │审计服务 │               │   │
│  │   └─────────┘  └─────────┘  └─────────┘               │   │
│  │                                                         │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### 事件驱动架构实现

```java
/**
 * 事件驱动架构完整实现
 */

// ==================== 1. 事件定义 ====================

/**
 * 事件基类
 */
@Data
public abstract class DomainEvent {
    private String eventId;
    private String eventType;
    private LocalDateTime occurredOn;
    private String aggregateId;
    private int version;

    protected DomainEvent() {
        this.eventId = UUID.randomUUID().toString();
        this.occurredOn = LocalDateTime.now();
        this.eventType = this.getClass().getSimpleName();
    }
}

/**
 * 订单创建事件
 */
@Data
@EqualsAndHashCode(callSuper = true)
public class OrderCreatedEvent extends DomainEvent {
    private String orderId;
    private String customerId;
    private BigDecimal totalAmount;
    private List<OrderItemDTO> items;
    private String shippingAddress;

    public OrderCreatedEvent(Order order) {
        super();
        this.aggregateId = order.getId();
        this.orderId = order.getId();
        this.customerId = order.getCustomerId();
        this.totalAmount = order.getTotalAmount();
        this.items = order.getItems().stream()
                .map(this::toDTO)
                .collect(Collectors.toList());
    }
}

/**
 * 订单支付完成事件
 */
@Data
@EqualsAndHashCode(callSuper = true)
public class OrderPaidEvent extends DomainEvent {
    private String orderId;
    private String paymentId;
    private BigDecimal paidAmount;
    private String paymentMethod;
    private LocalDateTime paidTime;
}

/**
 * 库存扣减事件
 */
@Data
@EqualsAndHashCode(callSuper = true)
public class InventoryDeductedEvent extends DomainEvent {
    private String orderId;
    private List<StockDeduction> deductions;

    @Data
    public static class StockDeduction {
        private String productId;
        private int quantity;
        private int remainingStock;
    }
}

// ==================== 2. 事件发布 ====================

/**
 * 事件发布器接口
 */
public interface EventPublisher {
    void publish(DomainEvent event);
    void publishAll(List<DomainEvent> events);
}

/**
 * Kafka事件发布器实现
 */
@Component
@Slf4j
public class KafkaEventPublisher implements EventPublisher {

    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate;

    @Autowired
    private ObjectMapper objectMapper;

    @Override
    public void publish(DomainEvent event) {
        try {
            String topic = getTopicName(event);
            String key = event.getAggregateId();
            String message = objectMapper.writeValueAsString(event);

            kafkaTemplate.send(topic, key, message)
                    .addCallback(
                        result -> log.debug("事件发布成功: {}", event.getEventId()),
                        ex -> log.error("事件发布失败: {}", event.getEventId(), ex)
                    );

        } catch (JsonProcessingException e) {
            throw new EventPublishException("事件序列化失败", e);
        }
    }

    @Override
    public void publishAll(List<DomainEvent> events) {
        events.forEach(this::publish);
    }

    private String getTopicName(DomainEvent event) {
        // 根据事件类型确定Topic
        // OrderCreatedEvent -> order-events
        String eventName = event.getClass().getSimpleName();
        String domain = eventName.replace("Event", "")
                                 .replaceAll("([A-Z])", "-$1")
                                 .toLowerCase()
                                 .substring(1);
        return domain + "-events";
    }
}

/**
 * 可靠事件发布 (本地事件表 + 定时发送)
 */
@Service
@Slf4j
public class ReliableEventPublisher {

    @Autowired
    private EventOutboxRepository outboxRepository;

    @Autowired
    private KafkaEventPublisher kafkaPublisher;

    /**
     * 保存事件到本地表 (与业务事务一起)
     */
    @Transactional
    public void saveEvent(DomainEvent event) {
        EventOutbox outbox = EventOutbox.builder()
                .eventId(event.getEventId())
                .eventType(event.getEventType())
                .aggregateId(event.getAggregateId())
                .payload(JSON.toJSONString(event))
                .status(EventStatus.PENDING)
                .createTime(LocalDateTime.now())
                .build();

        outboxRepository.save(outbox);
    }

    /**
     * 定时发送未发送的事件
     */
    @Scheduled(fixedDelay = 1000)
    public void publishPendingEvents() {
        List<EventOutbox> pendingEvents = outboxRepository
                .findByStatusOrderByCreateTime(EventStatus.PENDING, 100);

        for (EventOutbox outbox : pendingEvents) {
            try {
                // 反序列化事件
                DomainEvent event = deserializeEvent(outbox);

                // 发布到Kafka
                kafkaPublisher.publish(event);

                // 更新状态为已发送
                outbox.setStatus(EventStatus.PUBLISHED);
                outbox.setPublishTime(LocalDateTime.now());
                outboxRepository.save(outbox);

            } catch (Exception e) {
                log.error("事件发布失败: {}", outbox.getEventId(), e);
                outbox.setRetryCount(outbox.getRetryCount() + 1);
                if (outbox.getRetryCount() >= 3) {
                    outbox.setStatus(EventStatus.FAILED);
                }
                outboxRepository.save(outbox);
            }
        }
    }
}

// ==================== 3. 事件消费 ====================

/**
 * 订单事件消费者
 */
@Component
@Slf4j
public class OrderEventConsumer {

    @Autowired
    private InventoryService inventoryService;

    @Autowired
    private NotificationService notificationService;

    @Autowired
    private StringRedisTemplate redisTemplate;

    /**
     * 消费订单创建事件
     */
    @KafkaListener(topics = "order-created-events", groupId = "inventory-service")
    public void handleOrderCreated(ConsumerRecord<String, String> record) {
        OrderCreatedEvent event = JSON.parseObject(
                record.value(), OrderCreatedEvent.class);

        // 幂等性检查
        String idempotentKey = "event:processed:" + event.getEventId();
        Boolean isNew = redisTemplate.opsForValue()
                .setIfAbsent(idempotentKey, "1", Duration.ofDays(1));

        if (Boolean.FALSE.equals(isNew)) {
            log.info("事件已处理，跳过: {}", event.getEventId());
            return;
        }

        try {
            log.info("处理订单创建事件: orderId={}", event.getOrderId());

            // 锁定库存
            for (OrderItemDTO item : event.getItems()) {
                inventoryService.lockStock(item.getProductId(), item.getQuantity());
            }

            log.info("库存锁定成功: orderId={}", event.getOrderId());

        } catch (Exception e) {
            log.error("处理订单创建事件失败: {}", event.getEventId(), e);
            // 删除幂等key，允许重试
            redisTemplate.delete(idempotentKey);
            throw e;
        }
    }

    /**
     * 消费订单支付事件
     */
    @KafkaListener(topics = "order-paid-events", groupId = "notification-service")
    public void handleOrderPaid(ConsumerRecord<String, String> record) {
        OrderPaidEvent event = JSON.parseObject(
                record.value(), OrderPaidEvent.class);

        log.info("处理订单支付事件: orderId={}", event.getOrderId());

        // 发送支付成功通知
        notificationService.sendPaymentSuccessNotification(
                event.getOrderId(),
                event.getPaidAmount()
        );
    }
}

/**
 * 通用事件处理器注册
 */
@Component
public class EventHandlerRegistry {

    private final Map<Class<?>, List<EventHandler<?>>> handlers =
            new ConcurrentHashMap<>();

    @Autowired
    public EventHandlerRegistry(List<EventHandler<?>> allHandlers) {
        for (EventHandler<?> handler : allHandlers) {
            Class<?> eventType = getEventType(handler);
            handlers.computeIfAbsent(eventType, k -> new ArrayList<>())
                    .add(handler);
        }
    }

    @SuppressWarnings("unchecked")
    public <E extends DomainEvent> void dispatch(E event) {
        List<EventHandler<?>> eventHandlers = handlers.get(event.getClass());
        if (eventHandlers != null) {
            for (EventHandler<?> handler : eventHandlers) {
                ((EventHandler<E>) handler).handle(event);
            }
        }
    }
}

// ==================== 4. 事件溯源 (Event Sourcing) ====================

/**
 * 事件溯源聚合根基类
 */
public abstract class EventSourcedAggregate {

    private String id;
    private int version = 0;
    private List<DomainEvent> uncommittedEvents = new ArrayList<>();

    /**
     * 应用事件
     */
    protected void applyEvent(DomainEvent event) {
        // 使用反射调用对应的apply方法
        try {
            Method method = this.getClass().getMethod("apply", event.getClass());
            method.invoke(this, event);
        } catch (Exception e) {
            throw new RuntimeException("Apply event failed", e);
        }

        event.setVersion(++version);
        uncommittedEvents.add(event);
    }

    /**
     * 从历史事件重建状态
     */
    public void replayEvents(List<DomainEvent> events) {
        for (DomainEvent event : events) {
            try {
                Method method = this.getClass().getMethod("apply", event.getClass());
                method.invoke(this, event);
                this.version = event.getVersion();
            } catch (Exception e) {
                throw new RuntimeException("Replay event failed", e);
            }
        }
    }

    public List<DomainEvent> getUncommittedEvents() {
        return new ArrayList<>(uncommittedEvents);
    }

    public void markEventsAsCommitted() {
        uncommittedEvents.clear();
    }
}

/**
 * 事件溯源订单聚合
 */
public class OrderAggregate extends EventSourcedAggregate {

    private String orderId;
    private String customerId;
    private OrderStatus status;
    private BigDecimal totalAmount;
    private List<OrderItem> items = new ArrayList<>();

    // 私有构造，通过工厂方法或事件重建
    private OrderAggregate() {}

    /**
     * 创建订单 (发起事件)
     */
    public static OrderAggregate create(String customerId, List<OrderItem> items) {
        OrderAggregate order = new OrderAggregate();

        // 计算总金额
        BigDecimal total = items.stream()
                .map(OrderItem::getSubtotal)
                .reduce(BigDecimal.ZERO, BigDecimal::add);

        // 创建并应用事件
        OrderCreatedEvent event = new OrderCreatedEvent();
        event.setOrderId(UUID.randomUUID().toString());
        event.setCustomerId(customerId);
        event.setTotalAmount(total);

        order.applyEvent(event);
        return order;
    }

    /**
     * 支付订单
     */
    public void pay(String paymentId, BigDecimal amount) {
        if (status != OrderStatus.CREATED) {
            throw new IllegalStateException("订单状态不允许支付");
        }

        OrderPaidEvent event = new OrderPaidEvent();
        event.setOrderId(this.orderId);
        event.setPaymentId(paymentId);
        event.setPaidAmount(amount);
        event.setPaidTime(LocalDateTime.now());

        applyEvent(event);
    }

    // 事件应用方法 (状态变更)
    public void apply(OrderCreatedEvent event) {
        this.orderId = event.getOrderId();
        this.customerId = event.getCustomerId();
        this.totalAmount = event.getTotalAmount();
        this.status = OrderStatus.CREATED;
    }

    public void apply(OrderPaidEvent event) {
        this.status = OrderStatus.PAID;
    }
}

/**
 * 事件存储仓储
 */
@Repository
public class EventStoreRepository {

    @Autowired
    private JdbcTemplate jdbcTemplate;

    /**
     * 保存事件
     */
    @Transactional
    public void saveEvents(String aggregateId, List<DomainEvent> events,
                           int expectedVersion) {
        // 乐观锁检查
        Integer currentVersion = jdbcTemplate.queryForObject(
                "SELECT MAX(version) FROM event_store WHERE aggregate_id = ?",
                Integer.class, aggregateId);

        if (currentVersion != null && currentVersion != expectedVersion) {
            throw new ConcurrencyException("并发冲突");
        }

        // 保存事件
        for (DomainEvent event : events) {
            jdbcTemplate.update(
                    "INSERT INTO event_store (event_id, aggregate_id, event_type, " +
                    "version, payload, created_at) VALUES (?, ?, ?, ?, ?, ?)",
                    event.getEventId(),
                    aggregateId,
                    event.getEventType(),
                    event.getVersion(),
                    JSON.toJSONString(event),
                    event.getOccurredOn()
            );
        }
    }

    /**
     * 获取聚合的所有事件
     */
    public List<DomainEvent> getEvents(String aggregateId) {
        return jdbcTemplate.query(
                "SELECT * FROM event_store WHERE aggregate_id = ? ORDER BY version",
                (rs, rowNum) -> {
                    String eventType = rs.getString("event_type");
                    String payload = rs.getString("payload");
                    return deserializeEvent(eventType, payload);
                },
                aggregateId
        );
    }

    /**
     * 获取指定版本后的事件
     */
    public List<DomainEvent> getEventsAfterVersion(String aggregateId, int version) {
        return jdbcTemplate.query(
                "SELECT * FROM event_store WHERE aggregate_id = ? AND version > ? " +
                "ORDER BY version",
                (rs, rowNum) -> deserializeEvent(
                        rs.getString("event_type"),
                        rs.getString("payload")),
                aggregateId, version
        );
    }
}
```

---

### 200. 如何做技术选型？

```
┌─────────────────────────────────────────────────────────────────┐
│                    技术选型方法论                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                  选型流程                                 │   │
│  │                                                         │   │
│  │  1.需求分析 ──▶ 2.候选方案 ──▶ 3.评估对比 ──▶ 4.验证    │   │
│  │       │            │            │            │          │   │
│  │       ▼            ▼            ▼            ▼          │   │
│  │   明确场景     收集备选     多维度评分    POC验证       │   │
│  │   明确约束     行业调研     权重计算      压力测试       │   │
│  │                                                         │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                  评估维度                                 │   │
│  │                                                         │   │
│  │  ┌──────────────────────────────────────────────────┐  │   │
│  │  │ 功能性                                            │  │   │
│  │  │ • 是否满足业务需求                                │  │   │
│  │  │ • 功能完整性和成熟度                              │  │   │
│  │  │ • 扩展性和定制能力                                │  │   │
│  │  └──────────────────────────────────────────────────┘  │   │
│  │                                                         │   │
│  │  ┌──────────────────────────────────────────────────┐  │   │
│  │  │ 性能                                              │  │   │
│  │  │ • 吞吐量、响应时间                                │  │   │
│  │  │ • 资源消耗                                        │  │   │
│  │  │ • 可扩展性                                        │  │   │
│  │  └──────────────────────────────────────────────────┘  │   │
│  │                                                         │   │
│  │  ┌──────────────────────────────────────────────────┐  │   │
│  │  │ 可靠性                                            │  │   │
│  │  │ • 稳定性、可用性                                  │  │   │
│  │  │ • 容错能力                                        │  │   │
│  │  │ • 数据一致性保证                                  │  │   │
│  │  └──────────────────────────────────────────────────┘  │   │
│  │                                                         │   │
│  │  ┌──────────────────────────────────────────────────┐  │   │
│  │  │ 生态和社区                                        │  │   │
│  │  │ • 社区活跃度、更新频率                            │  │   │
│  │  │ • 文档质量                                        │  │   │
│  │  │ • 第三方集成支持                                  │  │   │
│  │  └──────────────────────────────────────────────────┘  │   │
│  │                                                         │   │
│  │  ┌──────────────────────────────────────────────────┐  │   │
│  │  │ 团队因素                                          │  │   │
│  │  │ • 学习成本                                        │  │   │
│  │  │ • 团队技术栈匹配度                                │  │   │
│  │  │ • 人才招聘难度                                    │  │   │
│  │  └──────────────────────────────────────────────────┘  │   │
│  │                                                         │   │
│  │  ┌──────────────────────────────────────────────────┐  │   │
│  │  │ 成本                                              │  │   │
│  │  │ • 许可费用                                        │  │   │
│  │  │ • 运维成本                                        │  │   │
│  │  │ • 迁移成本                                        │  │   │
│  │  └──────────────────────────────────────────────────┘  │   │
│  │                                                         │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
