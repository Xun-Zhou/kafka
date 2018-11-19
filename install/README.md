# kafka安装

- 下载kafka

    [下载kafka](http://kafka.apache.org/downloads)选择二进制下载，安装环境为centos7，kafka需要java环境不要忘记安装jdk

    解压文件
    > tar -xzf kafka_2.11-2.0.0.tgz
    > cd kafka_2.11-2.0.0
- 启动服务器

    Kafka使用ZooKeeper，因此没有ZooKeeper服务器，可以使用kafka自带的单节点ZooKeeper实例(生产中不推荐使用)。

    启动zookeeper服务
    > bin/zookeeper-server-start.sh config/zookeeper.properties
    
    启动kafka服务
    > bin/kafka-server-start.sh config/server.properties
- 创建主题

    创建名为test的主题，只有一个分区，一个副本
    > bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic test

    list topic命令，可以看到该主题列表：
    > bin/kafka-topics.sh --list --zookeeper localhost:2181
   
    可以配置为发布消息主题不存在时自动创建主题，而不是手动创建主题。
- 创建生产者

    Kafka附带一个命令行客户端，它将从文件或标准输入中获取输入，并将其作为消息发送到Kafka集群。默认情况下，每行将作为单独的消息发送。
    创建生产者，发送消息：
    > bin/kafka-console-producer.sh --broker-list localhost:9092 --topic test
    
    > This is a message
    
    > This is another message
- 创建消费者
    
    Kafka还有一个命令行客户端，它会将消息转储到标准输出。
    > bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic test --from-beginning
    
    > This is a message
    
    > This is another message
- kafka集群
    
    kafka使用zookeeper，只要zookeeper的地址相同，在不同机器上启动kafka实例，都能组成集群，资源有限，这里使用单机集群，3个kafka节点
    
    拷贝配置文件
    
    > cp config/server.properties config/server-1.properties
    
    > cp config/server.properties config/server-2.properties
    
    修改配置文件，自改broker.id，log文件保存地址和服务端口
    
    config/server-1.properties:
          
        broker.id=1
        listeners=PLAINTEXT://:9093
        log.dirs=/tmp/kafka-logs-1
        
    config/server-2.properties:
    
        broker.id=2
        listeners=PLAINTEXT://:9094
        log.dirs=/tmp/kafka-logs-2
        
    启动节点(&启动后后台运行)
    
    > bin/kafka-server-start.sh config/server-1.properties &

    > bin/kafka-server-start.sh config/server-2.properties &

    创建一个副本为3的topic
    
    > bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 3 --partitions 1 --topic my-replicated-topic
    
    使用命令describe topics查看topic运行情况
    
    > bin/kafka-topics.sh --describe --zookeeper localhost:2181 --topic my-replicated-topic
    
        Topic:my-replicated-topic   PartitionCount:1    ReplicationFactor:3 Configs:
            Topic: my-replicated-topic  Partition: 0    Leader: 1   Replicas: 1,2,0 Isr: 1,2,0
    
    第一行给出了所有分区的摘要，每一行提供有关一个分区的信息。由于我们只有一个分区用于此主题，因此只有一行。
    
        leader：分区中负责所有读取和写入的节点
        replicas：分区中负责同步leader的节点
        isr：所有完成同步的节点集(包括leader和flower)，是副本列表的子集
    