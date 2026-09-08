# 部署架构设计

## 1. 服务结构

完整 OJ 环境由多个服务组成：

```text
Frontend / Nginx
Gateway
User Service
Question Service
Judge Service
AI Service
Code Sandbox
MySQL
Redis
RabbitMQ
Nacos
```

核心业务服务采用 Spring Cloud 微服务架构。

## 2. 请求入口

外部用户首先访问：

```text
Browser
   |
   v
Nginx
   |
   v
Gateway
```

Gateway 是统一 API 安全边界。

业务微服务本身不建议直接暴露公网。

## 3. 服务发现

业务服务通过 Nacos 完成服务发现。

Feign Client 负责内部 HTTP 调用。

```text
Service A
   |
   | OpenFeign
   v
Nacos Discovery
   |
   v
Service B
```

关键 `/inner/**` 调用还叠加 HMAC 内部认证。

## 4. RabbitMQ

判题任务使用 RabbitMQ 解耦：

```text
Question Service
      |
      v
RabbitMQ
      |
      v
Judge Service
```

RabbitMQ 不参与 HTTP 请求同步响应，因此判题可以异步进行。

## 5. Redis

Redis 用于：

```text
Session
Rate Limit
Internal nonce replay protection
AI related cache
```

不同用途应使用独立 key namespace。

## 6. MySQL

MySQL 保存核心业务数据：

```text
User
Question
QuestionSubmit
Comment
AI record
Outbox
```

业务服务使用独立应用账号连接数据库，而不是直接使用 root。

## 7. Docker Compose

开发和面试演示环境采用 Docker Compose 编排。

这样可以一次性启动：

```text
database
cache
MQ
registry
backend
sandbox
frontend
```

并通过 healthcheck 控制服务依赖。

## 8. 网络边界

推荐只公开：

```text
HTTPS reverse proxy / frontend
Gateway
```

而：

```text
MySQL
Redis
RabbitMQ
Nacos
Sandbox
internal services
```

都应限制在受控网络内。

## 9. Independent AI Assistant

独立 AI Assistant 不属于 OJ 主 Compose 内部强耦合服务。

两个系统通过共享 integration network 联动：

```text
OJ Compose
    |
    | shared Docker network
    v
AI Code Assistant
```

独立助手可以单独启动、停止和升级。

## 10. 技术栈边界

OJ 主项目保持原有 Java 8 / Spring Boot 2.x 技术栈，以控制大版本迁移风险。

独立 AI Assistant 使用 Java 21 和较新的 Spring AI 技术栈。

这样形成：

```text
Stable legacy-compatible OJ core
            +
Modern AI subsystem
```

避免为了接入 AI 强制升级整个 OJ。

## 11. 生产部署建议

Docker Compose 适合作为：

```text
开发环境
演示环境
中小规模部署
```

更高要求环境应进一步考虑：

```text
TLS termination
secret management
observability
database HA
RabbitMQ HA
Nacos authentication
isolated code execution nodes
centralized logging
```

尤其 Code Sandbox 应优先部署到独立执行节点。

## 12. 面试说明

这个部署设计强调：

```text
服务边界
网络边界
数据边界
执行边界
AI 边界
```

而不是简单地把所有容器放在同一个 Docker Compose 中。
