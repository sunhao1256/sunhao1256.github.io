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

# kafkaController

- 所有的controller发生变更的时候，也会通知给所有的broker

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
  //kafkaController.scala 
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
        //把自己注册为controller
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

所以每个broker都会有KafkaApis这个类来接受的，并且将kafkaController的信息封装在一个ControllerNodeProvider，zk模式下则是MetadataCacheControllerNodeProvider。

```scala
        metadataCache = MetadataCache.zkMetadataCache( //元数据存储，保存例如partition等信息，在内存
          config.brokerId,
          config.interBrokerProtocolVersion,
          brokerFeatures,
          config.migrationEnabled)
//provider实际上是把上面的metadataCache传下去的。
        val controllerNodeProvider = new MetadataCacheControllerNodeProvider(metadataCache, config,
          () => Option(quorumControllerNodeProvider).map(_.getControllerInfo()))


```

而MetadataCache会在KafkaAPis接受到UPDATE_METADATA的时候进行更新

```scala
      request.header.apiKey match {
        case ApiKeys.PRODUCE => handleProduceRequest(request, requestLocal)
        case ApiKeys.FETCH => handleFetchRequest(request)
        case ApiKeys.LIST_OFFSETS => handleListOffsetRequest(request)
        case ApiKeys.METADATA => handleTopicMetadataRequest(request) //更新metadata
        case ApiKeys.LEADER_AND_ISR => handleLeaderAndIsrRequest(request)
        case ApiKeys.STOP_REPLICA => handleStopReplicaRequest(request)
        case ApiKeys.UPDATE_METADATA => handleUpdateMetadataRequest(request, requestLocal)
        case ApiKeys.CONTROLLED_SHUTDOWN => handleControlledShutdownRequest(request)
        
        
   def handleUpdateMetadataRequest(request: RequestChannel.Request, requestLocal: RequestLocal): Unit = {
    val zkSupport = metadataSupport.requireZkOrThrow(KafkaApis.shouldNeverReceive(request))
    val correlationId = request.header.correlationId
    val updateMetadataRequest = request.body[UpdateMetadataRequest]

    authHelper.authorizeClusterOperation(request, CLUSTER_ACTION)
    if (zkSupport.isBrokerEpochStale(updateMetadataRequest.brokerEpoch, updateMetadataRequest.isKRaftController)) {
      // When the broker restarts very quickly, it is possible for this broker to receive request intended
      // for its previous generation so the broker should skip the stale request.
      info(s"Received UpdateMetadata request with broker epoch ${updateMetadataRequest.brokerEpoch} " +
        s"smaller than the current broker epoch ${zkSupport.controller.brokerEpoch} from " +
        s"controller ${updateMetadataRequest.controllerId} with epoch ${updateMetadataRequest.controllerEpoch}.")
      requestHelper.sendResponseExemptThrottle(request,
        new UpdateMetadataResponse(new UpdateMetadataResponseData().setErrorCode(Errors.STALE_BROKER_EPOCH.code)))
    } else {
      //在这里更新maybeUpdateMetadataCache
      val deletedPartitions = replicaManager.maybeUpdateMetadataCache(correlationId, updateMetadataRequest)
      if (deletedPartitions.nonEmpty) {
        groupCoordinator.onPartitionsDeleted(deletedPartitions.asJava, requestLocal.bufferSupplier)
      }

      if (zkSupport.adminManager.hasDelayedTopicOperations) {
        updateMetadataRequest.partitionStates.forEach { partitionState =>
          zkSupport.adminManager.tryCompleteDelayedTopicOperations(partitionState.topicName)
        }
      }

      quotas.clientQuotaCallback.foreach { callback =>
        if (callback.updateClusterMetadata(metadataCache.getClusterMetadata(clusterId, request.context.listenerName))) {
          quotas.fetch.updateQuotaMetricConfigs()
          quotas.produce.updateQuotaMetricConfigs()
          quotas.request.updateQuotaMetricConfigs()
          quotas.controllerMutation.updateQuotaMetricConfigs()
        }
      }
      if (replicaManager.hasDelayedElectionOperations) {
        updateMetadataRequest.partitionStates.forEach { partitionState =>
          val tp = new TopicPartition(partitionState.topicName, partitionState.partitionIndex)
          replicaManager.tryCompleteElection(TopicPartitionOperationKey(tp))
        }
      }
      requestHelper.sendResponseExemptThrottle(request, new UpdateMetadataResponse(
        new UpdateMetadataResponseData().setErrorCode(Errors.NONE.code)))
    }
  }
        
          def maybeUpdateMetadataCache(correlationId: Int, updateMetadataRequest: UpdateMetadataRequest) : Seq[TopicPartition] =  {
    replicaStateChangeLock synchronized {
      if (updateMetadataRequest.controllerEpoch < controllerEpoch) {
        val stateControllerEpochErrorMessage = s"Received update metadata request with correlation id $correlationId " +
          s"from an old controller ${updateMetadataRequest.controllerId} with epoch ${updateMetadataRequest.controllerEpoch}. " +
          s"Latest known controller epoch is $controllerEpoch"
        stateChangeLogger.warn(stateControllerEpochErrorMessage)
        throw new ControllerMovedException(stateChangeLogger.messageWithPrefix(stateControllerEpochErrorMessage))
      } else {
        val zkMetadataCache = metadataCache.asInstanceOf[ZkMetadataCache]
        //调用metadataCache的updateMetadata
        val deletedPartitions = zkMetadataCache.updateMetadata(correlationId, updateMetadataRequest)
        controllerEpoch = updateMetadataRequest.controllerEpoch
        deletedPartitions
      }
    }
  }

        
  def updateMetadata(
    correlationId: Int,
    originalUpdateMetadataRequest: UpdateMetadataRequest
  ): Seq[TopicPartition] = {
    var updateMetadataRequest = originalUpdateMetadataRequest
    //加锁
    inWriteLock(partitionMetadataLock) {
      if (
        updateMetadataRequest.isKRaftController &&
        updateMetadataRequest.updateType() == AbstractControlRequest.Type.FULL
      ) {
        if (updateMetadataRequest.version() < 8) {
          stateChangeLogger.error(s"Received UpdateMetadataRequest with Type=FULL (2), but version of " +
            updateMetadataRequest.version() + ", which should not be possible. Not treating this as a full " +
            "metadata update")
        } else if (!zkMigrationEnabled) {
          stateChangeLogger.error(s"Received UpdateMetadataRequest with Type=FULL (2), but ZK migrations " +
            s"are not enabled on this broker. Not treating this as a full metadata update")
        } else {
          // When handling a UMR from a KRaft controller, we may have to insert some partition
          // deletions at the beginning, to handle the different way topic deletion works in KRaft
          // mode (and also migration mode).
          //
          // After we've done that, we re-create the whole UpdateMetadataRequest object using the
          // updated list of topic info. This ensures that UpdateMetadataRequest.normalize is called
          // on the new, updated topic data. Note that we don't mutate the old request object; it may
          // be used elsewhere.
          val newTopicStates = ZkMetadataCache.transformKRaftControllerFullMetadataRequest(
            metadataSnapshot,
            updateMetadataRequest.controllerEpoch(),
            updateMetadataRequest.topicStates(),
            logMessage => if (logMessage.startsWith("Error")) {
              stateChangeLogger.error(logMessage)
            } else {
              stateChangeLogger.info(logMessage)
            })

          // It would be nice if we could call duplicate() here, but we don't want to copy the
          // old topicStates array. That would be quite costly, and we're not going to use it anyway.
          // Instead, we copy each field that we need.
          val originalRequestData = updateMetadataRequest.data()
          val newData = new UpdateMetadataRequestData().
            setControllerId(originalRequestData.controllerId()).
            setIsKRaftController(originalRequestData.isKRaftController).
            setType(originalRequestData.`type`()).
            setControllerEpoch(originalRequestData.controllerEpoch()).
            setBrokerEpoch(originalRequestData.brokerEpoch()).
            setTopicStates(newTopicStates).
            setLiveBrokers(originalRequestData.liveBrokers())
          updateMetadataRequest = new UpdateMetadataRequest(newData, updateMetadataRequest.version())
        }
      }

      val aliveBrokers = new mutable.LongMap[Broker](metadataSnapshot.aliveBrokers.size)
      val aliveNodes = new mutable.LongMap[collection.Map[ListenerName, Node]](metadataSnapshot.aliveNodes.size)
      val controllerIdOpt: Option[CachedControllerId] = updateMetadataRequest.controllerId match {
        case id if id < 0 => None
        case id =>
          if (updateMetadataRequest.isKRaftController)
            Some(KRaftCachedControllerId(id))
          else
            Some(ZkCachedControllerId(id))
      }

      updateMetadataRequest.liveBrokers.forEach { broker =>
        // `aliveNodes` is a hot path for metadata requests for large clusters, so we use java.util.HashMap which
        // is a bit faster than scala.collection.mutable.HashMap. When we drop support for Scala 2.10, we could
        // move to `AnyRefMap`, which has comparable performance.
        val nodes = new java.util.HashMap[ListenerName, Node]
        val endPoints = new mutable.ArrayBuffer[EndPoint]
        broker.endpoints.forEach { ep =>
          val listenerName = new ListenerName(ep.listener)
          endPoints += new EndPoint(ep.host, ep.port, listenerName, SecurityProtocol.forId(ep.securityProtocol))
          nodes.put(listenerName, new Node(broker.id, ep.host, ep.port, broker.rack()))
        }
        aliveBrokers(broker.id) = Broker(broker.id, endPoints, Option(broker.rack))
        aliveNodes(broker.id) = nodes.asScala
      }
      aliveNodes.get(brokerId).foreach { listenerMap =>
        val listeners = listenerMap.keySet
        if (!aliveNodes.values.forall(_.keySet == listeners))
          error(s"Listeners are not identical across brokers: $aliveNodes")
      }

      val topicIds = mutable.Map.empty[String, Uuid]
      topicIds ++= metadataSnapshot.topicIds
      val (newTopicIds, newZeroIds) = updateMetadataRequest.topicStates().asScala
        .map(topicState => (topicState.topicName(), topicState.topicId()))
        .partition { case (_, topicId) => topicId != Uuid.ZERO_UUID }
      newZeroIds.foreach { case (zeroIdTopic, _) => topicIds.remove(zeroIdTopic) }
      topicIds ++= newTopicIds.toMap

      val deletedPartitions = new java.util.LinkedHashSet[TopicPartition]
      if (!updateMetadataRequest.partitionStates.iterator.hasNext) {
        //这里更新了
        metadataSnapshot = MetadataSnapshot(metadataSnapshot.partitionStates, topicIds.toMap,
          controllerIdOpt, aliveBrokers, aliveNodes)
      } else {
        //since kafka may do partial metadata updates, we start by copying the previous state
        val partitionStates = new mutable.AnyRefMap[String, mutable.LongMap[UpdateMetadataPartitionState]](metadataSnapshot.partitionStates.size)
        metadataSnapshot.partitionStates.forKeyValue { (topic, oldPartitionStates) =>
          val copy = new mutable.LongMap[UpdateMetadataPartitionState](oldPartitionStates.size)
          copy ++= oldPartitionStates
          partitionStates(topic) = copy
        }

        val traceEnabled = stateChangeLogger.isTraceEnabled
        val controllerId = updateMetadataRequest.controllerId
        val controllerEpoch = updateMetadataRequest.controllerEpoch
        val newStates = updateMetadataRequest.partitionStates.asScala
        newStates.foreach { state =>
          // per-partition logging here can be very expensive due going through all partitions in the cluster
          val tp = new TopicPartition(state.topicName, state.partitionIndex)
          if (state.leader == LeaderAndIsr.LeaderDuringDelete) {
            removePartitionInfo(partitionStates, topicIds, tp.topic, tp.partition)
            if (traceEnabled)
              stateChangeLogger.trace(s"Deleted partition $tp from metadata cache in response to UpdateMetadata " +
                s"request sent by controller $controllerId epoch $controllerEpoch with correlation id $correlationId")
            deletedPartitions.add(tp)
          } else {
            addOrUpdatePartitionInfo(partitionStates, tp.topic, tp.partition, state)
            deletedPartitions.remove(tp)
            if (traceEnabled)
              stateChangeLogger.trace(s"Cached leader info $state for partition $tp in response to " +
                s"UpdateMetadata request sent by controller $controllerId epoch $controllerEpoch with correlation id $correlationId")
          }
        }
        val cachedPartitionsCount = newStates.size - deletedPartitions.size
        stateChangeLogger.info(s"Add $cachedPartitionsCount partitions and deleted ${deletedPartitions.size} partitions from metadata cache " +
          s"in response to UpdateMetadata request sent by controller $controllerId epoch $controllerEpoch with correlation id $correlationId")
//这里也更新了,做了个快照。
        metadataSnapshot = MetadataSnapshot(partitionStates, topicIds.toMap, controllerIdOpt, aliveBrokers, aliveNodes)
      }
      deletedPartitions.asScala.toSeq
    }
  }


```

从上面的代码可以看出流程如下

- 在zk模式下kafka集群所有的broker尝试从zk取path是/controller的节点，如果取不到则把自己作为controller，尝试注册。

- 成功后，controller节点则会通知所有的broker进行更新controller信息。是通过UPDATE_METADATA接口发送给broker

- broker注册了kafkaAPis，用于接受所有的请求，在接受UPDATE_MEATADATA会更新本地的metadataCache视线的ZkMetaCache

- 封装一个controllerNodeProvider

  ```scala
          val controllerNodeProvider = new MetadataCacheControllerNodeProvider(metadataCache, config,
            () => Option(quorumControllerNodeProvider).map(_.getControllerInfo()))
  //为什么不直接使用metadataCache，而要封装一个provider，应该是为了支持多种模式，比如KRaft，面向接口嘛。
  ```

  

- 而broker在后续的操作中都会用到这个metadataCache获取controllerId，比如说NodeToControllerChannelManagerImpl 则是node节点和controller节点交互的封装，里面就使用了

  ```scala
      clientToControllerChannelManager = new NodeToControllerChannelManagerImpl(
        controllerNodeProvider = controllerNodeProvider, //
        time = time,
        metrics = metrics,
        config = config,
        channelName = "forwarding",
        s"zk-broker-${config.nodeId}-",
        retryTimeoutMs = config.requestTimeoutMs.longValue
      )
      clientToControllerChannelManager.start()
  ```

# NodeToControllerChannelManagerImpl

as the name implies，这个就是broker和KafkaController专门交互的东西

```scala
//启动实际上是启动一个requestThread 
def start(): Unit = {
    requestThread.start()
  }

//内置的broker处理请求的
public abstract class InterBrokerSendThread extends ShutdownableThread {

    protected volatile KafkaClient networkClient;

    private final int requestTimeoutMs;
    private final Time time;
    private final UnsentRequests unsentRequests;

    public InterBrokerSendThread(
        String name,
        KafkaClient networkClient,
        int requestTimeoutMs,
        Time time
    ) {
        this(name, networkClient, requestTimeoutMs, time, true);
    }

    public InterBrokerSendThread(
        String name,
        KafkaClient networkClient,
        int requestTimeoutMs,
        Time time,
        boolean isInterruptible
    ) {
        super(name, isInterruptible);
        this.networkClient = networkClient;
        this.requestTimeoutMs = requestTimeoutMs;
        this.time = time;
        this.unsentRequests = new UnsentRequests();
    }

  //子类必须实现的
    public abstract Collection<RequestAndCompletionHandler> generateRequests();

    public boolean hasUnsentRequests() {
        return unsentRequests.iterator().hasNext();
    }

    @Override
    public void shutdown() throws InterruptedException {
        initiateShutdown();
        networkClient.initiateClose();
        awaitShutdown();
        Utils.closeQuietly(networkClient, "InterBrokerSendThread network client");
    }

  //取所有的请求，封装成clientRequest
    private void drainGeneratedRequests() {
        generateRequests().forEach(request ->
            unsentRequests.put(
                request.destination,
                networkClient.newClientRequest(
                    request.destination.idString(),
                    request.request,
                    request.creationTimeMs,
                    true,
                    requestTimeoutMs,
                    request.handler
                )
            )
        );
    }

    protected void pollOnce(long maxTimeoutMs) {
        try {
          //抽光需要发送的请求
            drainGeneratedRequests();
            long now = time.milliseconds();
            final long timeout = sendRequests(now, maxTimeoutMs);
            networkClient.poll(timeout, now);
            now = time.milliseconds();
            checkDisconnects(now);
            failExpiredRequests(now);
            unsentRequests.clean();
        } catch (FatalExitError fee) {
            throw fee;
        } catch (Throwable t) {
            if (t instanceof DisconnectException && !networkClient.active()) {
                // DisconnectException is expected when NetworkClient#initiateClose is called
                return;
            }
            if (t instanceof InterruptedException && !isRunning()) {
                // InterruptedException is expected when shutting down. Throw the error to ShutdownableThread to handle
                throw t;
            }
            log.error("unhandled exception caught in InterBrokerSendThread", t);
            // rethrow any unhandled exceptions as FatalExitError so the JVM will be terminated
            // as we will be in an unknown state with potentially some requests dropped and not
            // being able to make progress. Known and expected Errors should have been appropriately
            // dealt with already.
            throw new FatalExitError();
        }
    }

    @Override //是父类的ShutdownableThread 的方法，无限循环取出发，标准的一个独立工作线程
    public void doWork() {
        pollOnce(Long.MAX_VALUE);
    }

  //真正出发的方法，内部的networkClient实际上又是取调用NIO的selector了。这里就不展开了
    private long sendRequests(long now, long maxTimeoutMs) {
        long pollTimeout = maxTimeoutMs;
        for (Node node : unsentRequests.nodes()) {
            final Iterator<ClientRequest> requestIterator = unsentRequests.requestIterator(node);
            while (requestIterator.hasNext()) {
                final ClientRequest request = requestIterator.next();
                if (networkClient.ready(node, now)) {
                    networkClient.send(request, now);
                    requestIterator.remove();
                } else {
                    pollTimeout = Math.min(pollTimeout, networkClient.connectionDelay(node, now));
                }
            }
        }
        return pollTimeout;
    }

    private void checkDisconnects(long now) {
        // any disconnects affecting requests that have already been transmitted will be handled
        // by NetworkClient, so we just need to check whether connections for any of the unsent
        // requests have been disconnected; if they have, then we complete the corresponding future
        // and set the disconnect flag in the ClientResponse
        final Iterator<Map.Entry<Node, ArrayDeque<ClientRequest>>> iterator = unsentRequests.iterator();
        while (iterator.hasNext()) {
            final Map.Entry<Node, ArrayDeque<ClientRequest>> entry = iterator.next();
            final Node node = entry.getKey();
            final ArrayDeque<ClientRequest> requests = entry.getValue();
            if (!requests.isEmpty() && networkClient.connectionFailed(node)) {
                iterator.remove();
                for (ClientRequest request : requests) {
                    final AuthenticationException authenticationException = networkClient.authenticationException(node);
                    if (authenticationException != null) {
                        log.error("Failed to send the following request due to authentication error: {}", request);
                    }
                    completeWithDisconnect(request, now, authenticationException);
                }
            }
        }
    }

    private void failExpiredRequests(long now) {
        // clear all expired unsent requests
        final Collection<ClientRequest> timedOutRequests = unsentRequests.removeAllTimedOut(now);
        for (ClientRequest request : timedOutRequests) {
            log.debug("Failed to send the following request after {} ms: {}", request.requestTimeoutMs(), request);
            completeWithDisconnect(request, now, null);
        }
    }

    private static void completeWithDisconnect(
        ClientRequest request,
        long now,
        AuthenticationException authenticationException
    ) {
        final RequestCompletionHandler handler = request.callback();
        handler.onComplete(
            new ClientResponse(
                request.makeHeader(request.requestBuilder().latestAllowedVersion()),
                handler,
                request.destination(),
                now /* createdTimeMs */,
                now /* receivedTimeMs */,
                true /* disconnected */,
                null /* versionMismatch */,
                authenticationException,
                null
            )
        );
    }

    public void wakeup() {
        networkClient.wakeup();
    }

    private static final class UnsentRequests {

        private final Map<Node, ArrayDeque<ClientRequest>> unsent = new HashMap<>();

        void put(Node node, ClientRequest request) {
            ArrayDeque<ClientRequest> requests = unsent.computeIfAbsent(node, n -> new ArrayDeque<>());
            requests.add(request);
        }

        Collection<ClientRequest> removeAllTimedOut(long now) {
            final List<ClientRequest> expiredRequests = new ArrayList<>();
            for (ArrayDeque<ClientRequest> requests : unsent.values()) {
                final Iterator<ClientRequest> requestIterator = requests.iterator();
                boolean foundExpiredRequest = false;
                while (requestIterator.hasNext() && !foundExpiredRequest) {
                    final ClientRequest request = requestIterator.next();
                    final long elapsedMs = Math.max(0, now - request.createdTimeMs());
                    if (elapsedMs > request.requestTimeoutMs()) {
                        expiredRequests.add(request);
                        requestIterator.remove();
                        foundExpiredRequest = true;
                    }
                }
            }
            return expiredRequests;
        }

        void clean() {
            unsent.values().removeIf(ArrayDeque::isEmpty);
        }

        Iterator<Entry<Node, ArrayDeque<ClientRequest>>> iterator() {
            return unsent.entrySet().iterator();
        }

        Iterator<ClientRequest> requestIterator(Node node) {
            ArrayDeque<ClientRequest> requests = unsent.get(node);
            return (requests == null) ? Collections.emptyIterator() : requests.iterator();
        }

        Set<Node> nodes() {
            return unsent.keySet();
        }
    }
}


//专门用于和controller处理的独立线程
class NodeToControllerRequestThread(
  initialNetworkClient: KafkaClient,
  var isNetworkClientForZkController: Boolean,
  networkClientFactory: ControllerInformation => KafkaClient,
  metadataUpdater: ManualMetadataUpdater,
  controllerNodeProvider: ControllerNodeProvider, //上述提到的保存了controller的meta信息
  config: KafkaConfig,
  time: Time,
  threadName: String,
  retryTimeoutMs: Long
) extends InterBrokerSendThread(
  threadName,
  initialNetworkClient,
  Math.min(Int.MaxValue, Math.min(config.controllerSocketTimeoutMs, retryTimeoutMs)).toInt,
  time,
  false
) with Logging {

  this.logIdent = logPrefix

  private def maybeResetNetworkClient(controllerInformation: ControllerInformation): Unit = {
    if (isNetworkClientForZkController != controllerInformation.isZkController) {
      debug("Controller changed to " + (if (isNetworkClientForZkController) "kraft" else "zk") + " mode. " +
        s"Resetting network client with new controller information : ${controllerInformation}")
      // Close existing network client.
      val oldClient = networkClient
      oldClient.initiateClose()
      oldClient.close()

      isNetworkClientForZkController = controllerInformation.isZkController
      updateControllerAddress(controllerInformation.node.orNull)
      controllerInformation.node.foreach(n => metadataUpdater.setNodes(Seq(n).asJava))
      networkClient = networkClientFactory(controllerInformation)
    }
  }

  private val requestQueue = new LinkedBlockingDeque[NodeToControllerQueueItem]()
  private val activeController = new AtomicReference[Node](null)

  // Used for testing
  @volatile
  private[server] var started = false

  def activeControllerAddress(): Option[Node] = {
    Option(activeController.get())
  }

  private def updateControllerAddress(newActiveController: Node): Unit = {
    activeController.set(newActiveController)
  }

  def enqueue(request: NodeToControllerQueueItem): Unit = {
    if (!started) {
      throw new IllegalStateException("Cannot enqueue a request if the request thread is not running")
    }
    requestQueue.add(request)
    if (activeControllerAddress().isDefined) {
      wakeup()
    }
  }

  def queueSize: Int = {
    requestQueue.size
  }

  override def generateRequests(): util.Collection[RequestAndCompletionHandler] = {
    val currentTimeMs = time.milliseconds()
    val requestIter = requestQueue.iterator()
    while (requestIter.hasNext) {
      val request = requestIter.next
      if (currentTimeMs - request.createdTimeMs >= retryTimeoutMs) {
        requestIter.remove()
        request.callback.onTimeout()
      } else {
        val controllerAddress = activeControllerAddress()
        if (controllerAddress.isDefined) {
          requestIter.remove()
          return util.Collections.singletonList(new RequestAndCompletionHandler(
            time.milliseconds(),
            controllerAddress.get,
            request.request,
            response => handleResponse(request)(response)
          ))
        }
      }
    }
    util.Collections.emptyList()
  }

  private[server] def handleResponse(queueItem: NodeToControllerQueueItem)(response: ClientResponse): Unit = {
    debug(s"Request ${queueItem.request} received $response")
    if (response.authenticationException != null) {
      error(s"Request ${queueItem.request} failed due to authentication error with controller. Disconnecting the " +
        s"connection to the stale controller ${activeControllerAddress().map(_.idString).getOrElse("null")}",
        response.authenticationException)
      maybeDisconnectAndUpdateController()
      queueItem.callback.onComplete(response)
    } else if (response.versionMismatch != null) {
      error(s"Request ${queueItem.request} failed due to unsupported version error",
        response.versionMismatch)
      queueItem.callback.onComplete(response)
    } else if (response.wasDisconnected()) {
      updateControllerAddress(null)
      requestQueue.putFirst(queueItem)
    } else if (response.responseBody().errorCounts().containsKey(Errors.NOT_CONTROLLER)) {
      debug(s"Request ${queueItem.request} received NOT_CONTROLLER exception. Disconnecting the " +
        s"connection to the stale controller ${activeControllerAddress().map(_.idString).getOrElse("null")}")
      maybeDisconnectAndUpdateController()
      requestQueue.putFirst(queueItem)
    } else {
      queueItem.callback.onComplete(response)
    }
  }

  private def maybeDisconnectAndUpdateController(): Unit = {
    // just close the controller connection and wait for metadata cache update in doWork
    activeControllerAddress().foreach { controllerAddress =>
      try {
        // We don't care if disconnect has an error, just log it and get a new network client
        networkClient.disconnect(controllerAddress.idString)
      } catch {
        case t: Throwable => error("Had an error while disconnecting from NetworkClient.", t)
      }
      updateControllerAddress(null)
    }
  }

  override def doWork(): Unit = {
    val controllerInformation = controllerNodeProvider.getControllerInfo()
    maybeResetNetworkClient(controllerInformation)
    if (activeControllerAddress().isDefined) {
      super.pollOnce(Long.MaxValue)
    } else {
      debug("Controller isn't cached, looking for local metadata changes")
      controllerInformation.node match {
        case Some(controllerNode) =>
          val controllerType = if (controllerInformation.isZkController) {
            "ZK"
          } else {
            "KRaft"
          }
          info(s"Recorded new $controllerType controller, from now on will use node $controllerNode")
          updateControllerAddress(controllerNode)
          metadataUpdater.setNodes(Seq(controllerNode).asJava)
        case None =>
          // need to backoff to avoid tight loops
          debug("No controller provided, retrying after backoff")
          super.pollOnce(100)
      }
    }
  }

  override def start(): Unit = {
    super.start()
    started = true
  }

```

提供的核心方法

```scala
  /**
   * Send request to the controller.
   *
   * @param request         The request to be sent.
   * @param callback        Request completion callback.
   */
  def sendRequest(
    request: AbstractRequest.Builder[_ <: AbstractRequest],
    callback: ControllerRequestCompletionHandler
  ): Unit = {
    requestThread.enqueue(NodeToControllerQueueItem(
      time.milliseconds(),
      request,
      callback
    ))
  }

```

总结下流程

- 在kafkaServer构建的时候，会同步构建controllerNodeProvider，然后传递给NodeToControllerChannelManager。
- NodeToControllerManager启动的时候实际上是独立启动了另一个NodeToControllerRequestThread
- 而NodeToControllerRequestThread内部维护了一个阻塞队列，用于阻塞底层不断循环的ShutdownableThread
- 而中间加了一个中间类InterBrokerSendThread，用于衔接NodeToControllerRequestThread和ShutdownableThread，从NodeToControllerRequestThread里的队列取数据，并调用ShutdownableThread的doWork
- NodeToControllerManager提供了sendRequest方法，实际上就是将对象扔进了NodeToControllerRequestThread的队列里

# Producer的流程

Producer作为客户端

## 使用方法

```kotlin
    private val producerProp = run {
        val props = Properties()
        props.setProperty("bootstrap.servers", "localhost:8092")
        props["linger.ms"] = 1;
        props.setProperty("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
        props.setProperty("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");
        props
    }


    @Test
    fun test_producer() {
        val kafkaProducer = KafkaProducer<String, String>(producerProp)
        val topic = "my-topic"
        for (x in 2..5) {
            println("what $x")
            kafkaProducer.send(ProducerRecord(topic,  "what $x"))
                .get()
        }
    }

```

## doSend()

在doSend()方法里 先会等待metaData的更新

```java
    private Future<RecordMetadata> doSend(ProducerRecord<K, V> record, Callback callback) {
        // Append callback takes care of the following:
        //  - call interceptors and user callback on completion
        //  - remember partition that is calculated in RecordAccumulator.append
        AppendCallbacks<K, V> appendCallbacks = new AppendCallbacks<K, V>(callback, this.interceptors, record);

        try {
            throwIfProducerClosed();
            // first make sure the metadata for the topic is available
            long nowMs = time.milliseconds();
            ClusterAndWaitTime clusterAndWaitTime;
            try {
              //更新下metadata
                clusterAndWaitTime = waitOnMetadata(record.topic(), record.partition(), nowMs, maxBlockTimeMs);
            } catch (KafkaException e) {
                if (metadata.isClosed())
                    throw new KafkaException("Producer closed while send in progress", e);
                throw e;
            }
            nowMs += clusterAndWaitTime.waitedOnMetadataMs;
            long remainingWaitMs = Math.max(0, maxBlockTimeMs - clusterAndWaitTime.waitedOnMetadataMs);
            Cluster cluster = clusterAndWaitTime.cluster;
            byte[] serializedKey;
          //序列化key
            try {
                serializedKey = keySerializer.serialize(record.topic(), record.headers(), record.key());
            } catch (ClassCastException cce) {
                throw new SerializationException("Can't convert key of class " + record.key().getClass().getName() +
                        " to class " + producerConfig.getClass(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG).getName() +
                        " specified in key.serializer", cce);
            }
          //序列化value
            byte[] serializedValue;
            try {
                serializedValue = valueSerializer.serialize(record.topic(), record.headers(), record.value());
            } catch (ClassCastException cce) {
                throw new SerializationException("Can't convert value of class " + record.value().getClass().getName() +
                        " to class " + producerConfig.getClass(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG).getName() +
                        " specified in value.serializer", cce);
            }

            // Try to calculate partition, but note that after this call it can be RecordMetadata.UNKNOWN_PARTITION,
            // which means that the RecordAccumulator would pick a partition using built-in logic (which may
            // take into account broker load, the amount of data produced to each partition, etc.).
          //获取topic的要写入额哪个分区 
          int partition = partition(record, serializedKey, serializedValue, cluster);

            setReadOnly(record.headers());
            Header[] headers = record.headers().toArray();

            int serializedSize = AbstractRecords.estimateSizeInBytesUpperBound(apiVersions.maxUsableProduceMagic(),
                    compressionType, serializedKey, serializedValue, headers);
          //确保size 别太大了
            ensureValidRecordSize(serializedSize);
            long timestamp = record.timestamp() == null ? nowMs : record.timestamp();

            // A custom partitioner may take advantage on the onNewBatch callback.
            boolean abortOnNewBatch = partitioner != null;

            // Append the record to the accumulator.  Note, that the actual partition may be
            // calculated there and can be accessed via appendCallbacks.topicPartition.
          //kafka的消息不是有一条消息就发出去，而是等到一个size阀值，或者一个时间的阀值，这样增加了啥呢，增加了吞吐，因为你一条一条发，每一条都是一次网络请求，给网络加压力了。打包一起发的话，可以提高broker的吞吐能力，还可以数据压缩打包，减少带宽消耗。
          //所以这个RecordAccumulator就是累积器，负责吧record 先保存下的意思,在sender里面，会去检查你这个accumulator的，然后他会发送。
            RecordAccumulator.RecordAppendResult result = accumulator.append(record.topic(), partition, timestamp, serializedKey,
                    serializedValue, headers, appendCallbacks, remainingWaitMs, abortOnNewBatch, nowMs, cluster);
            assert appendCallbacks.getPartition() != RecordMetadata.UNKNOWN_PARTITION;

          //是否为需要终止当前的batch，启动一个新的batch
            if (result.abortForNewBatch) {
                int prevPartition = partition;
                onNewBatch(record.topic(), cluster, prevPartition);
                partition = partition(record, serializedKey, serializedValue, cluster);
                if (log.isTraceEnabled()) {
                    log.trace("Retrying append due to new batch creation for topic {} partition {}. The old partition was {}", record.topic(), partition, prevPartition);
                }
                result = accumulator.append(record.topic(), partition, timestamp, serializedKey,
                    serializedValue, headers, appendCallbacks, remainingWaitMs, false, nowMs, cluster);
            }

          //事务啥的
            // Add the partition to the transaction (if in progress) after it has been successfully
            // appended to the accumulator. We cannot do it before because the partition may be
            // unknown or the initially selected partition may be changed when the batch is closed
            // (as indicated by `abortForNewBatch`). Note that the `Sender` will refuse to dequeue
            // batches from the accumulator until they have been added to the transaction.
            if (transactionManager != null) {
                transactionManager.maybeAddPartition(appendCallbacks.topicPartition());
            }

          //如果batch 满了，或者新batch，直接唤醒sender，触发发送的逻辑
            if (result.batchIsFull || result.newBatchCreated) {
                log.trace("Waking up the sender since topic {} partition {} is either full or getting a new batch", record.topic(), appendCallbacks.getPartition());
                this.sender.wakeup(); //在sender里会在sendProducerData里使用accumulator
            }
          //返回一步的future
            return result.future;
            // handling exceptions and record the errors;
            // for API exceptions return them in the future,
            // for other exceptions throw directly
        } catch (ApiException e) {
            log.debug("Exception occurred during message send:", e);
            if (callback != null) {
                TopicPartition tp = appendCallbacks.topicPartition();
                RecordMetadata nullMetadata = new RecordMetadata(tp, -1, -1, RecordBatch.NO_TIMESTAMP, -1, -1);
                callback.onCompletion(nullMetadata, e);
            }
            this.errors.record();
            this.interceptors.onSendError(record, appendCallbacks.topicPartition(), e);
            if (transactionManager != null) {
                transactionManager.maybeTransitionToErrorState(e);
            }
            return new FutureFailure(e);
        } catch (InterruptedException e) {
            this.errors.record();
            this.interceptors.onSendError(record, appendCallbacks.topicPartition(), e);
            throw new InterruptException(e);
        } catch (KafkaException e) {
            this.errors.record();
            this.interceptors.onSendError(record, appendCallbacks.topicPartition(), e);
            throw e;
        } catch (Exception e) {
            // we notify interceptor about all exceptions, since onSend is called before anything else in this method
            this.interceptors.onSendError(record, appendCallbacks.topicPartition(), e);
            throw e;
        }
    }

```

## waitOnMetaData

```java
    private ClusterAndWaitTime waitOnMetadata(String topic, Integer partition, long nowMs, long maxWaitMs) throws InterruptedException {
        // add topic to metadata topic list if it is not there already and reset expiry
        Cluster cluster = metadata.fetch();

        if (cluster.invalidTopics().contains(topic))
            throw new InvalidTopicException(topic);

      //添加到metadata里面，如果这个topic不在缓存里的话，则需要更新
        metadata.add(topic, nowMs);

        Integer partitionsCount = cluster.partitionCountForTopic(topic); //新的topic都会返回null
        // Return cached metadata if we have it, and if the record's partition is either undefined
        // or within the known partition range
        if (partitionsCount != null && (partition == null || partition < partitionsCount))
            return new ClusterAndWaitTime(cluster, 0);

        long remainingWaitMs = maxWaitMs;
        long elapsed = 0;
        // Issue metadata requests until we have metadata for the topic and the requested partition,
        // or until maxWaitTimeMs is exceeded. This is necessary in case the metadata
        // is stale and the number of partitions for this topic has increased in the meantime.
        long nowNanos = time.nanoseconds();
        do {
            if (partition != null) {
                log.trace("Requesting metadata update for partition {} of topic {}.", partition, topic);
            } else {
                log.trace("Requesting metadata update for topic {}.", topic);
            }
          //更新下时间
            metadata.add(topic, nowMs + elapsed);
            int version = metadata.requestUpdateForTopic(topic); 
          //更新topic的metadata,实际上是给metadata.needPartialUpdate = true
          //在timeToNextUpdate这个方法中会检测，如果needPartialUpdate 为true，则timeToNextUpdate方法则返回0
            sender.wakeup(); //让selector 立刻唤醒，并返回结果。
            try {
              //这里去等待 更新成功，标记就是最新的version> 上次的version
                metadata.awaitUpdate(version, remainingWaitMs);
            } catch (TimeoutException ex) {
                // Rethrow with original maxWaitMs to prevent logging exception with remainingWaitMs
                throw new TimeoutException(
                        String.format("Topic %s not present in metadata after %d ms.",
                                topic, maxWaitMs));
            }
            cluster = metadata.fetch();
            elapsed = time.milliseconds() - nowMs;
            if (elapsed >= maxWaitMs) {
                throw new TimeoutException(partitionsCount == null ?
                        String.format("Topic %s not present in metadata after %d ms.",
                                topic, maxWaitMs) :
                        String.format("Partition %d of topic %s with partition count %d is not present in metadata after %d ms.",
                                partition, topic, partitionsCount, maxWaitMs));
            }
            metadata.maybeThrowExceptionForTopic(topic);
            remainingWaitMs = maxWaitMs - elapsed;
            partitionsCount = cluster.partitionCountForTopic(topic);
        } while (partitionsCount == null || (partition != null && partition >= partitionsCount));

        producerMetrics.recordMetadataWait(time.nanoseconds() - nowNanos);

        return new ClusterAndWaitTime(cluster, elapsed);
    }

```

## Sender

一个独立线程，负责真正调用client 去send请求

```java
    @Override
    public void run() {
        log.debug("Starting Kafka producer I/O thread.");

        // main loop, runs until close is called
        while (running) {
            try {
                runOnce();
            } catch (Exception e) {
                log.error("Uncaught error in kafka producer I/O thread: ", e);
            }
        }

        log.debug("Beginning shutdown of Kafka producer I/O thread, sending remaining records.");

        // okay we stopped accepting requests but there may still be
        // requests in the transaction manager, accumulator or waiting for acknowledgment,
        // wait until these are completed.
        while (!forceClose && ((this.accumulator.hasUndrained() || this.client.inFlightRequestCount() > 0) || hasPendingTransactionalRequests())) {
            try {
                runOnce();
            } catch (Exception e) {
                log.error("Uncaught error in kafka producer I/O thread: ", e);
            }
        }

        // Abort the transaction if any commit or abort didn't go through the transaction manager's queue
        while (!forceClose && transactionManager != null && transactionManager.hasOngoingTransaction()) {
            if (!transactionManager.isCompleting()) {
                log.info("Aborting incomplete transaction due to shutdown");
                transactionManager.beginAbort();
            }
            try {
                runOnce();
            } catch (Exception e) {
                log.error("Uncaught error in kafka producer I/O thread: ", e);
            }
        }

        if (forceClose) {
            // We need to fail all the incomplete transactional requests and batches and wake up the threads waiting on
            // the futures.
            if (transactionManager != null) {
                log.debug("Aborting incomplete transactional requests due to forced shutdown");
                transactionManager.close();
            }
            log.debug("Aborting incomplete batches due to forced shutdown");
            this.accumulator.abortIncompleteBatches();
        }
        try {
            this.client.close();
        } catch (Exception e) {
            log.error("Failed to close network client", e);
        }

        log.debug("Shutdown of Kafka producer I/O thread has completed.");
    }

    /**
     * Run a single iteration of sending
     *
     */
    void runOnce() {
        if (transactionManager != null) {
            try {
                transactionManager.maybeResolveSequences();

                // do not continue sending if the transaction manager is in a failed state
                if (transactionManager.hasFatalError()) {
                    RuntimeException lastError = transactionManager.lastError();
                    if (lastError != null)
                        maybeAbortBatches(lastError);
                    client.poll(retryBackoffMs, time.milliseconds());
                    return;
                }

                // Check whether we need a new producerId. If so, we will enqueue an InitProducerId
                // request which will be sent below
                transactionManager.bumpIdempotentEpochAndResetIdIfNeeded();

                if (maybeSendAndPollTransactionalRequest()) {
                    return;
                }
            } catch (AuthenticationException e) {
                // This is already logged as error, but propagated here to perform any clean ups.
                log.trace("Authentication exception while processing transactional request", e);
                transactionManager.authenticationFailed(e);
            }
        }

        long currentTimeMs = time.milliseconds();
        long pollTimeout = sendProducerData(currentTimeMs);
        client.poll(pollTimeout, currentTimeMs);
    }

```

## producer更新metadata的流程

- producer创建完毕

- 内部的Sender是独立的一个线程，实际上是while true去一直通过selector去检查是否有事件

- 实际上是这个Sender里面有runOnce，里面最后都会调用kafka提供的NetworkClient的Poll方法，实际上里面是通过nio的select方法，只不过有一个timeout

  - while true
  - 执行一个select timeout方法
  - 如果需要发一个请求的话，则调用sender的wakerup，实际上就是调的networkClient里的nio selector的wakeup，如果当前在select阻塞中立刻重新返回结果。

- 核心方法networkClient的poll

  ```java
      public List<ClientResponse> poll(long timeout, long now) {
          ensureActive();
  
          if (!abortedSends.isEmpty()) {
              // If there are aborted sends because of unsupported version exceptions or disconnects,
              // handle them immediately without waiting for Selector#poll.
              List<ClientResponse> responses = new ArrayList<>();
              handleAbortedSends(responses);
              completeResponses(responses);
              return responses;
          }
        
   
  				//这里可以会去检查metadata是否需要更新
          long metadataTimeout = metadataUpdater.maybeUpdate(now);
          try {
              this.selector.poll(Utils.min(timeout, metadataTimeout, defaultRequestTimeoutMs));
          } catch (IOException e) {
              log.error("Unexpected error during I/O", e);
          }
  
          // process completed actions
          long updatedNow = this.time.milliseconds();
          List<ClientResponse> responses = new ArrayList<>();
          handleCompletedSends(responses, updatedNow);
          handleCompletedReceives(responses, updatedNow);
          handleDisconnections(responses, updatedNow);
          handleConnections();
        //调apiversion接口
          handleInitiateApiVersionRequests(updatedNow);
          handleTimedOutConnections(responses, updatedNow);
          handleTimedOutRequests(responses, updatedNow);
          completeResponses(responses);
  
          return responses;
      }
  
      public synchronized long timeToNextUpdate(long nowMs) {
        //标记需要更新，或者说已经到下一个过期时间，则需要更新哦
          long timeToExpire = updateRequested() ? 0 : Math.max(this.lastSuccessfulRefreshMs + this.metadataExpireMs - nowMs, 0);
          return Math.max(timeToExpire, timeToAllowUpdate(nowMs));
      }
  
  ```

- 因此producer有2种，1是更新时间到了，在while true循环里自然会更新，2是 设置updateRequested为true，调用sender的wakeup()立即出发下次请求，从而完成metadata的更新。
- producer 创建完成后就会进行首次的metadata获取

## calculate record的partition

kafka的msg，即代码中的record的应该发送给哪个partition是在producer中计算得来的

- 指定了partition，就用这个
- 没指定的话，通过record的key进行计算，如果record的key为null，则随机partition任意一个，当然kafka提供了RoundRobin的partitioner，可以对所有partition进行轮训

```java
    private int partition(ProducerRecord<K, V> record, byte[] serializedKey, byte[] serializedValue, Cluster cluster) {
        if (record.partition() != null)
            return record.partition();

      //给使用者自己一个实现partitioner的方法，可以自定义partition的分配规则
        if (partitioner != null) {
            int customPartition = partitioner.partition(
                record.topic(), record.key(), serializedKey, record.value(), serializedValue, cluster);
            if (customPartition < 0) {
                throw new IllegalArgumentException(String.format(
                    "The partitioner generated an invalid partition number: %d. Partition number should always be non-negative.", customPartition));
            }
            return customPartition;
        }

    
        if (serializedKey != null && !partitionerIgnoreKeys) {
            // hash the keyBytes to choose a partition
          //key不是空的，算hash得结果murmur2 算法
            return BuiltInPartitioner.partitionForKey(serializedKey, cluster.partitionsForTopic(record.topic()).size());
        } else {
            //用内置的，即RoundRobinPartitioner
            return RecordMetadata.UNKNOWN_PARTITION;
        }
    }

```

```java
private int nextPartition(Cluster cluster) {
    int random = mockRandom != null ? mockRandom.get() : Utils.toPositive(ThreadLocalRandom.current().nextInt());

    // Cache volatile variable in local variable.
    PartitionLoadStats partitionLoadStats = this.partitionLoadStats;
    int partition;

    if (partitionLoadStats == null) {
        // We don't have stats to do adaptive partitioning (or it's disabled), just switch to the next
        // partition based on uniform distribution.
        List<PartitionInfo> availablePartitions = cluster.availablePartitionsForTopic(topic);
        if (availablePartitions.size() > 0) {
          //随机数取余
            partition = availablePartitions.get(random % availablePartitions.size()).partition();
        } else {
            // We don't have available partitions, just pick one among all partitions.
          List<PartitionInfo> partitions = cluster.partitionsForTopic(topic);
            partition = random % partitions.size();
        }
```

## Sender的sendProducerData

```java
    private long sendProducerData(long now) {
        Cluster cluster = metadata.fetch();
        // get the list of partitions with data ready to send
        RecordAccumulator.ReadyCheckResult result = this.accumulator.ready(cluster, now);

        // if there are any partitions whose leaders are not known yet, force metadata update
      //还有位置leader partition topc的
        if (!result.unknownLeaderTopics.isEmpty()) {
            // The set of topics with unknown leader contains topics with leader election pending as well as
            // topics which may have expired. Add the topic again to metadata to ensure it is included
            // and request metadata update, since there are messages to send to the topic.
            for (String topic : result.unknownLeaderTopics)
                this.metadata.add(topic, now);

            log.debug("Requesting metadata update due to unknown leader topics from the batched records: {}",
                result.unknownLeaderTopics);
            this.metadata.requestUpdate();
        }

        // remove any nodes we aren't ready to send to
        Iterator<Node> iter = result.readyNodes.iterator();
        long notReadyTimeout = Long.MAX_VALUE;
        while (iter.hasNext()) {
            Node node = iter.next();
            if (!this.client.ready(node, now)) {
                // Update just the readyTimeMs of the latency stats, so that it moves forward
                // every time the batch is ready (then the difference between readyTimeMs and
                // drainTimeMs would represent how long data is waiting for the node).
                this.accumulator.updateNodeLatencyStats(node.id(), now, false);
                iter.remove();
                notReadyTimeout = Math.min(notReadyTimeout, this.client.pollDelayMs(node, now));
            } else {
                // Update both readyTimeMs and drainTimeMs, this would "reset" the node
                // latency.
                this.accumulator.updateNodeLatencyStats(node.id(), now, true);
            }
        }

        // create produce requests
        Map<Integer, List<ProducerBatch>> batches = this.accumulator.drain(cluster, result.readyNodes, this.maxRequestSize, now);
        addToInflightBatches(batches);
        if (guaranteeMessageOrder) {
            // Mute all the partitions drained
            for (List<ProducerBatch> batchList : batches.values()) {
                for (ProducerBatch batch : batchList)
                    this.accumulator.mutePartition(batch.topicPartition);
            }
        }

        accumulator.resetNextBatchExpiryTime();
        List<ProducerBatch> expiredInflightBatches = getExpiredInflightBatches(now);
        List<ProducerBatch> expiredBatches = this.accumulator.expiredBatches(now);
        expiredBatches.addAll(expiredInflightBatches);

        // Reset the producer id if an expired batch has previously been sent to the broker. Also update the metrics
        // for expired batches. see the documentation of @TransactionState.resetIdempotentProducerId to understand why
        // we need to reset the producer id here.
        if (!expiredBatches.isEmpty())
            log.trace("Expired {} batches in accumulator", expiredBatches.size());
        for (ProducerBatch expiredBatch : expiredBatches) {
            String errorMessage = "Expiring " + expiredBatch.recordCount + " record(s) for " + expiredBatch.topicPartition
                + ":" + (now - expiredBatch.createdMs) + " ms has passed since batch creation";
            failBatch(expiredBatch, new TimeoutException(errorMessage), false);
            if (transactionManager != null && expiredBatch.inRetry()) {
                // This ensures that no new batches are drained until the current in flight batches are fully resolved.
                transactionManager.markSequenceUnresolved(expiredBatch);
            }
        }
        sensors.updateProduceRequestMetrics(batches);

        // If we have any nodes that are ready to send + have sendable data, poll with 0 timeout so this can immediately
        // loop and try sending more data. Otherwise, the timeout will be the smaller value between next batch expiry
        // time, and the delay time for checking data availability. Note that the nodes may have data that isn't yet
        // sendable due to lingering, backing off, etc. This specifically does not include nodes with sendable data
        // that aren't ready to send since they would cause busy looping.
        long pollTimeout = Math.min(result.nextReadyCheckDelayMs, notReadyTimeout);
        pollTimeout = Math.min(pollTimeout, this.accumulator.nextExpiryTimeMs() - now);
        pollTimeout = Math.max(pollTimeout, 0);
        if (!result.readyNodes.isEmpty()) {
            log.trace("Nodes with data ready to send: {}", result.readyNodes);
            // if some partitions are already ready to be sent, the select time would be 0;
            // otherwise if some partition already has some data accumulated but not ready yet,
            // the select time will be the time difference between now and its linger expiry time;
            // otherwise the select time will be the time difference between now and the metadata expiry time;
            pollTimeout = 0;
        }
      //真正的发送消息
        sendProduceRequests(batches, now);
        return pollTimeout;
    }

```

最后真正的send是调用client的send方法

```java
private void doSend(ClientRequest clientRequest, boolean isInternalRequest, long now, AbstractRequest request) {
    String destination = clientRequest.destination();
    RequestHeader header = clientRequest.makeHeader(request.version());
    if (log.isDebugEnabled()) {
        log.debug("Sending {} request with header {} and timeout {} to node {}: {}",
            clientRequest.apiKey(), header, clientRequest.requestTimeoutMs(), destination, request);
    }
    Send send = request.toSend(header);
    InFlightRequest inFlightRequest = new InFlightRequest(
            clientRequest,
            header,
            isInternalRequest,
            request,
            send,
            now);
    this.inFlightRequests.add(inFlightRequest);
    selector.send(new NetworkSend(clientRequest.destination(), send));
}
```

实际是selector.send

```java
    public void send(NetworkSend send) {
        String connectionId = send.destinationId();
        KafkaChannel channel = openOrClosingChannelOrFail(connectionId);
        if (closingChannels.containsKey(connectionId)) {
            // ensure notification via `disconnected`, leave channel in the state in which closing was triggered
            this.failedSends.add(connectionId);
        } else {
            try {
              //nio的channel
                channel.setSend(send);
            } catch (Exception e) {
                // update the state for consistency, the channel will be discarded after `close`
                channel.state(ChannelState.FAILED_SEND);
                // ensure notification via `disconnected` when `failedSends` are processed in the next poll
                this.failedSends.add(connectionId);
                close(channel, CloseMode.DISCARD_NO_NOTIFY);
                if (!(e instanceof CancelledKeyException)) {
                    log.error("Unexpected exception during send, closing connection {} and rethrowing exception {}",
                            connectionId, e);
                    throw e;
                }
            }
        }
    }

    public void setSend(NetworkSend send) {
        if (this.send != null)
            throw new IllegalStateException("Attempt to begin a send operation with prior send operation still in progress, connection id is " + id);
        this.send = send;
      //nio 添加write的事件 ，然后底层就是selector那一套了
        this.transportLayer.addInterestOps(SelectionKey.OP_WRITE);
    }

```

## 什么地方设置的返回值呢？

还是在client.poll()里面

```java


        // process completed actions
        long updatedNow = this.time.milliseconds();
        List<ClientResponse> responses = new ArrayList<>();
        handleCompletedSends(responses, updatedNow);
        handleCompletedReceives(responses, updatedNow);
        handleDisconnections(responses, updatedNow);
        handleConnections();
        handleInitiateApiVersionRequests(updatedNow);
        handleTimedOutConnections(responses, updatedNow);
        handleTimedOutRequests(responses, updatedNow);
//处理返回值
        completeResponses(responses);

        return responses;
    private void completeResponses(List<ClientResponse> responses) {
        for (ClientResponse response : responses) {
            try {
                response.onComplete();
            } catch (Exception e) {
                log.error("Uncaught error in request completion:", e);
            }
        }
    }

    public void onComplete() {
        if (callback != null)
            callback.onComplete(this);
    }


```

在sendProducerRequest里 设置了callback

```java
        ProduceRequest.Builder requestBuilder = ProduceRequest.forMagic(minUsedMagic,
                new ProduceRequestData()
                        .setAcks(acks)
                        .setTimeoutMs(timeout)
                        .setTransactionalId(transactionalId)
                        .setTopicData(tpd));

//这里设置的callback
        RequestCompletionHandler callback = response -> handleProduceResponse(response, recordsByPartition, time.milliseconds());

        String nodeId = Integer.toString(destination);
        ClientRequest clientRequest = client.newClientRequest(nodeId, requestBuilder, now, acks != 0,
                requestTimeoutMs, callback);
        client.send(clientRequest, now);
        log.trace("Sent produce request to {}: {}", nodeId, requestBuilder);


    private void handleProduceResponse(ClientResponse response, Map<TopicPartition, ProducerBatch> batches, long now) {
        RequestHeader requestHeader = response.requestHeader();
        int correlationId = requestHeader.correlationId();
        if (response.wasTimedOut()) {
            log.trace("Cancelled request with header {} due to the last request to node {} timed out",
                requestHeader, response.destination());
            for (ProducerBatch batch : batches.values())
                completeBatch(batch, new ProduceResponse.PartitionResponse(Errors.REQUEST_TIMED_OUT, String.format("Disconnected from node %s due to timeout", response.destination())),
                        correlationId, now);
        } else if (response.wasDisconnected()) {
            log.trace("Cancelled request with header {} due to node {} being disconnected",
                requestHeader, response.destination());
            for (ProducerBatch batch : batches.values())
              //完成batch，实际上就给方法返回的future，done掉
                completeBatch(batch, new ProduceResponse.PartitionResponse(Errors.NETWORK_EXCEPTION, String.format("Disconnected from node %s", response.destination())),
                        correlationId, now);
        } else if (response.versionMismatch() != null) {
            log.warn("Cancelled request {} due to a version mismatch with node {}",
                    response, response.destination(), response.versionMismatch());
            for (ProducerBatch batch : batches.values())
                completeBatch(batch, new ProduceResponse.PartitionResponse(Errors.UNSUPPORTED_VERSION), correlationId, now);
        } else {
            log.trace("Received produce response from node {} with correlation id {}", response.destination(), correlationId);
            // if we have a response, parse it
            if (response.hasResponse()) {
                // Sender should exercise PartitionProduceResponse rather than ProduceResponse.PartitionResponse
                // https://issues.apache.org/jira/browse/KAFKA-10696
                ProduceResponse produceResponse = (ProduceResponse) response.responseBody();
                produceResponse.data().responses().forEach(r -> r.partitionResponses().forEach(p -> {
                    TopicPartition tp = new TopicPartition(r.name(), p.index());
                    ProduceResponse.PartitionResponse partResp = new ProduceResponse.PartitionResponse(
                            Errors.forCode(p.errorCode()),
                      //base offset
                            p.baseOffset(),
                            p.logAppendTimeMs(),
                      //offset，偏移量
                            p.logStartOffset(),
                            p.recordErrors()
                                .stream()
                                .map(e -> new ProduceResponse.RecordError(e.batchIndex(), e.batchIndexErrorMessage()))
                                .collect(Collectors.toList()),
                            p.errorMessage());
                    ProducerBatch batch = batches.get(tp);
                                //完成batch，实际上就给方法返回的future，done掉
                    completeBatch(batch, partResp, correlationId, now);
                }));
                this.sensors.recordLatency(response.destination(), response.requestLatencyMs());
            } else {
                // this is the acks = 0 case, just complete all requests
                for (ProducerBatch batch : batches.values()) {
                    completeBatch(batch, new ProduceResponse.PartitionResponse(Errors.NONE), correlationId, now);
                }
            }
        }
    }



```

# Consumer的流程

## ConsumerGroup

消费者组，就是只一个topic可以有多个消费者消费。也可以由多个消费者组成一个组去消费。

消费者组是说一群消费者当做一个消费者去消费topic，每个消费者之间是独立的，他们的offset就是各自的。

比如说消费者A和消费者B是2个组的，那么他们连接上kafka的时候各自获取到自己的offset，我给这个topic发一条消息，**两人都会收到**。

如果消费者A和消费者B都属于Demo组的话，那么他们连接上kafka的时候的offset是整个组的，也就是说我给这个topic发一条消息，**只会有一个人收到**。



## Consumer能多线程去消费吗

```java
//不能   
private void acquire() {
        final Thread thread = Thread.currentThread();
        final long threadId = thread.getId();
        if (threadId != currentThread.get() && !currentThread.compareAndSet(NO_CURRENT_THREAD, threadId))
            throw new ConcurrentModificationException("KafkaConsumer is not safe for multi-threaded access. " +
                    "currentThread(name: " + thread.getName() + ", id: " + threadId + ")" +
                    " otherThread(id: " + currentThread.get() + ")"
            );
        refcount.incrementAndGet();
    }

```



# kafka如何保证消息的消费顺序





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
