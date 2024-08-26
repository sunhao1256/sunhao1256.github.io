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
