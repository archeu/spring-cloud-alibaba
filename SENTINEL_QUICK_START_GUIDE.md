# Sentinel 快速入门 - 5分钟实现接口频次控制

> 手把手教你集成 Sentinel 并对指定接口进行 QPS 限流

---

## 📋 目录

1. [项目准备](#1-项目准备)
2. [核心依赖配置](#2-核心依赖配置)
3. [应用配置文件](#3-应用配置文件)
4. [启动 Sentinel Dashboard](#4-启动-sentinel-dashboard)
5. [实现接口频次控制](#5-实现接口频次控制)
6. [测试限流效果](#6-测试限流效果)
7. [常见问题排查](#7-常见问题排查)

---

## 1. 项目准备

### 项目结构

```
sentinel-demo/
├── pom.xml
└── src/main/
    ├── java/com/example/sentinel/
    │   ├── SentinelDemoApplication.java      # 启动类
    │   ├── controller/
    │   │   └── OrderController.java          # 业务接口
    │   └── config/
    │       └── SentinelConfig.java           # Sentinel 配置类（可选）
    └── resources/
        └── application.yml                    # 配置文件
```

---

## 2. 核心依赖配置

### pom.xml（完整版本）

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.0.2</version>
        <relativePath/>
    </parent>

    <groupId>com.example</groupId>
    <artifactId>sentinel-demo</artifactId>
    <version>1.0.0</version>
    <packaging>jar</packaging>

    <properties>
        <java.version>17</java.version>
        <spring-cloud.version>2022.0.0</spring-cloud.version>
        <spring-cloud-alibaba.version>2022.0.0.0</spring-cloud-alibaba.version>
    </properties>

    <dependencies>
        <!-- Spring Boot Web -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <!--
            Sentinel 核心依赖
            作用：
            1. 提供 Sentinel 核心限流/熔断能力
            2. 自动为所有 HTTP 接口埋点（拦截器）
            3. 提供 @SentinelResource 注解
            4. 集成 Dashboard 通信能力
        -->
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
        </dependency>

        <!--
            Actuator（可选，用于健康检查和监控）
            作用：提供 /actuator/sentinel 端点，查看 Sentinel 配置
        -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>

        <!-- Lombok（可选，简化代码） -->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
    </dependencies>

    <dependencyManagement>
        <dependencies>
            <!-- Spring Cloud 依赖管理 -->
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>${spring-cloud.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>

            <!-- Spring Cloud Alibaba 依赖管理 -->
            <dependency>
                <groupId>com.alibaba.cloud</groupId>
                <artifactId>spring-cloud-alibaba-dependencies</artifactId>
                <version>${spring-cloud-alibaba.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

**核心依赖说明：**

| 依赖 | 版本 | 作用 |
|------|------|------|
| `spring-cloud-starter-alibaba-sentinel` | 2022.0.0.0 | Sentinel 核心功能 |
| Spring Boot | 3.0.2 | 基础框架 |
| Spring Cloud | 2022.0.0 | 微服务框架 |

---

## 3. 应用配置文件

### application.yml（完整配置）

```yaml
# ============================================================
# 服务基础配置
# ============================================================
server:
  port: 8080  # 应用端口

spring:
  application:
    name: sentinel-demo  # 应用名称（会显示在 Dashboard 上）

  # ============================================================
  # Sentinel 配置
  # ============================================================
  cloud:
    sentinel:
      # ──────────────────────────────────────────────────────
      # Transport 配置（与 Dashboard 通信）
      # ──────────────────────────────────────────────────────
      transport:
        # Dashboard 地址（Sentinel 控制台）
        dashboard: localhost:8080

        # Client 通信端口（与 Dashboard 通信）
        # 默认 8719，如果被占用会自动 +1 尝试
        port: 8719

        # 心跳发送周期（毫秒）
        # 默认 null，表示不发送心跳（Dashboard 主动拉取）
        # heartbeat-interval-ms: 10000

      # ──────────────────────────────────────────────────────
      # 限流降级配置
      # ──────────────────────────────────────────────────────
      # 是否启用 Sentinel（默认 true）
      enabled: true

      # 饥饿加载（推荐开启）
      # 作用：应用启动时立即初始化 Sentinel，避免首次请求失败
      # 默认 false（懒加载，首次请求时才初始化）
      eager: true

      # ──────────────────────────────────────────────────────
      # HTTP 配置
      # ──────────────────────────────────────────────────────
      http-method-specify: true  # URL 限流时是否区分 HTTP 方法（GET/POST）

      # Web 上下文统一（默认 false）
      # true: 所有 URL 共享同一个 context（适合简单场景）
      # false: 每个 URL 独立 context（适合复杂链路追踪）
      web-context-unify: false

      # ──────────────────────────────────────────────────────
      # 自定义限流响应（可选）
      # ──────────────────────────────────────────────────────
      # 可以通过代码配置 WebCallbackManager.setUrlBlockHandler()
      # 或使用默认的 "Blocked by Sentinel (flow limiting)"

      # ──────────────────────────────────────────────────────
      # 规则持久化配置（可选，后续高级特性讲解）
      # ──────────────────────────────────────────────────────
      # datasource:
      #   # 从 Nacos 读取规则
      #   flow:
      #     nacos:
      #       server-addr: localhost:8848
      #       dataId: ${spring.application.name}-flow-rules
      #       groupId: SENTINEL_GROUP
      #       rule-type: flow

# ============================================================
# Actuator 配置（可选）
# ============================================================
management:
  endpoints:
    web:
      exposure:
        # 暴露所有端点（生产环境建议只暴露必要端点）
        include: '*'
  endpoint:
    sentinel:
      enabled: true  # 启用 Sentinel 端点

# ============================================================
# 日志配置（可选）
# ============================================================
logging:
  level:
    # Sentinel 日志级别
    com.alibaba.csp.sentinel: INFO
    # 业务日志级别
    com.example.sentinel: DEBUG
```

**关键配置说明：**

1. **`dashboard: localhost:8080`**
   - Dashboard 地址（控制台）
   - 如果 Dashboard 在其他机器，填写实际 IP

2. **`port: 8719`**
   - Client 监听端口（与 Dashboard 通信）
   - 确保此端口未被占用

3. **`eager: true`**
   - 饥饿加载，应用启动时立即初始化 Sentinel
   - 推荐开启，避免首次请求失败

4. **`http-method-specify: true`**
   - 限流时区分 HTTP 方法
   - 如 `GET:/order/{id}` 和 `POST:/order/{id}` 视为不同资源

---

## 4. 启动 Sentinel Dashboard

### 4.1 下载 Dashboard

```bash
# 方式一：直接下载（推荐）
# 访问 https://github.com/alibaba/Sentinel/releases
# 下载 sentinel-dashboard-1.8.6.jar（或最新版本）

# 方式二：使用 wget 下载
wget https://github.com/alibaba/Sentinel/releases/download/1.8.6/sentinel-dashboard-1.8.6.jar
```

### 4.2 启动 Dashboard

```bash
# 启动命令（默认端口 8080）
java -jar sentinel-dashboard-1.8.6.jar

# 自定义端口启动（如 8858）
java -Dserver.port=8858 -jar sentinel-dashboard-1.8.6.jar

# 自定义用户名密码（默认 sentinel/sentinel）
java -Dsentinel.dashboard.auth.username=admin \
     -Dsentinel.dashboard.auth.password=admin123 \
     -jar sentinel-dashboard-1.8.6.jar

# 完整启动命令（推荐）
java -Dserver.port=8858 \
     -Dcsp.sentinel.dashboard.server=localhost:8858 \
     -Dproject.name=sentinel-dashboard \
     -jar sentinel-dashboard-1.8.6.jar
```

**启动成功标志：**

```
INFO 16128 --- [main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 8080 (http)
INFO 16128 --- [main] c.a.c.s.dashboard.DashboardApplication   : Started DashboardApplication in 3.282 seconds
```

### 4.3 访问 Dashboard

```
URL: http://localhost:8080
用户名: sentinel
密码: sentinel
```

**Dashboard 主界面：**

```
┌────────────────────────────────────────────────────┐
│  Sentinel 控制台                                    │
├────────────────────────────────────────────────────┤
│                                                    │
│  左侧菜单：                                         │
│  ├─ 首页                                           │
│  ├─ 实时监控                                        │
│  ├─ 簇点链路                                        │
│  ├─ 流控规则                                        │
│  ├─ 降级规则                                        │
│  ├─ 热点规则                                        │
│  ├─ 系统规则                                        │
│  ├─ 授权规则                                        │
│  └─ 集群流控                                        │
│                                                    │
│  中间区域：                                         │
│  应用列表（目前为空，需要应用启动并发送心跳）         │
│                                                    │
└────────────────────────────────────────────────────┘
```

---

## 5. 实现接口频次控制

### 5.1 创建启动类

```java
package com.example.sentinel;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

/**
 * Sentinel 示例应用启动类
 */
@SpringBootApplication
public class SentinelDemoApplication {

    public static void main(String[] args) {
        SpringApplication.run(SentinelDemoApplication.class, args);

        System.out.println("\n" +
            "===========================================================\n" +
            "  Sentinel Demo 启动成功！\n" +
            "  应用地址: http://localhost:8080\n" +
            "  Sentinel Dashboard: http://localhost:8080\n" +
            "  测试接口: http://localhost:8080/order/create\n" +
            "===========================================================\n");
    }
}
```

---

### 5.2 创建业务接口（方式一：URL 自动限流）

**特点：无需添加任何注解，Sentinel 自动为所有 HTTP 接口埋点**

```java
package com.example.sentinel.controller;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.web.bind.annotation.*;

import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;
import java.util.HashMap;
import java.util.Map;

/**
 * 订单接口 - 演示 URL 自动限流
 *
 * Sentinel 会自动为所有 HTTP 接口创建资源，资源名格式：
 * - http-method-specify=true:  GET:/order/create
 * - http-method-specify=false: /order/create
 */
@RestController
@RequestMapping("/order")
public class OrderController {

    private static final Logger log = LoggerFactory.getLogger(OrderController.class);

    /**
     * 创建订单接口
     *
     * 资源名称：GET:/order/create（自动生成）
     * 无需添加 @SentinelResource 注解
     *
     * 访问方式：
     * curl http://localhost:8080/order/create?productId=1001&quantity=2
     *
     * @param productId 商品 ID
     * @param quantity 数量
     * @return 订单信息
     */
    @GetMapping("/create")
    public Map<String, Object> createOrder(
            @RequestParam(required = false, defaultValue = "1001") String productId,
            @RequestParam(required = false, defaultValue = "1") Integer quantity) {

        log.info("收到创建订单请求，商品ID: {}, 数量: {}", productId, quantity);

        // 模拟业务逻辑
        String orderId = "ORDER-" + System.currentTimeMillis();
        String currentTime = LocalDateTime.now()
            .format(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss"));

        Map<String, Object> result = new HashMap<>();
        result.put("success", true);
        result.put("orderId", orderId);
        result.put("productId", productId);
        result.put("quantity", quantity);
        result.put("timestamp", currentTime);
        result.put("message", "订单创建成功");

        return result;
    }

    /**
     * 查询订单接口
     *
     * 资源名称：GET:/order/{orderId}
     *
     * 访问方式：
     * curl http://localhost:8080/order/ORDER-123456
     *
     * @param orderId 订单 ID
     * @return 订单详情
     */
    @GetMapping("/{orderId}")
    public Map<String, Object> getOrder(@PathVariable String orderId) {

        log.info("收到查询订单请求，订单ID: {}", orderId);

        Map<String, Object> result = new HashMap<>();
        result.put("orderId", orderId);
        result.put("status", "已支付");
        result.put("totalAmount", 299.00);
        result.put("createTime", "2025-01-15 10:30:00");

        return result;
    }

    /**
     * 取消订单接口（演示 POST 请求）
     *
     * 资源名称：POST:/order/{orderId}/cancel
     *
     * 访问方式：
     * curl -X POST http://localhost:8080/order/ORDER-123456/cancel
     *
     * @param orderId 订单 ID
     * @return 取消结果
     */
    @PostMapping("/{orderId}/cancel")
    public Map<String, Object> cancelOrder(@PathVariable String orderId) {

        log.info("收到取消订单请求，订单ID: {}", orderId);

        Map<String, Object> result = new HashMap<>();
        result.put("success", true);
        result.put("orderId", orderId);
        result.put("message", "订单已取消");

        return result;
    }
}
```

**优点：**
- ✅ 无需修改代码，自动限流
- ✅ 适合 RESTful API

**缺点：**
- ❌ 资源名称固定为 URL 路径
- ❌ 无法自定义限流响应

---

### 5.3 创建业务接口（方式二：@SentinelResource 注解）

**特点：自定义资源名、自定义限流响应**

```java
package com.example.sentinel.controller;

import com.alibaba.csp.sentinel.annotation.SentinelResource;
import com.alibaba.csp.sentinel.slots.block.BlockException;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.web.bind.annotation.*;

import java.util.HashMap;
import java.util.Map;

/**
 * 用户接口 - 演示 @SentinelResource 注解
 */
@RestController
@RequestMapping("/user")
public class UserController {

    private static final Logger log = LoggerFactory.getLogger(UserController.class);

    /**
     * 用户注册接口
     *
     * @SentinelResource 注解说明：
     * - value: 资源名称（必填）
     * - blockHandler: 限流处理方法（可选）
     * - fallback: 异常降级方法（可选）
     *
     * 访问方式：
     * curl http://localhost:8080/user/register?username=zhangsan
     *
     * @param username 用户名
     * @return 注册结果
     */
    @PostMapping("/register")
    @SentinelResource(
        value = "userRegister",           // 资源名称（在 Dashboard 配置规则时使用）
        blockHandler = "handleRegisterBlock"  // 限流时调用的方法
    )
    public Map<String, Object> register(@RequestParam String username) {

        log.info("收到用户注册请求，用户名: {}", username);

        // 模拟业务逻辑
        Map<String, Object> result = new HashMap<>();
        result.put("success", true);
        result.put("userId", "USER-" + System.currentTimeMillis());
        result.put("username", username);
        result.put("message", "注册成功");

        return result;
    }

    /**
     * 限流处理方法
     *
     * 注意：
     * 1. 方法签名必须与原方法一致
     * 2. 最后一个参数必须是 BlockException
     * 3. 返回值类型必须与原方法一致
     *
     * @param username 用户名
     * @param ex 限流异常
     * @return 限流响应
     */
    public Map<String, Object> handleRegisterBlock(String username, BlockException ex) {
        log.warn("用户注册接口被限流，用户名: {}, 异常类型: {}", username, ex.getClass().getSimpleName());

        Map<String, Object> result = new HashMap<>();
        result.put("success", false);
        result.put("errorCode", "FLOW_LIMIT");
        result.put("message", "系统繁忙，请稍后再试！");
        result.put("hint", "当前注册人数过多，请等待 1 分钟后重试");

        return result;
    }

    /**
     * 用户登录接口（演示多种限流响应）
     *
     * 访问方式：
     * curl -X POST http://localhost:8080/user/login?username=zhangsan&password=123456
     *
     * @param username 用户名
     * @param password 密码
     * @return 登录结果
     */
    @PostMapping("/login")
    @SentinelResource(
        value = "userLogin",
        blockHandler = "handleLoginBlock",
        fallback = "handleLoginFallback"  // 异常降级方法
    )
    public Map<String, Object> login(
            @RequestParam String username,
            @RequestParam String password) {

        log.info("收到用户登录请求，用户名: {}", username);

        // 模拟业务逻辑（可能抛出异常）
        if ("error".equals(username)) {
            throw new RuntimeException("模拟系统异常");
        }

        Map<String, Object> result = new HashMap<>();
        result.put("success", true);
        result.put("token", "TOKEN-" + System.currentTimeMillis());
        result.put("username", username);

        return result;
    }

    /**
     * 登录限流处理方法
     */
    public Map<String, Object> handleLoginBlock(String username, String password, BlockException ex) {
        log.warn("用户登录接口被限流，用户名: {}", username);

        Map<String, Object> result = new HashMap<>();
        result.put("success", false);
        result.put("errorCode", "FLOW_LIMIT");
        result.put("message", "登录请求过于频繁，请 30 秒后重试");

        return result;
    }

    /**
     * 登录异常降级方法
     *
     * 注意：fallback 方法的最后一个参数是 Throwable
     */
    public Map<String, Object> handleLoginFallback(String username, String password, Throwable ex) {
        log.error("用户登录接口异常，用户名: {}, 异常信息: {}", username, ex.getMessage());

        Map<String, Object> result = new HashMap<>();
        result.put("success", false);
        result.put("errorCode", "SYSTEM_ERROR");
        result.put("message", "系统异常，请稍后重试");

        return result;
    }

    /**
     * 获取用户信息接口（演示使用 blockHandlerClass）
     *
     * 访问方式：
     * curl http://localhost:8080/user/info?userId=USER-123
     *
     * @param userId 用户 ID
     * @return 用户信息
     */
    @GetMapping("/info")
    @SentinelResource(
        value = "getUserInfo",
        blockHandler = "handleBlock",
        blockHandlerClass = UserBlockHandler.class  // 指定限流处理类
    )
    public Map<String, Object> getUserInfo(@RequestParam String userId) {

        log.info("收到获取用户信息请求，用户ID: {}", userId);

        Map<String, Object> result = new HashMap<>();
        result.put("userId", userId);
        result.put("username", "张三");
        result.put("email", "zhangsan@example.com");
        result.put("phone", "13800138000");

        return result;
    }
}

/**
 * 独立的限流处理类
 *
 * 作用：当多个接口使用相同的限流逻辑时，可以抽取到独立的类中
 *
 * 注意：
 * 1. 方法必须是 public static
 * 2. 方法签名必须与原方法一致 + BlockException 参数
 */
class UserBlockHandler {

    private static final Logger log = LoggerFactory.getLogger(UserBlockHandler.class);

    /**
     * 通用限流处理方法
     */
    public static Map<String, Object> handleBlock(String userId, BlockException ex) {
        log.warn("接口被限流，用户ID: {}, 规则类型: {}", userId, ex.getRule());

        Map<String, Object> result = new HashMap<>();
        result.put("success", false);
        result.put("errorCode", "FLOW_LIMIT");
        result.put("message", "访问过于频繁，请稍后再试");
        result.put("retryAfter", 60);  // 建议 60 秒后重试

        return result;
    }
}
```

**@SentinelResource 核心参数：**

| 参数 | 说明 | 示例 |
|------|------|------|
| `value` | 资源名称（必填） | `"userRegister"` |
| `blockHandler` | 限流/熔断处理方法 | `"handleBlock"` |
| `blockHandlerClass` | 限流处理类（静态方法） | `UserBlockHandler.class` |
| `fallback` | 异常降级方法 | `"handleFallback"` |
| `fallbackClass` | 异常降级类 | `UserFallback.class` |
| `exceptionsToIgnore` | 忽略的异常 | `IllegalArgumentException.class` |

---

### 5.4 自定义全局限流响应（可选）

**适用场景：统一处理所有 URL 限流响应**

```java
package com.example.sentinel.config;

import com.alibaba.csp.sentinel.adapter.spring.webmvc.callback.BlockExceptionHandler;
import com.alibaba.csp.sentinel.slots.block.BlockException;
import com.alibaba.csp.sentinel.slots.block.authority.AuthorityException;
import com.alibaba.csp.sentinel.slots.block.degrade.DegradeException;
import com.alibaba.csp.sentinel.slots.block.flow.FlowException;
import com.alibaba.csp.sentinel.slots.block.flow.param.ParamFlowException;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.http.MediaType;
import org.springframework.stereotype.Component;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.util.HashMap;
import java.util.Map;

/**
 * 全局限流异常处理器
 *
 * 作用：统一处理所有 URL 限流响应（不使用 @SentinelResource 的接口）
 */
@Component
public class CustomBlockExceptionHandler implements BlockExceptionHandler {

    private static final Logger log = LoggerFactory.getLogger(CustomBlockExceptionHandler.class);

    private final ObjectMapper objectMapper = new ObjectMapper();

    @Override
    public void handle(HttpServletRequest request, HttpServletResponse response, BlockException ex)
            throws Exception {

        log.warn("请求被限流，URL: {}, 规则类型: {}", request.getRequestURI(), ex.getClass().getSimpleName());

        // 构建响应数据
        Map<String, Object> result = new HashMap<>();
        result.put("success", false);
        result.put("timestamp", System.currentTimeMillis());
        result.put("path", request.getRequestURI());

        // 根据不同类型的限流异常，返回不同的提示
        if (ex instanceof FlowException) {
            // 流控异常（QPS 限流）
            result.put("errorCode", "FLOW_LIMIT");
            result.put("message", "访问过于频繁，请稍后再试");
            result.put("hint", "当前 QPS 超过限制");
        } else if (ex instanceof DegradeException) {
            // 熔断降级异常
            result.put("errorCode", "DEGRADE");
            result.put("message", "服务暂时不可用，请稍后再试");
            result.put("hint", "服务已熔断，正在恢复中");
        } else if (ex instanceof ParamFlowException) {
            // 热点参数限流异常
            result.put("errorCode", "PARAM_FLOW_LIMIT");
            result.put("message", "当前参数访问过于频繁");
            result.put("hint", "热点参数限流");
        } else if (ex instanceof AuthorityException) {
            // 授权异常
            result.put("errorCode", "AUTHORITY_DENIED");
            result.put("message", "无权访问");
        } else {
            // 其他异常
            result.put("errorCode", "BLOCKED");
            result.put("message", "请求被拦截");
        }

        // 设置响应
        response.setStatus(429);  // HTTP 429 Too Many Requests
        response.setContentType(MediaType.APPLICATION_JSON_VALUE);
        response.setCharacterEncoding("UTF-8");

        // 写入响应
        response.getWriter().write(objectMapper.writeValueAsString(result));
    }
}
```

---

## 6. 测试限流效果

### 6.1 启动应用

```bash
# 方式一：IDE 启动
# 直接运行 SentinelDemoApplication.main()

# 方式二：Maven 打包启动
mvn clean package
java -jar target/sentinel-demo-1.0.0.jar
```

**启动成功日志：**

```
INFO 12345 --- [main] c.e.s.SentinelDemoApplication : Started SentinelDemoApplication in 3.5 seconds
INFO 12345 --- [main] c.a.c.s.SentinelWebAutoConfiguration : [Sentinel Starter] register SentinelWebInterceptor
```

---

### 6.2 验证 Dashboard 连接

#### **Step 1：访问业务接口（触发懒加载）**

```bash
# 访问订单创建接口
curl http://localhost:8080/order/create?productId=1001&quantity=2

# 预期响应
{
  "success": true,
  "orderId": "ORDER-1705304567890",
  "productId": "1001",
  "quantity": 2,
  "timestamp": "2025-01-15 12:30:00",
  "message": "订单创建成功"
}
```

#### **Step 2：登录 Dashboard 查看应用**

```
1. 访问 http://localhost:8080
2. 登录（sentinel/sentinel）
3. 左侧菜单 → 点击 "sentinel-demo"
4. 可以看到：
   - 实时监控：QPS、RT、线程数曲线
   - 簇点链路：所有资源列表
```

**簇点链路示例：**

```
┌────────────────────────────────────────────────────┐
│  簇点链路                                           │
├────────────────────────────────────────────────────┤
│  资源名称              | 通过QPS | 阻塞QPS | RT(ms) │
├────────────────────────────────────────────────────┤
│  GET:/order/create    |    5    |    0    |  120   │
│  GET:/order/{orderId} |    2    |    0    |   95   │
│  POST:/user/register  |    0    |    0    |    0   │
└────────────────────────────────────────────────────┘
```

---

### 6.3 配置限流规则（Dashboard 方式）

#### **Step 1：进入流控规则页面**

```
Dashboard → sentinel-demo → 流控规则 → 新增流控规则
```

#### **Step 2：配置规则**

```
┌────────────────────────────────────────────────────┐
│  新增流控规则                                        │
├────────────────────────────────────────────────────┤
│                                                    │
│  资源名：GET:/order/create  （从下拉框选择）         │
│                                                    │
│  针对来源：default（默认，表示所有来源）              │
│                                                    │
│  阈值类型：○ QPS（每秒查询率）                       │
│           ○ 线程数                                 │
│                                                    │
│  单机阈值：10  （每秒最多 10 个请求）                │
│                                                    │
│  流控模式：○ 直接（默认）                            │
│           ○ 关联                                   │
│           ○ 链路                                   │
│                                                    │
│  流控效果：○ 快速失败（默认）                        │
│           ○ Warm Up（预热）                        │
│           ○ 排队等待                               │
│                                                    │
│  [新增]  [取消]                                    │
└────────────────────────────────────────────────────┘
```

**配置示例：**

| 配置项 | 值 | 说明 |
|--------|-----|------|
| 资源名 | `GET:/order/create` | 订单创建接口 |
| 阈值类型 | QPS | 每秒查询率 |
| 单机阈值 | 10 | 每秒最多 10 个请求 |
| 流控模式 | 直接 | 直接限流此资源 |
| 流控效果 | 快速失败 | 超过阈值立即拒绝 |

点击**"新增"**按钮保存规则。

---

### 6.4 测试限流效果

#### **方式一：使用 curl 手动测试**

```bash
# 快速发送 20 个请求
for i in {1..20}; do
  curl http://localhost:8080/order/create
  echo ""
done
```

**预期结果：**

```
# 前 10 个请求成功
{"success":true,"orderId":"ORDER-1705304567890",...}
{"success":true,"orderId":"ORDER-1705304567891",...}
...（共 10 个）

# 后 10 个请求被限流
Blocked by Sentinel (flow limiting)
Blocked by Sentinel (flow limiting)
...（共 10 个）
```

---

#### **方式二：使用 Apache Bench (ab) 压测**

```bash
# 安装 ab（Mac）
brew install httpd

# 安装 ab（Ubuntu）
sudo apt-get install apache2-utils

# 压测：100 个请求，并发 20
ab -n 100 -c 20 http://localhost:8080/order/create
```

**压测结果分析：**

```
Concurrency Level:      20
Time taken for tests:   1.234 seconds
Complete requests:      100
Failed requests:        90  ← 90 个请求被限流（429 状态码）
Non-2xx responses:      90

Requests per second:    81.03 [#/sec] (mean)
Time per request:       246.8 [ms] (mean)

Percentage of the requests served within a certain time (ms)
  50%    120
  90%    250
  95%    300
  99%    450
```

---

#### **方式三：使用 JMeter 压测**

**配置步骤：**

1. 下载 JMeter：https://jmeter.apache.org/download_jmeter.cgi
2. 新建线程组：
   - 线程数：20
   - 循环次数：10
3. 添加 HTTP 请求：
   - 服务器：localhost
   - 端口：8080
   - 路径：/order/create
4. 添加监听器：
   - 聚合报告
   - 查看结果树
5. 运行测试

**JMeter 结果：**

```
样本数：200
平均响应时间：150ms
吞吐量：130 req/sec
错误率：95%  ← 大部分请求被限流
```

---

### 6.5 观察实时监控

在 Dashboard 的"实时监控"页面，可以看到：

```
┌────────────────────────────────────────────────────┐
│  实时监控 - GET:/order/create                       │
├────────────────────────────────────────────────────┤
│                                                    │
│  通过 QPS (绿线)                                    │
│     │                                              │
│  10 │  ──────────────────────  ← 稳定在 10         │
│     │                                              │
│   0 └──────────────────────────────────> 时间      │
│                                                    │
│  阻塞 QPS (红线)                                    │
│     │                                              │
│  90 │        ████████████████  ← 大量请求被限流     │
│     │                                              │
│   0 └──────────────────────────────────────> 时间  │
│                                                    │
│  平均 RT (蓝线)                                     │
│     │                                              │
│ 150 │  ────────────────────── ← 响应时间稳定        │
│     │                                              │
│   0 └──────────────────────────────────────> 时间  │
│                                                    │
└────────────────────────────────────────────────────┘
```

---

### 6.6 配置规则（代码方式）

**适用场景：规则固定，无需动态调整**

```java
package com.example.sentinel.config;

import com.alibaba.csp.sentinel.slots.block.RuleConstant;
import com.alibaba.csp.sentinel.slots.block.flow.FlowRule;
import com.alibaba.csp.sentinel.slots.block.flow.FlowRuleManager;
import org.springframework.context.annotation.Configuration;

import javax.annotation.PostConstruct;
import java.util.ArrayList;
import java.util.List;

/**
 * Sentinel 规则配置类
 *
 * 作用：通过代码初始化限流规则
 */
@Configuration
public class SentinelRuleConfig {

    /**
     * 初始化流控规则
     *
     * 注意：代码配置的规则重启后会丢失，建议使用 Dashboard 或规则持久化
     */
    @PostConstruct
    public void initFlowRules() {
        List<FlowRule> rules = new ArrayList<>();

        // 规则 1：订单创建接口 QPS 限流
        FlowRule orderCreateRule = new FlowRule();
        orderCreateRule.setResource("GET:/order/create");  // 资源名
        orderCreateRule.setGrade(RuleConstant.FLOW_GRADE_QPS);  // QPS 模式
        orderCreateRule.setCount(10);  // QPS 阈值：10
        orderCreateRule.setLimitApp("default");  // 针对来源（default=所有）
        rules.add(orderCreateRule);

        // 规则 2：用户注册接口 QPS 限流
        FlowRule userRegisterRule = new FlowRule();
        userRegisterRule.setResource("userRegister");  // 资源名（@SentinelResource 定义）
        userRegisterRule.setGrade(RuleConstant.FLOW_GRADE_QPS);
        userRegisterRule.setCount(5);  // QPS 阈值：5
        rules.add(userRegisterRule);

        // 规则 3：订单查询接口 并发线程数限流
        FlowRule orderQueryRule = new FlowRule();
        orderQueryRule.setResource("GET:/order/{orderId}");
        orderQueryRule.setGrade(RuleConstant.FLOW_GRADE_THREAD);  // 线程数模式
        orderQueryRule.setCount(10);  // 最大并发线程数：10
        rules.add(orderQueryRule);

        // 加载规则
        FlowRuleManager.loadRules(rules);

        System.out.println("Sentinel 流控规则初始化完成，共加载 " + rules.size() + " 条规则");
    }
}
```

---

## 7. 常见问题排查

### 7.1 Dashboard 看不到应用

**原因：**
1. 应用未启动
2. 应用未发送心跳到 Dashboard
3. 端口配置错误
4. 防火墙阻止通信

**解决方案：**

```bash
# 1. 检查应用是否启动
curl http://localhost:8080/order/create

# 2. 检查 Sentinel 客户端端口是否监听
# Linux/Mac
lsof -i:8719

# Windows
netstat -ano | findstr 8719

# 3. 检查 Dashboard 地址配置
# application.yml
spring.cloud.sentinel.transport.dashboard: localhost:8080  # ← 确认正确

# 4. 手动触发心跳（访问任意业务接口）
curl http://localhost:8080/order/create

# 5. 查看 Sentinel 日志
# 日志位置：${user.home}/logs/csp/sentinel-record.log.xxx
tail -f ~/logs/csp/sentinel-record.log.*
```

---

### 7.2 规则不生效

**原因：**
1. 资源名配置错误
2. 规则未保存
3. Dashboard 规则未推送成功

**解决方案：**

```bash
# 1. 确认资源名是否正确
# Dashboard → 簇点链路 → 查看实际的资源名

# 2. 检查规则是否保存
# Dashboard → 流控规则 → 查看规则列表

# 3. 查看应用日志
tail -f logs/application.log | grep Sentinel

# 4. 通过 Actuator 查看规则
curl http://localhost:8080/actuator/sentinel
```

---

### 7.3 限流后无响应

**原因：**
- 自定义的 blockHandler 方法签名错误

**解决方案：**

```java
// ❌ 错误：缺少 BlockException 参数
public Map<String, Object> handleBlock(String username) {
    ...
}

// ✅ 正确：必须包含 BlockException 参数
public Map<String, Object> handleBlock(String username, BlockException ex) {
    ...
}

// ✅ 正确：blockHandlerClass 中的方法必须是 static
public static Map<String, Object> handleBlock(String username, BlockException ex) {
    ...
}
```

---

### 7.4 性能问题

**问题：Sentinel 是否影响性能？**

**答案：影响极小**

```
性能测试数据（10,000 QPS）：

无 Sentinel：
- 平均响应时间：100ms
- CPU 使用率：30%

有 Sentinel：
- 平均响应时间：102ms（+2ms）
- CPU 使用率：32%（+2%）

结论：Sentinel 开销极小（< 2%）
```

---

## 📚 总结

### 快速集成步骤回顾

1. ✅ **添加依赖**：`spring-cloud-starter-alibaba-sentinel`
2. ✅ **配置 Dashboard**：`spring.cloud.sentinel.transport.dashboard`
3. ✅ **启动 Dashboard**：`java -jar sentinel-dashboard.jar`
4. ✅ **定义资源**：
   - 方式一：URL 自动限流（无需注解）
   - 方式二：@SentinelResource 注解
5. ✅ **配置规则**：
   - 方式一：Dashboard 配置（推荐）
   - 方式二：代码配置
6. ✅ **测试验证**：压测工具（curl/ab/JMeter）

### 接口频次控制关键配置

```yaml
# 1. Dashboard 配置
spring.cloud.sentinel.transport.dashboard: localhost:8080

# 2. 饥饿加载（推荐）
spring.cloud.sentinel.eager: true

# 3. 区分 HTTP 方法
spring.cloud.sentinel.http-method-specify: true
```

### 限流规则核心参数

| 参数 | 说明 | 示例 |
|------|------|------|
| 资源名 | 需要限流的资源 | `GET:/order/create` |
| 阈值类型 | QPS 或 并发线程数 | QPS |
| 单机阈值 | 限流阈值 | 10 |
| 流控模式 | 直接/关联/链路 | 直接 |
| 流控效果 | 快速失败/预热/排队 | 快速失败 |

---

好了，**第3步：快速入门**就讲到这里。你现在应该能够：

- ✅ 成功集成 Sentinel 到 Spring Cloud 项目
- ✅ 启动 Sentinel Dashboard 控制台
- ✅ 对指定接口进行 QPS 限流
- ✅ 自定义限流响应
- ✅ 使用压测工具验证限流效果

如果你已经成功实现了接口频次控制，请告诉我**"下一步"**，我会继续讲解**第4步：核心功能演示**，深入学习流量控制、熔断降级、系统保护、热点参数限流等高级特性！🚀
