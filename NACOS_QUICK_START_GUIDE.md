# Nacos 服务发现实践指南

> 本指南提供一个最简单的 Spring Cloud + Nacos 服务发现示例
>
> 版本：Spring Cloud 2022.0.0 + Spring Cloud Alibaba 2022.0.0.0

---

## 📁 项目结构

```
nacos-practice-demo/
├── pom.xml                          # 父项目 POM（管理版本）
├── nacos-provider/                  # 服务提供者
│   ├── pom.xml
│   └── src/main/java/com/example/provider/
│       ├── ProviderApplication.java      # 启动类
│       └── controller/HelloController.java
│   └── src/main/resources/
│       └── application.yml               # 配置文件
├── nacos-consumer/                  # 服务消费者
│   ├── pom.xml
│   └── src/main/java/com/example/consumer/
│       ├── ConsumerApplication.java      # 启动类
│       ├── config/RestTemplateConfig.java # RestTemplate 配置
│       ├── feign/ProviderFeignClient.java # Feign 客户端接口
│       └── controller/ConsumerController.java
│   └── src/main/resources/
│       └── application.yml               # 配置文件
```

---

## 1️⃣ 父项目 POM 配置

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.example</groupId>
    <artifactId>nacos-practice-demo</artifactId>
    <version>1.0.0</version>
    <packaging>pom</packaging>

    <modules>
        <module>nacos-provider</module>
        <module>nacos-consumer</module>
    </modules>

    <properties>
        <java.version>17</java.version>
        <maven.compiler.source>17</maven.compiler.source>
        <maven.compiler.target>17</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>

        <!-- Spring Boot 版本：3.0.x 对应 Spring Cloud 2022.x -->
        <spring-boot.version>3.0.2</spring-boot.version>

        <!-- Spring Cloud 版本：2022.0.0 对应代号 Kilburn -->
        <spring-cloud.version>2022.0.0</spring-cloud.version>

        <!-- Spring Cloud Alibaba 版本：与 Spring Cloud 2022.x 适配 -->
        <spring-cloud-alibaba.version>2022.0.0.0</spring-cloud-alibaba.version>
    </properties>

    <!--
        dependencyManagement 作用：
        统一管理子模块的依赖版本，避免版本冲突
        子模块引用时无需指定版本号
    -->
    <dependencyManagement>
        <dependencies>
            <!-- Spring Boot 依赖管理 -->
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-dependencies</artifactId>
                <version>${spring-boot.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>

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
                <version>${spring-boot.version}</version>
            </plugin>
        </plugins>
    </build>
</project>
```

---

## 2️⃣ 服务提供者（Provider）

### 📄 nacos-provider/pom.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <!-- 继承父项目的版本管理 -->
    <parent>
        <groupId>com.example</groupId>
        <artifactId>nacos-practice-demo</artifactId>
        <version>1.0.0</version>
    </parent>

    <artifactId>nacos-provider</artifactId>

    <dependencies>
        <!--
            Spring Boot Web Starter
            作用：提供 Spring MVC、内嵌 Tomcat、JSON 序列化等 Web 开发能力
        -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <!--
            Nacos 服务发现 Starter
            作用：
            1. 自动配置 NacosServiceRegistry（服务注册器）
            2. 启动时自动将当前服务注册到 Nacos Server
            3. 定期发送心跳维持服务健康状态
            4. 应用关闭时自动注销服务
        -->
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
        </dependency>

        <!--
            Spring Cloud LoadBalancer（可选但推荐）
            作用：虽然 Provider 不需要调用其他服务，但添加此依赖可以：
            1. 避免某些情况下的启动警告
            2. 支持健康检查端点
        -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-loadbalancer</artifactId>
        </dependency>

        <!--
            Spring Boot Actuator（可选）
            作用：提供健康检查、监控端点
            可通过 /actuator/health 查看服务状态
        -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
    </dependencies>

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

### 📄 nacos-provider/src/main/resources/application.yml

```yaml
# ============================================================
# 服务提供者配置文件
# ============================================================

server:
  port: 8081  # 服务端口（可以启动多个实例，使用不同端口测试负载均衡）

spring:
  application:
    # 服务名称：非常重要！
    # 作用：
    # 1. 这是服务在 Nacos 注册中心的唯一标识
    # 2. 消费者通过这个名称来查找和调用服务
    # 3. 相同 name 的多个实例会被识别为同一个服务的不同副本（用于负载均衡）
    name: nacos-provider

  cloud:
    nacos:
      # 用户名和密码（Nacos 3.0+ 必须配置）
      # 原因：Nacos 3.0 版本增强了安全性，默认开启鉴权
      # 默认用户名密码都是 nacos（首次登录控制台会提示绑定）
      username: nacos
      password: nacos

      discovery:
        # Nacos Server 地址
        # 如果是集群部署，可以配置多个地址，逗号分隔：
        # server-addr: 192.168.1.10:8848,192.168.1.11:8848,192.168.1.12:8848
        server-addr: 127.0.0.1:8848

        # 命名空间 ID（可选）
        # 作用：逻辑隔离不同环境的服务（开发/测试/生产）
        # 默认为 public 命名空间
        # namespace: dev-namespace-id

        # 服务分组（可选）
        # 作用：在同一命名空间下进一步分组，隔离不同业务线的服务
        # 默认为 DEFAULT_GROUP
        # group: PROVIDER_GROUP

        # 集群名称（可选）
        # 作用：标识服务所属的集群（如北京机房、上海机房）
        # 负载均衡时可以优先选择同集群的实例
        cluster-name: DEFAULT

        # 实例权重（可选，默认 1.0）
        # 作用：控制负载均衡时的流量分配
        # 取值范围：0.01 ~ 100.0，权重越大，分配的流量越多
        # 应用场景：灰度发布时可以给新版本设置较小权重
        # weight: 1.0

        # 临时实例标识（可选，默认 true）
        # 作用：
        # - true（临时实例，AP 模式）：Client 主动心跳，适合云原生快速弹性伸缩
        # - false（持久实例，CP 模式）：Server 主动探测，适合需要强一致性的场景
        ephemeral: true

        # 元数据（可选）
        # 作用：存储自定义信息，可用于灰度路由、版本控制等
        # metadata:
        #   version: 1.0.0
        #   region: beijing

# ============================================================
# Actuator 配置（可选，用于健康检查和监控）
# ============================================================
management:
  endpoints:
    web:
      exposure:
        # 暴露所有端点（生产环境建议只暴露必要的端点）
        include: '*'
  endpoint:
    health:
      # 显示详细健康信息
      show-details: always

# ============================================================
# 日志配置（可选，方便调试）
# ============================================================
logging:
  level:
    # Nacos 相关日志级别设为 DEBUG，可以看到服务注册的详细过程
    com.alibaba.nacos: INFO
    com.alibaba.cloud.nacos: INFO
```

### 📄 nacos-provider/src/main/java/com/example/provider/ProviderApplication.java

```java
package com.example.provider;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;

/**
 * 服务提供者启动类
 *
 * 注解说明：
 * 1. @SpringBootApplication：Spring Boot 应用标识，包含自动配置、组件扫描等功能
 * 2. @EnableDiscoveryClient：启用服务发现功能
 *    作用：
 *    - 激活 Spring Cloud 的服务注册与发现能力
 *    - 触发 NacosServiceRegistry 的自动配置
 *    - 在应用启动完成后，自动将服务注册到 Nacos Server
 *    - 在应用关闭时，自动从 Nacos Server 注销服务
 *
 * 注意：
 * - Spring Cloud 2020.0.0 之后，@EnableDiscoveryClient 可以省略
 * - 只要引入了 nacos-discovery 依赖，服务发现功能就会自动启用
 * - 但显式添加此注解可以提高代码可读性，明确表明这是一个服务发现应用
 */
@SpringBootApplication
@EnableDiscoveryClient
public class ProviderApplication {

    public static void main(String[] args) {
        SpringApplication.run(ProviderApplication.class, args);

        System.out.println("\n" +
            "===========================================================\n" +
            "  服务提供者启动成功！\n" +
            "  服务名称: nacos-provider\n" +
            "  访问地址: http://localhost:8081\n" +
            "  测试接口: http://localhost:8081/hello?name=张三\n" +
            "  Nacos 控制台: http://127.0.0.1:8848/nacos\n" +
            "===========================================================\n");
    }
}
```

### 📄 nacos-provider/src/main/java/com/example/provider/controller/HelloController.java

```java
package com.example.provider.controller;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;

/**
 * 服务提供者的业务接口
 *
 * 作用：提供一个简单的 /hello 接口，供消费者调用
 */
@RestController
public class HelloController {

    private static final Logger log = LoggerFactory.getLogger(HelloController.class);

    /**
     * 从配置文件中读取服务端口
     * 作用：在响应中返回端口号，方便测试负载均衡效果
     * （启动多个实例时，可以通过端口号区分是哪个实例响应的请求）
     */
    @Value("${server.port}")
    private String serverPort;

    /**
     * Hello 接口
     *
     * @param name 请求参数（可选）
     * @return 返回问候消息，包含请求参数、实例端口号、时间戳
     *
     * 访问方式：
     * - http://localhost:8081/hello?name=张三
     * - http://localhost:8081/hello（不传参数）
     *
     * 响应示例：
     * "你好，张三！这是来自服务提供者 [端口: 8081] 的响应，时间: 2025-01-15 10:30:45"
     */
    @GetMapping("/hello")
    public String hello(@RequestParam(value = "name", defaultValue = "访客") String name) {
        // 记录日志，方便观察请求处理情况
        log.info("收到请求，参数 name = {}", name);

        // 格式化当前时间
        String currentTime = LocalDateTime.now()
            .format(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss"));

        // 构建响应消息
        String response = String.format(
            "你好，%s！这是来自服务提供者 [端口: %s] 的响应，时间: %s",
            name,
            serverPort,
            currentTime
        );

        return response;
    }

    /**
     * 健康检查接口（可选）
     *
     * 作用：提供一个简单的健康检查端点
     * 某些场景下（如 Kubernetes 健康探测）可能需要自定义健康检查接口
     *
     * 注意：Spring Boot Actuator 已经提供了 /actuator/health 端点
     * 这里只是演示如何自定义一个简单的健康检查接口
     */
    @GetMapping("/health")
    public String health() {
        return "Provider is healthy, port: " + serverPort;
    }
}
```

---

## 3️⃣ 服务消费者（Consumer）

### 📄 nacos-consumer/pom.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <!-- 继承父项目的版本管理 -->
    <parent>
        <groupId>com.example</groupId>
        <artifactId>nacos-practice-demo</artifactId>
        <version>1.0.0</version>
    </parent>

    <artifactId>nacos-consumer</artifactId>

    <dependencies>
        <!-- Spring Boot Web Starter -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <!-- Nacos 服务发现 Starter -->
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
        </dependency>

        <!--
            Spring Cloud LoadBalancer
            作用：提供负载均衡能力
            重要性：消费者必须引入此依赖！
            原因：
            1. @LoadBalanced 注解需要此依赖才能生效
            2. RestTemplate 才能通过服务名调用服务（如 http://nacos-provider/hello）
            3. 提供负载均衡算法（默认是轮询 Round Robin）
        -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-loadbalancer</artifactId>
        </dependency>

        <!--
            OpenFeign（可选，提供声明式 HTTP 客户端）
            作用：简化服务间的 HTTP 调用
            优势：
            1. 使用接口 + 注解的方式定义 HTTP 调用，无需手动拼接 URL
            2. 自动集成负载均衡（基于 LoadBalancer）
            3. 支持请求/响应拦截器、熔断降级等高级功能

            如果你不需要使用 Feign，可以注释掉此依赖，只用 RestTemplate
        -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-openfeign</artifactId>
        </dependency>

        <!-- Spring Boot Actuator（可选） -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
    </dependencies>

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

### 📄 nacos-consumer/src/main/resources/application.yml

```yaml
# ============================================================
# 服务消费者配置文件
# ============================================================

server:
  port: 8082  # 消费者端口

spring:
  application:
    # 服务名称：虽然消费者可以不注册到 Nacos，但建议也注册上去
    # 好处：
    # 1. 方便监控和管理（在 Nacos 控制台可以看到所有服务）
    # 2. 如果消费者也需要被其他服务调用，就必须注册
    name: nacos-consumer

  cloud:
    nacos:
      username: nacos
      password: nacos

      discovery:
        server-addr: 127.0.0.1:8848

        # 消费者特有配置（可选）
        # 是否启用服务发现功能
        # enabled: true

        # 是否将自己注册到 Nacos（可选，默认 true）
        # 如果消费者只调用服务，不被调用，可以设置为 false
        # register-enabled: true

# ============================================================
# LoadBalancer 配置（可选）
# ============================================================
spring:
  cloud:
    loadbalancer:
      # 启用 Nacos 的负载均衡策略
      # 作用：使用 Nacos 提供的负载均衡规则（如同集群优先）
      nacos:
        enabled: true

      # 负载均衡策略（可选）
      # ribbon:
      #   enabled: false  # 禁用旧版 Ribbon（Spring Cloud 2020+ 已废弃 Ribbon）

# ============================================================
# Feign 配置（如果使用 OpenFeign）
# ============================================================
feign:
  client:
    config:
      default:
        # 连接超时时间（毫秒）
        connectTimeout: 5000
        # 读取超时时间（毫秒）
        readTimeout: 5000
        # 日志级别：NONE, BASIC, HEADERS, FULL
        loggerLevel: BASIC

# ============================================================
# Actuator 配置（可选）
# ============================================================
management:
  endpoints:
    web:
      exposure:
        include: '*'

# ============================================================
# 日志配置（可选）
# ============================================================
logging:
  level:
    com.alibaba.nacos: INFO
    com.alibaba.cloud.nacos: INFO
    # Feign 日志（如果使用 OpenFeign）
    com.example.consumer.feign: DEBUG
```

### 📄 nacos-consumer/src/main/java/com/example/consumer/ConsumerApplication.java

```java
package com.example.consumer;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.cloud.openfeign.EnableFeignClients;

/**
 * 服务消费者启动类
 *
 * 注解说明：
 * 1. @SpringBootApplication：Spring Boot 应用标识
 * 2. @EnableDiscoveryClient：启用服务发现功能（可省略）
 * 3. @EnableFeignClients：启用 OpenFeign 客户端功能
 *    作用：
 *    - 扫描 @FeignClient 注解的接口
 *    - 为这些接口生成动态代理实现类
 *    - 自动配置 Feign 客户端（包括负载均衡、编解码器等）
 *
 *    注意：如果不使用 Feign，可以删除此注解
 */
@SpringBootApplication
@EnableDiscoveryClient
@EnableFeignClients  // 如果不使用 Feign，删除此注解
public class ConsumerApplication {

    public static void main(String[] args) {
        SpringApplication.run(ConsumerApplication.class, args);

        System.out.println("\n" +
            "===========================================================\n" +
            "  服务消费者启动成功！\n" +
            "  服务名称: nacos-consumer\n" +
            "  访问地址: http://localhost:8082\n" +
            "  测试 RestTemplate: http://localhost:8082/test-rest?name=李四\n" +
            "  测试 Feign: http://localhost:8082/test-feign?name=王五\n" +
            "===========================================================\n");
    }
}
```

### 📄 nacos-consumer/src/main/java/com/example/consumer/config/RestTemplateConfig.java

```java
package com.example.consumer.config;

import org.springframework.cloud.client.loadbalancer.LoadBalanced;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.client.RestTemplate;

/**
 * RestTemplate 配置类
 *
 * 作用：配置支持负载均衡的 RestTemplate
 */
@Configuration
public class RestTemplateConfig {

    /**
     * 创建 RestTemplate Bean
     *
     * @LoadBalanced 注解作用：
     * 1. 拦截 RestTemplate 的 HTTP 请求
     * 2. 将服务名（如 nacos-provider）解析为实际的 IP 地址和端口
     * 3. 应用负载均衡策略，从多个实例中选择一个进行调用
     *
     * 工作原理：
     * - 请求 URL：http://nacos-provider/hello
     * - LoadBalancer 拦截器识别出服务名 "nacos-provider"
     * - 从 Nacos 获取该服务的所有健康实例（如 [192.168.1.10:8081, 192.168.1.11:8081]）
     * - 根据负载均衡算法（默认轮询）选择一个实例
     * - 将 URL 替换为实际地址：http://192.168.1.10:8081/hello
     * - 发起真正的 HTTP 请求
     *
     * 注意：
     * - 必须添加 @LoadBalanced 注解，否则 RestTemplate 无法识别服务名
     * - 必须引入 spring-cloud-starter-loadbalancer 依赖
     * - 如果不加 @LoadBalanced，使用服务名会报错：UnknownHostException
     */
    @Bean
    @LoadBalanced
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }

    // ========== 扩展知识（可选） ==========

    /**
     * 如果需要同时使用带负载均衡和不带负载均衡的 RestTemplate：
     * 可以定义两个 Bean，通过 @Qualifier 注解区分
     */

    // @Bean
    // @Qualifier("normalRestTemplate")
    // public RestTemplate normalRestTemplate() {
    //     // 普通的 RestTemplate，用于调用外部 API（如第三方接口）
    //     return new RestTemplate();
    // }

    // 使用时：
    // @Autowired
    // @Qualifier("normalRestTemplate")
    // private RestTemplate normalRestTemplate;
}
```

### 📄 nacos-consumer/src/main/java/com/example/consumer/feign/ProviderFeignClient.java

```java
package com.example.consumer.feign;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;

/**
 * 服务提供者的 Feign 客户端接口
 *
 * @FeignClient 注解作用：
 * 1. 声明这是一个 Feign 客户端接口
 * 2. value/name 属性指定目标服务名（必须与提供者在 Nacos 注册的名称一致）
 * 3. Spring 会为这个接口生成动态代理实现类
 * 4. 自动集成负载均衡、服务发现、熔断降级等功能
 *
 * 使用方式：
 * 1. 定义接口方法，方法签名要与服务提供者的 Controller 方法一致
 * 2. 使用 Spring MVC 注解（@GetMapping, @PostMapping 等）声明 HTTP 请求
 * 3. 在需要调用的地方 @Autowired 注入即可，像调用本地方法一样调用远程服务
 *
 * 优势：
 * - 代码简洁：无需手动拼接 URL、处理请求响应
 * - 类型安全：编译期检查方法签名和参数类型
 * - 易于维护：接口定义即文档，清晰明了
 */
@FeignClient(
    name = "nacos-provider",  // 目标服务名（必须与 Provider 的 spring.application.name 一致）

    // path = "/api"  // （可选）统一前缀，如果提供者的所有接口都有 /api 前缀

    // fallback = ProviderFeignClientFallback.class  // （可选）降级处理类
    // fallbackFactory = ProviderFeignClientFallbackFactory.class  // （可选）降级工厂类

    // configuration = FeignClientConfig.class  // （可选）自定义配置类
)
public interface ProviderFeignClient {

    /**
     * 调用服务提供者的 /hello 接口
     *
     * 方法签名说明：
     * 1. @GetMapping：声明这是一个 GET 请求，路径为 /hello
     * 2. @RequestParam：声明请求参数 name
     *    - value：参数名称（必须与服务提供者一致）
     *    - required：是否必填（false 表示可选）
     *    - defaultValue：默认值（如果不传参数，使用此默认值）
     * 3. 返回值类型：必须与服务提供者的返回值类型匹配（会自动进行 JSON 反序列化）
     *
     * 注意事项：
     * - 方法名可以随意定义（如 hello, sayHello, callHello），不影响功能
     * - 但 @GetMapping 的 value 必须与提供者的接口路径一致
     * - @RequestParam 的 value 必须与提供者的参数名一致
     * - 如果参数是对象，使用 @RequestBody；如果是路径参数，使用 @PathVariable
     */
    @GetMapping("/hello")
    String hello(@RequestParam(value = "name", required = false, defaultValue = "访客") String name);

    // ========== 其他常见示例（供参考） ==========

    /**
     * POST 请求示例：传递 JSON 对象
     */
    // @PostMapping("/user/save")
    // String saveUser(@RequestBody UserDTO user);

    /**
     * 路径参数示例
     */
    // @GetMapping("/user/{id}")
    // UserDTO getUserById(@PathVariable("id") Long id);

    /**
     * 多个请求参数示例
     */
    // @GetMapping("/user/list")
    // List<UserDTO> listUsers(
    //     @RequestParam("page") int page,
    //     @RequestParam("size") int size
    // );

    /**
     * 请求头示例
     */
    // @GetMapping("/user/info")
    // UserDTO getUserInfo(@RequestHeader("Authorization") String token);
}
```

### 📄 nacos-consumer/src/main/java/com/example/consumer/controller/ConsumerController.java

```java
package com.example.consumer.controller;

import com.example.consumer.feign.ProviderFeignClient;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.cloud.client.ServiceInstance;
import org.springframework.cloud.client.discovery.DiscoveryClient;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.client.RestTemplate;

import java.util.List;

/**
 * 服务消费者的 Controller
 *
 * 作用：演示三种调用服务提供者的方式
 * 1. RestTemplate（通过服务名调用）
 * 2. OpenFeign（声明式 HTTP 客户端）
 * 3. DiscoveryClient（手动获取服务实例信息）
 */
@RestController
public class ConsumerController {

    private static final Logger log = LoggerFactory.getLogger(ConsumerController.class);

    /**
     * 注入带负载均衡功能的 RestTemplate
     * 这个 RestTemplate 由 RestTemplateConfig 类配置并注册为 Bean
     */
    @Autowired
    private RestTemplate restTemplate;

    /**
     * 注入 Feign 客户端接口
     * Spring 会自动创建这个接口的实现类（动态代理）
     * 如果不使用 Feign，删除此字段和相关方法
     */
    @Autowired
    private ProviderFeignClient providerFeignClient;

    /**
     * 注入 DiscoveryClient（可选）
     * 作用：Spring Cloud 提供的服务发现客户端
     * 可以用来获取服务实例列表、服务详情等信息
     * 一般用于需要手动控制服务调用逻辑的场景
     */
    @Autowired
    private DiscoveryClient discoveryClient;

    // ============================================================
    // 方式一：使用 RestTemplate 调用服务
    // ============================================================

    /**
     * 使用 RestTemplate 调用服务提供者
     *
     * 测试方式：
     * 浏览器访问：http://localhost:8082/test-rest?name=张三
     * 或 curl 'http://localhost:8082/test-rest?name=张三'
     *
     * @param name 请求参数
     * @return 服务提供者返回的消息
     */
    @GetMapping("/test-rest")
    public String testRestTemplate(@RequestParam(value = "name", defaultValue = "访客") String name) {
        log.info("使用 RestTemplate 调用服务，参数 name = {}", name);

        /**
         * RestTemplate 调用方式：
         *
         * 格式：http://{服务名}/{接口路径}?{参数}
         *
         * 关键点：
         * 1. 使用服务名（nacos-provider）而不是 IP 地址
         * 2. @LoadBalanced 注解的 RestTemplate 会自动解析服务名
         * 3. 负载均衡器会从 Nacos 获取所有健康实例，并选择一个进行调用
         * 4. 如果有多个实例，默认使用轮询算法（Round Robin）
         *
         * RestTemplate 常用方法：
         * - getForObject(url, 返回类型.class)：GET 请求，返回响应体对象
         * - getForEntity(url, 返回类型.class)：GET 请求，返回 ResponseEntity（包含状态码、响应头等）
         * - postForObject(url, 请求体, 返回类型.class)：POST 请求
         * - exchange(url, 请求方法, 请求实体, 返回类型)：通用方法，支持所有 HTTP 方法
         */
        String url = "http://nacos-provider/hello?name=" + name;
        String response = restTemplate.getForObject(url, String.class);

        log.info("RestTemplate 调用成功，响应: {}", response);
        return "【RestTemplate 方式】" + response;
    }

    // ============================================================
    // 方式二：使用 OpenFeign 调用服务（推荐）
    // ============================================================

    /**
     * 使用 OpenFeign 调用服务提供者
     *
     * 测试方式：
     * 浏览器访问：http://localhost:8082/test-feign?name=李四
     * 或 curl 'http://localhost:8082/test-feign?name=李四'
     *
     * @param name 请求参数
     * @return 服务提供者返回的消息
     */
    @GetMapping("/test-feign")
    public String testFeign(@RequestParam(value = "name", defaultValue = "访客") String name) {
        log.info("使用 Feign 调用服务，参数 name = {}", name);

        /**
         * Feign 调用方式：
         *
         * 优势：
         * 1. 代码简洁：像调用本地方法一样调用远程服务
         * 2. 无需拼接 URL：URL 路径、参数映射在接口中定义
         * 3. 类型安全：编译期检查方法签名和参数类型
         * 4. 易于维护：接口定义即文档
         * 5. 自动集成负载均衡、熔断降级等功能
         *
         * 适用场景：
         * - 服务间调用频繁，接口较多
         * - 需要统一管理服务调用逻辑
         * - 需要集成熔断降级、请求重试等高级功能
         */
        String response = providerFeignClient.hello(name);

        log.info("Feign 调用成功，响应: {}", response);
        return "【Feign 方式】" + response;
    }

    // ============================================================
    // 方式三：使用 DiscoveryClient 手动获取服务实例
    // ============================================================

    /**
     * 使用 DiscoveryClient 获取服务实例信息
     *
     * 测试方式：
     * 浏览器访问：http://localhost:8082/discovery-info
     *
     * 作用：
     * 演示如何手动从 Nacos 获取服务实例列表
     * 在需要自定义负载均衡策略、灰度路由等场景下很有用
     *
     * @return 服务实例信息
     */
    @GetMapping("/discovery-info")
    public String getDiscoveryInfo() {
        log.info("获取服务发现信息");

        // 获取所有已注册的服务名列表
        List<String> services = discoveryClient.getServices();
        log.info("所有已注册的服务: {}", services);

        StringBuilder info = new StringBuilder("=== Nacos 服务发现信息 ===\n\n");
        info.append("已注册的服务数量: ").append(services.size()).append("\n\n");

        // 遍历每个服务，获取其实例列表
        for (String serviceName : services) {
            List<ServiceInstance> instances = discoveryClient.getInstances(serviceName);
            info.append(String.format("服务名称: %s\n", serviceName));
            info.append(String.format("实例数量: %d\n", instances.size()));

            // 遍历每个实例，打印详细信息
            for (int i = 0; i < instances.size(); i++) {
                ServiceInstance instance = instances.get(i);
                info.append(String.format("  实例 %d:\n", i + 1));
                info.append(String.format("    - 服务ID: %s\n", instance.getServiceId()));
                info.append(String.format("    - Host: %s\n", instance.getHost()));
                info.append(String.format("    - Port: %d\n", instance.getPort()));
                info.append(String.format("    - URI: %s\n", instance.getUri()));
                info.append(String.format("    - Metadata: %s\n", instance.getMetadata()));
            }
            info.append("\n");
        }

        return info.toString();
    }

    /**
     * 手动调用服务示例（不推荐，仅供学习理解）
     *
     * 演示：如何使用 DiscoveryClient 手动获取服务实例并调用
     *
     * 注意：这种方式需要自己实现负载均衡逻辑，一般不推荐使用
     * 推荐使用 @LoadBalanced RestTemplate 或 Feign
     */
    @GetMapping("/test-manual")
    public String testManualCall(@RequestParam(value = "name", defaultValue = "访客") String name) {
        log.info("手动调用服务，参数 name = {}", name);

        // 1. 从 Nacos 获取服务实例列表
        List<ServiceInstance> instances = discoveryClient.getInstances("nacos-provider");

        if (instances == null || instances.isEmpty()) {
            return "错误：未找到服务 nacos-provider 的实例";
        }

        // 2. 手动选择一个实例（这里简单选择第一个，实际应该实现负载均衡算法）
        ServiceInstance instance = instances.get(0);
        log.info("选择的实例: {}:{}", instance.getHost(), instance.getPort());

        // 3. 拼接完整的 URL
        String url = String.format("http://%s:%d/hello?name=%s",
            instance.getHost(),
            instance.getPort(),
            name);

        // 4. 使用普通的 RestTemplate 调用（注意：这里不能用带 @LoadBalanced 的 RestTemplate）
        RestTemplate normalRestTemplate = new RestTemplate();
        String response = normalRestTemplate.getForObject(url, String.class);

        log.info("手动调用成功，响应: {}", response);
        return "【手动调用方式】" + response;
    }
}
```

---

## 4️⃣ 启动和测试步骤

### Step 1: 启动 Nacos Server

```bash
# 确保 Nacos Server 3.0.3 已下载并配置（参考前面的配置说明）

# Linux/Mac
cd nacos/bin
sh startup.sh -m standalone

# Windows
cd nacos\bin
startup.cmd
```

访问 Nacos 控制台：http://127.0.0.1:8848/nacos
- 用户名：nacos
- 密码：nacos

### Step 2: 启动服务提供者

```bash
# 方式一：IDE 启动
# 直接运行 ProviderApplication.java 的 main 方法

# 方式二：Maven 打包启动
cd nacos-provider
mvn clean package
java -jar target/nacos-provider-1.0.0.jar

# 方式三：启动多个实例（测试负载均衡）
java -jar target/nacos-provider-1.0.0.jar --server.port=8081
java -jar target/nacos-provider-1.0.0.jar --server.port=8082
java -jar target/nacos-provider-1.0.0.jar --server.port=8083
```

验证服务注册：
- 访问 Nacos 控制台 → 服务管理 → 服务列表
- 应该能看到 `nacos-provider` 服务，实例数为 1（或多个）

### Step 3: 测试服务提供者

```bash
# 直接调用提供者接口
curl 'http://localhost:8081/hello?name=张三'

# 预期响应：
# 你好，张三！这是来自服务提供者 [端口: 8081] 的响应，时间: 2025-01-15 10:30:45
```

### Step 4: 启动服务消费者

```bash
# 方式一：IDE 启动
# 直接运行 ConsumerApplication.java 的 main 方法

# 方式二：Maven 打包启动
cd nacos-consumer
mvn clean package
java -jar target/nacos-consumer-1.0.0.jar
```

### Step 5: 测试服务调用

```bash
# 1. 测试 RestTemplate 方式
curl 'http://localhost:8082/test-rest?name=李四'

# 预期响应：
# 【RestTemplate 方式】你好，李四！这是来自服务提供者 [端口: 8081] 的响应，时间: ...

# 2. 测试 Feign 方式
curl 'http://localhost:8082/test-feign?name=王五'

# 预期响应：
# 【Feign 方式】你好,王五！这是来自服务提供者 [端口: 8081] 的响应，时间: ...

# 3. 查看服务发现信息
curl 'http://localhost:8082/discovery-info'

# 预期响应：会显示所有已注册服务的详细信息

# 4. 测试负载均衡（前提：启动了多个 Provider 实例）
# 多次调用同一个接口，观察响应中的端口号是否轮流变化
for i in {1..6}; do
  curl 'http://localhost:8082/test-rest?name=Test'
  echo ""
done

# 如果启动了 8081, 8082, 8083 三个实例
# 预期结果：端口号会按照 8081 → 8082 → 8083 → 8081 → ... 的顺序轮询
```

---

## 5️⃣ 关键知识点总结

### 服务名的重要性

```yaml
# Provider 的 application.yml
spring:
  application:
    name: nacos-provider  # ← 这个名称非常重要！

# Consumer 中的调用方式：
# RestTemplate:  http://nacos-provider/hello
#                      ↑ 必须与上面的 name 一致
#
# Feign: @FeignClient(name = "nacos-provider")
#                             ↑ 必须与上面的 name 一致
```

### RestTemplate vs Feign 对比

| 特性 | RestTemplate | OpenFeign |
|------|--------------|-----------|
| **代码风格** | 手动拼接 URL | 声明式接口 |
| **类型安全** | ❌ 弱类型 | ✅ 强类型 |
| **易用性** | 较繁琐 | 简洁优雅 |
| **负载均衡** | ✅ 需要 @LoadBalanced | ✅ 自动集成 |
| **熔断降级** | 需要手动集成 Resilience4j | ✅ 原生支持 fallback |
| **适用场景** | 简单调用、外部 API | 内部服务调用（推荐）|

### 负载均衡原理

```
Consumer 发起请求: http://nacos-provider/hello
         ↓
LoadBalancerInterceptor 拦截
         ↓
从 Nacos 获取服务实例列表: [8081, 8082, 8083]
         ↓
应用负载均衡算法（轮询 Round Robin）
         ↓
选择一个实例: 8081
         ↓
替换 URL: http://192.168.1.10:8081/hello
         ↓
发起真实的 HTTP 请求
```

### 常见问题排查

#### 问题1：服务未注册到 Nacos

```
检查清单：
1. Nacos Server 是否启动成功？
   → 访问 http://127.0.0.1:8848/nacos

2. application.yml 配置是否正确？
   → server-addr: 127.0.0.1:8848
   → username/password 是否配置

3. 是否引入了 nacos-discovery 依赖？
   → spring-cloud-starter-alibaba-nacos-discovery

4. 查看应用启动日志，搜索关键字：
   → "nacos registry"
   → "register finished"
```

#### 问题2：消费者无法调用提供者

```
错误信息: java.net.UnknownHostException: nacos-provider

原因：RestTemplate 没有添加 @LoadBalanced 注解

解决方案：
@Bean
@LoadBalanced  // ← 必须添加此注解
public RestTemplate restTemplate() {
    return new RestTemplate();
}
```

#### 问题3：Feign 调用失败

```
错误信息: Load balancer does not contain an instance for the service nacos-provider

可能原因：
1. Provider 未启动或未注册到 Nacos
2. Consumer 的 spring.cloud.nacos.discovery.server-addr 配置错误
3. Provider 和 Consumer 在不同的 namespace 或 group

解决方案：
1. 检查 Nacos 控制台，确认 Provider 已注册
2. 检查 Consumer 的配置，确保与 Provider 在同一个 namespace/group
3. 重启 Consumer，确保获取到最新的服务列表
```

---

## 6️⃣ 扩展实验

### 实验1：测试服务下线

1. 启动 Provider（端口 8081）
2. 启动 Consumer，调用接口成功
3. 停止 Provider（Ctrl+C）
4. 等待 15 秒（Nacos 心跳超时时间）
5. 再次调用 Consumer 接口，观察是否报错

### 实验2：测试负载均衡

1. 启动 3 个 Provider 实例（8081, 8082, 8083）
2. 启动 Consumer
3. 多次调用 `/test-rest` 接口
4. 观察响应中的端口号是否轮流变化

### 实验3：测试灰度发布（权重）

1. 在 Nacos 控制台修改实例权重：
   - 8081 实例：权重 1
   - 8082 实例：权重 5
2. 多次调用接口，观察 8082 实例是否接收到更多请求

### 实验4：测试同集群优先

```yaml
# Provider 8081 的配置
spring:
  cloud:
    nacos:
      discovery:
        cluster-name: BJ  # 北京集群

# Provider 8082 的配置
spring:
  cloud:
    nacos:
      discovery:
        cluster-name: SH  # 上海集群

# Consumer 的配置
spring:
  cloud:
    nacos:
      discovery:
        cluster-name: BJ  # 北京集群
    loadbalancer:
      nacos:
        enabled: true  # 启用 Nacos 负载均衡（支持同集群优先）
```

启动后观察：Consumer 会优先调用同集群（BJ）的 Provider

---

## 7️⃣ 下一步学习

1. **配置管理**：学习 Nacos Config，实现动态配置刷新
2. **服务保护**：集成 Sentinel，实现熔断降级、限流
3. **分布式事务**：集成 Seata，解决分布式事务问题
4. **链路追踪**：集成 Sleuth + Zipkin，实现调用链追踪
5. **API 网关**：集成 Spring Cloud Gateway，统一入口管理

---

## 📚 参考资料

- [Nacos 官方文档](https://nacos.io/zh-cn/docs/quick-start.html)
- [Spring Cloud Alibaba 官方文档](https://sca.aliyun.com/zh-cn/)
- [Spring Cloud LoadBalancer 文档](https://docs.spring.io/spring-cloud-commons/docs/current/reference/html/#spring-cloud-loadbalancer)
- [OpenFeign 官方文档](https://docs.spring.io/spring-cloud-openfeign/docs/current/reference/html/)

---

**祝学习愉快！🎉**
