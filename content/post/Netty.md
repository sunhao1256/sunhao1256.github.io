---
title: "Netty"
date: 2022-01-11T14:43:18+08:00
tags: ['Netty']
---

# netty是在什么地方创建SeverChannel的

在BootStrap调用bind方法时创建

```java
private ChannelFuture doBind(final SocketAddress localAddress) {
    final ChannelFuture regFuture = this.initAndRegister();
}
```

initAndRegister方法

```java
final ChannelFuture initAndRegister() {
    Channel channel = null;

    try {
        channel = this.channelFactory.newChannel();
```

channelFactory是在bootstrap配置class时实例化的

在AbstractBootstrap中

```java
public B channel(Class<? extends C> channelClass) {
    return this.channelFactory((io.netty.channel.ChannelFactory)(new ReflectiveChannelFactory((Class)ObjectUtil.checkNotNull(channelClass, "channelClass"))));
}
```

因此是直接通过反射，创建了NioServerSocketChannel.class

```java
public class NioServerSocketChannel extends AbstractNioMessageChannel implements ServerSocketChannel {
    private static final ChannelMetadata METADATA = new ChannelMetadata(false, 16);
    private static final SelectorProvider DEFAULT_SELECTOR_PROVIDER = SelectorProvider.provider();
    private static final InternalLogger logger = InternalLoggerFactory.getInstance(NioServerSocketChannel.class);
    private final ServerSocketChannelConfig config;

    private static java.nio.channels.ServerSocketChannel newSocket(SelectorProvider provider) {
        try {
          //实际上就是调用了java的方法
            return provider.openServerSocketChannel();
        } catch (IOException var2) {
            throw new ChannelException("Failed to open a server socket.", var2);
        }
    }

    public NioServerSocketChannel() {
        this(newSocket(DEFAULT_SELECTOR_PROVIDER));
    }

    public NioServerSocketChannel(SelectorProvider provider) {
        this(newSocket(provider));
    }
  //super方法中配置了阻塞
    public NioServerSocketChannel(java.nio.channels.ServerSocketChannel channel) {
        super((Channel)null, channel, 16);
        this.config = new NioServerSocketChannel.NioServerSocketChannelConfig(this, this.javaChannel().socket());
    }
```

```java
protected AbstractNioChannel(Channel parent, SelectableChannel ch, int readInterestOp) {
    super(parent);
    this.ch = ch;
    this.readInterestOp = readInterestOp;

    try {
      //设置阻塞
        ch.configureBlocking(false);
    } catch (IOException var7) {
        try {
            ch.close();
        } catch (IOException var6) {
            logger.warn("Failed to close a partially initialized socket.", var6);
        }

        throw new ChannelException("Failed to enter non-blocking mode.", var7);
    }
}
```

```java
protected AbstractChannel(Channel parent) {
    this.parent = parent;
    this.id = this.newId();
    this.unsafe = this.newUnsafe();
    this.pipeline = this.newChannelPipeline();//创建了管道
}
```

# ServerBootstrapAcceptor

服务端不管之前增加了多少handler，最后都会增加ServerBootstrapAcceptor

在ServerBootstrap.class中

```java
void init(Channel channel) {
    setChannelOptions(channel, this.newOptionsArray(), logger);
    setAttributes(channel, (Entry[])this.attrs0().entrySet().toArray(EMPTY_ATTRIBUTE_ARRAY));
    ChannelPipeline p = channel.pipeline();
    final EventLoopGroup currentChildGroup = this.childGroup;
    final ChannelHandler currentChildHandler = this.childHandler;
    final Entry[] currentChildOptions;
    synchronized(this.childOptions) {
        currentChildOptions = (Entry[])this.childOptions.entrySet().toArray(EMPTY_OPTION_ARRAY);
    }

    final Entry<AttributeKey<?>, Object>[] currentChildAttrs = (Entry[])this.childAttrs.entrySet().toArray(EMPTY_ATTRIBUTE_ARRAY);
    p.addLast(new ChannelHandler[]{new ChannelInitializer<Channel>() {
        public void initChannel(final Channel ch) {
            final ChannelPipeline pipeline = ch.pipeline();
            ChannelHandler handler = ServerBootstrap.this.config.handler();
            if (handler != null) {
                pipeline.addLast(new ChannelHandler[]{handler});
            }

            ch.eventLoop().execute(new Runnable() {
                public void run() {
                  //在这里加入了默认的
                  //以后新的请求的都通过这个连接器处理，给新的连接分配一个nio线程
                    pipeline.addLast(new ChannelHandler[]{new ServerBootstrap.ServerBootstrapAcceptor(ch, currentChildGroup, currentChildHandler, currentChildOptions, currentChildAttrs)});
                }
            });
        }
    }});
}
```

netty服务端启动的几个过程：创建channel（创建jdk底层channel）、init初始化（最主要是添加连接器serverBootstrapAcceptor）、register（把第一步jdk底层创建channel注册到selector上面，并且把这个NioServerSocketChannel作为attachment添加进去），doBind（绑定端口，实现监听accept事件）

- 1、服务端的socket在哪里初始化？

  在bind方法中，initAndRegister中，实例化SocketChannel，使用的是反射方式，传入的就是bootstrap配置的channel.class。底层还是jdk的ServerSocketChannel对应普通方式的

  ```java
  serverSocketChannel = ServerSocketChannel.open();
  //设置为非阻塞模式
  serverSocketChannel.configureBlocking(false);
  ```

  register是把创建好的注册到selector上，但没有监听事件，对应的是

  ```java
  //监听客户端请求
  //0表示只注册，不监听事件
  serverSocketChannel.register(selector, 0);
  ```

  ```java
      private static java.nio.channels.ServerSocketChannel newSocket(SelectorProvider provider) {
          try {
              return provider.openServerSocketChannel();//创建jdk的ServerSocketChannel
          } catch (IOException var2) {
              throw new ChannelException("Failed to open a server socket.", var2);
          }
      }
  ```

  

- 2、在哪里进行accept连接？

  还是在bind方法中，doBind最终底层调用了jdk的端口绑定和监听accpet事件

  ```java
      public NioServerSocketChannel(java.nio.channels.ServerSocketChannel channel) {
          super((Channel)null, channel, 16);//16就是SelectionKey.OP_ACCEPT 1<<4
          this.config = new NioServerSocketChannel.NioServerSocketChannelConfig(this, this.javaChannel().socket());
      }
  ```

  对应普通的就是

  ```java
  serverSocketChannel.socket().bind(new InetSocketAddress(port),1024);
  //监听客户端请求
  serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);
  ```

  bind最终是在NioServerSocketChannel执行的

  ```java
  protected void doBind(SocketAddress localAddress) throws Exception {
      if (PlatformDependent.javaVersion() >= 7) {
          this.javaChannel().bind(localAddress, this.config.getBacklog());
      } else {
          this.javaChannel().socket().bind(localAddress, this.config.getBacklog());
      }
  
  }
  ```

  accpet最终是在AbstractNioChannel的doBeginRead方法中，注册了构造方法传来的16，即accpet事件

  ```java
  protected void doBeginRead() throws Exception {
      SelectionKey selectionKey = this.selectionKey;
      if (selectionKey.isValid()) {
          this.readPending = true;
          int interestOps = selectionKey.interestOps();
          if ((interestOps & this.readInterestOp) == 0) {
              selectionKey.interestOps(interestOps | this.readInterestOp);
          }
  
      }
  }
  ```

# NioEventLoopGroup、NioEventLoop的关系

在NioEventLoopGroup实例化的时候，会做如下动作

- 创建线程创建(执行)器：new ThreadPerTaskExecutor()

  ```java
  protected MultithreadEventExecutorGroup(int nThreads, Executor executor, EventExecutorChooserFactory chooserFactory, Object... args) {
      this.terminatedChildren = new AtomicInteger();
      this.terminationFuture = new DefaultPromise(GlobalEventExecutor.INSTANCE);
      if (nThreads <= 0) {
          throw new IllegalArgumentException(String.format("nThreads: %d (expected: > 0)", nThreads));
      } else {
          if (executor == null) {
            //创建线程执行器
              executor = new ThreadPerTaskExecutor(this.newDefaultThreadFactory());
          }
  
              this.children = new EventExecutor[nThreads];
  
              int j;
              for(int i = 0; i < nThreads; ++i) {
                  boolean success = false;
                  boolean var18 = false;
  
                  try {
                      var18 = true;
                    //填充child
                      this.children[i] = this.newChild((Executor)executor, args);
                      success = true;
                      var18 = false;
                  } catch (Exception var19) {
        
  ```

- 构造NioEventLoop：for{newChild()}

  ```java
      protected EventLoop newChild(Executor executor, Object... args) throws Exception {
        //填充的就是nioEventLoop 
        return new NioEventLoop(this, executor, (SelectorProvider)args[0], ((SelectStrategyFactory)args[1]).newSelectStrategy(), (RejectedExecutionHandler)args[2]);
      }
  ```

  

- 创建线程选择器：chooserFactory.newChooser()

  ```java
  
      private static final class PowerOfTowEventExecutorChooser implements EventExecutorChooser {
          private final AtomicInteger idx = new AtomicInteger();
          private final EventExecutor[] executors;
   
          PowerOfTowEventExecutorChooser(EventExecutor[] executors) {
              this.executors = executors;
          }
   
          @Override
          public EventExecutor next() {
              return executors[idx.getAndIncrement() & executors.length - 1];
          }
      }
   
      private static final class GenericEventExecutorChooser implements EventExecutorChooser {
          private final AtomicInteger idx = new AtomicInteger();
          private final EventExecutor[] executors;
   
          GenericEventExecutorChooser(EventExecutor[] executors) {
              this.executors = executors;
          }
   
          @Override
          public EventExecutor next() {
              return executors[Math.abs(idx.getAndIncrement() % executors.length)];
          }
      }
  ```

  模取运算除了%，还可以用&（length-1）

- 线程创建与运行的流程

  - NioEventLoopGroup顾名思义，就是NioEventLoopGroup的集合，内部维护了children数组，数组的内容就是NioEventLoop

  - 当调用Loop去execute的时候，实际上是调用线程执行器ThreadPerTaskExecutor去执行，内部DefaultThreadFactory就是创建了FastThreadLocalThread线程。

  - Loop的execute方法，除了用DefaultThreadFactory创建并运行FastThreadLocalThread线程外，还会将任务放入Loop自身维护的一个无锁队列。并且运行自身的run方法

    ```java
    private void doStartThread() {
        assert thread == null;
        executor.execute(new Runnable() {
            @Override
            public void run() {
                thread = Thread.currentThread();
                if (interrupted) {
                    thread.interrupt();
                }
    
                boolean success = false;
                updateLastExecutionTime();
                try {
                    SingleThreadEventExecutor.this.run();//调用run方法
                  
                  
    // nioEventLoop的run方法,无线循环去消费队列
                 for(;;)
                   .....
                   ....
                   ....
                  final int ioRatio = this.ioRatio;
                  boolean ranTasks;
                  if (ioRatio == 100) {
                    try {
                      if (strategy > 0) {
                        processSelectedKeys();
                      }
                    } finally {
                      // Ensure we always run tasks.
                      ranTasks = runAllTasks();
                    }
                  } else if (strategy > 0) {
                    final long ioStartTime = System.nanoTime();
                    try {
                      processSelectedKeys();
                    } finally {
                      // Ensure we always run tasks.
                      final long ioTime = System.nanoTime() - ioStartTime;
                      ranTasks = runAllTasks(ioTime * (100 - ioRatio) / ioRatio);//通过ioRatio控制消费任务的速度，这里就是不断的循环去消费
                    }
                  } else {
                    ranTasks = runAllTasks(0); // This will run the minimum number of tasks
                  }
    ```

- 总结

  在创建NioEventLoopGroup后，内部会维护了一个children，children的内容就是nioEventLoop，并且创建loop的时候为每个loop创建了内部的无锁队列，默认长度是系统核心的2倍，然后调用group的register或者execute方法，都会执行next去取一个loop，next方法就是模取循环下一个线程。

  loop会执行execute方法，实现会将任务Runnable放入队列。然后第一次运行时调用了ThreadFactory去创建了新线程，这个新线程并不是直接就开始执行任务，而是无限循环消费loop内部维护的无锁队列。达到了，一个loop一个线程。

# NioEventLoop启动

在绑定端口的时候做了启动线程的动作

```java
channel.eventLoop().execute(new Runnable() {
            @Override
            public void run() {
                if (regFuture.isSuccess()) {
                    channel.bind(localAddress, promise).addListener(ChannelFutureListener.CLOSE_ON_FAILURE);
                } else {
                    promise.setFailure(regFuture.cause());
                }
            }
        });
```

这里的channel的eventLoop是通过register方法去绑定的，是在一开始Bootstrap的intAndRegister方法中执行

```java

    final ChannelFuture initAndRegister() {
        Channel channel = null;

        try {
            channel = this.channelFactory.newChannel();
            this.init(channel);
        } catch (Throwable var3) {
            if (channel != null) {
                channel.unsafe().closeForcibly();
                return (new DefaultChannelPromise(channel, GlobalEventExecutor.INSTANCE)).setFailure(var3);
            }

            return (new DefaultChannelPromise(new FailedChannel(), GlobalEventExecutor.INSTANCE)).setFailure(var3);
        }

      //从group取一个loop绑定到channel上
        ChannelFuture regFuture = this.config().group().register(channel);
        if (regFuture.cause() != null) {
            if (channel.isRegistered()) {
                channel.close();
            } else {
                channel.unsafe().closeForcibly();
            }
        }

        return regFuture;
    }
```

# NioEventLoop的run方法

- select()检查是否有io事件

  netty在这里检查是否需要select，需要的话，直接调用jdk的select方法，进行阻塞。

  jdk的select方法，在linux下，会发生没有事件，但是select方法被触发导致的空轮训bug

  netty的解决方式

  ```java
  long time = System.nanoTime();
  //检查当前时间-本应该阻塞的时间>=阻塞之前的时间，即发生阻塞了。没有空轮训
  if (time - TimeUnit.MILLISECONDS.toNanos(timeoutMillis) >= currentTimeNanos) {
      selectCnt = 1;
    //否则，空轮训超过默认的512次，重建selector
  } else if (SELECTOR_AUTO_REBUILD_THRESHOLD > 0 && selectCnt >= SELECTOR_AUTO_REBUILD_THRESHOLD) {
      logger.warn("Selector.select() returned prematurely {} times in a row; rebuilding Selector {}.", selectCnt, selector);
      this.rebuildSelector();
      selector = this.selector;
      selector.selectNow();
      selectCnt = 1;
      break;
  }
  ```

  >rebuildSelector()方法就是将原来selector上的所有keys，绑定到新的selector上

- processSelectedKeys()：处理io任务，即selectionKey中ready的事件，如accept、connect、read、write等

  底层取轮询io事件，都是通过NioUnsafe去处理的

  ```java
      private void processSelectedKey(SelectionKey k, AbstractNioChannel ch) {
          final AbstractNioChannel.NioUnsafe unsafe = ch.unsafe();
          if (!k.isValid()) {
              final EventLoop eventLoop;
              try {
                  eventLoop = ch.eventLoop();
              } catch (Throwable ignored) {
                  // If the channel implementation throws an exception because there is no event loop, we ignore this
                  // because we are only trying to determine if ch is registered to this event loop and thus has authority
                  // to close ch.
                  return;
              }
              // Only close ch if ch is still registered to this EventLoop. ch could have deregistered from the event loop
              // and thus the SelectionKey could be cancelled as part of the deregistration process, but the channel is
              // still healthy and should not be closed.
              // See https://github.com/netty/netty/issues/5125
              if (eventLoop == this) {
                  // close the channel if the key is not valid anymore
                  unsafe.close(unsafe.voidPromise());
              }
              return;
          }
  
          try {
            //处理各种事件
              int readyOps = k.readyOps();
              // We first need to call finishConnect() before try to trigger a read(...) or write(...) as otherwise
              // the NIO JDK channel implementation may throw a NotYetConnectedException.
              if ((readyOps & SelectionKey.OP_CONNECT) != 0) {
                  // remove OP_CONNECT as otherwise Selector.select(..) will always return without blocking
                  // See https://github.com/netty/netty/issues/924
                  int ops = k.interestOps();
                  ops &= ~SelectionKey.OP_CONNECT;
                  k.interestOps(ops);
  
                  unsafe.finishConnect();
              }
  
              // Process OP_WRITE first as we may be able to write some queued buffers and so free memory.
              if ((readyOps & SelectionKey.OP_WRITE) != 0) {
                  // Call forceFlush which will also take care of clear the OP_WRITE once there is nothing left to write
                  ch.unsafe().forceFlush();
              }
  
              // Also check for readOps of 0 to workaround possible JDK bug which may otherwise lead
              // to a spin loop
              if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) {
                  unsafe.read();
              }
          } catch (CancelledKeyException ignored) {
              unsafe.close(unsafe.voidPromise());
          }
      }
  ```

  

- runAllTasks()：处理异步任务队列里面的任务，也就是添加到taskQueue中的任务，如bind、channelActive等

  这里有个ioRatio，就是执行io任务和非io任务的时间比。用户可以自行设置。默认为50，可以看源码。如果是100的话，先执行完io任务，执行完之后才执行非io任务

  ```java
                  selectCnt++;
                  cancelledKeys = 0;
                  needsToSelectAgain = false;
                  final int ioRatio = this.ioRatio;
                  boolean ranTasks;
                  if (ioRatio == 100) {
                      try {
                          if (strategy > 0) {
                              processSelectedKeys();
                          }
                      } finally {
                          // Ensure we always run tasks.
                          ranTasks = runAllTasks();
                      }
                  } else if (strategy > 0) {
                      final long ioStartTime = System.nanoTime();
                      try {
                          processSelectedKeys();
                      } finally {
                          // Ensure we always run tasks.
                          final long ioTime = System.nanoTime() - ioStartTime;
                          ranTasks = runAllTasks(ioTime * (100 - ioRatio) / ioRatio);
                      }
                  } else {
                      ranTasks = runAllTasks(0); // This will run the minimum number of tasks
                  }
  
                  if (ranTasks || strategy > 0) {
                      if (selectCnt > MIN_PREMATURE_SELECTOR_RETURNS && logger.isDebugEnabled()) {
                          logger.debug("Selector.select() returned prematurely {} times in a row for Selector {}.",
                                  selectCnt - 1, selector);
                      }
                      selectCnt = 0;
                  } else if (unexpectedSelectorWakeup(selectCnt)) { // Unexpected wakeup (unusual case)
                      selectCnt = 0;
                  }
  ```

  ```java
  protected boolean runAllTasks(long timeoutNanos) {
      fetchFromScheduledTaskQueue();//将定时任务队列里的到期任务放入队列中，如果队列满了，还放在定时任务队列里
      Runnable task = pollTask();//取一个任务出来
      if (task == null) {
          afterRunningAllTasks();
          return false;
      }
  
      final long deadline = timeoutNanos > 0 ? ScheduledFutureTask.nanoTime() + timeoutNanos : 0;
      long runTasks = 0;
      long lastExecutionTime;
      for (;;) {
          safeExecute(task);
  
          runTasks ++;
  
          // Check timeout every 64 tasks because nanoTime() is relatively expensive.
          // XXX: Hard-coded value - will make it configurable if it is really a problem.
          if ((runTasks & 0x3F) == 0) {
              lastExecutionTime = ScheduledFutureTask.nanoTime();
              if (lastExecutionTime >= deadline) {
                  break;
              }
          }
  
        //不断处理任务
          task = pollTask();
          if (task == null) {
              lastExecutionTime = ScheduledFutureTask.nanoTime();
              break;
          }
      }
  
      afterRunningAllTasks();
      this.lastExecutionTime = lastExecutionTime;
      return true;
  }
  ```

  > 定时任务的场景，例如心跳处理

# 服务端创建客户端连接流程

processSelectedKeys方法处理客户端连接，最后是落在底层的unsafe上。

read方法最后会通知到管道上，通知所有的handler，

```java
public void read() {
    assert eventLoop().inEventLoop();
    final ChannelConfig config = config();
    final ChannelPipeline pipeline = pipeline();
    final RecvByteBufAllocator.Handle allocHandle = unsafe().recvBufAllocHandle();
    allocHandle.reset(config);

    boolean closed = false;
    Throwable exception = null;
    try {
        try {
            do {
              //创建NioSocketChannel
                int localRead = doReadMessages(readBuf);
                if (localRead == 0) {
                    break;
                }
                if (localRead < 0) {
                    closed = true;
                    break;
                }

                allocHandle.incMessagesRead(localRead);
            } while (continueReading(allocHandle));
        } catch (Throwable t) {
            exception = t;
        }

        int size = readBuf.size();
        for (int i = 0; i < size; i ++) {
            readPending = false;
          //管道传入NioSocketChannel
            pipeline.fireChannelRead(readBuf.get(i));
        }
```

而我们知道在一开始创建NioServerSocketChannel的时候，在init中最后加入了ServerBootstrapAcceptor。而ServerBootstrapAcceptor就是处理客户端连接的

```java
        @Override
        @SuppressWarnings("unchecked")
        public void channelRead(ChannelHandlerContext ctx, Object msg) {
            final Channel child = (Channel) msg;

          //得到上面的NioSocketChannel
          //将一开始代码中的childHandler(new ChannelInitializer<SocketChannel>() {}传进来
            child.pipeline().addLast(childHandler);
          //设置通道配置
            setChannelOptions(child, childOptions, logger);
          //设置属性  
          setAttributes(child, childAttrs);

            try {
              //worker线程组注册channel
                childGroup.register(child).addListener(new ChannelFutureListener() {
                    @Override
                    public void operationComplete(ChannelFuture future) throws Exception {
                        if (!future.isSuccess()) {
                            forceClose(child, future.cause());
                        }
                    }
                });
            } catch (Throwable t) {
                forceClose(child, t);
            }
        }		
```

- 小结

  Server在获取到读事件后，创建了客户端的SocketChannel，并且通过pipeline通知到服务端初始化时创建的ServerBootstrapAcceptor，**通过ServerBootstrapAcceptor初始化childHandler**，使用worker组进行注册SocketChannel，并且创建客户端Channel默认的监听事件时read，然后就和服务端一样，通过默认的HeadContext进行read方法，默认都是自动读的。然后就是通过group创建新线程去工作。循环调底层的jdk方法。再通过pipeline通知到各个handler

# Pipeline

- netty是如何判断ChannelHandler类型的

  答：通过传入是inbound还是outbound

- 对于ChannelHandler的添加应该遵循什么样的顺序

  答：inbound传播是从Head结点开始的，outbound传播是从Tail节点开始的，所以需要根据需要从这两个方面选择

- 用户手动触发事件传播，不同的触发方式有什么样的区别？

  可以这里回答：ctx.channel().write()和ctx.write()的区别，ctx.channel().write()从tail节点开始传播（对于outBound事件），ctx.write()从当前节点开始传播。

## 初始化

Pipeline是在channel创建的时候进行初始化的，并且维护了双向链表，默认的添加HeadContext和TailContext，节点是AbstractChannelHandlerContext

```java
    protected AbstractChannel(Channel parent) {
        this.parent = parent;
        id = newId();
        unsafe = newUnsafe();
        pipeline = newChannelPipeline();
    }

    protected DefaultChannelPipeline(Channel channel) {
        this.channel = ObjectUtil.checkNotNull(channel, "channel");
        succeededFuture = new SucceededChannelFuture(channel, null);
        voidPromise =  new VoidChannelPromise(channel, true);

        tail = new TailContext(this);
        head = new HeadContext(this);

        head.next = tail;
        tail.prev = head;
    }
```

## ChannelHandlerContext

```java
public interface ChannelHandlerContext extends AttributeMap, ChannelInboundInvoker, ChannelOutboundInvoker {
 
    /**
     * Return the {@link Channel} which is bound to the {@link ChannelHandlerContext}.
     */
  //所在的channel
    Channel channel();
 
    /**
     * Returns the {@link EventExecutor} which is used to execute an arbitrary task.
     */
  //哪一个loop执行的
    EventExecutor executor();
 
    /**
     * The unique name of the {@link ChannelHandlerContext}.The name was used when then {@link ChannelHandler}
     * was added to the {@link ChannelPipeline}. This name can also be used to access the registered
     * {@link ChannelHandler} from the {@link ChannelPipeline}.
     */
    String name();
 
    /**
     * The {@link ChannelHandler} that is bound this {@link ChannelHandlerContext}.
     */
  //干活的业务handler
    ChannelHandler handler();
 
    /**
     * Return {@code true} if the {@link ChannelHandler} which belongs to this context was removed
     * from the {@link ChannelPipeline}. Note that this method is only meant to be called from with in the
     * {@link EventLoop}.
     */
    boolean isRemoved();
    ...
    /**
     * Return the assigned {@link ByteBufAllocator} which will be used to allocate {@link ByteBuf}s.
     */
    // 内存分配器
    ByteBufAllocator alloc();
    ...
}

```

## ChannelHandler的添加

DefaultChannelPipeline的addLast方法

```java
@Override
public final ChannelPipeline addLast(EventExecutorGroup group, String name, ChannelHandler handler) {
    final AbstractChannelHandlerContext newCtx;
    synchronized (this) {
      //检查handler是否重复，如果没@Shareable的话，不可以再添加了
        checkMultiplicity(handler);

      //使用DefaultChannelHandlerContext，内部封装了具体的业务handler
        newCtx = newContext(group, filterName(name, handler), handler);

      //连接到链表上去
        addLast0(newCtx);

        // If the registered is false it means that the channel was not registered on an eventLoop yet.
        // In this case we add the context to the pipeline and add a task that will call
        // ChannelHandler.handlerAdded(...) once the channel is registered.
        if (!registered) {
            newCtx.setAddPending();
            callHandlerCallbackLater(newCtx, true);
            return this;
        }

        EventExecutor executor = newCtx.executor();
        if (!executor.inEventLoop()) {
            callHandlerAddedInEventLoop(newCtx, executor);
            return this;
        }
    }
    callHandlerAdded0(newCtx);
    return this;
}
```

## ChannelHandler删除

```java
private AbstractChannelHandlerContext getContextOrDie(ChannelHandler handler) {
    AbstractChannelHandlerContext ctx = (AbstractChannelHandlerContext) context(handler);
    if (ctx == null) {
        throw new NoSuchElementException(handler.getClass().getName());
    } else {
        return ctx;
    }
}

//从头到尾找
    @Override
    public final ChannelHandlerContext context(ChannelHandler handler) {
        if (handler == null) {
            throw new NullPointerException("handler");
        }
 
        AbstractChannelHandlerContext ctx = head.next;
        for (;;) {
 
            if (ctx == null) {
                return null;
            }
 
            if (ctx.handler() == handler) {
                return ctx;
            }
 
            ctx = ctx.next;
        }
    }

//不删除head和tail
    private AbstractChannelHandlerContext remove(final AbstractChannelHandlerContext ctx) {
        assert ctx != head && ctx != tail;
 
        synchronized (this) {
            remove0(ctx);
 
            // If the registered is false it means that the channel was not registered on an eventloop yet.
            // In this case we remove the context from the pipeline and add a task that will call
            // ChannelHandler.handlerRemoved(...) once the channel is registered.
            if (!registered) {
           
                callHandlerCallbackLater(ctx, false);
                return ctx;
            }
 
            EventExecutor executor = ctx.executor();
            if (!executor.inEventLoop()) {
                executor.execute(new Runnable() {
                    @Override
                    public void run() {
                        callHandlerRemoved0(ctx);
                    }
                });
                return ctx;
            }
        }
        callHandlerRemoved0(ctx);
        return ctx;
    }
//真正的删除
 private static void remove0(AbstractChannelHandlerContext ctx) {
        AbstractChannelHandlerContext prev = ctx.prev;
        AbstractChannelHandlerContext next = ctx.next;
        prev.next = next;
        next.prev = prev;
    }



//通知删除
    final void callHandlerRemoved() throws Exception {
        try {
            // Only call handlerRemoved(...) if we called handlerAdded(...) before.
            if (handlerState == ADD_COMPLETE) {
                handler().handlerRemoved(this);
            }
        } finally {
            // Mark the handler as removed in any case.
          //标记handler删除了  
          setRemoved();
        }
    }



```

## Inboud事件

根据上面的描述，最终loop会绑定到pipeline中，当processSelectedKeys触发后，最后会调用到pipeline的fireChannelRead方法，找到下一个inboundHandlerContext

```java
    public ChannelHandlerContext fireChannelRead(Object msg) {
        invokeChannelRead(this.findContextInbound(), msg);
        return this;
    }


//找到下一个Inbound的handler
    private AbstractChannelHandlerContext findContextInbound() {
        AbstractChannelHandlerContext ctx = this;

        do {
            ctx = ctx.next;
        } while(!ctx.inbound);

        return ctx;
    }

//创建handlerContext
    DefaultChannelHandlerContext(DefaultChannelPipeline pipeline, EventExecutor executor, String name, ChannelHandler handler) {
      															//判断是否是inbound 
      super(pipeline, executor, name, isInbound(handler), isOutbound(handler));
        if (handler == null) {
            throw new NullPointerException("handler");
        } else {
            this.handler = handler;
        }
    }
    private static boolean isInbound(ChannelHandler handler) {
        return handler instanceof ChannelInboundHandler;
    }

//出发handler的方法，直接到用户的业务代码了
    private void invokeChannelRead(Object msg) {
        if (this.invokeHandler()) {
            try {
                ((ChannelInboundHandler)this.handler()).channelRead(this, msg);
            } catch (Throwable var3) {
                this.notifyHandlerException(var3);
            }
        } else {
            this.fireChannelRead(msg);
        }

    }

```

## outBound事件

包含bind、connect、disconnect方法、close、deregister、read、write、flush等方法，这些方法更多的是主动向用户发起的操作。

（而inBound事件更多的是事件的触发，如register、readComplete、active，比较被动的）

**outBound是从tail反过来在链表上传输的**

## ctx.channel().write()和ctx.write()的区别

ctx.channel().write()从tail节点开始传播，ctx.write()从当前节点开始传播（不包括当前节点，因为它首先会找到当前节点的下一个节点在执行write操作）。

# ByteBuf

- Pooled和Unpooled

  池化和不池化

- Unsafe和非Unsafe

  **unsafe通过操作底层unsafe的offset+index的方式去操作数据，非unsafe直接通过一个数组的下标（或者jdk底层的buffer）去操作数据**。

- Heap和Direct

  **heap是依赖一个数组，direct是依赖jdk底层的ByteBuffer**。

# ByteBuffAllocator

## UnpooledByteBufAllocator

### 分配堆内内存

```java

//如果jdk环境支持unsafe，则用unsafe操作内存空间
//heap 内存空间，底层就是一个数组
protected ByteBuf newHeapBuffer(int initialCapacity, int maxCapacity) {
    return PlatformDependent.hasUnsafe() ?
            new InstrumentedUnpooledUnsafeHeapByteBuf(this, initialCapacity, maxCapacity) :
            new InstrumentedUnpooledHeapByteBuf(this, initialCapacity, maxCapacity);
}

    public UnpooledHeapByteBuf(ByteBufAllocator alloc, int initialCapacity, int maxCapacity) {
        super(maxCapacity);

        if (initialCapacity > maxCapacity) {
            throw new IllegalArgumentException(String.format(
                    "initialCapacity(%d) > maxCapacity(%d)", initialCapacity, maxCapacity));
        }

        this.alloc = checkNotNull(alloc, "alloc");
      //声明数组
        setArray(allocateArray(initialCapacity));
        setIndex(0, 0);
    }


//创建数组
    protected byte[] allocateArray(int initialCapacity) {
        return new byte[initialCapacity];
    }

```

### 分配堆外内存

底层调用jdk的**DirectBuffer**

```java
@Override
protected ByteBuf newDirectBuffer(int initialCapacity, int maxCapacity) {
    final ByteBuf buf;
    if (PlatformDependent.hasUnsafe()) {
        buf = noCleaner ? new InstrumentedUnpooledUnsafeNoCleanerDirectByteBuf(this, initialCapacity, maxCapacity) :
                new InstrumentedUnpooledUnsafeDirectByteBuf(this, initialCapacity, maxCapacity);
    } else {
        buf = new InstrumentedUnpooledDirectByteBuf(this, initialCapacity, maxCapacity);
    }
    return disableLeakDetector ? buf : toLeakAwareBuffer(buf);
}

    public UnpooledDirectByteBuf(ByteBufAllocator alloc, int initialCapacity, int maxCapacity) {
        super(maxCapacity);
        ObjectUtil.checkNotNull(alloc, "alloc");
        checkPositiveOrZero(initialCapacity, "initialCapacity");
        checkPositiveOrZero(maxCapacity, "maxCapacity");
        if (initialCapacity > maxCapacity) {
            throw new IllegalArgumentException(String.format(
                    "initialCapacity(%d) > maxCapacity(%d)", initialCapacity, maxCapacity));
        }

        this.alloc = alloc;
        setByteBuffer(allocateDirect(initialCapacity), false);
    }

    protected ByteBuffer allocateDirect(int initialCapacity) {
        return ByteBuffer.allocateDirect(initialCapacity);
    }

    public static ByteBuffer allocateDirect(int capacity) {
        return new DirectByteBuffer(capacity);
    }

    void setByteBuffer(ByteBuffer buffer, boolean tryFree) {
        if (tryFree) {
            ByteBuffer oldBuffer = this.buffer;
            if (oldBuffer != null) {
                if (doNotFree) {
                    doNotFree = false;
                } else {
                    freeDirect(oldBuffer);
                }
            }
        }

        this.buffer = buffer;
        tmpNioBuf = null;
        capacity = buffer.remaining();
    }
    final void setByteBuffer(ByteBuffer buffer, boolean tryFree) {
        super.setByteBuffer(buffer, tryFree);
      //计算内存地址，使用Unsafe去操作
        memoryAddress = PlatformDependent.directBufferAddress(buffer);
    }

		//非 unsafe，直接调用jdk方法
    protected byte _getByte(int index) {
        return buffer.get(index);
    }

	//获取内存地址，在用unsafe去调用
    @Override
    protected byte _getByte(int index) {
        return UnsafeByteBufUtil.getByte(addr(index));
    }

	//计算地址
    final long addr(int index) {
        return memoryAddress + index;
    }


```

## PooledByteBufAllocator

### 分配堆内内存

```java
@Override
protected ByteBuf newHeapBuffer(int initialCapacity, int maxCapacity) {
    PoolThreadCache cache = threadCache.get();//取出当前线程的缓存，多线程情况下的处理
    PoolArena<byte[]> heapArena = cache.heapArena; //取出当前缓存里的arena

    final ByteBuf buf;
    if (heapArena != null) {
        buf = heapArena.allocate(cache, initialCapacity, maxCapacity);
    } else {
        buf = PlatformDependent.hasUnsafe() ?
                new UnpooledUnsafeHeapByteBuf(this, initialCapacity, maxCapacity) :
                new UnpooledHeapByteBuf(this, initialCapacity, maxCapacity);
    }

    return toLeakAwareBuffer(buf);
}



    final class PoolThreadLocalCache extends FastThreadLocal<PoolThreadCache> {
 
        @Override
        protected synchronized PoolThreadCache initialValue() {
            final PoolArena<byte[]> heapArena = leastUsedArena(heapArenas);
            final PoolArena<ByteBuffer> directArena = leastUsedArena(directArenas);
 
          //FastThreadLocal就是ThreacLocal的快速版本，首次没有取到缓存的时候，会进行初始化动作
            return new PoolThreadCache(
                    heapArena, directArena, tinyCacheSize, smallCacheSize, normalCacheSize,
                    DEFAULT_MAX_CACHED_BUFFER_CAPACITY, DEFAULT_CACHE_TRIM_INTERVAL);
        }
     ...
}


```

# ByteToMessageDecoder

抽象的字节解码器

```java
@Override
public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
    if (msg instanceof ByteBuf) {
        CodecOutputList out = CodecOutputList.newInstance();
        try {
            first = cumulation == null;
          //检查是否要扩容，默认netty实现是合并到一个ByteBuf上，需要内存复制，另一种是直接创建一个新的compositeByteBuf
            cumulation = cumulator.cumulate(ctx.alloc(),
                    first ? Unpooled.EMPTY_BUFFER : cumulation, (ByteBuf) msg);
            callDecode(ctx, cumulation, out);
        } catch (DecoderException e) {
            throw e;
        } catch (Exception e) {
            throw new DecoderException(e);
        } finally {
            try {
                if (cumulation != null && !cumulation.isReadable()) {
                    numReads = 0;
                    cumulation.release();
                    cumulation = null;
                } else if (++numReads >= discardAfterReads) {
                    // We did enough reads already try to discard some bytes so we not risk to see a OOME.
                    // See https://github.com/netty/netty/issues/4275
                    numReads = 0;
                    discardSomeReadBytes();
                }

                int size = out.size();
                firedChannelRead |= out.insertSinceRecycled();
                fireChannelRead(ctx, out, size);
            } finally {
                out.recycle();
            }
        }
    } else {
        ctx.fireChannelRead(msg);
    }
}


//默认Cumulator实现，内存复制到一个ByteBuf上

    public static final Cumulator MERGE_CUMULATOR = new Cumulator() {
        @Override
        public ByteBuf cumulate(ByteBufAllocator alloc, ByteBuf cumulation, ByteBuf in) {
            if (!cumulation.isReadable() && in.isContiguous()) {
                // If cumulation is empty and input buffer is contiguous, use it directly
                cumulation.release();
                return in;
            }
            try {
                final int required = in.readableBytes();
                if (required > cumulation.maxWritableBytes() ||
                        (required > cumulation.maxFastWritableBytes() && cumulation.refCnt() > 1) ||
                        cumulation.isReadOnly()) {
                    // Expand cumulation (by replacing it) under the following conditions:
                    // - cumulation cannot be resized to accommodate the additional data
                    // - cumulation can be expanded with a reallocation operation to accommodate but the buffer is
                    //   assumed to be shared (e.g. refCnt() > 1) and the reallocation may not be safe.
                    return expandCumulation(alloc, cumulation, in);
                }
                cumulation.writeBytes(in, in.readerIndex(), required);
                in.readerIndex(in.writerIndex());
                return cumulation;
            } finally {
                // We must release in in all cases as otherwise it may produce a leak if writeBytes(...) throw
                // for whatever release (for example because of OutOfMemoryError)
                in.release();
            }
        }
    };
```

```java
protected void callDecode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) {
    try {
        while (in.isReadable()) {
            final int outSize = out.size();

          //解析到了数据
            if (outSize > 0) {
              //通知到下一个handler
                fireChannelRead(ctx, out, outSize);
                out.clear();

                // Check if this handler was removed before continuing with decoding.
                // If it was removed, it is not safe to continue to operate on the buffer.
                //
                // See:
                // - https://github.com/netty/netty/issues/4635
                if (ctx.isRemoved()) {
                    break;
                }
            }

            int oldInputLength = in.readableBytes();
          //子类实现decode
            decodeRemovalReentryProtection(ctx, in, out);

            // Check if this handler was removed before continuing the loop.
            // If it was removed, it is not safe to continue to operate on the buffer.
            //
            // See https://github.com/netty/netty/issues/1664
            if (ctx.isRemoved()) {
                break;
            }

          //如果子类没有解析到对象
          //  1 数据不够解析道对象的
          //  2 
            if (out.isEmpty()) {
              
              //数据不够解析的，跳出while，等下一次继续
                if (oldInputLength == in.readableBytes()) {
                    break;
                } else {
                  // 解析出来的东西，不够合成一个对象的，继续解析
                    continue;
                }
            }

          // 解析出对象了，但是数据没有减少，有问题啊，明显对象你自己搞出来的，不是从数据里拿的，你有问题
            if (oldInputLength == in.readableBytes()) {
                throw new DecoderException(
                        StringUtil.simpleClassName(getClass()) +
                                ".decode() did not read anything but decoded a message.");
            }
				
          // 如果只要解析一个，直接跳出
            if (isSingleDecode()) {
                break;
            }
        }
    } catch (DecoderException e) {
        throw e;
    } catch (Exception cause) {
        throw new DecoderException(cause);
    }
}
```
