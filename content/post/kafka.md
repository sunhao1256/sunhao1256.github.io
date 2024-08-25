---
title: "Kafka"
date: 2024-07-04T10:18:22+08:00
tags: ["middleware"]
---

- difference between kafka and rabbitmq

  > reference https://tunghatbh.medium.com/kafka-vs-rabbitmq-a0ad83d0a820

  - rabbtimq used to delivery message between different application, it can make sure message recevied. just like post office
  - kafka use pull mechanism , so it doesn't guarantee the message whether consumed. but it supply durable ability , so consumer can re receive message via offset . hence, kafa just like book store.

  - Kafka is made of topics, not queues (unlike RabbitMQ, ActiveMQ, SQS). Queues are meant to scale to millions of consumers and to delete messages once processed. In Kafka data is not deleted once processed and consumers cannot scale beyond the number of partitions in a topic.

- kafka的核心组件相互直接的关系与作用

  - broker，kafka的集群实例单位。具体的运行的服务。
  - topic，业务逻辑上的主题。比如 order topic
  - partition ， 分区，一个topic会被分为多个partition，partition具体到系统中是文件，例如my-topic-0。
    - 为什么要分partition，topic分到多个broker上不就已经实现了灾备吗？
      - 首先partition内部是顺序的，因此并发去写入的话，一定会串行。而kafka的partition对于读写是使用的读写锁。如果partition部分，只有一个的话，partition本质上是一个log文件，时间久了文件过大，io性能会有压力，在日志追加、读取或者日志压缩的时候，此外灾备的效率也会低，因为备份一个大文件和备份多个小文件比起来当然不一样。此外，为了提高并发能力，将partition分割成一个一个独立的partition log是可以提高并发的，因为读写锁只是针对单个log文件，这样极大的提高了并发的读写能力。
  - producer，客户端之一，发消息的
  - consumer，消费者之一，收消息的
  - consumer group

- kafka为什么用日志文件来存储具体的partition

- kafka的consumer group是什么？怎么实现的？

- kafka如何保证消息顺序？消息不丢失？消息不重复消费 ？

- kafka为什么吞吐高延迟又低

- kafka是怎么实现的事务

- zookeeper在kafka中扮演什么角色，做了哪些事情，与kRaft模式的区别

- kafka说的这个日志是什么，随机写入和顺序写入的区别？java的RandomAccessFile 干什么的，java的NIO包又是干什么的？

- kafka的集群架构是什么样子的？

- 如何构建一个kafka集群

- 如何加入一个broker

- kafkaController干嘛的
  
  - 是一个角色
  
  - 在源码中的一个类。
  
  - 也是维护整个cluster的节点，通过zookeeper去选举。当client send a request的时候，
  
    eg: ./kafka-topics.sh --bootstrap-server localhost:9092  --topic demo1 --describe , localhost:9092只是一个broker，并不一定是kafkaController。
  
    也就是说如果我有个2个broker，一个8082和一个9092的话，8082的default.replication.factor=3，而9092的default.replication.factor=1的话，而8082是kafkaController的话 ，我这条command，会创建失败，因为整个集群里只有2个broker，不满足3。但是如果我的8082挂了，我执行这个命令，9092会成为kafkaController，从而创建一个replication.factor=1的topic
  
  - partition选举的leader

  - 在BrokerServer启动的是即startup方法里，会创建KafkaController
  
    ```scala
            /* start kafka controller */ //开启kafka controller
            _kafkaController = new KafkaController(config, zkClient, time, metrics, brokerInfo, brokerEpoch, tokenManager, brokerFeatures, metadataCache, threadNamePrefix)
            kafkaController.startup()
    ```
  
    kafkaController的startup里
  
    ```scala
      def startup(): Unit = {
        zkClient.registerStateChangeHandler(new StateChangeHandler {
          override val name: String = StateChangeHandlers.ControllerHandler
          override def afterInitializingSession(): Unit = {
            eventManager.put(RegisterBrokerAndReelect)
          }
          override def beforeInitializingSession(): Unit = {
            val queuedEvent = eventManager.clearAndPut(Expire)
    
            // Block initialization of the new session until the expiration event is being handled,
            // which ensures that all pending events have been processed before creating the new session
            queuedEvent.awaitProcessing()
          }
        })
        eventManager.put(Startup) //扔一个启动的event
        eventManager.start() // 线程启动处理event
      }
    
    ```
  
    ```scala
      private def processStartup(): Unit = {
        zkClient.registerZNodeChangeHandlerAndCheckExistence(controllerChangeHandler)
        elect()
      }
      private def elect(): Unit = {
        activeControllerId = zkClient.getControllerId.getOrElse(-1) //第一步就是去zookeeper获取/controller里有没有内容的，有的话，则controller已经是别的节点了
        /*
         * We can get here during the initial startup and the handleDeleted ZK callback. Because of the potential race condition,
         * it's possible that the controller has already been elected when we get here. This check will prevent the following
         * createEphemeralPath method from getting into an infinite loop if this broker is already the controller.
         */
        if (activeControllerId != -1) {
          debug(s"Broker $activeControllerId has been elected as the controller, so stopping the election process.")
          return
        }
    
        try {
          val (epoch, epochZkVersion) = zkClient.registerControllerAndIncrementControllerEpoch(config.brokerId)
          controllerContext.epoch = epoch
          controllerContext.epochZkVersion = epochZkVersion
          activeControllerId = config.brokerId
    
          info(s"${config.brokerId} successfully elected as the controller. Epoch incremented to ${controllerContext.epoch} " +
            s"and epoch zk version is now ${controllerContext.epochZkVersion}")
    
          onControllerFailover() //通知所有的broker，更新controller的信息,实际是通过ControllerBrokerRequestBatch这个去发送的，他给每个broker都创建了独立的ControllerBrokerRequestBatch，里面维护一个线程去接queue里的内容，kafka几乎所有的异步都是这么玩的。
        } catch {
          case e: ControllerMovedException =>
            maybeResign()
    
            if (activeControllerId != -1)
              debug(s"Broker $activeControllerId was elected as controller instead of broker ${config.brokerId}", e)
            else
              warn("A controller has been elected but just resigned, this will result in another round of election", e)
          case t: Throwable =>
            error(s"Error while electing or becoming controller on broker ${config.brokerId}. " +
              s"Trigger controller movement immediately", t)
            triggerControllerMove()
        }
      }
    
    
    ```
  
    在kafkaController被创建完成，或者后续有变更的时候，都会通知给所有的broker，并且api协议用的枚举是UPDATE_METADATA
  
    ```java
        public UpdateMetadataRequest(UpdateMetadataRequestData data, short version) {
            super(ApiKeys.UPDATE_METADATA, version);
            this.data = data;
            // Do this from the constructor to make it thread-safe (even though it's only needed when some methods are called)
            normalize();
        }
    
    ```
  
    所以每个broker都会有KafkaApis这个类来接受的
  
- 如何提高kafka的消费能力？

  增加topic的partition个数

- 如何配置topic的replication.factor

  broker的server.property可以配置

  ```properties
  default.replication.factor = 1  //默认是1
  ```

  - 如果是一个kafka cluster的话，

  kafka提供的command tool的话也有创建参数

  ```shell
  ./kafka-topics.sh --bootstrap-server localhost:8092  --topic demo1 --create --replication-factor 3
  ```

  client肯定也可以
