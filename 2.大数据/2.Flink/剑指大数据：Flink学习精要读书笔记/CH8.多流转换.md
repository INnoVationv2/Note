# CH-8 多流转换

多流转换分为两类：`分流`、`合流`

，合流的算子比较丰富

## 8.1 分流

一般通过侧输出流实现

[完整代码](https://github.com/INnoVationv2/FlinkLearnDemo/blob/main/src/main/scala/com/innovationv2/CH8/SideOutput.scala)

## 8.2 合流

### 1.`union()`

> `DataStream` → `DataStream`

合并多个DataStream，要求所有`stream`内的元素类型相同

```java
DataStreamSource<Integer> source_1 = env.fromElements(0, 2, 4, 6, 8),
											    source_2 = env.fromElements(1, 3, 5, 7, 9),
  	 							        source_3 = env.fromElements(10, 11, 12, 13, 14);
DataStream<Integer> union = source_1.union(source_2, source_3);
union.print();
```

合并后的水位线怎么算？

和6.2.4节讲过的，上游有多个算子的情况相同，union为每一条流维护一个水位线，当前水位线的值以水位线最小的那一条流的水位线为准。

### 2.`connect()`

>两个`DataStream`→`ConnectedStreams`

- 可以将两个数据类型一样或不一样 DataStream 连接成一个新的ConnectedStreams
- `connect`与`union`不同，调用`connect`方法将两个流连接成一个新的流，但是两个流内部是相互独立的

```java
DataStreamSource<Integer> source_1 = env.fromElements(0, 2, 4, 6, 8),
                          source_2 = env.fromElements(1, 3, 5, 7, 9);
ConnectedStreams<Integer, Integer> connect = source_1.connect(source_2);
```



`ConnectedStreams`也有相关的处理算子

#### 1.`coMap()`

>`ConnectedStreams` → `DataStream`

对`ConnectedStreams`调用`map`方法时需要传入`CoMapFunction`函数

```java
SingleOutputStreamOperator<Integer> map = connect
  .map(new CoMapFunction<Integer, Integer, Integer>() {
    // 处理第一条流中的数据
    @Override
    public Integer map1(Integer v1) {
      System.out.println("Map_1: " + v1);
      return v1;
    }

    // 处理第二条流中的数据
    @Override
    public Integer map2(Integer v2) throws Exception {
      System.out.println("Map_2: " + v2);
      return v2;
    }
  });
```

#### 2.`coFlatMap()`

> `ConnectedStreams` → `DataStream`

和FlatMap类似

```java
SingleOutputStreamOperator<Integer> map = connect
  .flatMap(new CoFlatMapFunction<>() {
    @Override
    public void flatMap1(Integer v1, Collector<Integer> collector){
      collector.collect(v1);
      collector.collect(v1);

      @Override
      public void flatMap2(Integer v2, Collector<Integer> collector){
        collector.collect(v2);
        collector.collect(v2);
      }
    });
    map.print();
```

#### 3.`CoProcessFunction`

```scala
public abstract class CoProcessFunction<IN1, IN2, OUT> extends AbstractRichFunction {
  // 处理第一条流中的数据
  public abstract void processElement1(IN1 value, Context ctx, Collector<OUT> out) throws Exception;

  // 处理第二条流中的数据
  public abstract void processElement2(IN2 value, Context ctx, Collector<OUT> out) throws Exception;

  public void onTimer(long timestamp, OnTimerContext ctx, Collector<OUT> out) throws Exception {}

  public abstract class Context{...}
}
```

同样可以通过上下文访问timestamp、水位线，并通过TimerService注册定时器

也提供了onTimer()方法，用于定义定时触发的处理操作。

### 3.`Join()`

Join使用示例

```java
stream.join(otherStream)
	.where(<KeySelector>)
	.equalTo(<KeySelector>)
	.window(<WindowAssigner>)
  [.trigger()]
  [.allowedLateness()]
	.apply(<JoinFunction>)
```

1. 使用`join()`合并两条流，得到`JoinedStreams`

2. 通过`where()`和`equalTo()`指定Join条件

   `where`的参数：KeySelector，指定第一条流中Join的key

   `equalTo`的参数：KeySelector，指定了第二条流中Join的key

3. 通过`window()`开窗

   三种时间窗口都可以用：滚动、滑动、会话窗口。

4. 通过`apply()`传入`JoinFunction`进行处理计算

   这里只能调用apply()

   `JoinFunction`有两个参数：分别表示两条流中匹配成功的数据。

   ```java
   public interface JoinFunction<IN1, IN2, OUT> extends Function, Serializable {
     OUT join(IN1 first, IN2 second) throws Exception;
   }
   ```

#### 内部处理流程

1. 两条流到来，按照key分组，进入相应的窗口中存储
2. 到达窗口结束时间，对两条流中的所有数据做笛卡尔积，然后进行遍历，检查是否匹配，把匹配成功的数据传给`JoinFunction`的`join()`进行处理，
3. 输出结果

### 4.间隔`Join`

<img src="http://pic-save-fury.oss-cn-shanghai.aliyuncs.com/uPic/image-20240812175029467.png" alt="image-20240812175029467" style="zoom:50%;" />

间隔联结：两条流A、B，时间字段分别为ts_a、ts_b

匹配时，设定上下界时间，比如上下各5秒，那么a中数据的匹配间隔就是`[ts_a-5, ts_a+5]`，之后通过这个界，筛选出b流数据，然后进行匹配。

给定两个时间点，`上界(upperBound)`和`下界”(lowerBound)`。对于一条流中的任意元素a，就可以开辟一段时间间隔：`[a.timestamp+lowerBound，a.timestamp+upperBound]`，看另一个流是否有数据匹配在这段间隔内

间隔联结目前只支持`事件时间`语义。

**案例**

两条流：订单流、网页浏览流。针对同一个用户，对用户下订单的事件和用户最近30s(向前向后各30s)的浏览数据进行联结查询。

[完整代码](https://github.com/INnoVationv2/FlinkLearnDemo/blob/main/src/main/scala/com/innovationv2/CH8/IntervalJoinDemo.scala)

### 5. 窗口同组`Join`

使用模板：

```scala
stream.join(otherStream)
	.where(<KeySelector>)
	.equalTo(<KeySelector>)
	.window(<WindowAssigner>)
  [.trigger()]
  [.allowedLateness()]
	.apply(<CoGroupFunction>)
```

`CoGroupFunction`定义如下

```scala
public interface CoGroupFunction<IN1, IN2, O> extends Function, Serializable {
  void coGroup(Iterable<IN1> first, Iterable<IN2> second, Collector<O> out) throws Exception;
}
```

三个参数：两条流中的数据以及用于输出的Collector，这里的前两个参数不是每一组“配对”数据，而是所有匹配的数据。

[使用案例](https://github.com/INnoVationv2/FlinkLearnDemo/blob/main/src/main/scala/com/innovationv2/CH8/CoGroupFunctionDemo.scala)

