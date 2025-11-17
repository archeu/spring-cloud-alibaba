# Sentinel 核心功能深度实战

> 完整掌握流量控制、熔断降级、系统保护、热点参数限流

---

## 📋 目录

1. [流量控制深度实战](#1-流量控制深度实战)
   - 1.1 QPS 限流
   - 1.2 并发线程数限流
   - 1.3 流控模式（直接、关联、链路）
   - 1.4 流控效果（快速失败、Warm Up、排队等待）
2. [熔断降级实战](#2-熔断降级实战)
   - 2.1 慢调用比例熔断（RT）
   - 2.2 异常比例熔断
   - 2.3 异常数熔断
3. [系统自适应保护](#3-系统自适应保护)
4. [热点参数限流](#4-热点参数限流)

---

## 1. 流量控制深度实战

### 1.1 QPS 限流（已在快速入门中学习）

**回顾：** QPS（Queries Per Second）每秒查询率限流

**配置：**
```
资源名：GET:/order/create
阈值类型：QPS
单机阈值：10
```

**效果：** 每秒最多通过 10 个请求，超出部分拒绝

---

### 1.2 并发线程数限流

**原理：** 限制同时处理该资源的线程数量

**应用场景：**
```
问题：某个接口调用外部服务，响应慢（如 5 秒）
风险：如果 QPS=100，5 秒内会有 500 个线程阻塞等待
      → 线程池耗尽 → 整个应用无法响应

解决：限制并发线程数 = 50
效果：最多 50 个线程同时处理该接口
      超出的请求立即失败，避免线程池耗尽
```

#### **代码示例**

```java
package com.example.sentinel.controller;

import com.alibaba.csp.sentinel.annotation.SentinelResource;
import com.alibaba.csp.sentinel.slots.block.BlockException;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.web.bind.annotation.*;

import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.TimeUnit;

/**
 * 支付接口 - 演示并发线程数限流
 */
@RestController
@RequestMapping("/payment")
public class PaymentController {

    private static final Logger log = LoggerFactory.getLogger(PaymentController.class);

    /**
     * 支付接口（模拟慢调用）
     *
     * 场景：调用第三方支付网关，响应时间 3 秒
     *
     * 资源名：paymentProcess
     * 限流策略：并发线程数 ≤ 10
     *
     * @param orderId 订单 ID
     * @param amount 金额
     * @return 支付结果
     */
    @PostMapping("/process")
    @SentinelResource(
        value = "paymentProcess",
        blockHandler = "handlePaymentBlock"
    )
    public Map<String, Object> processPayment(
            @RequestParam String orderId,
            @RequestParam Double amount) {

        log.info("开始处理支付，订单ID: {}, 金额: {}, 当前线程: {}",
                 orderId, amount, Thread.currentThread().getName());

        try {
            // 模拟调用第三方支付网关（耗时 3 秒）
            TimeUnit.SECONDS.sleep(3);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }

        log.info("支付处理完成，订单ID: {}", orderId);

        Map<String, Object> result = new HashMap<>();
        result.put("success", true);
        result.put("orderId", orderId);
        result.put("amount", amount);
        result.put("transactionId", "TXN-" + System.currentTimeMillis());
        result.put("message", "支付成功");

        return result;
    }

    /**
     * 限流处理方法
     */
    public Map<String, Object> handlePaymentBlock(String orderId, Double amount, BlockException ex) {
        log.warn("支付接口被限流，订单ID: {}, 当前并发线程数已达上限", orderId);

        Map<String, Object> result = new HashMap<>();
        result.put("success", false);
        result.put("errorCode", "THREAD_LIMIT");
        result.put("message", "当前支付请求过多，请稍后重试");
        result.put("hint", "系统繁忙，建议 30 秒后重试");

        return result;
    }

    /**
     * 查询支付状态接口（演示对比）
     *
     * 资源名：paymentQuery
     * 限流策略：QPS ≤ 100（查询接口通常用 QPS 限流）
     */
    @GetMapping("/query")
    @SentinelResource(value = "paymentQuery")
    public Map<String, Object> queryPayment(@RequestParam String orderId) {

        log.info("查询支付状态，订单ID: {}", orderId);

        Map<String, Object> result = new HashMap<>();
        result.put("orderId", orderId);
        result.put("status", "SUCCESS");
        result.put("amount", 299.00);
        result.put("transactionId", "TXN-1705304567890");

        return result;
    }
}
```

#### **Dashboard 配置**

```
流控规则配置：

资源名：paymentProcess
针对来源：default
阈值类型：○ QPS
         ● 线程数  ← 选择线程数
单机阈值：10      ← 最多 10 个线程同时处理
流控模式：直接
流控效果：快速失败
```

#### **测试验证**

**测试工具：JMeter**

```
配置：
- 线程数：50（模拟 50 个并发用户）
- 循环次数：1
- Ramp-Up 时间：1 秒（1 秒内启动所有线程）

HTTP 请求：
- URL: http://localhost:8080/payment/process
- 方法：POST
- 参数：
  - orderId: ORDER-${__Random(1000,9999)}
  - amount: 299.00

预期结果：
- 前 10 个请求：正常处理（耗时 3 秒）
- 后 40 个请求：立即被限流（耗时 < 10ms）

聚合报告：
- 样本数：50
- 成功数：10
- 失败数：40
- 平均响应时间：约 600ms（(10*3000 + 40*10) / 50）
```

**测试脚本（curl）**

```bash
# 并发测试脚本
#!/bin/bash

# 启动 20 个后台进程并发请求
for i in {1..20}; do
  (
    curl -X POST "http://localhost:8080/payment/process?orderId=ORDER-$i&amount=299" &
    echo "请求 $i 已发送"
  ) &
done

# 等待所有后台进程完成
wait

echo "所有请求已完成"

# 预期结果：
# 前 10 个请求成功（耗时 3 秒）
# 后 10 个请求被限流（立即返回）
```

#### **对比：QPS vs 线程数**

| 场景 | 使用 QPS 限流 | 使用线程数限流 |
|------|--------------|---------------|
| **快速接口**（RT < 100ms） | ✅ 推荐 | ❌ 不适合 |
| **慢接口**（RT > 1s） | ❌ 可能导致线程池耗尽 | ✅ 推荐 |
| **外部依赖**（如数据库、RPC） | ❌ 不精确 | ✅ 精确控制 |
| **计算密集型** | ✅ 推荐 | ⚠️ 可选 |

**实战建议：**

```
1. 快速接口（RT < 100ms）：使用 QPS 限流
   示例：查询订单、获取用户信息

2. 慢接口（RT > 1s）：使用线程数限流
   示例：调用支付网关、上传文件、导出报表

3. 组合使用：
   资源1：QPS ≤ 100
   资源2：线程数 ≤ 10
   双重保护，更安全
```

---

### 1.3 流控模式（直接、关联、链路）

#### **1.3.1 直接模式（默认）**

**定义：** 直接限流当前资源

**配置：**
```
资源名：GET:/order/create
流控模式：直接
单机阈值：10
```

**效果：** `/order/create` 的 QPS > 10 时，直接限流

---

#### **1.3.2 关联模式**

**定义：** 当关联资源达到阈值时，限流当前资源

**应用场景：**

```
场景：电商系统

资源A：/order/query（查询订单，读操作）
资源B：/order/create（创建订单，写操作）

问题：
- 创建订单（写）比查询订单（读）更重要
- 如果查询请求过多，占用数据库连接，导致创建订单失败

解决：
- 配置关联规则：
  - 资源名：/order/query（查询）
  - 关联资源：/order/create（创建）
  - 阈值：当 /order/create 的 QPS > 50 时，限流 /order/query

效果：
- 优先保证创建订单的资源
- 查询订单被限流，释放资源给创建订单
```

**代码示例：**

```java
package com.example.sentinel.controller;

import com.alibaba.csp.sentinel.annotation.SentinelResource;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.web.bind.annotation.*;

import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.TimeUnit;

/**
 * 订单接口 - 演示关联模式
 */
@RestController
@RequestMapping("/order")
public class OrderController {

    private static final Logger log = LoggerFactory.getLogger(OrderController.class);

    /**
     * 创建订单接口（写操作，重要）
     *
     * 资源名：orderCreate
     */
    @PostMapping("/create")
    @SentinelResource(value = "orderCreate")
    public Map<String, Object> createOrder(
            @RequestParam String productId,
            @RequestParam Integer quantity) {

        log.info("创建订单，商品ID: {}, 数量: {}", productId, quantity);

        // 模拟数据库写操作（耗时 200ms）
        try {
            TimeUnit.MILLISECONDS.sleep(200);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }

        Map<String, Object> result = new HashMap<>();
        result.put("success", true);
        result.put("orderId", "ORDER-" + System.currentTimeMillis());
        result.put("productId", productId);
        result.put("quantity", quantity);

        return result;
    }

    /**
     * 查询订单接口（读操作，次要）
     *
     * 资源名：orderQuery
     * 关联资源：orderCreate
     *
     * 规则：当 orderCreate 的 QPS > 50 时，限流 orderQuery
     */
    @GetMapping("/query")
    @SentinelResource(value = "orderQuery")
    public Map<String, Object> queryOrder(@RequestParam String orderId) {

        log.info("查询订单，订单ID: {}", orderId);

        // 模拟数据库查询（耗时 100ms）
        try {
            TimeUnit.MILLISECONDS.sleep(100);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }

        Map<String, Object> result = new HashMap<>();
        result.put("orderId", orderId);
        result.put("status", "已支付");
        result.put("totalAmount", 299.00);

        return result;
    }
}
```

**Dashboard 配置：**

```
流控规则配置：

资源名：orderQuery（查询订单，被限流的资源）
针对来源：default
阈值类型：QPS
单机阈值：100
流控模式：● 关联  ← 选择关联模式
关联资源：orderCreate  ← 输入关联资源名
流控效果：快速失败

说明：
当 orderCreate 的 QPS > 100 时，限流 orderQuery
保护创建订单接口的资源
```

**测试验证：**

```bash
# 终端1：持续请求创建订单（触发关联限流）
for i in {1..200}; do
  curl -X POST "http://localhost:8080/order/create?productId=1001&quantity=1" &
done

# 终端2：同时请求查询订单
for i in {1..50}; do
  curl "http://localhost:8080/order/query?orderId=ORDER-123"
  echo ""
done

# 预期结果：
# 创建订单：正常通过（QPS 很高）
# 查询订单：大部分被限流（因为创建订单的 QPS 超过阈值）
```

---

#### **1.3.3 链路模式**

**定义：** 只对指定调用链路限流

**应用场景：**

```
场景：
Service A → Service C → 数据库
Service B → Service C → 数据库

问题：
- Service C 是共享服务
- 不希望 Service A 的大流量影响 Service B

解决：
- 配置链路规则：
  - 资源名：Service C
  - 入口资源：Service A
  - 阈值：QPS ≤ 10

效果：
- 只限流 Service A → Service C 的调用
- Service B → Service C 不受影响
```

**代码示例：**

```java
package com.example.sentinel.service;

import com.alibaba.csp.sentinel.annotation.SentinelResource;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Service;

/**
 * 通用服务 - 演示链路限流
 */
@Service
public class CommonService {

    private static final Logger log = LoggerFactory.getLogger(CommonService.class);

    /**
     * 查询数据库（共享方法）
     *
     * 资源名：queryDatabase
     */
    @SentinelResource(value = "queryDatabase")
    public String queryDatabase(String key) {
        log.info("查询数据库，key: {}", key);
        return "数据：" + key;
    }
}

/**
 * 业务服务 A
 */
@Service
public class ServiceA {

    private static final Logger log = LoggerFactory.getLogger(ServiceA.class);

    @Autowired
    private CommonService commonService;

    @SentinelResource(value = "serviceA")
    public String process() {
        log.info("ServiceA 处理中...");
        return commonService.queryDatabase("from-A");
    }
}

/**
 * 业务服务 B
 */
@Service
public class ServiceB {

    private static final Logger log = LoggerFactory.getLogger(ServiceB.class);

    @Autowired
    private CommonService commonService;

    @SentinelResource(value = "serviceB")
    public String process() {
        log.info("ServiceB 处理中...");
        return commonService.queryDatabase("from-B");
    }
}

/**
 * Controller
 */
@RestController
@RequestMapping("/service")
public class ServiceController {

    @Autowired
    private ServiceA serviceA;

    @Autowired
    private ServiceB serviceB;

    @GetMapping("/a")
    public String callServiceA() {
        return serviceA.process();
    }

    @GetMapping("/b")
    public String callServiceB() {
        return serviceB.process();
    }
}
```

**Dashboard 配置：**

```
流控规则配置：

资源名：queryDatabase
针对来源：default
阈值类型：QPS
单机阈值：10
流控模式：● 链路  ← 选择链路模式
入口资源：serviceA  ← 指定入口
流控效果：快速失败

说明：
只限流 serviceA → queryDatabase 的调用
serviceB → queryDatabase 不受影响
```

**重要配置（必须）：**

```yaml
# application.yml
spring:
  cloud:
    sentinel:
      # 必须设置为 false，否则链路模式不生效
      web-context-unify: false
```

**测试验证：**

```bash
# 并发请求 Service A（会被限流）
for i in {1..20}; do
  curl "http://localhost:8080/service/a"
  echo ""
done

# 并发请求 Service B（不受影响）
for i in {1..20}; do
  curl "http://localhost:8080/service/b"
  echo ""
done

# 预期结果：
# Service A：超过 10 个请求被限流
# Service B：全部通过
```

---

### 1.4 流控效果（快速失败、Warm Up、排队等待）

#### **1.4.1 快速失败（默认）**

**定义：** 超过阈值立即拒绝

**特点：**
- ✅ 简单直接
- ✅ 响应快
- ❌ 对突发流量不友好

---

#### **1.4.2 Warm Up（预热）**

**定义：** 冷启动，逐步提升 QPS 阈值

**原理：**

```
场景：系统刚启动，缓存未预热

如果立即承受 100 QPS：
→ 缓存穿透 → 数据库压力大 → 系统崩溃

使用 Warm Up：
T=0s   → QPS 阈值：10（初始值 = 阈值 / 3）
T=30s  → QPS 阈值：55（逐步增加）
T=60s  → QPS 阈值：100（达到设定阈值）

效果：系统逐步预热，避免冷启动压垮系统
```

**应用场景：**
- 系统启动
- 缓存失效后重建
- 秒杀活动开始前

**配置：**

```
资源名：GET:/product/list
阈值类型：QPS
单机阈值：100
流控模式：直接
流控效果：● Warm Up  ← 选择 Warm Up
预热时长：60（秒）  ← 60 秒内从 33 QPS 升到 100 QPS

计算公式：
初始 QPS = 阈值 / coldFactor（默认 3）
         = 100 / 3
         = 33.3

时间线：
T=0s   → QPS: 33
T=15s  → QPS: 50
T=30s  → QPS: 66
T=45s  → QPS: 83
T=60s  → QPS: 100（达到设定阈值）
```

**代码示例：**

```java
@RestController
@RequestMapping("/product")
public class ProductController {

    /**
     * 商品列表接口（演示 Warm Up）
     *
     * 资源名：productList
     * 流控效果：Warm Up，预热时长 60 秒
     */
    @GetMapping("/list")
    @SentinelResource(value = "productList")
    public List<Map<String, Object>> getProductList() {
        // 模拟查询数据库 + 缓存
        List<Map<String, Object>> products = new ArrayList<>();
        products.add(Map.of("id", 1, "name", "iPhone 15", "price", 5999));
        products.add(Map.of("id", 2, "name", "MacBook Pro", "price", 12999));
        return products;
    }
}
```

**测试验证：**

```bash
# 模拟系统启动后的流量增长

# T=0s - 10s：QPS 约 33（前 10 秒）
ab -n 500 -c 50 -t 10 http://localhost:8080/product/list

# T=30s - 40s：QPS 约 66（30 秒后）
sleep 20
ab -n 800 -c 80 -t 10 http://localhost:8080/product/list

# T=60s 之后：QPS 可达 100（预热完成）
sleep 20
ab -n 1000 -c 100 -t 10 http://localhost:8080/product/list
```

---

#### **1.4.3 排队等待**

**定义：** 请求匀速通过，超时则失败

**原理：**

```
场景：消息发送、批量任务处理

如果使用快速失败：
T=0s → 1000 个请求同时到达 → 900 个被拒绝

使用排队等待：
T=0s  → 请求1 通过
T=0.1s → 请求2 通过（间隔 100ms）
T=0.2s → 请求3 通过
...
T=99.9s → 请求1000 通过

效果：所有请求都能被处理（只要在超时时间内）
```

**应用场景：**
- 消息队列消费
- 批量数据处理
- 定时任务调度

**配置：**

```
资源名：messageSend
阈值类型：QPS
单机阈值：10（每秒处理 10 个）
流控模式：直接
流控效果：● 排队等待  ← 选择排队等待
超时时间：5000（毫秒）  ← 最多等待 5 秒

说明：
- 请求匀速通过，间隔 100ms（1000ms / 10 = 100ms）
- 如果等待时间 > 5 秒，则超时失败
```

**代码示例：**

```java
@RestController
@RequestMapping("/message")
public class MessageController {

    private static final Logger log = LoggerFactory.getLogger(MessageController.class);

    /**
     * 发送消息接口（演示排队等待）
     *
     * 资源名：messageSend
     * 流控效果：排队等待，超时时间 5000ms
     */
    @PostMapping("/send")
    @SentinelResource(value = "messageSend")
    public Map<String, Object> sendMessage(@RequestParam String content) {

        long startTime = System.currentTimeMillis();

        log.info("开始发送消息，内容: {}", content);

        // 模拟消息发送（耗时 50ms）
        try {
            TimeUnit.MILLISECONDS.sleep(50);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }

        long endTime = System.currentTimeMillis();
        long duration = endTime - startTime;

        log.info("消息发送完成，耗时: {}ms", duration);

        Map<String, Object> result = new HashMap<>();
        result.put("success", true);
        result.put("messageId", "MSG-" + System.currentTimeMillis());
        result.put("content", content);
        result.put("waitTime", duration);  // 包含排队等待时间

        return result;
    }
}
```

**测试验证：**

```bash
# 并发发送 100 个消息
for i in {1..100}; do
  (
    start=$(date +%s%3N)
    curl -X POST "http://localhost:8080/message/send?content=消息$i"
    end=$(date +%s%3N)
    duration=$((end - start))
    echo "消息 $i 完成，耗时: ${duration}ms"
  ) &
done

wait

# 预期结果：
# 消息1：耗时 50ms（立即处理）
# 消息2：耗时 150ms（排队 100ms + 处理 50ms）
# 消息3：耗时 250ms（排队 200ms + 处理 50ms）
# ...
# 消息50：耗时 4950ms（排队 4900ms + 处理 50ms，接近超时）
# 消息51+：超时失败（等待时间 > 5000ms）
```

---

## 2. 熔断降级实战

**核心概念：** 断路器（Circuit Breaker）状态机

```
┌────────────────────────────────────────────────────┐
│              断路器三种状态                          │
├────────────────────────────────────────────────────┤
│                                                    │
│  1. CLOSED（关闭）- 正常状态                        │
│     所有请求正常通过                                │
│     实时统计 RT/异常比例/异常数                      │
│     ↓                                              │
│     满足熔断条件（如 RT > 500ms）                   │
│     ↓                                              │
│  2. OPEN（打开）- 熔断状态                          │
│     所有请求立即拒绝（快速失败）                     │
│     持续时间：熔断时长（如 10 秒）                   │
│     ↓                                              │
│     熔断时长结束                                    │
│     ↓                                              │
│  3. HALF_OPEN（半开）- 探测状态                     │
│     允许一个请求通过（探测）                         │
│     ├─ 成功 → 转为 CLOSED（恢复正常）               │
│     └─ 失败 → 转为 OPEN（继续熔断）                 │
│                                                    │
└────────────────────────────────────────────────────┘
```

---

### 2.1 慢调用比例熔断（RT）

**定义：** 当慢调用比例超过阈值时，触发熔断

**核心参数：**
1. **RT 阈值**：认为是"慢调用"的响应时间（如 500ms）
2. **比例阈值**：慢调用占总请求的比例（如 80%）
3. **最小请求数**：触发熔断的最小请求数（如 5 个）
4. **熔断时长**：熔断持续时间（如 10 秒）
5. **统计时长**：统计窗口大小（如 1 秒）

**应用场景：**

```
场景：调用外部 API

正常情况：RT = 100ms
故障情况：外部 API 变慢，RT = 2000ms

如果不熔断：
- 所有请求都等待 2 秒
- 线程池耗尽
- 整个系统崩溃

使用熔断：
- 检测到 RT > 500ms 的请求占比 > 80%
- 立即熔断，所有请求快速失败
- 10 秒后尝试恢复
```

#### **代码示例**

```java
package com.example.sentinel.controller;

import com.alibaba.csp.sentinel.annotation.SentinelResource;
import com.alibaba.csp.sentinel.slots.block.BlockException;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.web.bind.annotation.*;

import java.util.HashMap;
import java.util.Map;
import java.util.Random;
import java.util.concurrent.TimeUnit;

/**
 * 外部 API 接口 - 演示慢调用比例熔断
 */
@RestController
@RequestMapping("/api")
public class ExternalApiController {

    private static final Logger log = LoggerFactory.getLogger(ExternalApiController.class);

    private final Random random = new Random();

    /**
     * 调用外部推荐服务（模拟慢调用）
     *
     * 资源名：recommendService
     * 熔断策略：慢调用比例
     *
     * @param userId 用户 ID
     * @return 推荐结果
     */
    @GetMapping("/recommend")
    @SentinelResource(
        value = "recommendService",
        blockHandler = "handleRecommendBlock",
        fallback = "handleRecommendFallback"
    )
    public Map<String, Object> getRecommendations(@RequestParam String userId) {

        log.info("调用推荐服务，用户ID: {}", userId);

        // 模拟外部 API 响应时间（80% 概率慢调用）
        int rt = random.nextInt(100) < 80 ? 1000 : 100;  // 80% 概率 1000ms，20% 概率 100ms

        try {
            TimeUnit.MILLISECONDS.sleep(rt);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }

        log.info("推荐服务响应，RT: {}ms", rt);

        Map<String, Object> result = new HashMap<>();
        result.put("success", true);
        result.put("userId", userId);
        result.put("recommendations", new String[]{"商品A", "商品B", "商品C"});
        result.put("rt", rt);

        return result;
    }

    /**
     * 熔断处理方法（BlockException）
     */
    public Map<String, Object> handleRecommendBlock(String userId, BlockException ex) {
        log.warn("推荐服务被熔断，用户ID: {}, 熔断规则: {}", userId, ex.getRule());

        Map<String, Object> result = new HashMap<>();
        result.put("success", false);
        result.put("errorCode", "CIRCUIT_BREAKER");
        result.put("message", "推荐服务暂时不可用");
        result.put("fallbackData", new String[]{"默认推荐1", "默认推荐2"});

        return result;
    }

    /**
     * 降级处理方法（Throwable）
     */
    public Map<String, Object> handleRecommendFallback(String userId, Throwable ex) {
        log.error("推荐服务异常，用户ID: {}, 异常: {}", userId, ex.getMessage());

        Map<String, Object> result = new HashMap<>();
        result.put("success", false);
        result.put("errorCode", "SERVICE_ERROR");
        result.put("message", "系统异常，返回默认推荐");
        result.put("fallbackData", new String[]{"默认推荐1", "默认推荐2"});

        return result;
    }
}
```

#### **Dashboard 配置**

```
降级规则配置：

资源名：recommendService
熔断策略：● 慢调用比例  ← 选择慢调用比例
最大 RT：500  ← RT > 500ms 视为慢调用
比例阈值：0.8  ← 慢调用占比 > 80% 触发熔断
熔断时长：10  ← 熔断 10 秒
最小请求数：5  ← 至少 5 个请求才触发熔断
统计时长：1000  ← 统计窗口 1 秒

说明：
1 秒内至少 5 个请求
如果 RT > 500ms 的请求占比 > 80%
触发熔断，持续 10 秒
```

#### **测试验证**

```bash
# 快速发送 20 个请求
for i in {1..20}; do
  curl "http://localhost:8080/api/recommend?userId=USER-$i"
  echo ""
  sleep 0.1
done

# 预期结果：
# 前 5-10 个请求：正常通过（但响应慢，80% 的请求 RT > 500ms）
# 检测到慢调用比例 > 80% → 触发熔断
# 后续请求：立即返回降级响应（不再调用外部 API）
# 10 秒后：进入半开状态，允许一个请求通过探测
```

**观察日志：**

```
INFO  调用推荐服务，用户ID: USER-1
INFO  推荐服务响应，RT: 1000ms  ← 慢调用
INFO  调用推荐服务，用户ID: USER-2
INFO  推荐服务响应，RT: 100ms   ← 正常
INFO  调用推荐服务，用户ID: USER-3
INFO  推荐服务响应，RT: 1000ms  ← 慢调用
INFO  调用推荐服务，用户ID: USER-4
INFO  推荐服务响应，RT: 1000ms  ← 慢调用
INFO  调用推荐服务，用户ID: USER-5
INFO  推荐服务响应，RT: 1000ms  ← 慢调用
WARN  推荐服务被熔断 ← 触发熔断（5 个请求中 4 个慢调用，比例 80%）
WARN  推荐服务被熔断
WARN  推荐服务被熔断
...（10 秒内所有请求都被熔断）
INFO  调用推荐服务，用户ID: USER-15  ← 10 秒后，半开状态，探测请求
INFO  推荐服务响应，RT: 100ms  ← 探测成功
INFO  调用推荐服务，用户ID: USER-16  ← 恢复正常
```

---

### 2.2 异常比例熔断

**定义：** 当异常比例超过阈值时，触发熔断

**核心参数：**
1. **比例阈值**：异常请求占总请求的比例（如 50%）
2. **最小请求数**：触发熔断的最小请求数（如 5 个）
3. **熔断时长**：熔断持续时间（如 10 秒）
4. **统计时长**：统计窗口大小（如 1 秒）

**应用场景：**

```
场景：调用下游服务

正常情况：成功率 99%
故障情况：下游服务异常，成功率 30%

使用熔断：
- 检测到异常比例 > 50%
- 立即熔断，快速失败
- 避免大量请求失败，浪费资源
```

#### **代码示例**

```java
@RestController
@RequestMapping("/downstream")
public class DownstreamController {

    private static final Logger log = LoggerFactory.getLogger(DownstreamController.class);

    private final Random random = new Random();

    /**
     * 调用下游服务（模拟异常）
     *
     * 资源名：downstreamService
     * 熔断策略：异常比例
     */
    @GetMapping("/call")
    @SentinelResource(
        value = "downstreamService",
        blockHandler = "handleBlock",
        fallback = "handleFallback"
    )
    public Map<String, Object> callDownstream(@RequestParam String requestId) {

        log.info("调用下游服务，请求ID: {}", requestId);

        // 模拟 60% 概率抛出异常
        if (random.nextInt(100) < 60) {
            throw new RuntimeException("下游服务异常");
        }

        Map<String, Object> result = new HashMap<>();
        result.put("success", true);
        result.put("requestId", requestId);
        result.put("data", "下游服务响应数据");

        return result;
    }

    /**
     * 熔断处理方法
     */
    public Map<String, Object> handleBlock(String requestId, BlockException ex) {
        log.warn("下游服务被熔断，请求ID: {}", requestId);

        Map<String, Object> result = new HashMap<>();
        result.put("success", false);
        result.put("errorCode", "CIRCUIT_BREAKER");
        result.put("message", "下游服务熔断中，请稍后重试");

        return result;
    }

    /**
     * 异常降级处理方法
     */
    public Map<String, Object> handleFallback(String requestId, Throwable ex) {
        log.error("下游服务异常降级，请求ID: {}, 异常: {}", requestId, ex.getMessage());

        Map<String, Object> result = new HashMap<>();
        result.put("success", false);
        result.put("errorCode", "SERVICE_ERROR");
        result.put("message", "服务异常，已降级");

        return result;
    }
}
```

#### **Dashboard 配置**

```
降级规则配置：

资源名：downstreamService
熔断策略：● 异常比例  ← 选择异常比例
比例阈值：0.5  ← 异常比例 > 50% 触发熔断
熔断时长：10  ← 熔断 10 秒
最小请求数：5  ← 至少 5 个请求才触发熔断
统计时长：1000  ← 统计窗口 1 秒

说明：
1 秒内至少 5 个请求
如果异常请求占比 > 50%
触发熔断，持续 10 秒
```

---

### 2.3 异常数熔断

**定义：** 当异常数量超过阈值时，触发熔断

**核心参数：**
1. **异常数阈值**：触发熔断的异常数量（如 10 个）
2. **熔断时长**：熔断持续时间（如 10 秒）
3. **统计时长**：统计窗口大小（如 1 分钟）

**应用场景：**

```
场景：定时任务、批处理

正常情况：偶尔 1-2 个异常
故障情况：连续 10 个异常

使用异常数熔断：
- 检测到 1 分钟内异常数 > 10
- 立即熔断，暂停任务
- 避免持续失败
```

#### **Dashboard 配置**

```
降级规则配置：

资源名：batchTask
熔断策略：● 异常数  ← 选择异常数
异常数：10  ← 异常数 > 10 触发熔断
熔断时长：60  ← 熔断 60 秒
统计时长：60000  ← 统计窗口 60 秒

说明：
60 秒内异常数 > 10
触发熔断，持续 60 秒
```

---

## 3. 系统自适应保护

**定义：** 根据系统指标（CPU、Load、RT、线程数、入口 QPS）自适应限流

**应用场景：**

```
问题：流量波动大，无法预估合理的 QPS 阈值

传统方式：
- 手动设置 QPS = 1000
- 流量高峰时可能不够
- 流量低谷时可能过多

系统自适应保护：
- 实时监控 CPU 使用率
- CPU > 80% 时自动限流
- CPU 下降后自动恢复
```

### 系统规则配置

#### **Dashboard 配置**

```
系统规则配置：

阈值类型：
○ Load（系统负载，仅 Linux）
● CPU 使用率  ← 推荐
○ 平均 RT
○ 并发线程数
○ 入口 QPS

阈值：0.8  ← CPU > 80% 时限流

说明：
当系统 CPU 使用率 > 80% 时
自动限流所有资源
保护系统稳定性
```

#### **代码配置**

```java
@Configuration
public class SentinelSystemRuleConfig {

    @PostConstruct
    public void initSystemRules() {
        List<SystemRule> rules = new ArrayList<>();

        // 规则1：CPU 保护
        SystemRule cpuRule = new SystemRule();
        cpuRule.setHighestSystemLoad(-1);  // 不启用 Load 保护
        cpuRule.setHighestCpuUsage(0.8);   // CPU > 80% 限流
        cpuRule.setAvgRt(-1);               // 不启用 RT 保护
        cpuRule.setMaxThread(-1);           // 不启用线程数保护
        cpuRule.setQps(-1);                 // 不启用 QPS 保护
        rules.add(cpuRule);

        // 规则2：平均 RT 保护
        SystemRule rtRule = new SystemRule();
        rtRule.setAvgRt(1000);  // 平均 RT > 1000ms 限流
        rules.add(rtRule);

        // 加载规则
        SystemRuleManager.loadRules(rules);

        log.info("Sentinel 系统规则初始化完成");
    }
}
```

### 测试验证

```bash
# 使用压测工具模拟高并发
ab -n 100000 -c 100 http://localhost:8080/order/create

# 观察 CPU 使用率
# Linux
top -p $(pgrep -f 'java.*sentinel-demo')

# Mac
top | grep java

# 当 CPU > 80% 时，Sentinel 自动限流
# 请求会返回：
# Blocked by Sentinel (system_load)
```

---

## 4. 热点参数限流

**定义：** 针对特定参数值进行限流

**应用场景：**

```
场景：电商秒杀

资源：/product/detail
参数：productId

问题：
- 热门商品（如 iPhone）被疯狂访问
- 商品 ID=1001 的 QPS 达到 10000
- 数据库被打爆

解决：
- 配置热点参数规则：
  - 参数索引：0（第一个参数 productId）
  - 单机阈值：100
  - 热点值：
    - productId=1001（iPhone）：QPS ≤ 10
    - productId=1002（MacBook）：QPS ≤ 20
    - 其他商品：QPS ≤ 100

效果：
- 热门商品被特殊保护
- 其他商品正常访问
```

### 代码示例

```java
@RestController
@RequestMapping("/product")
public class ProductController {

    private static final Logger log = LoggerFactory.getLogger(ProductController.class);

    /**
     * 商品详情接口（演示热点参数限流）
     *
     * 资源名：productDetail
     * 热点参数：productId（索引 0）
     *
     * @param productId 商品 ID
     * @param userId 用户 ID（索引 1，可选）
     * @return 商品详情
     */
    @GetMapping("/detail")
    @SentinelResource(
        value = "productDetail",
        blockHandler = "handleProductBlock"
    )
    public Map<String, Object> getProductDetail(
            @RequestParam String productId,
            @RequestParam(required = false) String userId) {

        log.info("查询商品详情，商品ID: {}, 用户ID: {}", productId, userId);

        Map<String, Object> product = new HashMap<>();
        product.put("productId", productId);
        product.put("name", "商品-" + productId);
        product.put("price", 5999.00);
        product.put("stock", 1000);

        return product;
    }

    /**
     * 热点限流处理方法
     */
    public Map<String, Object> handleProductBlock(
            String productId,
            String userId,
            BlockException ex) {

        log.warn("商品详情接口被限流，商品ID: {}", productId);

        Map<String, Object> result = new HashMap<>();
        result.put("success", false);
        result.put("errorCode", "HOT_PARAM_LIMIT");
        result.put("message", "该商品访问人数过多，请稍后重试");
        result.put("productId", productId);

        return result;
    }
}
```

### Dashboard 配置

```
热点规则配置：

资源名：productDetail
参数索引：0  ← 第一个参数（productId）
单机阈值：100  ← 默认 QPS 阈值
统计窗口时长：1000  ← 1 秒

高级选项 → 参数例外项：
┌────────────────────────────────────┐
│ 参数类型：String                    │
│ 参数值：1001（iPhone）              │
│ 限流阈值：10  ← 特殊商品 QPS 限制   │
├────────────────────────────────────┤
│ 参数值：1002（MacBook）             │
│ 限流阈值：20                        │
└────────────────────────────────────┘

说明：
- 商品 1001（iPhone）：QPS ≤ 10
- 商品 1002（MacBook）：QPS ≤ 20
- 其他商品：QPS ≤ 100
```

### 测试验证

```bash
# 并发访问热门商品（iPhone，ID=1001）
for i in {1..50}; do
  curl "http://localhost:8080/product/detail?productId=1001" &
done

# 预期结果：
# 前 10 个请求成功
# 后 40 个请求被限流

# 并发访问普通商品（ID=2001）
for i in {1..50}; do
  curl "http://localhost:8080/product/detail?productId=2001" &
done

# 预期结果：
# 全部通过（默认阈值 100）
```

---

## 📚 总结

### 核心功能对比

| 功能 | 限流维度 | 适用场景 | 配置难度 |
|------|---------|---------|---------|
| **QPS 限流** | 每秒请求数 | 快速接口 | ⭐ 简单 |
| **线程数限流** | 并发线程 | 慢接口、外部依赖 | ⭐ 简单 |
| **慢调用熔断** | 响应时间 | 调用外部服务 | ⭐⭐ 中等 |
| **异常比例熔断** | 异常率 | 不稳定服务 | ⭐⭐ 中等 |
| **系统保护** | CPU/Load | 全局保护 | ⭐⭐⭐ 复杂 |
| **热点限流** | 参数值 | 热点数据 | ⭐⭐⭐ 复杂 |

### 实战建议

1. **快速接口**：QPS 限流
2. **慢接口**：线程数限流
3. **外部依赖**：慢调用熔断
4. **不稳定服务**：异常比例熔断
5. **流量波动大**：系统自适应保护
6. **热点数据**：热点参数限流

### 下一步学习

- 集群流控（分布式限流）
- 规则持久化（Nacos、Apollo）
- 网关流控（Spring Cloud Gateway）
- 自定义 Slot（扩展 Sentinel）

---

好了，**第4步：核心功能演示**就讲到这里。你现在应该能够：

- ✅ 配置 QPS 和并发线程数限流
- ✅ 理解流控模式（直接、关联、链路）
- ✅ 使用流控效果（快速失败、Warm Up、排队等待）
- ✅ 配置三种熔断策略（慢调用、异常比例、异常数）
- ✅ 启用系统自适应保护
- ✅ 实现热点参数限流

如果你理解了这些核心功能，请告诉我**"下一步"**，我会继续讲解**第5步：高级特性**，学习集群流控、网关流控、规则持久化等生产级特性！🚀
