# linux-centos7搭建kafak集群

## 下载kafka，这里我们使用2.6.0版本

https://www.apache.org/dyn/closer.cgi?path=/kafka/2.7.0/kafka_2.13-2.7.0.tgz

##  解压
tar -zxvf kafka_2.13-2.7.0.tgz

## 单节点zookeeper和单节点kafka启动
### 修改配置文件
vim ./config/server.properties、
```
打开 listeners=PLAINTEXT://localhost:9092
```
### 后台启动zookeeper，和kafka
```
   cd kafka_2.13-2.7.0/
  ./bin/zookeeper-server-start.sh  config/zookeeper.properties
   nohup  ./bin/zookeeper-server-start.sh  config/zookeeper.properties &
  ./bin/kafka-server-start.sh config/server.properties
```
### 使用工具查看zookeeper和kafka信息

下载：zookeeper-dev-ZooInspector.jar
java -jar zookeeper-dev-ZooInspector.jar

### 相关命令
```
// 查看kafka的内容
bin/kafka-topics.sh --zookeeper localhost:2181 --list

//创建toptic为testk 4个分区，一个副本
bin/kafka-topics.sh --zookeeper localhost:2181 --create --topic testk --partitions 4 --replication-factor 1

//查看刚刚创建的topic
bin/kafka-topics.sh --zookeeper localhost:2181 --describe --topic testk

```
### 简单性能测试
```
    bin/kafka-producer-perf-test.sh --topic testk --num-records 100000 --record-size1000 --throughput 2000 --producer-props bootstrap.servers=localhost:9092
    bin/kafka-consumer-perf-test.sh ---bootstrap-server localhost:9092 --topic testk --fetch-size 1048576 --messages 100000 --threads 1
```
##  单节点zookeeper和集群kafka

### 清空之前zookeeper的信息
在ZooInspector界面上清空节点信息，除了zookeeper的信息

###  复制三份server.properties 到kafka_2.13-2.7.0目录下
```
[root@vm-JZ7A72FWtFRk config]# pwd
/root/myapp/kafka/kafka_2.13-2.7.0/config
[root@vm-JZ7A72FWtFRk config]# cp server.properties  ../kafka9001.properties
[root@vm-JZ7A72FWtFRk config]# cp server.properties  ../kafka9002.properties
[root@vm-JZ7A72FWtFRk config]# cp server.properties  ../kafka9003.properties
```

### 编辑 vim kafka9001.properties   vim kafka9002.properties   vim kafka9003.properties
 四个属性修改下如下
broker.id=1  2  3
listeners=PLAINTEXT://localhost:9092  localhost:9093   localhost:9094
advertised.listeners
log.dirs=/tmp/kafka-logs1  kafka-logs2   kafka-logs3
kafka9001.properties 信息如下
```
# Licensed to the Apache Software Foundation (ASF) under one or more
# contributor license agreements.  See the NOTICE file distributed with
# this work for additional information regarding copyright ownership.
# The ASF licenses this file to You under the Apache License, Version 2.0
# (the "License"); you may not use this file except in compliance with
# the License.  You may obtain a copy of the License at
#
#    http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# Licensed to the Apache Software Foundation (ASF) under one or more
# contributor license agreements.  See the NOTICE file distributed with
# this work for additional information regarding copyright ownership.
# The ASF licenses this file to You under the Apache License, Version 2.0
# (the "License"); you may not use this file except in compliance with
# the License.  You may obtain a copy of the License at
#
#    http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

# see kafka.server.KafkaConfig for additional details and defaults

############################# Server Basics #############################

# The id of the broker. This must be set to a unique integer for each broker.
broker.id=1

############################# Socket Server Settings #############################

# The address the socket server listens on. It will get the value returned from
# java.net.InetAddress.getCanonicalHostName() if not configured.
#   FORMAT:
#     listeners = listener_name://host_name:port
#   EXAMPLE:
#     listeners = PLAINTEXT://your.host.name:9092
listeners=PLAINTEXT://114.67.171.251:9093

# Hostname and port the broker will advertise to producers and consumers. If not set,
# it uses the value for "listeners" if configured.  Otherwise, it will use the value
# returned from java.net.InetAddress.getCanonicalHostName().
advertised.listeners=PLAINTEXT://114.67.171.251:9093

# Maps listener names to security protocols, the default is for them to be the same. See the config documentation for more details
#listener.security.protocol.map=PLAINTEXT:PLAINTEXT,SSL:SSL,SASL_PLAINTEXT:SASL_PLAINTEXT,SASL_SSL:SASL_SSL

# The number of threads that the server uses for receiving requests from the network and sending responses to the network
num.network.threads=3

# The number of threads that the server uses for processing requests, which may include disk I/O
num.io.threads=8

# The send buffer (SO_SNDBUF) used by the socket server
socket.send.buffer.bytes=102400

# The receive buffer (SO_RCVBUF) used by the socket server
socket.receive.buffer.bytes=102400

# The maximum size of a request that the socket server will accept (protection against OOM)
socket.request.max.bytes=104857600

############################# Log Basics #############################

# A comma separated list of directories under which to store log files
log.dirs=/tmp/kafka-logs1

# The default number of log partitions per topic. More partitions allow greater
# parallelism for consumption, but this will also result in more files across
# the brokers.
num.partitions=1
```
### kafka listeners 和 advertised.listeners 的区别及应用
（记得修改配置，不然代码服务会报错：could not be established. Broker may not be available）
* listeners: 学名叫监听器，其实就是告诉外部连接者要通过什么协议访问指定主机名和端口开放的 Kafka 服务。
* advertised.listeners：和 listener相比多了个
* advertised。Advertise的含义表示宣称的、公布的，就是组监听器是 Broker 用于对外发布的。
#### 只有内网
比如在公司搭建的 kafka 集群，只有内网中的服务可以用，这种情况下，只需要用 listeners 就行
```
listeners=<协议名称>://<内网ip>:<端口>
listeners=SASL_PLAINTEXT://192.168.0.4:9092
```

#### 内外网
在 docker 中或者 在类似阿里云主机上部署 kafka 集群，这种情况下是 需要用到 advertised_listeners。以 docker 为例
```
listeners=INSIDE://0.0.0.0:9092,OUTSIDE://<公网 ip>:端口(或者 0.0.0.0:端口)
advertised.listeners=INSIDE://localhost:9092,OUTSIDE://<宿主机ip>:<宿主机暴露的端口>
listener.security.protocol.map=INSIDE:SASL_PLAINTEXT,OUTSIDE:SASL_PLAINTEXT
kafka_inter_broker_listener_name:inter.broker.listener.name=INSIDE
```
总结：advertised_listeners 是对外暴露的服务端口，真正建立连接用的是 listeners


### 启动三个kafka
```
./bin/kafka-server-start.sh kafka9001.properties
./bin/kafka-server-start.sh kafka9002.properties
./bin/kafka-server-start.sh kafka9003.properties
```
jps查看信息
```
[root@vm-JZ7A72FWtFRk ~]# jps
6705 Bootstrap
17173 Kafka
16712 Kafka
17450 jar
18026 Kafka
15021 Bootstrap
7710 QuorumPeerMain
18479 Jps
[root@vm-JZ7A72FWtFRk ~]# ^C
[root@vm-JZ7A72FWtFRk ~]# ^C
[root@vm-JZ7A72FWtFRk ~]#
```

### 创建topic
```
[root@vm-JZ7A72FWtFRk ~]# cd /root/myapp/kafka/kafka_2.13-2.7.0/
[root@vm-JZ7A72FWtFRk kafka_2.13-2.7.0]# bin/kafka-topics.sh --zookeeper localhost:2181 --create --topic test32 --partitions 3 --replication-factor 2
[root@vm-JZ7A72FWtFRk kafka_2.13-2.7.0]# bin/kafka-topics.sh --zookeeper localhost:2181 --describe --topic test32
Topic: test32	PartitionCount: 3	ReplicationFactor: 2	Configs:
	Topic: test32	Partition: 0	Leader: 1	Replicas: 1,2	Isr: 1,2
	Topic: test32	Partition: 1	Leader: 2	Replicas: 2,3	Isr: 2,3
	Topic: test32	Partition: 2	Leader: 3	Replicas: 3,1	Isr: 3,1

//解析
三个parition在三个不同机器上，Leader代表Broker的id
Topic: test32	Partition: 0 （分区）	Leader: 1 （Broker的id localhost:9092） Replicas: 1,2 （Partition0在Broker1 和Broker2上）
                                                                               Isr: 1,2 (Isr同步的副本，高可用，选举，每一个Partition0副本在其他节点Broker有一个是Leader其他不是Leader)
Topic: test32	Partition: 1 （分区）	Leader: 2 （Broker的id localhost:9093）	Replicas: 2,3 （Partition1在Broker2 和Broker3上）	Isr: 2,3
Topic: test32	Partition: 2 （分区）	Leader: 3 （Broker的id localhost:9094）	Replicas: 3,1 （Partition2在Broker3 和Broker1上）    Isr: 3,1

Isr这个是重点，后续还需要深入分析
```

```
bin/kafka-console-producer.sh --bootstrap-server localhost:9003 --topic test32
bin/kafka-console-consumer.sh --bootstrap-server localhost:9001 --topic test32 --from-beginning
```