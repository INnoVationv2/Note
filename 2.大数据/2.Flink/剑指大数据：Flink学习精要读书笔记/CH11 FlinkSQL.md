# CH-11 FlinkSQL

## 11.1 快速上手

1. 引入依赖

   ```xml
   <dependency>
     <groupId>org.apache.flink</groupId>
     <artifactId>flink-table-api-scala-bridge_2.12</artifactId>
     <version>${flink.version}</version>
     <scope>test</scope>
   </dependency>
   
   <dependency>
     <groupId>org.apache.flink</groupId>
     <artifactId>flink-table-planner_2.12</artifactId>
     <version>${flink.version}</version>
     <scope>test</scope>
   </dependency>
   
   <dependency>
     <groupId>org.apache.flink</groupId>
     <artifactId>flink-table-common</artifactId>
     <version>${flink.version}</version>
     <scope>provided</scope>
   </dependency>
   ```

   1. Scala桥接器，连接`Table API`和`DataStream API`，按照不同的语言分为Java版和Scala版。

   2. 计划器，是Table API的核心组件，负责提供运时环境，生成程序执行计划。Flink安装包的lib目录下会自带planner，所以在生产环境中提交的作业不需要打包这个依赖。

   3. 如果要实现自定义的数据格式来做序列化，需要引入。

2. 简单示例

   [示例代码](https://github.com/INnoVationv2/FlinkLearnDemo/blob/main/src/main/scala/com/innovationv2/CH11/TableDemo.scala)

## 11.2 基本API

程序的基本架构

### 1.创建表环境

```scala
// 方法1
val settings = EnvironmentSettings
.newInstance()
.inBatchMode()
//.inStreamingMode()
.build()
val tableEnv = TableEnvironment.create(settings)

// 方法2
val env = StreamExecutionEnvironment.getExecutionEnvironment
val tableEnv = StreamTableEnvironment.create(env)
```

创建流处理场景，使用`方法2`就够用了

### 2.创建表

表有一个唯一的ID，由三部分组成：`目录名`、`数据库名`、`表名`。默认情况下，目录名为`default_catalog`，数据库名为`default_database`。如果需要修改目录和数据库名，可以在环境中进行设置。

```scala
tableEnv.useCatalog("..")
tableEnv.useDatabase("...")
```

创建表的方式有两种：连接器和虚拟表

1. 连接器表

   可使用SQL或TableAPI创建

   **SQL：**

   ```sql
   CREATE TABLE event_kafka(
     id INT,
     `user` STRING,
     url STRING,
     ts_int BIGINT
   ) WITH (
     'connector' = 'kafka',
     'topic' = 'events',
     'properties.bootstrap.servers' = 'localhost:9092',
     'properties.group.id' = 'CreateTableDemoConsumer',
     'scan.startup.mode' = 'latest-offset',
     'format' = 'avro'
   )
   ```

   **TableAPI**

   ```scala
   val tableSchema = Schema.newBuilder()
   	.column("id", DataTypes.INT)
   	.column("user", DataTypes.STRING)
   	.column("url", DataTypes.STRING)
   	.column("ts_int", DataTypes.BIGINT)
   	.build()
   val tableDescriptor = TableDescriptor
   	.forConnector("kafka")
   	.schema(tableSchema)
   	.format("avro")
   	.option("topic", "events")
   	.option("properties.bootstrap.servers", "localhost:9092")
   	.option("properties.group.id", "CreateTableDemoConsumer")
   	.option("scan.startup.mode", "latest-offset")
   	.build()
   tableEnv.createTable("event_kafka", tableDescriptor)

2. 虚拟表

   ```scala
   val eventTable = tableEnv.fromDataStream(sourceStream)
   tableEnv.createTemporaryView("event_table", eventTable)
   ```

   虚拟表由中间结果创建而来，创建后可通过sql进行查询

[案例代码](https://github.com/INnoVationv2/FlinkLearnDemo/blob/main/src/main/scala/com/innovationv2/CH11/CreateTableDemo.scala)

### 3.查询表

```scala
val eventTable = tableEnv.fromDataStream(sourceStream)
tableEnv.createTemporaryView("event_table", eventTable)

// SQL查询
val queryResult = tableEnv.sqlQuery(s"select id, user from event_table")
// Table API
import org.apache.flink.table.api.Expressions.$
val table = tableEnv.from("event_table")
val queryResult = table
	.where($("user").isEqual("Alice"))
	.select($("id"), $("user"))
tableEnv.toDataStream(queryResult).print()
```

[案例代码](https://github.com/INnoVationv2/FlinkLearnDemo/blob/main/src/main/scala/com/innovationv2/CH11/QueryTableDemo.scala)

### 4.输出表

使用connector创建表后，向这个表写入数据，数据就会自动输出到对应的外部系统

比如之前创建的event_kafka表，当写入数据到event_kafka时，数据会自动写入kafka中

```scala
private def _sendEventToKafka(): Unit = {
  println("Sending Event Data To Kafka")
  val env = StreamExecutionEnvironment.getExecutionEnvironment
  val tableEnv = StreamTableEnvironment.create(env)

  tableEnv.createTable("event_kafka", getEventKafkaDescriptor())

  // 将流转换成表
  val eventStream = env.addSource(new BasicEventSource(cnt = 100))
  // 把数据写入Kafka
  tableEnv.fromDataStream(eventStream)
  .executeInsert("event_kafka")
  .print()

  env.execute()
}
```

### 5.表和流转换

1. Table --> Stream

   1. `toDataStream()`

   2. `toChangelogStream()`


2. Stream --> Table

   1. `fromDataStream()`

   2. `createTemporaryView()`

   3. `fromChangelogStream()`

## 11.3 时间属性和窗口

### 1.事件时间

两种方式指定时间字段

1. 在创建表的DDL中定义

   ```sql
   CREATE TABLE EventTable(
     id INT,
   	user STRING,
   	url STRING,
     ts TIMESTAMP(3)
     WATERMARK FOR ts AS ts - INTERVAL '5' SECOND
   ) WITH (
     ...
   )
   ```

   - 把`ts`定义为了事件时间属性，而且设置了5s的水位线延迟

   - 数值必须用单引号引起来，单位用SECOND和SECONDS是等效的

   - 事件时间字段类型必须为`TIMESTAMP`或者`TIMESTAMP_LTZ`。

     -  `TIMESTAMP_LTZ`是指带有本地时区信息的时间戳，如果数据格式是`年-月-日-时-分-秒`，那就是不带时区信息的TIMESTAMP类型。

   - 如果时间存储为长整型毫秒数，就需要先进行转换

     ```sql
     ...
       ts BIGINT,
       ts_ltz AS TO_TIMESTAMP_LTZ(ts, 3)
       WATERMARK FOR ts_ltz AS ts_ltz - INTERVAL '5' SECOND
     ...
     ```

2. 在数据流转换为表时定义

   ```scala
   val stream = env.addSource(new BasicEventSource())
     .assignAscendingTimestamps(_.timestamp)
   val eventTable = tableEnv.fromDataStream(stream, $("id"), $("user"), $("url"), $("timestamp").rowtime().as("ts"))
   ```

   1. 为DataStream指定时间戳
   2. 创建table时，指定时间属性字段

### 2.处理时间

处理时间就是系统时间，不需要提取时间戳和生成水位线。需要额外声明一个字段，保存当前的处理时间。

1. 在创建表的DDL中定义

   ```sql
   // PROCTIME()返回当前时间，数据类型：TIMESTAMP_LTZ
   CREATE TABLE EventTable(
     id INT,
   	user STRING,
   	url STRING,
     ts AS PROCTIME()
   ) WITH (
     ...
   )
   ```

2. 在数据流转换为表时定义

   ```scala
   val eventTable = tableEnv.fromDataStream(stream, $("id"), $("user"), $("timestamp").proctime())
   ```

处理时间字段必须是一个新字段

## 3.窗口

从Flink 1.13开始使用窗口表值函数`(Windowing Table-Valued Function，Windowing TVF)`来定义窗口，窗口表值函数会对原表进行扩展，然后返回新表，我们对新表进行处理

窗口函数返回值：除原始表中的列，额外增加3个列：窗口起始点`window_start`、窗口结束点`window_start`、窗口时间（`窗口时间`等于`window_end-1 ms`，相当于窗口能够包含的最大时间戳）。

1. 滚动窗口

   ```sql
   TUMBLE(TABLE [table_name]， descriptor([time_field]), INTERVAL `1` HOUR)
   ```

2. 滑动窗口

   ```sql
   HOP(TABLE [table_name]， descriptor([time_field]), INTERVAL `5` MINUTES, INTERVAL `1` HOUR)
   ```

   第三个参数为滑动步长，第四个参数是窗口长度

3. 累积窗口

   该窗口面向这种需求，以1天为窗口统计数据，但是希望每个小时输出一次结果

   参数：`窗口长度`、`累积步长`

   ```sql
   CUMULATE(TABLE [table_name]， DESCRIPTOR([time_field]), INTERVAL `1` HOURS， INTERVAL `1` DAYS)
   ```

   第三个参数为累积步长，第四个参数是窗口长度

[案例代码](https://github.com/INnoVationv2/FlinkLearnDemo/blob/main/src/main/scala/com/innovationv2/CH11/WindowDemo.scala)

## 11.4 聚合查询

### 1.分组聚合

`SUM()`、`MAX()`、`MIN()`、`AVG()`、`COUNT()`等

```SQL
SELECT COUNT(score) FROM TABLE_XXX GROUP BY user
```

### 2.窗口聚合

之前已经[讲过](##3.窗口)，窗口表值函数会对原表进行扩展，我们在新表的基础上进行处理，但是窗口表值函数只是增加3个字段，具体怎么处理需要自行确定

比如`滚动窗口`，新增3个字段后，每行除了原有数据，新增`window_start`和`window_end`字段

如果想进行窗口聚合，那就使用`GROUP BY`

```sql
SELECT
	COUNT(score) 
FROM 
	TABLE(TUMBLE(TABLE table_xx, DESCRIPTOR(ts), INTERVAL '10' SECOND)) 
GROUP BY 
	window_start, window_end
```

[案例代码](https://github.com/INnoVationv2/FlinkLearnDemo/blob/main/src/main/scala/com/innovationv2/CH11/WindowDemo.scala)

### 3.开窗聚合

开窗聚合面向这么一种需求：比如有学生表，三个属性：姓名、年龄，成绩，现在每一行新增一个字段，计算本行往前10个数据的成绩和。

开窗聚合针对每一行计算一个聚合值。比如以每一行数据为基准，计算之前1小时内所有数据的平均值，或计算之前10个数的平均值。就好像在每一行上打开了一扇窗户、收集数据进行统计，这就是开窗函数。

开窗聚合与之前聚合的区别：窗口TVF聚合是`多对一`，数据分组后每组得到一个聚合结果。而开窗函数对每行做一次聚合，因此聚合之后行数不会改变，是`多对多`的关系。

格式

```sql
SELECT 
	<聚合函数> OVER(
    [PARTITION BY <字段1> [, <字段2>, ...]]
    ORDER BY <时间属性字段> 
    <开窗范围>)
  ...
  )
FROM ...
```

1. `PARTITION BY`（可选）：

   指定分区的键，类似于GROUP BY，这部分是可选的。

2. `ORDER BY`

   进行开窗时，无论是基于时间还是基于行数，数据都必须有序，不然每次聚合结果都会不一样，所以必须指明基于哪个字段排序，Flink目前只支持时间属性的升序排列，所以这里的字段必须是定义好的时间属性。

3. 开窗范围。

   指定开窗范围，即扩展多少行做聚合。这个范围由`BETWEEN <下界> AND <上界> `来定义。目前支持的上界只能是`CURRENT ROW`，也就是`从之前某一行到当前行`的范围.

   开窗范围可以基于时间，也可以基于数据的数量。对应两种模式：范围间隔和行间隔。

   1. 范围间隔

      基于ORDER BY指定的时间字段选取范围，一般指当前行时间戳之前的一段时间。

      例如，开窗范围选择当前行之前1小时的数据：

      ```sql
      RANGE BETWEEN INTERVAL '1' HOUR PRECEDING AND CURRENT ROW

   2. 行间隔

      直接确定要选多少行，由当前行向前选取。

      例如，开窗范围选择当前行之前的5行数据（最终聚合会包括当前行，所以一共6条数据）：

      ```SQL
      ROWS BETWEEN 5 PRECEDING AND CURRENT ROW

[案例代码](https://github.com/INnoVationv2/FlinkLearnDemo/blob/main/src/main/scala/com/innovationv2/CH11/OverAggDemo.scala)

在窗口那一节讲过，TVF窗口相当于对原表进行扩展，增添了时间字段，我们对扩展后的表进行处理。开窗函数实际上也差不多：都是在无界的数据流上划定了一个范围，截取出数据集进行聚合统计：

```sql
SELECT id, user,
 SUM(id) OVER w AS sum_id,
 MAX(CHAR_LENGTH(url)) OVER w as max_url
FROM EventTable
WINDOW w AS (
  ORDER BY ts
  ROWS BETWEEN 1 PRECEDING AND CURRENT ROW
)
```

上面的SQL，`WINDOW w`就相当于包含本条和前一条数据的数据集，对这个数据集进行聚合，就能得到结果。

### 代码案例--`TOP N`

求数据某个字段的最大值时，有时不仅希望得到最大/最小值，还希望得到前N个最大/最小值。这时每次聚合的结果就不是一行，而是N行了。这就是经典的`Top N`应用场景。

TopN函数

```sql
SELECT [column_list]
FROM (
  SELECT [column_list],
  ROW_NUMBER() OVER (
    [PARTITION BY col1[, col2...]]
    ORDER BY col1 [asc|desc][, col2 [asc|desc]...]) AS rownum
  FROM table_name)
WHERE rownum <= N [AND conditions]
```

OVER定义与之前基本一致，用`ROW_NUMBER()`为每一行数据聚合得到一个排序后的行号。行号命名为`row_num`，在外层以`row_num<=N`作为条件进行筛选，就可以得到Top N结果。

关键字说明

1. `WHERE`

   用来指定Top N选取的条件，这里必须通过row_num<=N或者row_num<N+1指定一个排名结束点(rank end)，以保证结果有界。

2. `PARTITION BY`可选的

   指定分区的字段，这样我们就可以针对不同的分组分别统计Top N

3.  `ORDER BY`

   指定排序字段，才能进行前N个最大/最小的选取。每个字段排序后可以用asc或者desc来指定排序规则：asc为升序排列，取出的就是最小的N个值。desc为降序排序，对应的就是最大的N个值。默认情况下为升序，asc可以省略。

之前介绍OVER窗口时说过，`ORDER BY`后面只能跟时间字段，并且只支持升序，这里为什么不一样？

这是Flink SQL针对Top N这个场景专门做的实现。要想实现Top N，必须按照上面的格式，否则将无法解析。Table API也并不支持ROW_NUMBER()函数，只有SQL中这一种Top N实现方式。

#### 案例1

> 需求：按照url长度进行排序，输出Top2

```sql
SELECT id, user, url, ts, row_num
FROM (
 SELECT *,
   ROW_NUMBER() OVER(
     ORDER BY CHAR_LENGTH(url) DESC
   ) AS row_num
 FROM EventTable
)
WHERE row_num <= 2
```

[完整代码](https://github.com/INnoVationv2/FlinkLearnDemo/blob/main/src/main/scala/com/innovationv2/CH11/TopNDemo.scala)中的`testTopN1()`

#### 实现2

> 需求：以10秒为一个窗口，统计用户访问量，输出访问量前2名的用户信息

分为两步

1. 先使用滚动窗口对数据进行分割

   ```sql
   SELECT
    window_start, window_end, user, COUNT(url) as cnt
   FROM
    TABLE(TUMBLE(TABLE EventTable, DESCRIPTOR(ts), INTERVAL '10' SECOND))
   GROUP BY window_start, window_end, user

2. 使用开窗函数输出访问量前2名的用户

   ```sql
   SELECT user, cnt, window_start, window_end
   FROM (
     SELECT *,
     	ROW_NUMBER() OVER (
       	PARTITION BY window_start, window_end
      		ORDER BY cnt DESC
    		) AS row_num
     FROM ($subSql)
   )
   WHERE row_num <= 2
   ```

   subSql就是第1步中的sql，查询时需要拼接到一起

[完整代码](https://github.com/INnoVationv2/FlinkLearnDemo/blob/main/src/main/scala/com/innovationv2/CH11/TopNDemo.scala)中的`testTopN2()`

## 11.5 Join查询

1. `INNER JOIN`

   全相连Join，对满足条件的数据生成笛卡尔积

2. `外连接`

   支持`LEFT JOIN`、 `RIGHT JOIN`、`FULL OUTER JOIN`

3. 间隔联结(`Interval Join`)

   ```sql
   SELECT 
   	u._2, p.*
   FROM 
   	userTable u, pvTable p
   WHERE 
   	p._2 BETWEEN u._2 - 30000 AND u._2 + 30000
   ```

[完整代码](https://github.com/INnoVationv2/FlinkLearnDemo/blob/main/src/main/scala/com/innovationv2/CH11/JoinDemo.scala)

## 11.7 自定义函数

系统函数尽管庞大，但不可能涵盖所有功能，如果有系统函数不支持的需求，可以用自定义函数(User-Defined-Function，UDF)来实现。

Flink提供了多种自定义函数的接口，以抽象类的形式定义。当前UDF主要有以下几类：

- `标量函数`：将输入的标量值转换成一个新的标量值。
- `表函数`：将标量值转换成一个或多个新的行数据，也就是扩展成一个表。
- `聚合函数`：将多行数据里的标量值转换成一个新的标量值。
- `表聚合函数`：将多行数据里的标量值转换成一个或多个新的行数据。

### 1.使用流程

1. 注册函数

   ```scala
   // 创建全局函数
   tableEnv.createTemporarySystemFunction("MyFunction", classOf[MyFunction])
   
   // 为当前目录创建函数,注册的函数只在当前数据库和目录有效
   createTemporaryFunction("MyFunction", classOf[MyFunction])
   ```

2. 调用函数

   1. 通过TableAPI调用

      ```scala
      // 通过TableAPI调用注册好的函数
      tableEnv.from("MyTable").select(call("MyFunction", $("myField")))
      
      // 通过TableAPI直接调用未注册过的函数
      tableEnv.from("MyTable").select(call(MyFunction.class, $("myField")))

   2. 通过SQL调用

      ```sql
      SELECT MyFunction(myField) FROM MyTable
      ```

### 2. UDF介绍

#### 1.标量函数

可以把`0、1或多个`标量值转换成一个标量值，输入：一行数据中的字段，输出唯一的值。从输入和输出来看，标量函数是`一对一`的转换。

```scala
//用DataTypeHint(inputGroup=InputGroup.ANY)对输入参数的类型做了标注，表示eval的参数可以是任意类型
class HashFunction extends ScalarFunction {
  def eval(@DataTypeHint(inputGroup = InputGroup.ANY) o: AnyRef): Int = {
    o.hashCode()
  }
}
```

[代码实现](https://github.com/INnoVationv2/FlinkLearnDemo/blob/main/src/main/scala/com/innovationv2/CH11/UDFDemo.scala)

#### 2.表函数

输入参数可以是0、1或多个标量值。返回任意行数据。`多行数据`事实上就构成了一个表，所以`表函数`就是返回一个表的函数，是一个`一对多`的关系。之前介绍的窗口TVF，本质上就是表函数。

```scala
@FunctionHint(output = new DataTypeHint("Row<word String, length INT>"))
class SplitFunction extends TableFunction[Row] {
  def eval(str: String): Unit = {
    str.split(" ").foreach(s => collect(Row.of(s, Int.box(s.length))))
  }
}
```

[代码实现](https://github.com/INnoVationv2/FlinkLearnDemo/blob/main/src/main/scala/com/innovationv2/CH11/UDFDemo.scala)

#### 3.聚合函数

把一行或多行数据（也就是一个表）聚合成一个标量值，SUM()、MAX()、MIN()、AVG()、COUNT()都是聚合函数。

需要继承抽象类AggregateFunction。两个泛型参数<T，ACC>，T：聚合输出的结果类型，ACC：聚合的中间状态类型。

聚合函数的工作原理如下：

1. 通过`createAccumulator()`创建累加器(Accumulator)，存储聚合中间结果。
2. 每输入一行数据，都会调用`accumulate()`更新累加器。
3. 数据处理完后，调用`getValue()`获取最终结果。

> 案例：实现加权平均值
>
> 若n个数$x_1, x_2, ... x_n$的权分别是$w_1, w2_,...,w_n$，那么加权平均值就是
>
> ![img](http://pic-save-fury.oss-cn-shanghai.aliyuncs.com/uPic/12a715d6dcbd221ca1a88facc2e31191.svg)
>
> 输入成绩和权重，输出加权平均值

[代码实现](https://github.com/INnoVationv2/FlinkLearnDemo/blob/main/src/main/scala/com/innovationv2/CH11/UDFDemo.scala)

#### 4.表聚合函数

把一行或多行数据（也就是一个表）聚合成另一张表，结果表中可以有多行多列

需要继承抽象类`TableAggregateFunction`。两个泛型参数<T，ACC>，必须实现的三个方法

1. `createAccumulator()`：创建累加器

2. `accumulate()`：聚合计算

3. `emitValue()`：

   没有输出，输入参数有两个：第一个是累加器ACC，第二个是收集器out，类型为Collect\<T>，通过调用`out.collect()`输出数据

   `emitValue()`在抽象类中没有定义，无法`override`，须手动实现。

## 附录:一些优化

1. 设置状态过期时间

   为了防止状态无限增长耗尽资源，Flink Table API和SQL可以在表环境中配置状态的生存时间(TTL)，状态超过一段时间没被访问过就丢弃

   ```scala
   val config = tableEnv.getConfig
   config.setIdleStateRetention(Duration.ofSeconds(60))
   ```
