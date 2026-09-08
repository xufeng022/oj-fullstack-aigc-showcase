# Docker Code Sandbox 安全设计

## 1. 信任边界

在线判题最大的风险之一是：

```text
用户提交的代码
=
不可信代码
```

因此用户代码不能直接运行在：

```text
业务 JVM
数据库服务器
Judge Service 主机环境
```

项目将执行过程放入独立 Code Sandbox。

```text
Judge Service
     |
     v
Code Sandbox Service
     |
     v
Disposable Docker Container
```

## 2. 一次提交一个执行容器

编译和运行发生在一次性容器中。

```text
create
  |
compile
  |
execute
  |
collect result
  |
destroy
```

执行结束后容器会被清理。

这样可以减少不同提交之间共享运行状态。

## 3. 网络隔离

用户程序默认不需要访问互联网。

子容器禁用网络，可以减少：

```text
主动扫描
外部下载
数据外传
攻击内网服务
```

等风险。

## 4. 文件系统限制

容器采用只读根文件系统，并仅提供必要的临时可写目录。

目标是避免用户程序任意修改系统目录或长期保存文件。

## 5. 非 Root

用户代码以非 root 身份运行。

同时减少 Linux capabilities，并启用：

```text
no-new-privileges
```

降低容器内提权风险。

## 6. 资源控制

执行环境限制：

```text
CPU
Memory
PID
JVM Heap
JVM Stack
Execution Time
Output Size
```

这些限制分别用于抵御：

```text
死循环
内存炸弹
Fork Bomb
超深递归
无限输出
```

## 7. 时间预算

不仅限制单个测试用例，还限制单次提交的总体执行预算。

原因是攻击者可以构造：

```text
每个 case 都没有超时
但大量 case 累积执行很久
```

因此：

```text
single case timeout
+
total request budget
```

需要同时存在。

## 8. 输出限制

stdout 和 stderr 都存在大小上限。

否则用户可以不断输出：

```text
while (true) {
    print(...)
}
```

最终消耗宿主服务内存。

## 9. 稳定错误协议

沙箱会区分：

```text
Compile Error
Runtime Error
Time Limit Exceeded
Memory Limit Exceeded
Output Limit Exceeded
System Error
```

执行失败不会全部退化成 HTTP 500。

## 10. Docker Socket 风险

当前设计仍存在一个非常重要的边界：

```text
Code Sandbox Service
        |
        v
/var/run/docker.sock
```

Docker Socket 本身是高权限接口。

因此即使子容器进行了较强限制，也不能把整个方案描述成“绝对安全”。

更高安全等级的生产环境应进一步使用：

```text
独立执行节点
独立 VM
rootless runtime
受限远程执行平台
```

并避免代码执行节点与数据库等关键业务服务共用 Docker daemon。
