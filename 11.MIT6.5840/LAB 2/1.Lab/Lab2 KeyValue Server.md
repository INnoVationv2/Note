# 6.5840 Lab 2: Key/Value Server

## 1.Introduction

本次Lab将构建一个单机的键值服务器，该服务器保证即使存在网络故障，每个操作也都`只执行一次`，并且这些操作线性化执行。后续Lab中，将复制这样的服务器来处理服务器崩溃的情况。

键值服务器支持三种RPC(远程过程调用)操作:`Put(key, value)`、`Append(key, arg)`和`Get(key)`。服务器在内存中维护一个Map。键和值都是字符串。

- `Put(key, value)` 在Map中添加或替换特定键的值，并返回新的值
- `Append(key, arg)` 将`arg`追加到Key对应的Value后面，并返回旧值
- `Get(key)` 获取值。如果key不存在，返回空字符串

对不存在的Key进行`Append`操作应当视原值为一个空字符串。客户端通过`Clerk`与服务器交互，`Clerk`提供`Put/Append/Get`方法，并管理与服务器的RPC交互。

你的服务器必须确保对`Clerk`的`Get/Put/Append`方法的调用是线性化的。如果客户端请求没有并发，每个客户端的`Get/Put/Append`调用应观察到之前一系列调用对状态的修改。对于并发调用，返回值和最终状态必须与这些操作按某种顺序一次执行时的结果相同。如果调用在时间上重叠，例如客户端 X 调用了`Clerk.Put()`，同时客户端 Y 调用了`Clerk.Append()`，然后客户端 X 的调用返回。一个调用必须观察到在该调用开始前已经完成的所有调用的效果。

## Getting Started

`src/kvsrv`中提供了基础代码和测试。你可以修改`kvsrv/client.go`、`kvsrv/server.go`和`kvsrv/common.go`。

可通过以下命令启动Server，别忘了经常执行`git pull`更新代码

```shell
$ cd ~/6.5840
$ git pull
...
$ cd src/kvsrv
$ go test
...
$
```

### 没有网络故障的KV服务器([easy](https://pdos.csail.mit.edu/6.824/labs/guidance.html))

> 第一个任务是完成一个在没有网络故障、不会丢失消息情景下的KV服务器。
>
> 在`client.go`中的Clerk Put/Append/Get方法中添加RPC发送代码，并在server.go中实现``Put`、`Append()`和`Get()`RPC处理程序。
>
> 当通过前两个测试`one client`和`many clients`时，此任务就算完成了。

> 提示：使用`go test -race`检查你的代码是否无竞态问题。

### 会丢失消息的Key/value服务器([easy](https://pdos.csail.mit.edu/6.824/labs/guidance.html))

修改代码，以便在消息丢失的情况下正常执行。如果消息丢失，则客户端的`ck.server.Call()`将返回`false`(更准确地说，`Call()`等待一段时间，如果在这段时间内没有回复，则返回false)。您将面临的一个问题是，Clerk可能需要多次发送RPC才能成功。但是，每次调用`Clerk.Put()`或`Clerk.Append()`都应该只执行一次，因此您必须确保重新发送不会导致服务器执行两次请求。

> 如果未收到回复，请向`Clerk`添加代码以重试，如果操作需要，请向`server.go添加代码以过滤重复项。这些说明包括有关`[重复检测](https://pdos.csail.mit.edu/6.824/notes/l-raft-QA.txt)的指导。

> 提示
>
> - 您需要唯一地标识客户端操作以确保键/值服务器只执行一次。
> - 您必须仔细考虑服务器必须维持什么状态来处理重复的`Get()`、`Put()`和`Append()`请求（如果有）。
> - 您的重复检测方案应快速释放服务器内存，例如通过让每个RPC暗示客户端已看到其上一个RPC的回复。可以假设客户端每次只会调用Clerk一次。

你的代码现在应该通过所有测试，如下所示：

```shell
$ go test
Test: one client ...
  ... Passed -- t  3.8 nrpc 31135 ops 31135
Test: many clients ...
  ... Passed -- t  4.7 nrpc 102853 ops 102853
Test: unreliable net, many clients ...
  ... Passed -- t  4.1 nrpc   580 ops  496
Test: concurrent append to same key, unreliable ...
  ... Passed -- t  0.6 nrpc    61 ops   52
Test: memory use get ...
  ... Passed -- t  0.4 nrpc     4 ops    0
Test: memory use put ...
  ... Passed -- t  0.2 nrpc     2 ops    0
Test: memory use append ...
  ... Passed -- t  0.4 nrpc     2 ops    0
Test: memory use many puts ...
  ... Passed -- t 11.5 nrpc 100000 ops    0
Test: memory use many gets ...
  ... Passed -- t 12.2 nrpc 100001 ops    0
PASS
ok      6.5840/kvsrv    39.000s
```

`每个Passed` 后的数字是实际时间（以秒为单位）、发送的 RPC 数量（包括客户端 RPC）以及执行的键/值操作数量（`Clerk` Get/Put/Append 调用）。

## 思路

### Lab 2.1

很简单，需要注意，使用Clerk/server进行RPC调用，不用复制Lab1中的远程调用实现

```golang
type Clerk struct {
	server *labrpc.ClientEnd
}
```

### Lab 2.2

1. 为消息添加ClientID和ID以便Server判断消息是否重复
2. 对于Put和Append方法，每次更新前，保存旧值，以便之后响应重复请求
3. 添加新的RPC方法：Report，当Client接收到结果后，使用Report报告Server该记录已经处理完成，Server此时就可清除用于响应重复请求保存的旧值。

### 结果

代码：

https://github.com/INnoVationv2/6.5840/commit/66b34ae954ee875c5a7bad035ae8b9c33cbb55ef

测试：

<img src="http://pic-save-fury.oss-cn-shanghai.aliyuncs.com/uPic/image-20240827171655249.png" alt="image-20240827171655249" style="zoom:50%;" />

