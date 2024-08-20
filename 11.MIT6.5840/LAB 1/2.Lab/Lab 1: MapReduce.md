# Lab 1: MapReduce

目标：实现一个MapReduce系统。其中包含：

1. `worker`进程：调用Map和Reduce程序并处理文件的读写
2. `coordinator`进程：负责将任务分发给worker并处理失败的worker。(注:本Lab使用`coordinator`而不是论文的`master`进行管理。)

## Getting started

`src/main/mrsequential.go`中提供了单线程版本的MapReduce。每次只启动一个Map和一个Reduce。还提供了几个MapReduce应用程序: 

- 单词计数器:`mrapps/wc.go`
- 文本索引器`mrapps/indexer.go`.

可以使用下列指令运行wordCount:

```
$ cd ~/6.824
$ cd src/main
$ go build -race -buildmode=plugin ../mrapps/wc.go
$ rm mr-out*
$ go run -race mrsequential.go wc.so pg*.txt
$ more mr-out-0
A 509
ABOUT 2
ACT 8
...
```

(注意:**-race**会启用Go竞态检测器。建议测试代码时启用它)

`mrsequential.go` 读取输入`pg-xxx.txt`，输出结果到 `mr-out-0`。

可以借鉴`mrsequential.go`中的代码，同时也看看`mrapps/wc.go `。了解应用程序如何MapReduce。

## 任务

实现一个分布式的`MapReduce`，其由两种程序组成:`coordinator`和`worker`。整个MapReduce将会只有一个`coordinator`和一个或多个并行执行的`worker`。

现实中，`workers`会在分布在不同的机器上运行，但本次Lab将在一台机器上运行。

- `worker：`
  
  - 通过RPC与`coordinator`通信。
  - 向`coordinator`请求任务，从一个或多个文件中读取输入，
  - 执行任务，将任务的输出写入一个或多个文件。
  
- `coordinator`：

  如果Worker未在设定时间内完成任务(本次实验为 10 秒)，`coordinator`应将该任务交给其他`worker`。

提供了一些初始代码. `coordinator` 和 `worker` 的代码分别在`main/mrcoordinator.go` 和 `main/mrworker.go`; 不要更改这些文件。把你的实现放在 `mr/coordinator.go`, `mr/worker.go`, 以及 `mr/rpc.go`.

下面展示如何运行WordCount。首先，确保wc是最新编译版本：

```
$ go build -race -buildmode=plugin ../mrapps/wc.go
```

在`main`目录下，运行`coordinator`

```
$ rm mr-out*
$ go run -race mrcoordinator.go pg-*.txt
```

将输入文件 `pg-*.txt` 的名字作为参数传递给`mrcoordinator.go` ；每个文件对应一个 `split`, 是一个Map任务的输入.

在一个或多个窗口运行一些worker:

```
$ go run -race mrworker.go wc.so
```

当`worker`和`coordinator`完成后，查看`mr-out-*`中的输出。结果应该和单Worker版本的结果相同。

```
$ cat mr-out-* | sort | more
A 509
ABOUT 2
ACT 8
...
```

测试脚本为`main/test-mr.sh`。输入`pg-xxx.txt`文件时，测试会检查`wc`和`indexer`能否产生正确结果。测试还会检查是否并行运行Map和Reduce任务，以及能否从运行任务时崩溃的`worker`中恢复。

现在运行测试，它会挂起，因为`coordinator`永远不会结束:

```
$ cd ~/6.824/src/main
$ bash test-mr.sh
*** Starting wc test.
```

可将`mr/coordinator.go`的`Done函数`中的`ret:= false`改为true。这样`coordinator`会立即退出：

```
$ bash test-mr.sh
*** Starting wc test.
sort: No such file or directory
cmp: EOF on mr-wc-all
--- wc output is not the same as mr-correct-wc.txt
--- wc test: FAIL
$
```

测试脚本没有在`mr-out-X`中看到结果，每个文件对应一个reduce任务。当前`mr/coordinator.go`和`mr/worker.go`没有实现，因此不会生成这些文件，测试失败。

代码正确实现时，测试脚本的输出应该像这样:

```
$ bash test-mr.sh
*** Starting wc test.
--- wc test: PASS
*** Starting indexer test.
--- indexer test: PASS
*** Starting map parallelism test.
--- map parallelism test: PASS
*** Starting reduce parallelism test.
--- reduce parallelism test: PASS
*** Starting crash test.
--- crash test: PASS
*** PASSED ALL TESTS
$
```

您还将从Go RPC包中看到一些错误

```
2019/12/16 13:27:09 rpc.Register: method `Done` has 1 input parameters; needs exactly three
```

忽略这些信息;将`coordinator`注册为[RPC服务器](https://golang.org/src/net/rpc/server.go)，检查它的所有方法是否适合RPCS(有3个输入);我们知道`Done`不是通过RPC调用的。

## 一些规则

- Map Worker：
  - 将输入划分到各个桶中，以供`nReduce`个reduce任务使用
  -  `nReduce`指reduce worker个数(由`main/mrcoordinator.go` 传递给 `MakeCoordinator()`)，每个`map worker`都应创建`nReduce`个中间文件，供reduce使用。

- worker应该把第X个reduce的输出放在`mr-out-X`中。
- `mr-out-X`文件的每一行对应Reduce函数的一次输出。这行使用Go ` %v %v` 格式打印生成，分别是Key和Value。在`main/mrsequential.go`中查看注释为`this is the correct format`的行。如果您的输出与这种格式偏离太多，测试脚本将会失败。
- 你可以修改 `mr/worker.go`, `mr/coordinator.go`, 以及 `mr/rpc.go`. 你可以临时修改其他文件进行测试，但要确保你的代码能够与原始版本兼容;我们将使用原始版本进行测试。
- worker应该把Map的中间输出放在当前目录下的文件中，之后将这些文件作为输入传递给Reduce任务。
- `main/mrcoordinator.go` 希望 `mr/coordinator.go` 实现 `Done()` 方法，当MapReduce任务完成时，返回true; 之后,`mrcoordinator.go` 会退出.
- 作业完成时，worker应该退出. 一种简单的实现是使用`call()`的返回值，如果worker未能联系到`coordinator`，可以默认`coordinator`已退出。也可由`coordinator`向worker发送`please exit`指令。

## 提示

- [指南页面](https://pdos.csail.mit.edu/6.824/labs/guidance.html)提供有一些关于开发和调试的提示

- 一种开始的方法是修改 `mr/worker.go`的 `Worker()` 发送一个RPC到`coordinator` 请求任务.，`coordinator`返回尚未处理的文件名。之后修改worker以读取该文件并调用应用程序Map函数, 如`mrsequential.go`.

- 应用程序的Map和Reduce函数在运行时使用Go插件包，从以`.so`结尾的文件中加载。

- 如果你修改了' mr/ '目录中的任何内容，你可能需要重新构建你使用的MapReduce插件，比如`go build -race -buildmode=plugin ../mrapps/wc.go`

- 这个LAB依赖于工作人员共享一个文件系统。当所有工作程序运行在同一台机器上时，这很简单，但如果工作程序运行在不同的机器上，则需要像GFS这样的全局文件系统。

- 中间文件的命名约定是`mr-X-Y`，其中X是Map任务号，Y是reduce任务号。

- worker的map任务代码需要在文件中存储中间键值对，并且保证reduce任务可以正确读取。一种解决方案是使用Go's `encoding/json` 包。将JSON格式的键值对写入打开的文件:

  ```go
  enc := json.NewEncoder(file)
  for _, kv := ... {
  	err := enc.Encode(&kv)
  ```

   之后这样将文件读出:

    ```go
  dec := json.NewDecoder(file)
  for {
  	var kv KeyValue
  	if err := dec.Decode(&kv); err != nil {
  		break
  	}
  	kva = append(kva, kv)
  }
    ```

- worker的map可以使用`ihash(key)`函数(在`worker.go`中)来为给定的Key选择reduce任务。

- 你可以参考 `mrsequential.go` 的代码，用于读取Map输入文件， 在Map和Reduce之间排序中间K\V对，以及将Reduce输出存储在文件中。

- 作为RPC服务器，`coordinator`将是并发的;不要忘记给共享数据加锁。

- 使用Go的竞态检测器, 使用 `go build -race` 和 `go run -race`. `test-mr.sh` 默认使用竞争检测器运行测试。

- Workers 有时需要等待, 例如，直到最后一个map完成，reduce才能启动. 一种解决方案是worker定期向coordinator请求工作, 在每次请求之间使用`time.Sleep()`休眠. 另一种方案是coordinator中相关的RPC处理程序有一个等待循环, 可以使用`time.Sleep()`或`sync.Cond`.Go在自己的线程中为每个RPC运行处理程序, 因此一个正在等待的处理程序不会阻止coordinator处理其他RPC。

- coordinator无法准确地区分崩溃的、还活着但由于某种原因停止工作的、以及正在执行但速度太慢而无法发挥作用的worker。你能做的最好的事情是让协调器等待一段时间，然后放弃并重新将任务发送给另一个worker。对于这个LAB，让协调者等待十秒钟了；十秒内未完成，`coordinator`就可假定worker已经死亡(当然，它也可能没有死亡)。

- 如果你选择实现备份任务`(Backup Tasks)`(论文第3.6节)，请注意，我们测试了你的代码在worker执行任务没有崩溃时不会调度多余的任务。备份任务应该只安排在一段相对较长的时间之后(例如10秒)。

- 要测试崩溃恢复，你可以使用`mrapps/crash.go`插件。它随机地存在于Map和Reduce函数中。

- 为了确保没有人在崩溃的情况下观察到部分写入的文件，那篇关于MapReduce的论文提到了使用临时文件的技巧，在文件写入完成后原子性地重命名它。你可以使用`ioutil.TempFile`创建一个临时文件，并使用`os.Rename`对其进行原子重命名。

- `test-mr.sh`运行子目录`mr-tmp`中的所有进程，所以如果出现问题后你想查看中间文件或输出文件，请查看那里。你可以在测试失败后临时将`test-mr.sh`修改为`exit`，这样脚本就不会继续测试(并覆盖输出文件)。

- `test-mr-many.sh`提供了一个带有超时功能的运行`test-mr.sh`的基本脚本(这也是我们之后测试代码的方式)。它接受运行测试的次数作为参数。你不应该并行运行多个`test-mr.sh`实例，因为协调器将重用相同的socket，从而导致冲突。

- Go RPC只发送以大写字母开头的结构体字段。子结构的字段名也必须是大写。

- 当将一个reply结构体的指针传递给RPC系统时，`*reply`指向的对象应该是零分配的。RPC调用的代码应该是这样的

  ```go
    reply := SomeType{}
    call(..., &reply)
  ```

  在调用前不设置任何回复字段，如果不遵循这个规定，RPC 系统可能会返回不正确的值。


## 实现

## Map

