# 判题可靠性设计

## 1. 问题背景

在线判题并不是简单的“收到代码 -> 执行 -> 返回结果”。

一次提交通常会经过：

```text
HTTP Request
    |
    v
Question Service
    |
    v
Database
    |
    v
RabbitMQ
    |
    v
Judge Service
    |
    v
Code Sandbox
```

这里最大的工程问题之一是：

```text
数据库写入成功
+
MQ 发送失败
=
提交记录永久停留在等待状态
```

如果直接在业务事务提交之后发送 MQ，就会产生典型的双写一致性问题。

## 2. Transactional Outbox

项目采用 Transactional Outbox。

提交记录和待发送消息在同一个数据库事务内写入：

```text
Database Transaction
  |
  |-- question_submit
  |
  '-- judge_message_outbox
```

事务成功意味着：

```text
提交存在
=
对应的待发送事件一定存在
```

随后由 Outbox Publisher 异步把消息投递到 RabbitMQ。

```text
judge_message_outbox
        |
        v
 Outbox Publisher
        |
        v
    RabbitMQ
```

这样避免了：

```text
DB success
MQ failure
```

造成的永久消息丢失。

## 3. Outbox Claim

多个 Question Service 实例可能同时扫描待发送记录。

因此不能简单：

```text
SELECT pending
-> send
```

否则同一个事件可能被多个实例重复发送。

项目采用原子 claim 思路：

```text
pending
   |
   | atomic claim
   v
publishing
   |
   | MQ success
   v
sent
```

只有成功取得 claim 的实例负责当前事件。

## 4. Self-Healing

服务可能在：

```text
pending -> publishing
```

之后崩溃。

因此 publishing 不能永久代表“消息正在发送”。

系统会检测长时间未完成的 publishing 状态并恢复，使事件重新进入可发送状态。

这使 Outbox 具备一定的自愈能力。

## 5. Judge Lease

MQ 本身通常提供至少一次投递语义，因此重复消费不能被当作异常情况。

Judge 在处理提交前先申请一个带过期时间的 lease：

```text
QuestionSubmit
    |
    | acquire
    v
judgeToken
judgeExpireTime
```

只有持有当前有效 lease 的 Judge 实例才能写入最终结果。

这样可以处理：

```text
MQ duplicate delivery
Judge process crash
multiple Judge instances
expired task takeover
```

## 6. 最终结果写入

最终更新需要满足：

```text
submission id
+
current judge token
```

只有 lease owner 可以完成最终写入。

这样可以避免：

```text
Judge A 超时后恢复
Judge B 已经重新判题
Judge A 又回来覆盖 Judge B 的新结果
```

## 7. RabbitMQ Retry / Dead Letter

判题消息采用：

```text
main
 |
 | failure
 v
retry
 |
 | TTL
 v
main
```

超过最大重试次数后：

```text
dead-letter
```

并将提交标记为可识别的系统错误状态，而不是无限重试。

## 8. Accepted 计数

Accepted 数量不能在 MQ 每消费一次时直接：

```text
acceptedNum++
```

因为重复消息会导致重复计数。

项目只在提交真正从非 Accepted 状态原子迁移到 Accepted 时更新计数。

## 9. 设计结果

最终判题链路具备：

```text
Transactional Outbox
        +
Atomic Claim
        +
Retry / Dead Letter
        +
Judge Lease
        +
Idempotent Final Write
        +
Recovery
```

它解决的是分布式系统中非常典型的：

```text
消息可靠性
并发竞争
重复消费
故障恢复
最终一致性
```

问题。

## 10. 面试说明

如果面试官问“为什么不用分布式事务”，可以回答：

项目中的核心需求并不要求数据库和 RabbitMQ 在一个强一致两阶段事务中完成，而是要求业务提交最终一定能够进入判题流程。因此使用 Transactional Outbox，把本地数据库事务作为可靠事实源，再异步投递 MQ，可以降低复杂度，同时获得更容易恢复和观测的最终一致性方案。
