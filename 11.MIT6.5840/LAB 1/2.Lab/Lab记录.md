# 一、非并行版本分析

## 1.非并行版本MapReduce流程

1. 通过第一个参数，传入Map和Reduce 函数
2. 之后的参数为待处理文件名
3. 读取文件
4. 调用Map函数，对文件内容进行处理，生成KV对
5. 对KV对进行sort
6. 按照Key进行分组，然后对每组数据调用Reduce
7. 将结果写入文件

# 二、Lab思路

概述：Worker向Coordinator申请任务

## 1. Coordinator

代码位置： `mr/coordinator.go`

结构体介绍

```go
type Coordinator struct {
	nReduce int32
	stage   int32

	workerCnt int32

	reduceJobIds []int

	mapJobPendingList    *HashSet
	reduceJobPendingList *HashSet
	jobProcessingList    *HashSet
}
```

1. 启动

   1. 启动时，设置Coordinator状态为Map，
   2. 初始化Job List，map Job的名字为文件名，reduce Job的名字为nReduce的编号

2. 初始化`Worker`

   有新`Worker`来时，为其分配编号，并传回nReduce

3. 分发任务

   Worker会定时请求任务

   1. 从`JobPendingList`中选取Job
   2. 将这个Job放入`JobProcessingList`
   3. 如果任务完成，将任务从JobProcessingList彻底删除
   4. 发送任务后，注册一个回调函数，如果10s后这个任务还在`JobProcessingList`，说明任务超时，通过回调函数将任务放回`JobWaitingList`

4. 任务完成

   Worker任务完成时，将Job从List中删除

5. 状态转换

   如果所有任务都已完成，

任务名就是文件名：

- Map的任务名是`输入文件名`
- Reduce的任务名是`nReduce编号`

1. 分发任务

   Worker会定时发来任务请求，

   1. 从`JobWaitingList`中选取任务给他
   2. 将这个任务放入`JobProcessingList`
   3. 如果任务完成，将任务从JobProcessingList彻底删除

   4. 发送任务后，注册一个回调函数，如果10s后这个任务还在`JobProcessingList`，说明任务超时，通过回调函数将任务放回`JobWaitingList`

2. 当`JobWaitingList`和`JobProcessingList`皆为空时，意味着任务完成，`Coordinator`可以退出

3. 只有当Map完成时，才可进行Reduce

   1. Map阶段

   2. Reduce阶段


## Worker

代码地址：`mr/worker.go`

### 1.`Map Worer`

读取文件，调用Map函数处理，将结果按照Hash值分配到nReduce个文件

中间文件名`mr-X-Y`

- X：Map编号
- Y：Reduce编号

### 2.`Reduce Worker`

结果文件名：`mr-out-X`

- X：Reduce编号

注：为应对两个Worker同时处理某个任务、以及任务失败时的情况，Worker创建文件时，为其添加特殊后缀，比如`mr-X-Y`创建为`mr-X-Y_123456`。在任务处理完成，向coordinate汇报时，修改回正确文件名`mr-X-Y`。

# 结果

代码地址：

[Github](https://github.com/INnoVationv2/6.5840)

测试结果

<img src="http://pic-save-fury.oss-cn-shanghai.aliyuncs.com/uPic/image-20240825175144678.png" alt="image-20240825175144678" style="zoom:50%;" />
