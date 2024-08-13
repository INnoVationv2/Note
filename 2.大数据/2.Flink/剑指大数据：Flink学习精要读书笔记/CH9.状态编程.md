# CH-9 状态编程

## 9.1 状态简介

状态分为两种：

1. 托管状态

   由Flink统一管理，状态的存储、故障恢复等都由Flink实现，直接使用即可

2. 原始状态

   自定义，相当于开辟了一块内存，需要自己管理，实现状态的序列化和故障恢复。

状态类型包含值状态(ValueState)、列表状态(ListState)、映射状态(MapState)、聚合状态(AggregateState)等

本章主要介绍`托管状态`



按照`生效范围`，状态又分为算子状态和按键分区状态

1. 算子状态

   作用范围为当前的算子，也就是只对当前并行子任务实例有效

2. 按键分区状态

   为每个Key维护一个状态

## 9.2 按键分区状态

Flink在算子中，为每个Key维护一个`(Key, State)`键值对

Key不同，访问到的状态就不同

状态的存储默认在内存，但是可以通过`状态后端`配置存储方式。

当算子并行度改变时，Flink会对所有键值对进行重分配，使得任务平均分配

### 9.2.1 支持的状态类型

1. 值状态`(ValueState)`

   ```scala
   val valueState = getRuntimeContext.getState(new ValueStateDescriptor[Int]("value-state", classOf[Int]))
   ```

   Demo：输入`(User,Score)`，对每个User计算成绩

   [代码](https://github.com/INnoVationv2/FlinkLearnDemo/blob/main/src/main/scala/com/innovationv2/CH9/KeyedState/ValueStateDemo.scala)

2. 列表状态`(ListState)`

   ```scala
   lazy val listState = getRuntimeContext.getListState(new ListStateDescriptor[(Int, String)]("list-state", classOf[(Int, String)]))
   ```

3. 映射状态`(MapState)`

   ```scala
   lazy val mapState = getRuntimeContext.getMapState(new MapStateDescriptor[Int, String]("map-list", classOf[Int], classOf[String]))
   ```

4. 归约状态`(ReducingState)`

   ReduceFunction的输入与输出的数据类型是一样的

   [示例代码](https://github.com/INnoVationv2/FlinkLearnDemo/blob/main/src/main/scala/com/innovationv2/CH9/KeyedState/ReduceStateDemo.scala)

5. 聚合状态`(AggregatingState)`

   AggregateFunction的输入和输出的数据类型可以不同

   [示例代码](https://github.com/INnoVationv2/FlinkLearnDemo/blob/main/src/main/scala/com/innovationv2/CH9/KeyedState/AggregatingStateDemo.scala)

### 9.2.2 状态TTL

可以对状态设置生存时长，超出一段时间不使用就删除，防止耗尽存储空间。

[实例代码](https://github.com/INnoVationv2/FlinkLearnDemo/blob/main/src/main/scala/com/innovationv2/CH9/TTLDemo.scala)

## 9.3 算子状态

算子状态更底层，对当前算子的并行任务有效，不需要考虑不同key的隔离。算子状态功能不如按键分区状态丰富，应用场景较少，调用方法也不同。

### 9.3.1 基本概念

算子状态的作用范围为当前算子。算子状态跟数据的key无关，不同key的数据只要被分发到同一并行子任务，就会访问到同一State。

一般用在Source或Sink等与外部系统连接的算子上，或完全没有key定义的场景，在给Source设置并行度后，每一个并行实例会为对应的kafka topic分区维护一个偏移量，保存在算子状态中，从而保证消费时的精确一次(exactly-once)。

当算子并行度变化时，算子状态也支持重组分配。根据状态类型不同，重组方案也会不同。

### 9.3.2 状态类型

主要三种：列表状态`(ListState)`、联合列表状态`(UnionListState)`、广播状态`(BroadcastState)`。

1. 列表状态

   必须配合`CheckpointedFunction`使用，自定义状态初始化和创建备份快照的逻辑

   ```scala
   class UserSink(threshold: Int) extends SinkFunction[(String, Int)] with CheckpointedFunction {
   	...
     override def snapshotState(context: FunctionSnapshotContext): Unit = {
       // 创建快照,将buffer保存下来
       checkpointState.clear()
       buffer.foreach(checkpointState.add)
     }
   
     override def initializeState(context: FunctionInitializationContext): Unit = {
       val listStateDescriptor = new ListStateDescriptor[(String, Int)]("Buffered-elements", classOf[(String, Int)])
       //创建ListState
       checkpointState = context.getOperatorStateStore.getListState(listStateDescriptor)
       // 如果是Restored,还需要恢复状态
       if (context.isRestored)
       checkpointState.get().forEach(element => buffer += element)
     }
   }
   ```

   [完整代码](https://github.com/INnoVationv2/FlinkLearnDemo/blob/main/src/main/scala/com/innovationv2/CH9/OperatorState/ListStateDemo.scala)

   当Flink从检查点恢复时，ListState会将原有状态平均分配给算子所有子任务

2. 联合列表状态

   使用方法和列表状态相同，区别在于并行度调整时，常规列表状态是轮询分配状态项，而联合列表状态会广播完整的状态列表，自行决定状态的去留。

## 9.4 广播状态

      1. 将状态广播出去，所有并行子任务访问到的状态都是相同的
      2. 并行度调整时直接复制一份即可

案例：对用户行为和模式进行匹配，如果模式相同，就输出

[完整代码](https://github.com/INnoVationv2/FlinkLearnDemo/blob/main/src/main/scala/com/innovationv2/CH9/BroadcastStateDemo.scala)

### 检查点恢复状态分配算法

<img src="http://pic-save-fury.oss-cn-shanghai.aliyuncs.com/uPic/image-20240806200050424.png" alt="image-20240806200050424" style="zoom:50%;" />

## 9.5 状态持久化和状态后端

### 1.`CheckPoint`

`checkpoint`：所有任务状态在某个时间的一个快照。

```java
// 参数是创建CheckPoint间隔
env.enableCheckpointing(100)
```

除了CheckPoint，Flink还提供了保存点，保存点是自定义的镜像保存，由用户手动触发。这在有计划地停止、重启应用时非常有用。

### 2.状态后端