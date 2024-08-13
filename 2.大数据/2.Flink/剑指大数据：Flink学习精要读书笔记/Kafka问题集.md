# Kafka问题集

## 1.设置`auto.offset.reset`

Kafka默认读取模式是`group-offsets`，即从当前group上次读的停止位置继续

但是如果该group是第一次出现，kafka中没有他的offset，就会报错，需要设置`auto.offset.reset`参数

在Flink中，设置了该参数仍然会报错

```
Caused by: org.apache.flink.table.api.ValidationException: Unsupported options found for 'kafka'.
Unsupported options:
auto.offset.reset
```

需要修改为

```scala
.option("properties.auto.offset.reset", "earliest")
```

