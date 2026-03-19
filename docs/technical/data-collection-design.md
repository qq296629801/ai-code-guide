# 数据采集方案 - 详细设计文档

> 版本：v1.0  
> 日期：2026-03-19

---

## 一、概述

### 1.1 设计原则

- **合规优先** - 只采集公开数据，遵守平台规则
- **稳定性** - 支持断点续爬、错误重试、异常恢复
- **可扩展** - 模块化设计，易于新增数据源
- **高效率** - 分布式采集，支持并发处理

### 1.2 数据源概览

| 数据源 | 采集方式 | 数据类型 | 更新频率 |
|--------|----------|----------|----------|
| 抖音 | 网页爬虫 + APP接口 | 直播、商品、达人 | 实时/每小时 |
| 淘宝 | 开放API + 网页爬虫 | 商品、订单、评价 | 每小时 |
| 1688 | 网页爬虫 | 商品、批发价 | 每天 |
| 第三方平台 | API对接 | 汇总数据 | 每小时 |

---

## 二、抖音数据采集

### 2.1 网页端采集

#### 2.1.1 技术选型

```
核心工具：Playwright (Python/Java)
代理方案：动态代理池
验证码处理：打码平台 / 手动接管
```

#### 2.1.2 采集流程

```
┌─────────────────────────────────────────────────────────────┐
│                      采集调度器                              │
│         任务队列 │ 优先级管理 │ 失败重试                     │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                      代理管理器                              │
│     IP池管理 │ 健康检测 │ 自动切换 │ 指纹伪装               │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    浏览器实例池                              │
│  ┌─────┐  ┌─────┐  ┌─────┐  ┌─────┐  ┌─────┐              │
│  │实例1│  │实例2│  │实例3│  │实例4│  │实例N│              │
│  └─────┘  └─────┘  └─────┘  └─────┘  └─────┘              │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                      数据解析器                              │
│     HTML解析 │ JSON提取 │ 字段清洗 │ 格式转换               │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                      数据存储                                │
│     MySQL │ Redis │ Elasticsearch │ MongoDB                │
└─────────────────────────────────────────────────────────────┘
```

#### 2.1.3 核心代码实现

**Java + Playwright 实现：**

```java
@Service
@Slf4j
public class DouyinWebCrawler {
    
    private final Playwright playwright;
    private final BrowserContextPool contextPool;
    private final ProxyManager proxyManager;
    
    /**
     * 采集用户主页数据
     */
    public UserInfo crawlUserInfo(String secUserId) {
        BrowserContext context = contextPool.acquire();
        Page page = context.newPage();
        
        try {
            // 设置请求拦截，屏蔽不必要资源
            page.route("**/*", route -> {
                String resourceType = route.request().resourceType();
                if (List.of("image", "media", "font", "stylesheet").contains(resourceType)) {
                    route.abort();
                } else {
                    route.resume();
                }
            });
            
            // 访问用户主页
            String url = "https://www.douyin.com/user/" + secUserId;
            page.navigate(url, new Page.NavigateOptions()
                .setWaitUntil(WaitUntilState.NETWORKIDLE)
                .setTimeout(30000));
            
            // 等待数据加载
            page.waitForSelector(".user-info", new Page.WaitForSelectorOptions()
                .setTimeout(10000));
            
            // 提取数据
            UserInfo user = new UserInfo();
            user.setSecUserId(secUserId);
            user.setNickname(page.locator(".nickname").textContent());
            user.setSignature(page.locator(".signature").textContent());
            user.setFollowerCount(extractNumber(page.locator(".follower-count").textContent()));
            user.setVideoCount(extractNumber(page.locator(".video-count").textContent()));
            
            return user;
            
        } catch (Exception e) {
            log.error("采集用户信息失败: {}", secUserId, e);
            throw new CrawlException("采集失败", e);
        } finally {
            page.close();
            contextPool.release(context);
        }
    }
    
    /**
     * 采集直播数据
     */
    public LiveInfo crawlLiveInfo(String roomId) {
        BrowserContext context = contextPool.acquire();
        Page page = context.newPage();
        
        try {
            // 监听 API 响应
            CompletableFuture<LiveInfo> future = new CompletableFuture<>();
            
            page.onResponse(response -> {
                if (response.url().contains("/webcast/room/reflow/info/")) {
                    try {
                        JsonObject json = JsonParser.parseString(response.text()).getAsJsonObject();
                        LiveInfo live = parseLiveInfo(json);
                        future.complete(live);
                    } catch (Exception e) {
                        future.completeExceptionally(e);
                    }
                }
            });
            
            // 访问直播间
            page.navigate("https://live.douyin.com/" + roomId);
            page.waitForTimeout(5000);
            
            return future.get(10, TimeUnit.SECONDS);
            
        } catch (Exception e) {
            throw new CrawlException("直播数据采集失败", e);
        } finally {
            page.close();
            contextPool.release(context);
        }
    }
    
    /**
     * 采集商品数据
     */
    public ProductInfo crawlProduct(String productId) {
        String apiUrl = "https://haohuo.jinritemai.com/eweapon/api/product/info";
        
        // 构造请求
        Map<String, Object> params = new HashMap<>();
        params.put("product_id", productId);
        
        // 发送请求（带签名）
        String response = httpClient.post(apiUrl, params, headers);
        
        return parseProductInfo(response);
    }
}
```

#### 2.1.4 反爬策略应对

| 反爬机制 | 应对策略 |
|----------|----------|
| 登录验证 | Cookie池管理、扫码登录 |
| 滑块验证码 | 打码平台自动识别 / 人工介入 |
| 频率限制 | 代理IP轮换、随机延迟 |
| 设备指纹 | 指纹伪装、浏览器实例复用 |
| 行为检测 | 模拟人类行为（滚动、点击） |

**代理池实现：**

```java
@Service
public class ProxyManager {
    
    private final Queue<ProxyInfo> proxyPool = new ConcurrentLinkedQueue<>();
    private final Map<String, ProxyStats> proxyStats = new ConcurrentHashMap<>();
    
    /**
     * 获取可用代理
     */
    public ProxyInfo acquire() {
        int maxRetries = 3;
        for (int i = 0; i < maxRetries; i++) {
            ProxyInfo proxy = proxyPool.poll();
            if (proxy != null && checkProxy(proxy)) {
                return proxy;
            }
            // 不可用，移除
            proxyPool.remove(proxy);
        }
        // 池中无可用代理，获取新的
        return fetchNewProxy();
    }
    
    /**
     * 代理健康检查
     */
    private boolean checkProxy(ProxyInfo proxy) {
        try {
            HttpURLConnection conn = (HttpURLConnection) 
                new URL("https://www.douyin.com").openConnection(new Proxy(
                    Proxy.Type.HTTP, 
                    new InetSocketAddress(proxy.getHost(), proxy.getPort())
                ));
            conn.setConnectTimeout(5000);
            conn.setReadTimeout(5000);
            return conn.getResponseCode() == 200;
        } catch (Exception e) {
            return false;
        }
    }
    
    /**
     * 从代理服务商获取新代理
     */
    private ProxyInfo fetchNewProxy() {
        // 调用代理服务商API
        String apiUrl = "https://api.proxy-provider.com/get?format=json";
        String response = httpClient.get(apiUrl);
        
        JsonObject json = JsonParser.parseString(response).getAsJsonObject();
        return new ProxyInfo(
            json.get("ip").getAsString(),
            json.get("port").getAsInt(),
            json.get("expire").getAsLong()
        );
    }
    
    /**
     * 释放代理（记录使用情况）
     */
    public void release(ProxyInfo proxy, boolean success) {
        ProxyStats stats = proxyStats.computeIfAbsent(
            proxy.getKey(), k -> new ProxyStats()
        );
        
        if (success) {
            stats.incrementSuccess();
        } else {
            stats.incrementFailure();
        }
        
        // 成功率高于50%才放回池中
        if (stats.getSuccessRate() > 0.5 && !proxy.isExpired()) {
            proxyPool.offer(proxy);
        }
    }
}
```

---

### 2.2 APP 端接口采集

#### 2.2.1 接口签名分析

抖音 APP 的 API 请求通常包含以下签名参数：

```
常见签名参数：
- _signature: 请求签名
- X-Gorgon: 设备指纹签名
- X-Khronos: 时间戳
- X-Argus: 反作弊参数
- X-Ladon: 加密参数
```

#### 2.2.2 签名生成方案

**方案一：逆向分析（推荐）**

```
步骤：
1. 使用 Frida/Xposed Hook 关键函数
2. 定位签名生成逻辑
3. 提取加密算法
4. 还原签名生成代码
```

**Frida Hook 示例：**

```javascript
// hook_signature.js
Java.perform(function() {
    var SignUtils = Java.use("com.ss.sys.ces.a");
    
    SignUtils.sign.implementation = function(timestamp, url, body) {
        console.log("timestamp: " + timestamp);
        console.log("url: " + url);
        console.log("body: " + body);
        
        var result = this.sign(timestamp, url, body);
        console.log("signature: " + result);
        
        return result;
    };
});
```

**方案二：协议转发**

```
架构：
┌─────────┐     ┌─────────────┐     ┌─────────┐
│  APP    │────▶│  中间人代理  │────▶│ 抖音API │
└─────────┘     └─────────────┘     └─────────┘
                      │
                      ▼
                ┌─────────────┐
                │  签名服务    │
                │  (Frida RPC)│
                └─────────────┘
```

#### 2.2.3 接口封装

```java
@Service
public class DouyinApiCrawler {
    
    private final SignService signService;
    private final DeviceManager deviceManager;
    
    /**
     * 获取用户详细信息
     */
    public UserDetail getUserDetail(String userId) {
        String url = "https://api.amemv.com/aweme/v1/web/user/profile/";
        
        // 获取设备信息
        DeviceInfo device = deviceManager.getAvailableDevice();
        
        // 构造参数
        Map<String, String> params = new LinkedHashMap<>();
        params.put("user_id", userId);
        params.put("device_id", device.getDeviceId());
        params.put("iid", device.getInstallId());
        params.put("openudid", device.getOpenUdid());
        // ... 其他参数
        
        // 生成签名
        String timestamp = String.valueOf(System.currentTimeMillis() / 1000);
        String signature = signService.generateSignature(url, params, timestamp);
        
        // 设置请求头
        Map<String, String> headers = new HashMap<>();
        headers.put("X-Gorgon", signature);
        headers.put("X-Khronos", timestamp);
        headers.put("User-Agent", device.getUserAgent());
        headers.put("Cookie", device.getCookies());
        
        // 发送请求
        String response = httpClient.get(url, params, headers);
        
        return parseUserDetail(response);
    }
    
    /**
     * 获取商品列表
     */
    public List<ProductInfo> getProductList(String categoryId, int page) {
        String url = "https://api.amemv.com/aweme/v1/promotion/product/list/";
        
        DeviceInfo device = deviceManager.getAvailableDevice();
        
        Map<String, String> params = new LinkedHashMap<>();
        params.put("category_id", categoryId);
        params.put("page", String.valueOf(page));
        params.put("page_size", "20");
        params.put("sort_type", "1"); // 1:销量 2:价格
        
        String signature = signService.generateSignature(url, params);
        
        // ... 发送请求
        return parseProductList(response);
    }
    
    /**
     * 获取直播间数据
     */
    public LiveRoomInfo getLiveRoomInfo(String roomId) {
        String url = "https://api.amemv.com/aweme/v1/webcast/room/info/";
        
        Map<String, String> params = new LinkedHashMap<>();
        params.put("room_id", roomId);
        params.put("type", "1");
        
        // ... 发送请求
        return parseLiveRoomInfo(response);
    }
}
```

---

### 2.3 直播流截取

#### 2.3.1 直播流获取

```
直播流地址格式：
https://pull-flv-xxx.douyin.com/webcast/xxx.flv
https://rtmp-xxx.douyin.com/live/xxx
```

**获取流程：**

```java
@Service
public class LiveStreamCrawler {
    
    /**
     * 获取直播流地址
     */
    public List<String> getLiveStreamUrls(String roomId) {
        // 从直播页面或API获取流地址
        String apiUrl = "https://api.amemv.com/aweme/v1/webcast/room/info/?room_id=" + roomId;
        String response = httpClient.get(apiUrl);
        
        JsonObject json = JsonParser.parseString(response).getAsJsonObject();
        JsonObject streamUrl = json.getAsJsonObject("data")
            .getAsJsonObject("stream_url");
        
        List<String> urls = new ArrayList<>();
        
        // FLV流
        if (streamUrl.has("flv_pull_url")) {
            JsonObject flvUrls = streamUrl.getAsJsonObject("flv_pull_url");
            for (String key : flvUrls.keySet()) {
                urls.add(flvUrls.get(key).getAsString());
            }
        }
        
        // HLS流
        if (streamUrl.has("hls_pull_url")) {
            urls.add(streamUrl.get("hls_pull_url").getAsString());
        }
        
        return urls;
    }
}
```

#### 2.3.2 FFmpeg 截帧

```java
@Service
public class LiveFrameCapture {
    
    /**
     * 从直播流截取帧
     */
    public byte[] captureFrame(String streamUrl) throws Exception {
        // 使用FFmpeg截取一帧
        ProcessBuilder pb = new ProcessBuilder(
            "ffmpeg",
            "-i", streamUrl,
            "-ss", "0",
            "-frames:v", "1",
            "-f", "image2pipe",
            "-vcodec", "png",
            "-"
        );
        
        pb.redirectErrorStream(true);
        Process process = pb.start();
        
        ByteArrayOutputStream output = new ByteArrayOutputStream();
        IOUtils.copy(process.getInputStream(), output);
        
        int exitCode = process.waitFor();
        if (exitCode != 0) {
            throw new RuntimeException("FFmpeg截帧失败");
        }
        
        return output.toByteArray();
    }
    
    /**
     * 定时截帧任务
     */
    @Scheduled(fixedRate = 5000) // 每5秒截一帧
    public void scheduledCapture() {
        List<String> activeRooms = getActiveLiveRooms();
        
        for (String roomId : activeRooms) {
            try {
                List<String> streamUrls = getLiveStreamUrls(roomId);
                if (streamUrls.isEmpty()) continue;
                
                byte[] frameData = captureFrame(streamUrls.get(0));
                
                // OCR识别
                String text = ocrService.recognize(frameData);
                
                // 解析商品信息、价格等
                LiveFrameInfo info = parseFrameInfo(text);
                info.setRoomId(roomId);
                info.setTimestamp(System.currentTimeMillis());
                
                // 存储
                liveFrameRepository.save(info);
                
            } catch (Exception e) {
                log.error("截帧失败: {}", roomId, e);
            }
        }
    }
}
```

---

## 三、淘宝数据采集

### 3.1 开放平台 API

#### 3.1.1 接入流程

```
1. 注册开放平台账号
2. 创建应用，获取 AppKey 和 AppSecret
3. 申请相关 API 权限
4. 实现 OAuth2.0 授权
5. 调用 API
```

#### 3.1.2 API 封装

```java
@Service
public class TaobaoApiService {
    
    @Value("${taobao.app-key}")
    private String appKey;
    
    @Value("${taobao.app-secret}")
    private String appSecret;
    
    /**
     * 商品搜索
     */
    public List<ItemInfo> searchItems(String keyword, int page) throws TaobaoApiException {
        TaobaoClient client = new DefaultTaobaoClient(
            "https://eco.taobao.com/router/rest",
            appKey,
            appSecret
        );
        
        TbkItemGetRequest req = new TbkItemGetRequest();
        req.setQ(keyword);
        req.setPageNo((long) page);
        req.setPageSize(20L);
        req.setSort("tk_rate_des"); // 按佣金率降序
        
        TbkItemGetResponse response = client.execute(req);
        
        return response.getResults().stream()
            .map(this::convertToItemInfo)
            .collect(Collectors.toList());
    }
    
    /**
     * 商品详情
     */
    public ItemDetail getItemDetail(String itemId) throws TaobaoApiException {
        TaobaoClient client = new DefaultTaobaoClient(
            "https://eco.taobao.com/router/rest",
            appKey,
            appSecret
        );
        
        TbkItemInfoGetRequest req = new TbkItemInfoGetRequest();
        req.setNumIids(Long.parseLong(itemId));
        
        TbkItemInfoGetResponse response = client.execute(req);
        
        return convertToItemDetail(response.getResults().get(0));
    }
    
    /**
     * 获取爆款商品
     */
    public List<ItemInfo> getHotItems(String categoryId) throws TaobaoApiException {
        TaobaoClient client = new DefaultTaobaoClient(
            "https://eco.taobao.com/router/rest",
            appKey,
            appSecret
        );
        
        TbkItemGetRequest req = new TbkItemGetRequest();
        req.setCat(categoryId);
        req.setPageNo(1L);
        req.setPageSize(100L);
        req.setSort("total_sales"); // 按销量排序
        
        TbkItemGetResponse response = client.execute(req);
        
        return response.getResults().stream()
            .map(this::convertToItemInfo)
            .collect(Collectors.toList());
    }
}
```

### 3.2 网页爬虫补充

对于开放平台未提供的接口，使用网页爬虫补充：

```java
@Service
public class TaobaoWebCrawler {
    
    /**
     * 获取商品评价
     */
    public List<ReviewInfo> getItemReviews(String itemId, int page) {
        BrowserContext context = contextPool.acquire();
        Page page1 = context.newPage();
        
        try {
            String url = "https://item.taobao.com/item.htm?id=" + itemId;
            page1.navigate(url);
            
            // 等待评价区域加载
            page1.waitForSelector(".tm-rate");
            
            // 滚动加载更多
            for (int i = 0; i < 5; i++) {
                page1.evaluate("window.scrollBy(0, 500)");
                page1.waitForTimeout(500);
            }
            
            // 提取评价
            List<ElementHandle> reviews = page1.locator(".rate-grid .rate-item").elementHandles();
            
            return reviews.stream()
                .map(this::parseReview)
                .collect(Collectors.toList());
                
        } finally {
            page1.close();
            contextPool.release(context);
        }
    }
}
```

---

## 四、1688 数据采集

### 4.1 商品搜索采集

```java
@Service
public class Ali1688Crawler {
    
    /**
     * 搜索商品
     */
    public List<WholesaleProduct> searchProducts(String keyword, int page) {
        BrowserContext context = contextPool.acquire();
        Page page1 = context.newPage();
        
        try {
            String url = "https://s.1688.com/selloffer/offer_search.htm?keywords=" 
                + URLEncoder.encode(keyword, "UTF-8") 
                + "&beginPage=" + page;
            
            page1.navigate(url);
            page1.waitForSelector(".offer-item");
            
            List<ElementHandle> items = page1.locator(".offer-item").elementHandles();
            
            return items.stream()
                .map(this::parseProduct)
                .collect(Collectors.toList());
                
        } finally {
            page1.close();
            contextPool.release(context);
        }
    }
    
    /**
     * 获取商品详情
     */
    public WholesaleProductDetail getProductDetail(String productId) {
        BrowserContext context = contextPool.acquire();
        Page page1 = context.newPage();
        
        try {
            String url = "https://detail.1688.com/offer/" + productId + ".html";
            page1.navigate(url);
            
            // 提取价格区间
            String priceRange = page1.locator(".price-range").textContent();
            
            // 提取起批量
            String minOrder = page1.locator(".min-order").textContent();
            
            // 提取供应商信息
            String supplier = page1.locator(".company-name").textContent();
            
            // 提取规格选项
            List<String> specs = page1.locator(".sku-item").elementHandles()
                .stream()
                .map(ElementHandle::textContent)
                .collect(Collectors.toList());
            
            return WholesaleProductDetail.builder()
                .productId(productId)
                .priceRange(priceRange)
                .minOrder(minOrder)
                .supplier(supplier)
                .specs(specs)
                .build();
                
        } finally {
            page1.close();
            contextPool.release(context);
        }
    }
    
    /**
     * 获取类目热销商品
     */
    public List<WholesaleProduct> getCategoryHotProducts(String categoryId) {
        String url = "https://s.1688.com/selloffer/offer_search.htm?category=" 
            + categoryId + "&sortType=tradegrowthdown"; // 按交易额降序
        
        // ... 类似搜索逻辑
    }
}
```

---

## 五、第三方数据接入

### 5.1 蝉妈妈 API

```java
@Service
public class ChanmamaApiService {
    
    @Value("${chanmama.api-key}")
    private String apiKey;
    
    @Value("${chanmama.base-url}")
    private String baseUrl;
    
    /**
     * 获取达人榜单
     */
    public List<InfluencerRank> getInfluencerRank(String category, String date) {
        String url = baseUrl + "/openapi/rank/influencer";
        
        Map<String, String> params = new HashMap<>();
        params.put("category", category);
        params.put("date", date);
        params.put("page", "1");
        params.put("size", "100");
        
        // 生成签名
        String sign = generateSign(params);
        params.put("sign", sign);
        
        String response = httpClient.get(url, params);
        
        return parseInfluencerRank(response);
    }
    
    /**
     * 获取商品榜单
     */
    public List<ProductRank> getProductRank(String category, String date) {
        String url = baseUrl + "/openapi/rank/product";
        
        // ... 类似逻辑
    }
    
    /**
     * 生成API签名
     */
    private String generateSign(Map<String, String> params) {
        // 按key排序
        List<String> keys = new ArrayList<>(params.keySet());
        Collections.sort(keys);
        
        // 拼接字符串
        StringBuilder sb = new StringBuilder();
        for (String key : keys) {
            sb.append(key).append("=").append(params.get(key)).append("&");
        }
        sb.append("key=").append(apiKey);
        
        // MD5
        return DigestUtils.md5Hex(sb.toString()).toUpperCase();
    }
}
```

### 5.2 飞瓜数据 API

```java
@Service
public class FeiguaApiService {
    
    /**
     * 获取直播监控数据
     */
    public LiveMonitorData getLiveMonitor(String roomId) {
        String url = "https://api.feigua.cn/v1/live/monitor";
        
        Map<String, String> params = new HashMap<>();
        params.put("room_id", roomId);
        
        String response = httpClient.get(url, withAuth(params));
        
        return parseLiveMonitor(response);
    }
    
    /**
     * 获取商品趋势
     */
    public ProductTrend getProductTrend(String productId, int days) {
        String url = "https://api.feigua.cn/v1/product/trend";
        
        Map<String, String> params = new HashMap<>();
        params.put("product_id", productId);
        params.put("days", String.valueOf(days));
        
        String response = httpClient.get(url, withAuth(params));
        
        return parseProductTrend(response);
    }
}
```

---

## 六、数据存储设计

### 6.1 数据库表设计

```sql
-- 商品表
CREATE TABLE `product` (
    `id` BIGINT PRIMARY KEY AUTO_INCREMENT,
    `platform` VARCHAR(20) NOT NULL COMMENT '平台：douyin/taobao/1688',
    `product_id` VARCHAR(64) NOT NULL COMMENT '商品ID',
    `title` VARCHAR(500) COMMENT '商品标题',
    `price` DECIMAL(10,2) COMMENT '价格',
    `original_price` DECIMAL(10,2) COMMENT '原价',
    `sales` INT COMMENT '销量',
    `commission_rate` DECIMAL(5,4) COMMENT '佣金率',
    `category_id` VARCHAR(64) COMMENT '类目ID',
    `category_name` VARCHAR(100) COMMENT '类目名称',
    `images` JSON COMMENT '图片列表',
    `detail_url` VARCHAR(500) COMMENT '详情页URL',
    `status` TINYINT DEFAULT 1 COMMENT '状态：1上架 0下架',
    `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
    `updated_at` DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    UNIQUE KEY `uk_platform_product` (`platform`, `product_id`),
    INDEX `idx_sales` (`sales`),
    INDEX `idx_category` (`category_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 直播表
CREATE TABLE `live_room` (
    `id` BIGINT PRIMARY KEY AUTO_INCREMENT,
    `platform` VARCHAR(20) NOT NULL,
    `room_id` VARCHAR(64) NOT NULL,
    `user_id` VARCHAR(64) NOT NULL COMMENT '主播ID',
    `nickname` VARCHAR(100) COMMENT '主播昵称',
    `title` VARCHAR(500) COMMENT '直播标题',
    `cover_url` VARCHAR(500) COMMENT '封面URL',
    `viewer_count` INT COMMENT '观看人数',
    `like_count` INT COMMENT '点赞数',
    `start_time` DATETIME COMMENT '开播时间',
    `end_time` DATETIME COMMENT '结束时间',
    `duration` INT COMMENT '时长(秒)',
    `sales_amount` DECIMAL(12,2) COMMENT '销售额',
    `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
    UNIQUE KEY `uk_platform_room` (`platform`, `room_id`),
    INDEX `idx_start_time` (`start_time`),
    INDEX `idx_user_id` (`user_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 商品销量快照表
CREATE TABLE `product_sales_snapshot` (
    `id` BIGINT PRIMARY KEY AUTO_INCREMENT,
    `product_id` BIGINT NOT NULL,
    `snapshot_date` DATE NOT NULL,
    `sales` INT COMMENT '当日销量',
    `total_sales` INT COMMENT '累计销量',
    `price` DECIMAL(10,2) COMMENT '当日价格',
    `viewer_count` INT COMMENT '曝光量',
    `click_count` INT COMMENT '点击量',
    `order_count` INT COMMENT '订单量',
    `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
    UNIQUE KEY `uk_product_date` (`product_id`, `snapshot_date`),
    INDEX `idx_snapshot_date` (`snapshot_date`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 爆品预测结果表
CREATE TABLE `hot_product_prediction` (
    `id` BIGINT PRIMARY KEY AUTO_INCREMENT,
    `product_id` BIGINT NOT NULL,
    `predict_date` DATE NOT NULL COMMENT '预测日期',
    `predict_score` DECIMAL(5,4) COMMENT '爆品指数 0-1',
    `predict_sales` INT COMMENT '预测销量',
    `confidence` DECIMAL(5,4) COMMENT '置信度',
    `factors` JSON COMMENT '影响因素',
    `model_version` VARCHAR(32) COMMENT '模型版本',
    `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
    INDEX `idx_predict_date` (`predict_date`),
    INDEX `idx_score` (`predict_score`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

### 6.2 数据分层存储

```
┌─────────────────────────────────────────────────────────────┐
│                       应用层                                 │
│     API服务 │ 分析服务 │ 报表服务 │ 预测服务                │
└─────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────────────────────────────────────┐
│                       数据仓库层 (ClickHouse)                │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │ DWD明细层   │  │ DWS汇总层   │  │ ADS应用层   │         │
│  │ 原始数据    │  │ 按日/周汇总 │  │ 报表数据    │         │
│  └─────────────┘  └─────────────┘  └─────────────┘         │
└─────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────────────────────────────────────┐
│                       数据湖层 (MinIO + Iceberg)             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │ 原始数据    │  │ 清洗数据    │  │ 特征数据    │         │
│  │ (JSON/Parquet)│ │ (Parquet)   │ │ (Parquet)   │         │
│  └─────────────┘  └─────────────┘  └─────────────┘         │
└─────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────────────────────────────────────┐
│                       实时层 (Kafka + Flink)                 │
│     实时采集 │ 实时清洗 │ 实时计算 │ 实时预警               │
└─────────────────────────────────────────────────────────────┘
```

---

## 七、采集调度系统

### 7.1 任务调度架构

```java
@Service
public class CrawlScheduler {
    
    @Autowired
    private XXLJobExecutor xxlJobExecutor;
    
    /**
     * 全量商品采集任务
     * 每天凌晨2点执行
     */
    @XxlJob("fullProductCrawl")
    public void fullProductCrawl() {
        // 获取所有类目
        List<Category> categories = categoryService.getAllCategories();
        
        for (Category category : categories) {
            // 提交异步任务
            CompletableFuture.runAsync(() -> {
                crawlProductsByCategory(category.getId());
            }, crawlExecutor);
        }
    }
    
    /**
     * 增量商品更新任务
     * 每小时执行
     */
    @XxlJob("incrementalProductUpdate")
    public void incrementalProductUpdate() {
        // 获取近24小时有变化的热门商品
        List<String> hotProductIds = getHotProductIds();
        
        for (String productId : hotProductIds) {
            CompletableFuture.runAsync(() -> {
                updateProductInfo(productId);
            }, crawlExecutor);
        }
    }
    
    /**
     * 直播监控任务
     * 每5分钟执行
     */
    @XxlJob("liveRoomMonitor")
    public void liveRoomMonitor() {
        // 获取正在直播的房间
        List<String> activeRooms = getActiveLiveRooms();
        
        for (String roomId : activeRooms) {
            CompletableFuture.runAsync(() -> {
                monitorLiveRoom(roomId);
            }, crawlExecutor);
        }
    }
    
    /**
     * 销量快照任务
     * 每天凌晨0点执行
     */
    @XxlJob("salesSnapshot")
    public void salesSnapshot() {
        List<Product> products = productRepository.findActiveProducts();
        
        for (Product product : products) {
            ProductSalesSnapshot snapshot = new ProductSalesSnapshot();
            snapshot.setProductId(product.getId());
            snapshot.setSnapshotDate(LocalDate.now());
            snapshot.setTotalSales(product.getSales());
            snapshot.setPrice(product.getPrice());
            
            snapshotRepository.save(snapshot);
        }
    }
}
```

### 7.2 任务监控

```java
@Service
public class CrawlMonitor {
    
    /**
     * 任务执行监控
     */
    @EventListener
    public void onTaskComplete(CrawlTaskEvent event) {
        // 记录执行日志
        CrawlTaskLog log = new CrawlTaskLog();
        log.setTaskId(event.getTaskId());
        log.setSuccess(event.isSuccess());
        log.setDataCount(event.getDataCount());
        log.setDuration(event.getDuration());
        log.setErrorMessage(event.getErrorMessage());
        
        taskLogRepository.save(log);
        
        // 更新统计
        if (!event.isSuccess()) {
            alertService.sendAlert("采集任务失败: " + event.getTaskId());
        }
    }
    
    /**
     * 数据质量监控
     */
    @Scheduled(cron = "0 0 * * * ?") // 每小时
    public void dataQualityCheck() {
        // 检查今日数据量
        long todayCount = productRepository.countByDate(LocalDate.now());
        long yesterdayCount = productRepository.countByDate(LocalDate.now().minusDays(1));
        
        if (todayCount < yesterdayCount * 0.5) {
            alertService.sendAlert("今日采集数据量异常偏低");
        }
    }
}
```

---

## 八、容错与恢复

### 8.1 断点续爬

```java
@Service
public class CheckpointManager {
    
    /**
     * 保存采集进度
     */
    public void saveCheckpoint(String taskId, int currentPage, String lastId) {
        Checkpoint checkpoint = new Checkpoint();
        checkpoint.setTaskId(taskId);
        checkpoint.setCurrentPage(currentPage);
        checkpoint.setLastId(lastId);
        checkpoint.setUpdateTime(LocalDateTime.now());
        
        checkpointRepository.save(checkpoint);
    }
    
    /**
     * 恢复采集进度
     */
    public Checkpoint loadCheckpoint(String taskId) {
        return checkpointRepository.findById(taskId).orElse(null);
    }
    
    /**
     * 支持续爬的采集任务
     */
    public void crawlWithCheckpoint(String taskId, String category) {
        Checkpoint checkpoint = loadCheckpoint(taskId);
        int startPage = checkpoint != null ? checkpoint.getCurrentPage() : 1;
        
        for (int page = startPage; page <= MAX_PAGE; page++) {
            try {
                List<Product> products = crawlProducts(category, page);
                
                if (products.isEmpty()) {
                    break;
                }
                
                productRepository.saveAll(products);
                saveCheckpoint(taskId, page, products.get(products.size() - 1).getProductId());
                
            } catch (Exception e) {
                log.error("采集失败，page={}", page, e);
                // 失败时保存进度，下次可恢复
                saveCheckpoint(taskId, page, null);
                throw e;
            }
        }
    }
}
```

### 8.2 异常重试

```java
@Service
public class RetryableCrawler {
    
    private static final int MAX_RETRY = 3;
    private static final long RETRY_DELAY_MS = 1000;
    
    @Retryable(
        value = {CrawlException.class, TimeoutException.class},
        maxAttempts = MAX_RETRY,
        backoff = @Backoff(delay = RETRY_DELAY_MS, multiplier = 2)
    )
    public Product crawlProductWithRetry(String productId) {
        return douyinCrawler.crawlProduct(productId);
    }
    
    @Recover
    public Product recoverCrawlProduct(Exception e, String productId) {
        log.error("采集商品失败，已重试{}次: {}", MAX_RETRY, productId, e);
        // 记录失败任务，后续人工处理
        failedTaskRepository.save(new FailedTask(productId, e.getMessage()));
        return null;
    }
}
```

---

## 九、性能优化

### 9.1 并发控制

```java
@Configuration
public class CrawlExecutorConfig {
    
    @Bean("crawlExecutor")
    public ThreadPoolTaskExecutor crawlExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10);
        executor.setMaxPoolSize(50);
        executor.setQueueCapacity(1000);
        executor.setThreadNamePrefix("crawl-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.initialize();
        return executor;
    }
}
```

### 9.2 批量处理

```java
@Service
public class BatchProductCrawler {
    
    @Autowired
    private JdbcTemplate jdbcTemplate;
    
    /**
     * 批量插入商品
     */
    public void batchInsert(List<Product> products) {
        jdbcTemplate.batchUpdate(
            "INSERT INTO product (platform, product_id, title, price, sales) " +
            "VALUES (?, ?, ?, ?, ?) " +
            "ON DUPLICATE KEY UPDATE title=VALUES(title), price=VALUES(price), sales=VALUES(sales)",
            products,
            100,
            (ps, product) -> {
                ps.setString(1, product.getPlatform());
                ps.setString(2, product.getProductId());
                ps.setString(3, product.getTitle());
                ps.setBigDecimal(4, product.getPrice());
                ps.setInt(5, product.getSales());
            }
        );
    }
}
```

---

## 十、安全与合规

### 10.1 数据脱敏

```java
@Service
public class DataMaskService {
    
    /**
     * 敏感信息脱敏
     */
    public String maskPhone(String phone) {
        if (phone == null || phone.length() < 7) {
            return phone;
        }
        return phone.substring(0, 3) + "****" + phone.substring(phone.length() - 4);
    }
    
    public String maskIdCard(String idCard) {
        if (idCard == null || idCard.length() < 8) {
            return idCard;
        }
        return idCard.substring(0, 4) + "**********" + idCard.substring(idCard.length() - 4);
    }
}
```

### 10.2 访问控制

```java
@Aspect
@Component
public class CrawlRateLimitAspect {
    
    private final Map<String, RateLimiter> limiters = new ConcurrentHashMap<>();
    
    @Around("@annotation(crawlRateLimit)")
    public Object around(ProceedingJoinPoint point, CrawlRateLimit crawlRateLimit) throws Throwable {
        String key = crawlRateLimit.key();
        RateLimiter limiter = limiters.computeIfAbsent(key, 
            k -> RateLimiter.create(crawlRateLimit.permitsPerSecond()));
        
        if (!limiter.tryAcquire(crawlRateLimit.timeout(), TimeUnit.MILLISECONDS)) {
            throw new RateLimitException("请求频率超限");
        }
        
        return point.proceed();
    }
}

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface CrawlRateLimit {
    String key();
    double permitsPerSecond();
    long timeout() default 1000;
}
```

---

## 十一、总结

### 11.1 技术要点

| 技术点 | 解决方案 |
|--------|----------|
| 反爬对抗 | 代理池 + 指纹伪装 + 行为模拟 |
| 高并发 | 线程池 + 异步处理 + 批量写入 |
| 容错恢复 | 断点续爬 + 重试机制 + 监控告警 |
| 数据质量 | 去重清洗 + 质量检测 + 异常预警 |

### 11.2 下一步工作

1. 完善抖音签名算法逆向
2. 接入更多第三方数据源
3. 优化采集性能
4. 建立数据质量监控体系