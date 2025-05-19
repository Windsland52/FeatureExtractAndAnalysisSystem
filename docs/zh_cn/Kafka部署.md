# Kafka 部署

前置完成[Zeek部署](./Zeek部署.md)

## Apache Kafka

```bash
apt install openjdk-11-jdk -y
java -version

# 考虑到稳定性，选择 3.7.2 版本
mkdir kafka_setup
cd kafka_setup
wget https://archive.apache.org/dist/kafka/3.7.2/kafka_2.13-3.7.2.tgz
tar -xzf kafka_2.13-3.7.2.tgz
mv kafka_2.13-3.7.2 kafka
cd kafka
# 运行 ZooKeeper 启动脚本，启动时无报错便保留该终端，新开终端
bin/zookeeper-server-start.sh config/zookeeper.properties

# 先回到 kafka 目录，启动 Kafka Broker 服务
bin/kafka-server-start.sh config/server.properties

# 创建一个名为 my-test-topic 的主题。这个主题将有 1 个分区 (partition) 和 1 个副本因子 (replication factor)。对于单节点集群，副本因子只能是 1
bin/kafka-topics.sh --create --topic my-test-topic --bootstrap-server localhost:9092 --partitions 1 --replication-factor 1
```

```plaintext
--create: 表示我们要创建一个新的主题。
--topic my-test-topic: 指定主题的名称。
--bootstrap-server localhost:9092: 指定连接 Kafka Broker 的地址和端口。localhost:9092 是默认配置。
--partitions 1: 指定主题的分区数量。
--replication-factor 1: 指定主题的副本因子。
```

```bash
# 验证
bin/kafka-topics.sh --list --bootstrap-server localhost:9092
bin/kafka-console-producer.sh --topic my-test-topic --bootstrap-server localhost:9092
# 后面随便输几条信息

# 再开个终端，可以看到前面输的信息，前面生成者部分也可以继续输入几条，可以看到消费者部分这边有输出
bin/kafka-console-consumer.sh --topic my-test-topic --from-beginning --bootstrap-server localhost:9092
```

## 本地编译 librdkafka

这是 Apache Kafka C/C++ library，安装 librdkafka 是为了让 Zeek 系统能够作为 Kafka 客户端，将数据成功发送到 Kafka 服务器。

Kafka 和 Zeek 连接的核心是 `Zeek 的 Kafka 插件`，因为插件要求 librdkafka 版本 1.4.2，距离当前最新版（2.3.0-1build2）差距过大，选择从 1.4.2版本源码构建。

```bash
apt install -y git build-essential pkg-config zlib1g-dev libssl-dev libsasl2-dev libzstd-dev cmake
```

```bash
cd /home
git clone https://github.com/confluentinc/librdkafka.git
cd librdkafka
git checkout v1.4.2
./configure --prefix=/usr/local  # 指定安装路径为 /usr/local
make
make install
ldconfig
```

## Zeek Kafka 插件

### 安装

```bash
zkg refresh
zkg search kafka
zkg install zeek/seisollc/zeek-kafka
```

### 配置

修改 `share/zeek/site/local.zeek` 文件，参考 [SeisoLLC/zeek-kafka](https://github.com/SeisoLLC/zeek-kafka)，以下是我的修改：

```zeek
# Installation-wide salt value that is used in some digest hashes, e.g., for
# the creation of file IDs. Please change this to a hard to guess value.
redef digest_salt = "Please change this";

@load packages/zeek-kafka
redef Kafka::topic_name = "";
redef Kafka::send_all_active_logs = T;
redef Kafka::kafka_conf = table(
    ["metadata.broker.list"] = "localhost:9092"
);
```

保存后可以 `zeekctl deploy` 重新加载 zeek，加载插件日志（由 misc/loaded-scripts 插件生成）可在 `logs/current/loaded_scripts.log` 查看。
