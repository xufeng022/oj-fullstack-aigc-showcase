# 微服务内部 HMAC 认证设计

## 1. 问题背景

微服务中的 `/inner/**` 接口只应该由其他内部服务调用。

仅依赖：

```text
Gateway 不暴露路由
```

是不够的。

如果攻击者能够直接访问某个 Service 端口，就可能绕过 Gateway。

因此内部接口自身也需要认证。

## 2. HMAC-SHA256

项目使用共享密钥生成 HMAC-SHA256 签名。

签名绑定：

```text
caller
HTTP method
canonical path
timestamp
nonce
```

逻辑上可以理解为：

```text
payload =
caller
+ method
+ path
+ timestamp
+ nonce

signature =
HMAC-SHA256(secret, payload)
```

共享 secret 不通过 HTTP 请求发送。

## 3. Caller 白名单

每个服务只允许指定服务身份访问其内部接口。

例如：

```text
Question Service
    |
    v
User Service /inner/**
```

服务端会验证 caller 是否位于允许列表中。

## 4. Canonical Path

客户端和服务端必须对同一路径得到完全相同的签名输入。

因此路径会先规范化。

不同形式：

```text
/get/id

/inner/get/id

/api/user/inner/get/id
```

可以映射到统一的 canonical internal path：

```text
/inner/get/id
```

这样避免不同 servlet context-path 导致客户端和服务端签名不一致。

## 5. Timestamp

签名带有毫秒时间戳。

服务器只接受一定时间窗口内的请求。

因此攻击者即使获得过去的合法请求，也不能无限期重放。

## 6. Nonce

仅靠 timestamp 仍然无法阻止时间窗口内重放。

因此每个请求额外生成随机 nonce：

```text
timestamp
+
nonce
```

服务器把已使用 nonce 保存到 Redis。

重复 nonce 会被拒绝。

Redis 使 nonce 防重放能够跨多个服务实例共享状态。

## 7. Constant-Time Comparison

签名比较使用常量时间比较方式，避免普通字符串比较带来的潜在时序侧信道问题。

## 8. Gateway 边界

Gateway 同时负责：

```text
阻止公网 /inner/**
移除客户端伪造的内部认证 Header
生成可信 Request ID
```

因此内部认证形成两层防护：

```text
Public Client
    |
    v
Gateway boundary
    |
    v
Service-level HMAC verification
```

## 9. 当前边界

当前方案不是 mTLS。

同时签名主要绑定：

```text
caller
method
path
timestamp
nonce
```

并没有把整个请求 body / query digest 都纳入 MAC。

因此跨主机或更高安全等级部署仍应使用：

```text
TLS / mTLS
private network ACL
service mesh
```

HMAC 不能替代传输层加密。
