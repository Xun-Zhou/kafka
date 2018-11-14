# kafka概述

    Kakfa是由LinkedIn公司开发的一个分布式的消息系统，后成为Apache顶级开源项目，它使用Scala编写，以可水平扩展和高吞吐率的特性而被广泛使用。
## Kafka体系架构

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
## Topic & Partition

    一个topic可以看做一类消息，每个topic可以分成多个partition，每个partition在存储层面是append log文件。
    任何发布到partition的消息都会被追加到log文件的尾部，每条消息在文件中的位置称为offset(偏移量)，offset为一个long型的数字，它唯一标记一条消息。
    每条消息都被append到partition中，是线性写磁盘，因此效率非常高(经验证，线性写磁盘效率比随机写内存要高，这是Kafka高吞吐率的一个很重要的保证)。

![KafkaTopic & Partition](https://github.com/Xun-Zhou/kafka/blob/master/introduce/log_anatomy.png "KafkaTopic & Partition")

    每一条消息被发送到broker中，会根据partition规则选择被存储到哪一个partition。
    如果partition规则设置的合理，所有消息可以均匀分布到不同的partition里，这样就实现了水平扩展。
    (如果一个topic对应一个日志文件，那这个文件所在的机器I/O将会成为这个topic的性能瓶颈，而partition解决了这个问题)。
    在创建topic时可以在/config/server.properties中配置(num.partitions)中指定这个partition的数量，当然也可以在topic创建之后去修改partition的数量。
## 高可靠性存储分析

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
      以“.index”索引文件中的元数据[3, 348]为例，在“.log”数据文件表示第3个消息，在全局partition中表示第170410+3=170413个消息，该消息的物理偏移地址为348。
      
      读取消息：以上图为例，读取offset=170418的消息，首先查找segment文件，
      其中00000000000000000000.index为最开始的文件，
      第二个文件为00000000000000170410.index(起始偏移为170410+1=170411)，
      第三个文件为00000000000000239430.index(起始偏移为239430+1=239431)，
      所以offset=170418在第二个文件之中。
      其次根据00000000000000170410.index文件中的[8,1325]定位到00000000000000170410.log文件中的1325的位置进行读取。
      消息都具有固定的物理结构，包括：offset(8 Bytes)、消息体的大小(4 Bytes)等字段，可以确定一条消息的大小，即读取到哪里截止，判断消息是否读完。
- 复制原理和同步方式
      
      topic的每个partition有一个预写式的日志文件，虽然partition可以继续细分为若干个segment文件，但是对于上层应用来说可以将partition看成最小的存储单元，
      每个partition都由一些列有序的、不可变的消息组成，这些消息被连续的追加到partition中。

![partition_log](https://github.com/Xun-Zhou/kafka/blob/master/introduce/partition_log.png "partition_log")

      LEO:LogEndOffset表示每个partition的log最后一条Message的位置
      HW:每个replica都有HW,leader和follower各自负责更新自己的HW的状态。
      对于leader新写入的消息，consumer不能立刻消费，leader会等待该消息被所有的flower同步后更新HW，此时消息才能被consumer消费。
      这样就保证了如果leader所在的broker失效，该消息仍然可以从新选举的leader中获取。对于来自内部broKer的读取请求，没有HW的限制。
      flower在做同步操作的时候，先将log文件截断到之前自己的HW的位置，之后再从leader中拉取消息进行同步。
      
      为了提高消息的可靠性，Kafka每个topic的partition有N个副本(replicas)，其中N(大于等于1)是topic的复制因子(replica fator)的个数。
      Kafka通过多副本机制实现故障自动转移，当Kafka集群中一个broker失效情况下仍能保证服务可用。
      在Kafka中发生复制时确保partition的日志能有序地写到其他节点上，N个replicas中，其中一个replica为leader，其他都为follower, 
      leader处理partition的所有读写请求，与此同时，follower会被动定期地去复制leader上的数据。
      
      ISR:副本同步队列
      副本数对Kafka的吞吐率是有一定的影响，但极大的增强了可用性。默认情况下Kafka的replica数量为1，即每个partition都有一个唯一的leader，为了确保消息的可靠性，
      通常应用中将其值(bin/server.server.properties offsets.topic.replication.factor指定)大小设置为大于1。 
      所有的副本(replicas)统称为Assigned Replicas，即AR。ISR是AR中的一个子集，由leader维护ISR列表(follower同步leader完成写入ISR)，
      follower从leader同步数据有一些延迟
      (包括延迟时间replica.lag.time.max.ms和延迟条数replica.lag.max.messages两个维度, 当前最新的版本0.10.x中只支持replica.lag.time.max.ms这个维度)，
      超过任意一个阈值都会把follower剔除出ISR, 存入OSR(Outof-Sync Replicas)列表，新加入的follower也会先存放在OSR中。
      AR=ISR+OSR。ISR中包括：leader和follower。Kafka的ISR的管理最终都会反馈到Zookeeper节点上。
- 数据可靠性和持久性保证
      
      1(默认)：producer发送消息leader成功接收立即返回确认，producer收到确认后发送下一条message。如果leader宕机了，则会丢失数据(follower未复制消息)。
      0：这意味着producer无需等待来自broker的确认而继续发送下一批消息。这种情况下数据传输效率最高，但是数据可靠性确是最低的。
      -1：producer需要等待所有follower都确认接收到数据后才算一次发送完成，可靠性最高，没有follow的情况下就成了默认的情况。
      
     - request.required.acks=1
         
      producer发送数据到leader，leader写本地日志成功，返回客户端成功;此时follow还没有同步该消息，leader就宕机了，那么此次发送的消息就会丢失。
      
     - request.required.acks=-1
     
      同步(Kafka默认为同步，即producer.type=sync)的发送模式，replication.factor>=2且min.insync.replicas>=2的情况下，不会丢失数据。
      有两种典型情况：
      1.数据发送到leader, follower全部完成数据同步后，leader此时挂掉，那么会选举出新的leader，数据不会丢失。
      2.数据发送到leader后 ，部分follow同步，leader此时挂掉。比如follower1和follower2都有可能变成新的leader, producer端会得到返回异常，producer端会重新发送数据，数据可能会重复。
- Leader选举

      Kafka在Zookeeper中为每一个partition动态的维护了一个ISR，这个ISR里的所有replica都跟上了leader，
      只有ISR里的成员才能有被选为leader的可能。
      在这种模式下，对于f+1个副本，一个Kafka topic能在保证不丢失已经commit消息的前提下容忍f个副本的失败，在大多数使用场景下，这种模式是十分有利的。
        
      当partition的所有replica都挂了，就无法保证数据不丢失了。这种情况下有两种可行的方案：
      1.unclean.leader.election.enable=false
      等待ISR中任意一个replica“活”过来，并且选它作为leader
      2.unclean.leader.election.enable=true
      选择第一个“活”过来的replica(并不一定是在ISR中)作为leader
      Kafka默认采用第二种策略

      实例分析：
      假设某个partition中的flower为3，replica-0, replica-1, replica-2分别存放在broker0, broker1和broker2中。AR=(0,1,2)，ISR=(0,1)
      设置request.required.acks=-1, min.insync.replicas=2，unclean.leader.election.enable=false
      当ISR中的replica-0出现crash的情况时，broker1选举为新的leader[ISR=(1)]，因为受min.insync.replicas=2影响，write不能服务，但是read能继续正常服务。
      此种情况恢复方案：
            1.尝试恢复replica-0，如果能起来，系统正常;
            2.如果replica-0不能恢复，需要将min.insync.replicas设置为1，恢复write功能。