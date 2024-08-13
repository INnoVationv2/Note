## Kafka Quick Onboard

## 1.安装

我们用docker进行安装

1. 去Docker官网下载并安装`Docker Desktop`

2. 去[Docker Hub](https://hub.docker.com/r/apache/kafka/tags)查看Kafka最新版本，我这里使用3.8.0，打开指令，输入以下安装指令

   ```sh
   docker pull apache/kafka:3.8.0
   ```

3. 下载完成后，启动kafka

   ```sh
   docker run -p 9092:9092 apache/kafka-native:3.8.0
   ```

   > 记得把`apache/kafka-native:3.8.0`换成你自己的镜像名

## 2.创建Topic

1. 打开Docker Desktop，进入Container，复制kafka容器的ID，即下图的`30ac9c2da5d9`

   <img src="http://pic-save-fury.oss-cn-shanghai.aliyuncs.com/uPic/image-20240810221651149.png" style="zoom:40%;" />

2. 打开`terminal`，进入kafka容器内

   ```sh
   docker exec -it 30ac9c2da5d9 sh
   ```

   > 把`30ac9c2da5d9`换成你的容器ID

3. 进入Kafka安装目录，默认在`/opt/kafka`

4. 创建Topic

   ```sh
   bin/kafka-topics.sh --create --topic events --bootstrap-server localhost:9092
   ```

   > `events`换成你的topic名

## 3.常用指令

```sh
// 创建topic
bin/kafka-topics.sh --create --topic [Topic_Name] --bootstrap-server localhost:9092

// 查看Topic详情
bin/kafka-topics.sh --describe --topic [Topic_Name] --bootstrap-server localhost:9092

// 删除Topic
bin/kafka-topics.sh --delete --topic [Topic_Name] --bootstrap-server localhost:9092

// 查看Topic数据数量
./kafka-get-offsets.sh --topic [Topic_Name] --bootstrap-server localhost:9092

//清空Topic
先删除然后创建
```

