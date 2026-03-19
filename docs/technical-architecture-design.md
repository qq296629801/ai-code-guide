# 技术架构 - 详细设计文档

> 版本：v1.0  
> 日期：2026-03-19

---

## 一、系统架构总览

### 1.1 整体架构图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              用户接入层                                       │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│  │  Web 控制台  │  │  移动端 APP  │  │  QQ 机器人  │  │ 微信/企微   │        │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘        │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                          │
│  │  Telegram   │  │  Discord    │  │   API SDK   │                          │
│  └─────────────┘  └─────────────┘  └─────────────┘                          │
└─────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              API 网关层                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                         Kong / Spring Cloud Gateway                  │    │
│  │    认证鉴权 │ 限流熔断 │ 路由转发 │ 日志追踪 │ 灰度发布               │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              业务服务层                                       │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│  │ 用户服务    │  │ 商品服务    │  │ 预测服务    │  │ 报告服务    │        │
│  │ user-svc    │  │ product-svc │  │ predict-svc │  │ report-svc  │        │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘        │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│  │ 采集服务    │  │ 分析服务    │  │ 通知服务    │  │ 调度服务    │        │
│  │ crawl-svc   │  │ analyze-svc │  │ notify-svc  │  │ scheduler   │        │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘        │
└─────────────────────────────────────────────────────────────────────────────┘
                                        │
                    ┌───────────────────┼───────────────────┐
                    ▼                   ▼                   ▼
┌─────────────────────────┐ ┌─────────────────────────┐ ┌─────────────────────┐
│       消息队列层         │ │        缓存层           │ │      搜索层         │
│  ┌─────────────────┐    │ │  ┌─────────────────┐    │ │ ┌─────────────────┐ │
│  │     Kafka       │    │ │  │  Redis Cluster  │    │ │ │ Elasticsearch   │ │
│  │  数据采集队列   │    │ │  │  热点数据缓存   │    │ │ │  商品搜索索引   │ │
│  │  预测任务队列   │    │ │  │  会话管理       │    │ │ │  日志检索       │ │
│  │  通知队列       │    │ │  │  分布式锁       │    │ │ └─────────────────┘ │
│  └─────────────────┘    │ │  └─────────────────┘    │ └─────────────────────┘
└─────────────────────────┘ └─────────────────────────┘
                                        │
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              数据存储层                                       │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│  │   MySQL     │  │  MongoDB    │  │ ClickHouse  │  │   Neo4j     │        │
│  │  业务数据   │  │  日志数据   │  │  数据仓库   │  │  知识图谱   │        │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘        │
│  ┌─────────────┐  ┌─────────────┐                                           │
│  │   MinIO     │  │    OSS      │                                           │
│  │  对象存储   │  │  云存储     │                                           │
│  └─────────────┘  └─────────────┘                                           │
└─────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              AI 能力层                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                        AI Gateway (统一网关)                          │    │
│  │     模型路由 │ 负载均衡 │ 成本控制 │ 内容审核 │ 日志审计              │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│  │ OpenAI API  │  │ Claude API  │  │ DeepSeek    │  │ 通义千问    │        │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘        │
│  ┌─────────────┐  ┌─────────────┐                                           │
│  │ 本地模型    │  │ 向量数据库  │                                           │
│  │ Ollama      │  │ Milvus      │                                           │
│  └─────────────┘  └─────────────┘                                           │
└─────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              外部数据源                                       │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│  │ 抖音数据    │  │ 淘宝数据    │  │ 1688数据    │  │ 第三方API   │        │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘        │
│  ┌─────────────┐  ┌─────────────┐                                           │
│  │ 蝉妈妈      │  │ 飞瓜数据    │                                           │
│  └─────────────┘  └─────────────┘                                           │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 技术栈清单

| 层级 | 技术选型 | 版本 | 说明 |
|------|----------|------|------|
| **接入层** | | | |
| Web 前端 | Vue 3 + TypeScript | 3.x | 响应式管理后台 |
| 移动端 | Flutter | 3.x | 跨平台 APP |
| QQ 机器人 | OpenClaw / NoneBot | - | QQ 消息推送 |
| 微信 | WCFerry / ComWechat | - | 微信消息推送 |
| **网关层** | | | |
| API 网关 | Spring Cloud Gateway | 4.x | 路由、限流、鉴权 |
| 负载均衡 | Nginx | 1.24+ | 反向代理、负载 |
| **服务层** | | | |
| 后端框架 | Spring Boot | 3.2+ | 微服务框架 |
| 微服务 | Spring Cloud | 2023.x | 服务治理 |
| 任务调度 | XXL-Job | 2.4+ | 分布式调度 |
| 消息队列 | Kafka | 3.x | 高吞吐消息 |
| 缓存 | Redis Cluster | 7.x | 分布式缓存 |
| **数据层** | | | |
| 关系数据库 | MySQL | 8.0+ | 核心业务数据 |
| 文档数据库 | MongoDB | 7.x | 日志、非结构化 |
| 时序数据库 | ClickHouse | 24.x | 分析查询 |
| 图数据库 | Neo4j | 5.x | 知识图谱 |
| 搜索引擎 | Elasticsearch | 8.x | 全文检索 |
| 对象存储 | MinIO | - | 文件存储 |
| **AI 层** | | | |
| AI 框架 | LangChain4j | 0.35+ | Java AI 框架 |
| 向量数据库 | Milvus | 2.x | 向量检索 |
| 本地模型 | Ollama | - | 本地推理 |
| **运维层** | | | |
| 容器 | Docker | 24+ | 容器化部署 |
| 编排 | Kubernetes | 1.28+ | 集群管理 |
| 监控 | Prometheus + Grafana | - | 指标监控 |
| 日志 | ELK Stack | 8.x | 日志收集分析 |
| 链路追踪 | SkyWalking | 9.x | 分布式追踪 |

---

## 二、微服务设计

### 2.1 服务拆分

```
ai-predict-platform/
├── gateway-service/          # API 网关服务
├── user-service/             # 用户服务
├── product-service/          # 商品服务
├── crawl-service/            # 数据采集服务
├── predict-service/          # 预测服务
├── analyze-service/          # 分析服务
├── report-service/           # 报告服务
├── notify-service/           # 通知服务
├── scheduler-service/        # 调度服务
├── ai-gateway-service/       # AI 网关服务
└── common/                   # 公共模块
    ├── common-core/          # 核心工具类
    ├── common-redis/         # Redis 封装
    ├── common-kafka/         # Kafka 封装
    ├── common-mybatis/       # MyBatis 封装
    └── common-ai/            # AI 能力封装
```

### 2.2 核心服务设计

#### 2.2.1 用户服务 (user-service)

```java
@RestController
@RequestMapping("/api/user")
public class UserController {
    
    @Autowired
    private UserService userService;
    
    @PostMapping("/register")
    public Result<UserVO> register(@RequestBody RegisterDTO dto) {
        return Result.success(userService.register(dto));
    }
    
    @PostMapping("/login")
    public Result<LoginVO> login(@RequestBody LoginDTO dto) {
        return Result.success(userService.login(dto));
    }
    
    @GetMapping("/profile")
    public Result<UserVO> getProfile() {
        Long userId = SecurityUtil.getCurrentUserId();
        return Result.success(userService.getUserById(userId));
    }
}

@Service
@Transactional
public class UserServiceImpl implements UserService {
    
    @Autowired
    private UserRepository userRepository;
    
    @Autowired
    private PasswordEncoder passwordEncoder;
    
    @Autowired
    private JwtUtil jwtUtil;
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    @Override
    public UserVO register(RegisterDTO dto) {
        // 检查用户名是否存在
        if (userRepository.existsByUsername(dto.getUsername())) {
            throw new BusinessException("用户名已存在");
        }
        
        // 创建用户
        User user = new User();
        user.setUsername(dto.getUsername());
        user.setPassword(passwordEncoder.encode(dto.getPassword()));
        user.setNickname(dto.getNickname());
        user.setEmail(dto.getEmail());
        user.setPhone(dto.getPhone());
        user.setStatus(1);
        
        userRepository.save(user);
        
        return convertToVO(user);
    }
    
    @Override
    public LoginVO login(LoginDTO dto) {
        // 查找用户
        User user = userRepository.findByUsername(dto.getUsername())
            .orElseThrow(() -> new AuthException("用户名或密码错误"));
        
        // 验证密码
        if (!passwordEncoder.matches(dto.getPassword(), user.getPassword())) {
            throw new AuthException("用户名或密码错误");
        }
        
        // 检查状态
        if (user.getStatus() != 1) {
            throw new AuthException("账号已被禁用");
        }
        
        // 生成 Token
        String token = jwtUtil.generateToken(user.getId(), user.getUsername());
        
        // 存入 Redis
        redisTemplate.opsForValue().set(
            "user:token:" + user.getId(),
            token,
            7,
            TimeUnit.DAYS
        );
        
        LoginVO vo = new LoginVO();
        vo.setToken(token);
        vo.setUser(convertToVO(user));
        
        return vo;
    }
}
```

#### 2.2.2 商品服务 (product-service)

```java
@Service
@Slf4j
public class ProductServiceImpl implements ProductService {
    
    @Autowired
    private ProductRepository productRepository;
    
    @Autowired
    private ElasticsearchRestTemplate esTemplate;
    
    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;
    
    @Override
    public Product saveProduct(Product product) {
        // 保存到 MySQL
        product = productRepository.save(product);
        
        // 同步到 ES
        esTemplate.save(convertToDocument(product));
        
        // 发送消息，触发特征计算
        kafkaTemplate.send("product.update", product.getId().toString(), product);
        
        return product;
    }
    
    @Override
    public Page<Product> searchProducts(ProductSearchDTO dto) {
        // ES 搜索
        NativeSearchQueryBuilder queryBuilder = new NativeSearchQueryBuilder();
        
        // 关键词搜索
        if (StringUtils.hasText(dto.getKeyword())) {
            queryBuilder.withQuery(
                QueryBuilders.multiMatchQuery(dto.getKeyword(), 
                    "title", "description", "categoryName")
                    .analyzer("ik_max_word")
            );
        }
        
        // 类目筛选
        if (StringUtils.hasText(dto.getCategoryId())) {
            queryBuilder.withFilter(
                QueryBuilders.termQuery("categoryId", dto.getCategoryId())
            );
        }
        
        // 价格范围
        if (dto.getMinPrice() != null || dto.getMaxPrice() != null) {
            RangeQueryBuilder rangeQuery = QueryBuilders.rangeQuery("price");
            if (dto.getMinPrice() != null) {
                rangeQuery.gte(dto.getMinPrice());
            }
            if (dto.getMaxPrice() != null) {
                rangeQuery.lte(dto.getMaxPrice());
            }
            queryBuilder.withFilter(rangeQuery);
        }
        
        // 排序
        if ("sales".equals(dto.getSortBy())) {
            queryBuilder.withSort(SortBuilders.fieldSort("sales").order(SortOrder.DESC));
        } else if ("price".equals(dto.getSortBy())) {
            queryBuilder.withSort(SortBuilders.fieldSort("price").order(SortOrder.ASC));
        }
        
        // 分页
        queryBuilder.withPageable(PageRequest.of(dto.getPage(), dto.getSize()));
        
        SearchHits<ProductDocument> hits = esTemplate.search(
            queryBuilder.build(),
            ProductDocument.class
        );
        
        // 转换结果
        List<Product> products = hits.stream()
            .map(hit -> convertToEntity(hit.getContent()))
            .collect(Collectors.toList());
        
        return new PageImpl<>(products, 
            PageRequest.of(dto.getPage(), dto.getSize()), 
            hits.getTotalHits());
    }
    
    @Override
    @Cacheable(value = "product:detail", key = "#productId")
    public Product getProductById(Long productId) {
        return productRepository.findById(productId)
            .orElseThrow(() -> new NotFoundException("商品不存在"));
    }
    
    @Override
    public List<Product> getHotProducts(String categoryId, int limit) {
        return productRepository.findTopByCategoryIdOrderByPredictScoreDesc(
            categoryId, 
            PageRequest.of(0, limit)
        );
    }
}
```

#### 2.2.3 预测服务 (predict-service)

```java
@Service
@Slf4j
public class PredictServiceImpl implements PredictService {
    
    @Autowired
    private ProductRepository productRepository;
    
    @Autowired
    private ProductSalesSnapshotRepository snapshotRepository;
    
    @Autowired
    private HotProductPredictionRepository predictionRepository;
    
    @Autowired
    private FeatureEngineClient featureClient;
    
    @Autowired
    private ModelService modelService;
    
    @Override
    public HotProductPrediction predict(Long productId) {
        // 获取商品信息
        Product product = productRepository.findById(productId)
            .orElseThrow(() -> new NotFoundException("商品不存在"));
        
        // 获取历史数据
        List<ProductSalesSnapshot> snapshots = snapshotRepository
            .findByProductIdOrderBySnapshotDateDesc(productId, PageRequest.of(0, 30));
        
        if (snapshots.size() < 7) {
            throw new BusinessException("数据不足，无法预测");
        }
        
        // 构建特征
        FeatureRequest featureRequest = buildFeatureRequest(product, snapshots);
        FeatureResponse features = featureClient.extractFeatures(featureRequest);
        
        // 调用模型预测
        PredictResult result = modelService.predict(features.getFeatures());
        
        // 构建预测结果
        HotProductPrediction prediction = new HotProductPrediction();
        prediction.setProductId(productId);
        prediction.setPredictDate(LocalDate.now().plusDays(7));
        prediction.setPredictScore(result.getScore());
        prediction.setConfidence(result.getConfidence());
        prediction.setFactors(result.getFactors());
        prediction.setModelVersion(modelService.getModelVersion());
        
        // 保存
        return predictionRepository.save(prediction);
    }
    
    @Override
    @Async
    public void batchPredict(List<Long> productIds) {
        log.info("开始批量预测，数量: {}", productIds.size());
        
        int success = 0;
        int failed = 0;
        
        for (Long productId : productIds) {
            try {
                predict(productId);
                success++;
            } catch (Exception e) {
                log.error("预测失败: productId={}", productId, e);
                failed++;
            }
        }
        
        log.info("批量预测完成，成功: {}, 失败: {}", success, failed);
    }
    
    @Override
    public List<HotProductPrediction> getTopPredictions(String categoryId, LocalDate date, int limit) {
        return predictionRepository.findByCategoryIdAndPredictDateOrderByPredictScoreDesc(
            categoryId, date, PageRequest.of(0, limit)
        );
    }
}
```

#### 2.2.4 采集服务 (crawl-service)

```java
@Service
@Slf4j
public class CrawlServiceImpl implements CrawlService {
    
    @Autowired
    private PlaywrightPool playwrightPool;
    
    @Autowired
    private ProxyManager proxyManager;
    
    @Autowired
    private ProductRepository productRepository;
    
    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;
    
    @Value("${crawl.douyin.base-url}")
    private String douyinBaseUrl;
    
    @Override
    public CrawlResult crawlDouyinProduct(String productId) {
        BrowserContext context = playwrightPool.acquire();
        Page page = context.newPage();
        
        try {
            // 设置请求拦截
            page.route("**/*", route -> {
                String type = route.request().resourceType();
                if (List.of("image", "media", "font", "stylesheet").contains(type)) {
                    route.abort();
                } else {
                    route.resume();
                }
            });
            
            // 访问商品页面
            String url = douyinBaseUrl + "/product/" + productId;
            page.navigate(url, new Page.NavigateOptions()
                .setWaitUntil(WaitUntilState.NETWORKIDLE)
                .setTimeout(30000));
            
            // 等待数据加载
            page.waitForSelector(".product-info", new Page.WaitForSelectorOptions()
                .setTimeout(10000));
            
            // 提取数据
            Product product = new Product();
            product.setPlatform("douyin");
            product.setProductId(productId);
            product.setTitle(page.locator(".product-title").textContent());
            product.setPrice(parsePrice(page.locator(".price").textContent()));
            product.setOriginalPrice(parsePrice(page.locator(".original-price").textContent()));
            product.setSales(parseSales(page.locator(".sales").textContent()));
            product.setDescription(page.locator(".description").textContent());
            
            // 图片
            List<String> images = page.locator(".product-images img").elementHandles()
                .stream()
                .map(el -> el.getAttribute("src"))
                .collect(Collectors.toList());
            product.setImages(String.join(",", images));
            
            // 保存
            product = productRepository.save(product);
            
            // 发送消息
            kafkaTemplate.send("crawl.product.success", product.getId().toString(), product);
            
            return CrawlResult.success(product);
            
        } catch (Exception e) {
            log.error("采集失败: {}", productId, e);
            return CrawlResult.fail(productId, e.getMessage());
            
        } finally {
            page.close();
            playwrightPool.release(context);
        }
    }
    
    @Override
    @Async("crawlExecutor")
    public void crawlCategory(String categoryId, int pages) {
        log.info("开始采集类目: {}, 页数: {}", categoryId, pages);
        
        for (int page = 1; page <= pages; page++) {
            try {
                List<String> productIds = crawlCategoryPage(categoryId, page);
                
                for (String productId : productIds) {
                    crawlDouyinProduct(productId);
                    Thread.sleep(1000 + new Random().nextInt(2000)); // 随机延迟
                }
                
            } catch (Exception e) {
                log.error("采集类目页失败: categoryId={}, page={}", categoryId, page, e);
            }
        }
        
        log.info("类目采集完成: {}", categoryId);
    }
}
```

---

## 三、数据库设计

### 3.1 MySQL 表结构

```sql
-- 用户表
CREATE TABLE `user` (
    `id` BIGINT PRIMARY KEY AUTO_INCREMENT,
    `username` VARCHAR(50) NOT NULL UNIQUE,
    `password` VARCHAR(255) NOT NULL,
    `nickname` VARCHAR(100),
    `email` VARCHAR(100),
    `phone` VARCHAR(20),
    `avatar` VARCHAR(500),
    `status` TINYINT DEFAULT 1 COMMENT '1正常 0禁用',
    `role` VARCHAR(20) DEFAULT 'USER' COMMENT 'USER ADMIN VIP',
    `vip_expire_time` DATETIME COMMENT 'VIP过期时间',
    `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
    `updated_at` DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX `idx_username` (`username`),
    INDEX `idx_phone` (`phone`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='用户表';

-- 用户配置表
CREATE TABLE `user_config` (
    `id` BIGINT PRIMARY KEY AUTO_INCREMENT,
    `user_id` BIGINT NOT NULL,
    `config_key` VARCHAR(100) NOT NULL,
    `config_value` TEXT,
    `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
    `updated_at` DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    UNIQUE KEY `uk_user_key` (`user_id`, `config_key`),
    FOREIGN KEY (`user_id`) REFERENCES `user`(`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='用户配置表';

-- 商品表
CREATE TABLE `product` (
    `id` BIGINT PRIMARY KEY AUTO_INCREMENT,
    `platform` VARCHAR(20) NOT NULL COMMENT 'douyin/taobao/1688',
    `product_id` VARCHAR(100) NOT NULL COMMENT '平台商品ID',
    `title` VARCHAR(500) NOT NULL,
    `description` TEXT,
    `price` DECIMAL(12,2) COMMENT '当前价格',
    `original_price` DECIMAL(12,2) COMMENT '原价',
    `sales` INT DEFAULT 0 COMMENT '销量',
    `monthly_sales` INT DEFAULT 0 COMMENT '月销量',
    `stock` INT DEFAULT 0 COMMENT '库存',
    `category_id` VARCHAR(100),
    `category_name` VARCHAR(200),
    `category_level` INT,
    `brand` VARCHAR(100),
    `supplier` VARCHAR(200),
    `commission_rate` DECIMAL(5,4) COMMENT '佣金率',
    `images` TEXT COMMENT '图片URL，逗号分隔',
    `video_url` VARCHAR(500),
    `detail_url` VARCHAR(500),
    `status` TINYINT DEFAULT 1 COMMENT '1上架 0下架 -1删除',
    `is_hot` TINYINT DEFAULT 0 COMMENT '是否爆品',
    `predict_score` DECIMAL(5,4) COMMENT '预测爆品分数',
    `tags` JSON COMMENT '标签',
    `extra` JSON COMMENT '扩展字段',
    `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
    `updated_at` DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    UNIQUE KEY `uk_platform_product` (`platform`, `product_id`),
    INDEX `idx_category` (`category_id`),
    INDEX `idx_sales` (`sales`),
    INDEX `idx_price` (`price`),
    INDEX `idx_predict_score` (`predict_score`),
    INDEX `idx_created_at` (`created_at`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='商品表';

-- 商品销量快照表（按日）
CREATE TABLE `product_sales_snapshot` (
    `id` BIGINT PRIMARY KEY AUTO_INCREMENT,
    `product_id` BIGINT NOT NULL,
    `snapshot_date` DATE NOT NULL,
    `sales` INT COMMENT '当日销量',
    `total_sales` INT COMMENT '累计销量',
    `price` DECIMAL(12,2) COMMENT '当日价格',
    `stock` INT COMMENT '库存',
    `viewer_count` INT COMMENT '曝光量',
    `click_count` INT COMMENT '点击量',
    `cart_count` INT COMMENT '加购量',
    `order_count` INT COMMENT '订单量',
    `refund_count` INT COMMENT '退款量',
    `commission` DECIMAL(12,2) COMMENT '佣金',
    `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
    UNIQUE KEY `uk_product_date` (`product_id`, `snapshot_date`),
    INDEX `idx_snapshot_date` (`snapshot_date`),
    FOREIGN KEY (`product_id`) REFERENCES `product`(`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='商品销量快照表';

-- 爆品预测表
CREATE TABLE `hot_product_prediction` (
    `id` BIGINT PRIMARY KEY AUTO_INCREMENT,
    `product_id` BIGINT NOT NULL,
    `category_id` VARCHAR(100),
    `predict_date` DATE NOT NULL COMMENT '预测日期',
    `predict_score` DECIMAL(5,4) NOT NULL COMMENT '爆品指数 0-1',
    `predict_sales` INT COMMENT '预测销量',
    `predict_range_low` INT COMMENT '预测销量下限',
    `predict_range_high` INT COMMENT '预测销量上限',
    `confidence` DECIMAL(5,4) COMMENT '置信度',
    `factors` JSON COMMENT '影响因素',
    `feature_importance` JSON COMMENT '特征重要性',
    `model_version` VARCHAR(32) COMMENT '模型版本',
    `actual_sales` INT COMMENT '实际销量（回填）',
    `is_accurate` TINYINT COMMENT '预测是否准确（回填）',
    `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
    INDEX `idx_product` (`product_id`),
    INDEX `idx_predict_date` (`predict_date`),
    INDEX `idx_predict_score` (`predict_score`),
    INDEX `idx_category_date` (`category_id`, `predict_date`),
    FOREIGN KEY (`product_id`) REFERENCES `product`(`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='爆品预测表';

-- 直播间表
CREATE TABLE `live_room` (
    `id` BIGINT PRIMARY KEY AUTO_INCREMENT,
    `platform` VARCHAR(20) NOT NULL,
    `room_id` VARCHAR(100) NOT NULL,
    `user_id` VARCHAR(100) NOT NULL COMMENT '主播ID',
    `nickname` VARCHAR(100),
    `avatar` VARCHAR(500),
    `title` VARCHAR(500),
    `cover_url` VARCHAR(500),
    `category` VARCHAR(100),
    `viewer_count` INT DEFAULT 0 COMMENT '观看人数',
    `like_count` INT DEFAULT 0 COMMENT '点赞数',
    `comment_count` INT DEFAULT 0 COMMENT '评论数',
    `share_count` INT DEFAULT 0 COMMENT '分享数',
    `gift_count` INT DEFAULT 0 COMMENT '礼物数',
    `sales_amount` DECIMAL(14,2) COMMENT '销售额',
    `order_count` INT COMMENT '订单数',
    `product_count` INT COMMENT '商品数',
    `start_time` DATETIME COMMENT '开播时间',
    `end_time` DATETIME COMMENT '结束时间',
    `duration` INT COMMENT '时长(秒)',
    `status` TINYINT DEFAULT 1 COMMENT '1直播中 0已结束',
    `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
    `updated_at` DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    UNIQUE KEY `uk_platform_room` (`platform`, `room_id`),
    INDEX `idx_user_id` (`user_id`),
    INDEX `idx_start_time` (`start_time`),
    INDEX `idx_status` (`status`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='直播间表';

-- 直播商品关联表
CREATE TABLE `live_product` (
    `id` BIGINT PRIMARY KEY AUTO_INCREMENT,
    `live_room_id` BIGINT NOT NULL,
    `product_id` BIGINT NOT NULL,
    `show_time` DATETIME COMMENT '展示时间',
    `show_duration` INT COMMENT '展示时长(秒)',
    `click_count` INT COMMENT '点击次数',
    `order_count` INT COMMENT '订单数',
    `sales_amount` DECIMAL(12,2) COMMENT '销售额',
    `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
    UNIQUE KEY `uk_live_product` (`live_room_id`, `product_id`),
    INDEX `idx_product_id` (`product_id`),
    FOREIGN KEY (`live_room_id`) REFERENCES `live_room`(`id`),
    FOREIGN KEY (`product_id`) REFERENCES `product`(`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='直播商品关联表';

-- 达人表
CREATE TABLE `influencer` (
    `id` BIGINT PRIMARY KEY AUTO_INCREMENT,
    `platform` VARCHAR(20) NOT NULL,
    `user_id` VARCHAR(100) NOT NULL COMMENT '平台用户ID',
    `nickname` VARCHAR(100),
    `avatar` VARCHAR(500),
    `signature` VARCHAR(500),
    `follower_count` INT DEFAULT 0,
    `following_count` INT DEFAULT 0,
    `video_count` INT DEFAULT 0,
    `live_count` INT DEFAULT 0,
    `total_sales` DECIMAL(14,2) COMMENT '总销售额',
    `total_order` INT COMMENT '总订单数',
    `avg_sales` DECIMAL(12,2) COMMENT '场均销售额',
    `category_expertise` JSON COMMENT '擅长类目',
    `tags` JSON COMMENT '标签',
    `level` INT COMMENT '达人等级',
    `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
    `updated_at` DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    UNIQUE KEY `uk_platform_user` (`platform`, `user_id`),
    INDEX `idx_follower` (`follower_count`),
    INDEX `idx_total_sales` (`total_sales`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='达人表';

-- 采集任务表
CREATE TABLE `crawl_task` (
    `id` BIGINT PRIMARY KEY AUTO_INCREMENT,
    `task_id` VARCHAR(64) NOT NULL UNIQUE,
    `task_name` VARCHAR(200),
    `platform` VARCHAR(20) NOT NULL,
    `task_type` VARCHAR(50) COMMENT 'PRODUCT/CATEGORY/LIVE/INFLUENCER',
    `target_id` VARCHAR(100) COMMENT '目标ID',
    `params` JSON COMMENT '任务参数',
    `priority` INT DEFAULT 5 COMMENT '优先级 1-10',
    `status` VARCHAR(20) DEFAULT 'PENDING' COMMENT 'PENDING/RUNNING/SUCCESS/FAILED',
    `progress` INT DEFAULT 0 COMMENT '进度百分比',
    `data_count` INT DEFAULT 0 COMMENT '采集数据量',
    `error_message` TEXT,
    `start_time` DATETIME,
    `end_time` DATETIME,
    `duration` INT COMMENT '执行时长(秒)',
    `retry_count` INT DEFAULT 0,
    `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
    `updated_at` DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX `idx_status` (`status`),
    INDEX `idx_platform_type` (`platform`, `task_type`),
    INDEX `idx_created_at` (`created_at`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='采集任务表';

-- 预警规则表
CREATE TABLE `alert_rule` (
    `id` BIGINT PRIMARY KEY AUTO_INCREMENT,
    `name` VARCHAR(100) NOT NULL,
    `type` VARCHAR(50) COMMENT 'SALES/PRICE/COMPETITOR/PREDICTION',
    `condition` JSON COMMENT '触发条件',
    `channels` JSON COMMENT '通知渠道',
    `receivers` JSON COMMENT '接收人',
    `enabled` TINYINT DEFAULT 1,
    `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
    `updated_at` DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='预警规则表';

-- 预警记录表
CREATE TABLE `alert_log` (
    `id` BIGINT PRIMARY KEY AUTO_INCREMENT,
    `rule_id` BIGINT,
    `product_id` BIGINT,
    `alert_type` VARCHAR(50),
    `alert_level` VARCHAR(20) COMMENT 'INFO/WARNING/ERROR',
    `message` TEXT,
    `detail` JSON,
    `is_read` TINYINT DEFAULT 0,
    `is_handled` TINYINT DEFAULT 0,
    `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
    INDEX `idx_product` (`product_id`),
    INDEX `idx_created_at` (`created_at`),
    FOREIGN KEY (`rule_id`) REFERENCES `alert_rule`(`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='预警记录表';
```

### 3.2 MongoDB 集合设计

```javascript
// 商品详情日志
db.product_detail_log.insertOne({
    product_id: Long("123456"),
    platform: "douyin",
    snapshot_date: ISODate("2026-03-19"),
    raw_data: {
        // 原始采集数据
    },
    processed_data: {
        // 处理后数据
    },
    created_at: ISODate("2026-03-19T10:00:00Z")
});

// 直播帧截图日志
db.live_frame_log.insertOne({
    live_room_id: Long("789"),
    frame_time: ISODate("2026-03-19T10:00:00Z"),
    image_url: "https://xxx/frame.jpg",
    ocr_text: "今日特价 99元",
    parsed_data: {
        products: [
            { name: "商品A", price: 99 }
        ]
    },
    created_at: ISODate("2026-03-19T10:00:00Z")
});

// AI 分析日志
db.ai_analysis_log.insertOne({
    request_id: "uuid-xxx",
    model: "gpt-4",
    prompt_tokens: 1000,
    completion_tokens: 500,
    total_tokens: 1500,
    cost: 0.03,
    latency_ms: 2000,
    request: "...",
    response: "...",
    created_at: ISODate("2026-03-19T10:00:00Z")
});
```

### 3.3 Elasticsearch 索引设计

```json
// 商品索引
PUT /product
{
    "mappings": {
        "properties": {
            "id": { "type": "long" },
            "platform": { "type": "keyword" },
            "productId": { "type": "keyword" },
            "title": {
                "type": "text",
                "analyzer": "ik_max_word",
                "search_analyzer": "ik_smart"
            },
            "description": {
                "type": "text",
                "analyzer": "ik_max_word"
            },
            "price": { "type": "scaled_float", "scaling_factor": 100 },
            "originalPrice": { "type": "scaled_float", "scaling_factor": 100 },
            "sales": { "type": "integer" },
            "monthlySales": { "type": "integer" },
            "categoryId": { "type": "keyword" },
            "categoryName": { "type": "keyword" },
            "brand": { "type": "keyword" },
            "commissionRate": { "type": "scaled_float", "scaling_factor": 10000 },
            "images": { "type": "keyword" },
            "status": { "type": "integer" },
            "isHot": { "type": "boolean" },
            "predictScore": { "type": "scaled_float", "scaling_factor": 10000 },
            "tags": { "type": "keyword" },
            "createdAt": { "type": "date" },
            "updatedAt": { "type": "date" }
        }
    }
}
```

---

## 四、缓存设计

### 4.1 Redis 缓存策略

```java
@Configuration
public class RedisConfig {
    
    @Bean
    public RedisTemplate<String, Object> redisTemplate(
            RedisConnectionFactory connectionFactory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(connectionFactory);
        
        // Key 序列化
        template.setKeySerializer(new StringRedisSerializer());
        template.setHashKeySerializer(new StringRedisSerializer());
        
        // Value 序列化
        template.setValueSerializer(new GenericJackson2JsonRedisSerializer());
        template.setHashValueSerializer(new GenericJackson2JsonRedisSerializer());
        
        template.afterPropertiesSet();
        return template;
    }
    
    @Bean
    public CacheManager cacheManager(RedisConnectionFactory connectionFactory) {
        RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofHours(1))
            .serializeKeysWith(RedisSerializationContext.SerializationPair
                .fromSerializer(new StringRedisSerializer()))
            .serializeValuesWith(RedisSerializationContext.SerializationPair
                .fromSerializer(new GenericJackson2JsonRedisSerializer()));
        
        return RedisCacheManager.builder(connectionFactory)
            .cacheDefaults(config)
            .build();
    }
}
```

### 4.2 缓存 Key 设计

```
# 商品详情
product:detail:{productId}              # 商品详情，TTL=1h
product:stats:{productId}               # 商品统计，TTL=5min

# 预测结果
predict:hot:{productId}:{date}          # 预测结果，TTL=24h
predict:top:{categoryId}:{date}         # 热门预测列表，TTL=1h

# 用户相关
user:token:{userId}                     # 用户Token，TTL=7d
user:config:{userId}                    # 用户配置，TTL=24h

# 采集相关
crawl:checkpoint:{taskId}               # 采集进度，TTL=7d
crawl:proxy:pool                        # 代理池，TTL=根据代理有效期
crawl:rate:{platform}:{target}          # 频率限制，TTL=1min

# 分布式锁
lock:predict:{productId}                # 预测锁，TTL=30s
lock:crawl:{taskId}                     # 采集锁，TTL=根据任务

# 实时数据
live:realtime:{roomId}                  # 直播实时数据，TTL=1min
```

---

## 五、消息队列设计

### 5.1 Kafka Topic 设计

```yaml
# 商品相关
product.created     # 商品创建事件
product.updated     # 商品更新事件
product.deleted     # 商品删除事件

# 采集相关
crawl.task.created  # 采集任务创建
crawl.task.progress # 采集进度更新
crawl.data.raw      # 原始采集数据
crawl.data.parsed   # 解析后数据

# 预测相关
predict.task.created    # 预测任务创建
predict.task.completed  # 预测任务完成
predict.alert.hot       # 爆品预警

# 通知相关
notify.email        # 邮件通知
notify.sms          # 短信通知
notify.qq           # QQ 消息
notify.wechat       # 微信消息
```

### 5.2 消费者设计

```java
@Component
@Slf4j
public class ProductEventConsumer {
    
    @Autowired
    private ElasticsearchRestTemplate esTemplate;
    
    @Autowired
    private FeatureService featureService;
    
    @KafkaListener(
        topics = "product.updated",
        groupId = "product-es-sync",
        containerFactory = "kafkaListenerContainerFactory"
    )
    public void syncToES(ConsumerRecord<String, Product> record) {
        Product product = record.value();
        log.info("同步商品到ES: {}", product.getId());
        
        try {
            ProductDocument doc = convertToDocument(product);
            esTemplate.save(doc);
        } catch (Exception e) {
            log.error("同步ES失败", e);
            // 发送到死信队列
        }
    }
    
    @KafkaListener(
        topics = "product.updated",
        groupId = "product-feature-update",
        containerFactory = "kafkaListenerContainerFactory"
    )
    public void updateFeatures(ConsumerRecord<String, Product> record) {
        Product product = record.value();
        log.info("更新商品特征: {}", product.getId());
        
        try {
            featureService.calculateFeatures(product.getId());
        } catch (Exception e) {
            log.error("特征计算失败", e);
        }
    }
}
```

---

## 六、AI 能力层设计

### 6.1 AI Gateway

```java
@Service
@Slf4j
public class AIGateway {
    
    private final Map<String, AIProvider> providers = new ConcurrentHashMap<>();
    
    @Value("${ai.default-provider}")
    private String defaultProvider;
    
    @PostConstruct
    public void init() {
        // 注册各 AI 提供商
        providers.put("openai", new OpenAIProvider());
        providers.put("claude", new ClaudeProvider());
        providers.put("deepseek", new DeepSeekProvider());
        providers.put("qwen", new QwenProvider());
    }
    
    /**
     * 统一聊天接口
     */
    public ChatResponse chat(ChatRequest request) {
        // 选择提供商
        String provider = request.getProvider() != null ? 
            request.getProvider() : defaultProvider;
        
        AIProvider aiProvider = providers.get(provider);
        if (aiProvider == null) {
            throw new BusinessException("不支持的AI提供商: " + provider);
        }
        
        // 检查配额
        checkQuota(request.getUserId());
        
        // 内容审核
        if (needContentReview(request)) {
            reviewContent(request);
        }
        
        // 调用 AI
        long startTime = System.currentTimeMillis();
        ChatResponse response = aiProvider.chat(request);
        long latency = System.currentTimeMillis() - startTime;
        
        // 记录日志
        logAIRequest(request, response, latency);
        
        // 扣除配额
        deductQuota(request.getUserId(), response.getUsage());
        
        return response;
    }
    
    /**
     * 智能路由 - 根据任务选择最优模型
     */
    public ChatResponse smartChat(ChatRequest request) {
        // 分析任务类型
        TaskType taskType = analyzeTask(request);
        
        // 选择最优模型
        String bestProvider = selectBestProvider(taskType);
        request.setProvider(bestProvider);
        
        return chat(request);
    }
    
    private String selectBestProvider(TaskType taskType) {
        return switch (taskType) {
            case CODE -> "claude";
            case CREATIVE -> "openai";
            case ANALYSIS -> "claude";
            case TRANSLATION -> "deepseek";
            default -> defaultProvider;
        };
    }
}
```

### 6.2 向量检索

```java
@Service
public class VectorSearchService {
    
    @Autowired
    private MilvusClient milvusClient;
    
    @Autowired
    private EmbeddingService embeddingService;
    
    /**
     * 索引商品向量
     */
    public void indexProduct(Product product) {
        // 生成文本嵌入
        String text = product.getTitle() + " " + product.getDescription();
        float[] embedding = embeddingService.embed(text);
        
        // 插入 Milvus
        InsertParam insertParam = InsertParam.newBuilder()
            .withCollectionName("products")
            .withFields(Arrays.asList(
                new InsertParam.Field("id", Collections.singletonList(product.getId())),
                new InsertParam.Field("embedding", Collections.singletonList(embedding)),
                new InsertParam.Field("title", Collections.singletonList(product.getTitle()))
            ))
            .build();
        
        milvusClient.insert(insertParam);
    }
    
    /**
     * 相似商品搜索
     */
    public List<Long> searchSimilarProducts(String query, int topK) {
        // 生成查询向量
        float[] queryEmbedding = embeddingService.embed(query);
        
        // 搜索
        SearchParam searchParam = SearchParam.newBuilder()
            .withCollectionName("products")
            .withMetricType(MetricType.L2)
            .withTopK(topK)
            .withVectors(Collections.singletonList(queryEmbedding))
            .withVectorFieldName("embedding")
            .build();
        
        R<SearchResults> result = milvusClient.search(searchParam);
        
        return result.getData().getResults().get(0).stream()
            .map(score -> score.getLongId())
            .collect(Collectors.toList());
    }
}
```

---

## 七、监控与运维

### 7.1 Prometheus 指标

```java
@Component
public class BusinessMetrics {
    
    private final Counter predictCounter;
    private final Counter crawlCounter;
    private final Histogram predictLatency;
    private final Gauge activeUsers;
    
    public BusinessMetrics(MeterRegistry registry) {
        // 预测计数
        predictCounter = Counter.builder("predict_total")
            .description("Total predictions")
            .tag("status", "success")
            .register(registry);
        
        // 采集计数
        crawlCounter = Counter.builder("crawl_total")
            .description("Total crawl tasks")
            .tag("platform", "unknown")
            .register(registry);
        
        // 预测延迟
        predictLatency = Histogram.builder("predict_latency_seconds")
            .description("Prediction latency")
            .register(registry);
        
        // 活跃用户
        activeUsers = Gauge.builder("active_users")
            .description("Active users count")
            .register(registry);
    }
    
    public void recordPredict(String status) {
        predictCounter.increment();
    }
    
    public void recordPredictLatency(double seconds) {
        predictLatency.observe(seconds);
    }
}
```

### 7.2 告警规则

```yaml
# Prometheus 告警规则
groups:
  - name: ai-predict-alerts
    rules:
      # 服务可用性
      - alert: ServiceDown
        expr: up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "服务 {{ $labels.instance }} 已下线"
      
      # 预测延迟
      - alert: PredictLatencyHigh
        expr: histogram_quantile(0.95, rate(predict_latency_seconds_bucket[5m])) > 5
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "预测延迟过高"
      
      # 错误率
      - alert: HighErrorRate
        expr: sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m])) / sum(rate(http_server_requests_seconds_count[5m])) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "错误率超过 5%"
      
      # 采集失败
      - alert: CrawlFailureHigh
        expr: sum(rate(crawl_total{status="failed"}[1h])) / sum(rate(crawl_total[1h])) > 0.1
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "采集失败率超过 10%"
      
      # AI API 调用失败
      - alert: AIAPIFailure
        expr: increase(ai_api_errors_total[5m]) > 10
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "AI API 调用频繁失败"
```

---

## 八、部署架构

### 8.1 Kubernetes 部署

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: predict-service
  namespace: ai-predict
spec:
  replicas: 3
  selector:
    matchLabels:
      app: predict-service
  template:
    metadata:
      labels:
        app: predict-service
    spec:
      containers:
        - name: predict-service
          image: registry.example.com/ai-predict/predict-service:latest
          ports:
            - containerPort: 8080
          resources:
            requests:
              memory: "512Mi"
              cpu: "250m"
            limits:
              memory: "1Gi"
              cpu: "500m"
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: "prod"
            - name: JAVA_OPTS
              value: "-Xms512m -Xmx768m"
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5
          volumeMounts:
            - name: config
              mountPath: /config
      volumes:
        - name: config
          configMap:
            name: predict-service-config
---
apiVersion: v1
kind: Service
metadata:
  name: predict-service
  namespace: ai-predict
spec:
  selector:
    app: predict-service
  ports:
    - port: 8080
      targetPort: 8080
  type: ClusterIP
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: predict-service-hpa
  namespace: ai-predict
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: predict-service
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
```

### 8.2 基础设施清单

```yaml
# Kubernetes 资源清单
Namespace: ai-predict

# StatefulSets (有状态服务)
- mysql (主从复制)
- redis-cluster (6节点)
- kafka (3节点)
- zookeeper (3节点)
- elasticsearch (3节点)
- milvus (单节点开发/集群生产)

# Deployments (无状态服务)
- gateway-service (2+ replicas)
- user-service (2+ replicas)
- product-service (2+ replicas)
- predict-service (3+ replicas)
- crawl-service (3+ replicas)
- notify-service (2+ replicas)
- scheduler-service (2+ replicas)

# Jobs (定时任务)
- daily-prediction-job (CronJob)
- daily-report-job (CronJob)
- data-cleanup-job (CronJob)

# ConfigMaps
- application-config
- ai-provider-config
- alert-rules-config

# Secrets
- db-credentials
- redis-credentials
- ai-api-keys
- notification-secrets

# Ingress
- ai-predict-ingress (域名路由)

# PersistentVolumes
- mysql-pv (100Gi)
- kafka-pv (50Gi x 3)
- es-pv (100Gi x 3)
- minio-pv (500Gi)
```

---

## 九、安全设计

### 9.1 认证授权

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(AbstractHttpConfigurer::disable)
            .sessionManagement(session -> 
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**").permitAll()
                .requestMatchers("/api/public/**").permitAll()
                .requestMatchers("/actuator/health/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .addFilterBefore(jwtAuthenticationFilter(), 
                UsernamePasswordAuthenticationFilter.class);
        
        return http.build();
    }
    
    @Bean
    public JwtAuthenticationFilter jwtAuthenticationFilter() {
        return new JwtAuthenticationFilter();
    }
}
```

### 9.2 数据脱敏

```java
@Aspect
@Component
public class DataMaskAspect {
    
    @AfterReturning(pointcut = "execution(* com.aipredict..service.*.*(..))", returning = "result")
    public void maskSensitiveData(JoinPoint joinPoint, Object result) {
        if (result instanceof UserVO) {
            UserVO user = (UserVO) result;
            user.setPhone(maskPhone(user.getPhone()));
            user.setEmail(maskEmail(user.getEmail()));
        }
    }
    
    private String maskPhone(String phone) {
        if (phone == null || phone.length() < 7) return phone;
        return phone.substring(0, 3) + "****" + phone.substring(phone.length() - 4);
    }
    
    private String maskEmail(String email) {
        if (email == null || !email.contains("@")) return email;
        int at = email.indexOf("@");
        return email.substring(0, Math.min(2, at)) + "***" + email.substring(at);
    }
}
```

---

## 十、总结

### 10.1 架构亮点

1. **微服务架构** - 服务解耦，独立部署扩展
2. **数据分层** - MySQL + ES + ClickHouse 多引擎协作
3. **AI 能力层** - 统一网关，多模型支持
4. **实时处理** - Kafka + Flink 实时数据流
5. **弹性伸缩** - K8s HPA 自动扩缩容

### 10.2 技术选型理由

| 选择 | 理由 |
|------|------|
| Spring Boot 3 | 生态成熟，团队熟悉 |
| Kafka | 高吞吐，适合采集数据流 |
| ClickHouse | 列式存储，分析查询快 |
| Milvus | 开源向量数据库，商品相似度搜索 |
| Kubernetes | 云原生，自动化运维 |

### 10.3 后续优化

1. 引入 Service Mesh (Istio)
2. 完善可观测性 (OpenTelemetry)
3. 多活容灾架构
4. 边缘计算节点部署