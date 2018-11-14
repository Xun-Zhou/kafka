#kafka概述

    Kakfa是由LinkedIn公司开发的一个分布式的消息系统，后成为Apache顶级开源项目，它使用Scala编写，以可水平扩展和高吞吐率的特性而被广泛使用。
##Kafka体系架构

![Kafka体系架构](https://github.com/Xun-Zhou/kafka/blob/master/introduce/kafka-apis.png "Kafka体系架构")

    如上图所示，一个典型的Kafka体系架构包括若干Producer，Kafka集群，若干Consumer，以及一个Zookeeper集群（高版本的kafka自带zookeeper）。
    Kafka通过Zookeeper管理集群配置，选举leader，以及在consumer group发生变化时进行rebalance。
    Producer使用push模式将消息发布到broker，Consumer使用pull模式从broker订阅并消费消息（传统的JMS和AMQP由server将消息push到Consumer）。
- 名词解释
    
      Broker:kafka集群节点,一个节点就是一个broker
      Topic:kafka根据topic进行消息分类,发布消息到kafka需要指定一个topic
      Partition:一个topic可以分为多个partition,发布消息通过kafka负载均衡算法指定发送到哪个partition
      Producer:消息生产者
      Consumer:消息消费者
      ConsumerGroup:每个consumer都属于一个特定的consumerGroup,每个consumerGroup都能获取到topic下的所有partition中的消息，
      一个partition中的一条消息只会被consumerGroup中的一个consumer消费,
      partition到group类似广播，group到consumer类似轮询
##Topic & Partition

    一个topic可以看做一类消息，每个topic可以分成多个partition，每个partition在存储层面是append log文件。
    任何发布到partition的消息都会被追加到log文件的尾部，每条消息在文件中的位置称为offset(偏移量)，offset为一个long型的数字，它唯一标记一条消息。
    每条消息都被append到partition中，是线性写磁盘，因此效率非常高(经验证，线性写磁盘效率比随机写内存要高，这是Kafka高吞吐率的一个很重要的保证)。

![KafkaTopic & Partition](https://github.com/Xun-Zhou/kafka/blob/master/introduce/log_anatomy.png "KafkaTopic & Partition")

    每一条消息被发送到broker中，会根据partition规则选择被存储到哪一个partition。
    如果partition规则设置的合理，所有消息可以均匀分布到不同的partition里，这样就实现了水平扩展。
    (如果一个topic对应一个日志文件，那这个文件所在的机器I/O将会成为这个topic的性能瓶颈，而partition解决了这个问题)。
    在创建topic时可以在/config/server.properties中配置(num.partitions)中指定这个partition的数量，当然也可以在topic创建之后去修改partition的数量。
##高可靠性存储分析

    Kafka的高可靠性的保障来源于其健壮的副本(replication)策略。通过调节其副本相关参数，可以使得Kafka在性能和可靠性之间平衡。
    Kafka从0.8.x版本开始提供partition级别的复制,replication的数量可以在/config/server.properties中配置(default.replication.refactor)。
- Kafka文件存储机制

      Kafka中消息以topic进行分类，producer向topic发送消息，消费者通过topic获取消息。
      topic在物理层面又能分为多个partition，partition还可以细分为segment，一个partition物理上由多个segment组成。
      假设有一个Kafka集群，这个集群只有一个Kafka broker，即只有一台物理机。
      配置/config/server.properties(log.dirs=/tmp/kafka-logs)，设置Kafka消息文件存储目录，
      创建一个topic：my_test，partition的数量为3(现实中只有一个broker是不能创建多个partition的):
      /bin/kafka-topics.sh --create --zookeeper localhost:2181 --topic my_test --partitions 3 --replication-factor 3。
      此时可以在/tmp/kafka-logs目录中可以看到生成了3个目录：
      ----------------------------------------
      my_test-0
      my_test-1
      my_test-2
      ----------------------------------------
      一个topic下有多个的partition，每个partiton为一个目录，partition的名称规则为：topic名称+有序序号，第一个序号从0开始计，最大的序号为partition数量减1，
      partition是实际物理上的概念，而topic是逻辑上的概念
      
      一个partition物理层面由多个相等的segment组成，相当于一个巨型文件被平均分配到多个大小相等的segment，这种特性方便与old segment的删除(清理过期消息)，partition只支持顺序读写
      segment文件由两部分组成，“.index”文件和“.log”文件，分别表示为segment索引文件和数据文件。
      这两个文件的命令规则为：partition全局的第一个segment从0开始，后续每个segment文件名为上一个segment文件最后一条消息的offset值，数值大小为64位，20位数字字符长度，没有数字用0填充
      ----------------------------------------
      00000000000000000000.index
      00000000000000000000.log
      00000000000000170410.index
      00000000000000170410.log
      00000000000000239430.index
      00000000000000239430.log
      ----------------------------------------
      
     ![segment](https://github.com/Xun-Zhou/kafka/blob/master/introduce/segment.png "segment")
      
      “.index”索引文件存储大量的元数据，“.log”数据文件存储大量的消息，索引文件中的元数据指向对应数据文件中message的物理偏移地址。
      以“.index”索引文件中的元数据[3, 348]为例，在“.log”数据文件表示第3个消息，即在全局partition中表示170410+3=170413个消息，该消息的物理偏移地址为348。

    