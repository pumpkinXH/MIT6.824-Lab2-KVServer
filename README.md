# MIT 6.824 分布式系统

## Lab2 Key/Value Server Golang实现

### 实验网站地址
[https://pdos.csail.mit.edu/6.824/labs/lab-mr.html](https://pdos.csail.mit.edu/6.824/labs/lab-kvsrv.html)

#### **1、实验要求**

在本次 Lab 中，你将在单机上构建一个 key/value 服务器，以确保即使网络出现故障，每个操作也只能执行一次，并且操作是可线性化的。

客户端可以向 key/value 服务器发送三个不同的 RPC： `Put(key, value)` 、 `Append(key, arg)` 和` Get(key)` 。服务器在内存中维护 key/value 对的`map`。键和值是字符串。 `Put(key, value)` 设置或替换map中给定键的值， `Append(key, arg) `将 arg 附加到键的值并返回旧值，` Get(key) `获取键的当前值。不存在的键的 Get请求应返回空字符串；对于不存在的键的 Append 请求应该表现为现有值是零长度字符串。每个客户端都通过`Clerk`的 `Put/Append/Get `方法与服务器进行通信。 `Clerk` 管理与服务器的 RPC 交互。

你的服务器必须保证应用程序对`Clerk Get/Put/Append `方法的调用是线性一致的。 如果客户端请求不是**并发**的，每个客户端 `Get/Put/Append `调用时能够看到之前调用序列导致的状态变更。 对于并发的请求来说，返回的结果和最终状态都必须和这些操作顺序执行的结果一致。如果一些请求在时间上重叠，则它们是并发的：例如，如果客户端 X 调用 `Clerk.Put()` ，并且客户端 Y 调用 `Clerk.Append() `，然后客户端 X 的调用 返回。 一个请求必须能够看到已完成的所有调用导致的状态变更。

一个应用实现线性一致性就像一台单机服务器一次处理一个请求的行为一样简单。 例如，如果一个客户端发起一个更新请求并从服务器获取了响应，随后从其他客户端发起的读操作可以保证能看到改更新的结果。在单台服务器上提供线性一致性是相对比较容易的。

Lab 在 `src/kvsrv` 中提供了框架代码和单元测试。你需要更改` kvsrv/client.go`、`kvsrv/server.go `和 `kvsrv/common.go` 文件。

#### 2、无网络故障的key/value 服务器

您的第一个任务是实现一个在没有丢失消息的情况下有效的解决方案。需要在 `client.go` 中，在 `Clerk`的 `Put/Append/Get` 方法中添加 RPC 的发送代码；并且实现 `server.go` 中 Put、Append、Get 三个 RPC handler。

当你通过了前两个测试 case：one client、many clients 时表示完成该任务。

使用 go test -race 检查您的代码是否没有争用。

#### 3、可能丢弃消息的Key/value 服务器

现在，您应该修改您的解决方案，以便在遇到丢失的消息（例如 RPC 请求和 RPC 回复）时继续工作。如果消息丢失，则客户端的` ck.server.Call() `将返回 `false` （更准确地说， `Call()` 等待响应直至超时，如果在此时间内没有响应就返回`false`）。您将面临的一个问题是 `Clerk `可能需要多次发送 RPC，直到成功为止。但是，**每次调用 `Clerk.Put()` 或` Clerk.Append() `应该只会导致一次执行，因此您必须确保重新发送不会导致服务器执行请求两次。**

你的任务是在 `Clerk` 中添加重试逻辑，并且在 `server.go` 中来过滤重复请求。

1. 您需要唯一地标识client操作，以确保Key/value服务器仅执行每个操作一次。
2. 您必须仔细考虑server必须维持什么状态来处理重复的` Get()` 、 `Put() `和` Append() `请求（如果有的话）。
3. 您的重复检测方案应该**快速释放服务器内存**，例如让每个 RPC 暗示client已看到其前一个 RPC 的回复。可以假设client一次只向Clerk发起一次调用。

**方案**

为`Put`和`Append`请求添加标识ID（`Get`请求只需不断重试，不会有影响）

使用`Map`用于在Key/value服务器中跟踪处理过的请求ID，以防止重复处理请求。

为解决请求丢失的现象，给`Put`和`Append`请求设置两个状态，首先是`修改`状态，待接收到修改完成的消息后改为`汇报`状态，向server汇报，server接收到完成请求后，再把`Map`中的请求记录删除，代表完成

![image-20241218170248257](https://github.com/user-attachments/assets/be20e536-1685-4012-b552-cdb1b7284259)

#### 4、结果
`cd kvsrv`

`go test`

![1734591921007](https://github.com/user-attachments/assets/5cc28f6f-534c-4e0c-b7d1-ec378efe6112)

