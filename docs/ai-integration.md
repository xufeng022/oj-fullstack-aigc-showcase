# AI Integration 架构

## 1. 总体设计

项目中 AI 分成两个层次：

```text
OJ Built-in AI Service
        +
Independent AI Code Assistant
```

两者职责不同，并保持独立部署边界。

## 2. OJ 内置 AI Service

OJ 自身包含独立 AI Service。

当前模型 Provider 使用：

```text
DeepSeek
```

但业务代码通过 Provider abstraction 调用模型，而不是把所有逻辑直接绑定到具体供应商。

AI Service 提供：

```text
AI Chat
Question Analysis
Code Analysis
Programming Assessment
AI Comment Assistant
AI Judge Second Review
```

## 3. AI Judge 的边界

传统 Judge Verdict 是确定性的：

```text
test case
+
program output
+
judge strategy
=
deterministic verdict
```

AI 不应该覆盖：

```text
AC
WA
CE
RE
TLE
MLE
```

等核心结果。

因此 AI Judge 被设计为 Second Review：

```text
Deterministic Judge
        |
        +---- final verdict
        |
        '-- AI review / explanation
```

AI 失败不会破坏核心判题。

## 4. Independent AI Code Assistant

项目还能通过 OpenFeign 调用独立部署的 AI Code Assistant。

```text
Browser
   |
   v
OJ Gateway
   |
   v
OJ AI Service
   |
   | versioned integration API
   v
Independent AI Code Assistant
```

独立助手采用：

```text
Java 21
Spring Boot
Spring AI
Agent Workflow
RAG
PGVector
Redis Conversation Memory
DashScope / Qwen-Plus
```

## 5. Versioned Integration API

OJ 与独立助手之间不是直接依赖内部实现，而是通过版本化接口联动。

例如逻辑边界：

```text
/api/integration/oj/v1/*
```

这样未来独立助手内部更换：

```text
LLM
RAG implementation
Agent workflow
storage
```

时，不要求 OJ 同步修改内部实现。

## 6. Failure Degradation

独立 AI Assistant 属于外部能力。

调用链设计为：

```text
try Independent AI Assistant
          |
          | success
          v
     return result

          |
          | failure
          v

fallback to OJ AI Provider
```

因此外部 AI 服务不可用不会让 OJ 的核心业务失效。

## 7. Redis Conversation Memory

独立 AI Assistant 可以维护会话记忆。

OJ 请求使用独立命名空间，避免普通聊天会话与 OJ 集成会话相互污染。

## 8. RAG / PGVector

编程学习类知识通过 RAG 增强模型上下文。

```text
User Question
      |
      v
Embedding
      |
      v
PGVector Search
      |
      v
Relevant Context
      |
      v
LLM
```

这样可以把知识检索与基础模型能力分开。

## 9. 为什么 AI 不进入核心 Judge

外部 LLM 调用具有：

```text
延迟高
结果非确定
供应商可能不可用
存在限流
存在网络依赖
```

如果把它放进核心判题事务：

```text
Submit
  ->
LLM
  ->
Judge
```

会直接降低 OJ 可用性。

因此项目坚持：

```text
Deterministic Judge
        |
        +-- always available core path
        |
        '-- AI enhancement
```
