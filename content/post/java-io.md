---
title: "Java Io"
date: 2023-11-10T17:14:11+08:00
tags: ["java"]
---

# IO流

## 字节流 InputStream，OutStream

字节为单位

## 字符流 Writer，Reader

字符为单位，java使用的是unicode编码，一个char字符是2个字节大小。

为什么要字符流？因为中文或者其他语言在转换为byte字节的时候，可以能乱码等原因导致的。所以需要字符流，更方便操作字符即字符串，所以我们如果只需要输出文字的话，最好用字符流

## 包装流，Buffered

包装流，其实类似代理，即例如BufferedInputStream会包装原始的InputStream，为其准备个缓存，当read的时候，会直接从系统内核，读取默认的8192的字节数组，将其填充满。

为什么要Buffer，即为什么要这个样的一个数组？***为了减少与系统的IO次数***

看下面这个例子，写一个20MB的文件

```java
public class OutputStreamDemo {
    public static void main(String[] args) {
        String filePath = "/Users/zero/Downloads/OutputStream.txt";
        File file = new File(filePath);

        try {
            FileOutputStream outputStream = new FileOutputStream(file);
            

            long start = System.currentTimeMillis();
            int total = 20 * 1000 * 1000;
            while (total > 0) {
                total--;
              //一个一个的byte写
                outputStream.write(1);
            }

            System.out.println("cost:" + (System.currentTimeMillis() - start));
            outputStream.close();

        } catch (IOException e) {
            throw new RuntimeException(e);
        }
    }
}
```

```java
Connected to the target VM, address: '127.0.0.1:57150', transport: 'socket'
cost:34082  //写了34秒
```

通过buffered包装类

```java
public class OutputStreamDemo {
    public static void main(String[] args) {
        String filePath = "/Users/zero/Downloads/OutputStream.txt";
        File file = new File(filePath);

        try {
            FileOutputStream outputStream = new FileOutputStream(file);
            BufferedOutputStream bufferedOutputStream = new BufferedOutputStream(outputStream);

            long start = System.currentTimeMillis();
            int total = 20 * 1000 * 1000;
            while (total > 0) {
                total--;
                bufferedOutputStream.write(1);
            }

            System.out.println("cost:" + (System.currentTimeMillis() - start));
            bufferedOutputStream.close();

        } catch (IOException e) {
            throw new RuntimeException(e);
        }
    }
}
```

```java
Connected to the target VM, address: '127.0.0.1:57293', transport: 'socket'
cost:205 //200毫秒
```

因为，包装类的存在，意味着这20MB的数据会分很多次写入8192的大小的字节数组，而与每次数组满了，才会与系统交互，即IO操作。因此耗时这么大的原因就是因为IO操作的存在。当然我们平时写代码不会一个字节一个字节去写，也是定义了一个1024的byte数组，其实就是实现了这个Buffered的操作罢了。

# RandomAccessFile

这个Random被大部分人翻译为“随机”，读起来是随机访问文件。听起来很奇怪，这个类的作用是，可以从某个文件的指定位置进行读或者写操作。

***Random还有任意的意思***任意访问文件，听起来就合理多了。

他的很大一个使用场景就是大文件的断点续传。

***目前RandomAccessFile大部分功能都被Java的Nio代替了***

一个简单的断点续传Demo

```java
public class RandomAccessFIleDemo {

    public static void main(String[] args) {

        String sourcePath = "/Users/zero/Downloads/source.txt";
        String targetPath = "/Users/zero/Downloads/target.txt";


        String content = "abcdefg";
        int interruptedPosition = 3;
        try {
            File sourceFile = new File(sourcePath);
            if (!sourceFile.exists())
                sourceFile.createNewFile();
            File targetFile = new File(targetPath);
            if (!targetFile.exists())
                targetFile.createNewFile();

            RandomAccessFile source = new RandomAccessFile(sourceFile, "rw");
            RandomAccessFile target = new RandomAccessFile(targetFile,"rw");
            source.writeChars(content);

            //from 0 position
            source.seek(0);

            int i=0;
            while ((i =source.read())!=-1){
                target.write(i);
                if (source.getFilePointer()==interruptedPosition){
                    throw new InterruptedException();
                }
            }

        } catch (IOException e) {
            throw new RuntimeException(e);
        } catch (InterruptedException e) {
            try {
                Thread.sleep(1000*10);
                //keep going
                File sourceFile = new File(sourcePath);
                File targetFile = new File(targetPath);
                long targetSize = targetFile.length();
                RandomAccessFile source = new RandomAccessFile(sourceFile,"r");
                RandomAccessFile target= new RandomAccessFile(targetFile,"rw");
                source.seek(targetSize);
                target.seek(targetSize);
                int i=0;
                while ((i =source.read())!=-1){
                    target.write(i);
                }
            } catch (InterruptedException ex) {
                throw new RuntimeException(ex);
            } catch (FileNotFoundException ex) {
                throw new RuntimeException(ex);
            } catch (IOException ex) {
                throw new RuntimeException(ex);
            }
        }


    }
}
```

# NIO

NIO首先要区分，IO模型中的NIO和Java中的NIO的区别。

所谓的IO模型，即Unix与内核交互的IO方式，参考https://notes.shichao.io/unp/ch6/

其中IO模型里的NIO指的是No Blocking IO，即通过调用内核的Select方法，不管是否有链接建立成功，都有返回，用01区分。然后用户程序，通过轮训不断Select来检查的，这样会带来的问题是空转，CPU负载过高，实际很少使用。

而Java的NIO指的是Java的NIO包，也叫New IO，对应的底层IO模型是IO multiplexing，即IO多路复用。

IO Multiplexing和BIO的区别是什么？与Multiple Thread BIO的区别是什么？

BIO的问题，在于每一次的请求都会产生一个线程去进行阻塞。问题是线程带来的资源开销与内核用户态切换带来的CPU消耗。

而IO Multiplexing是优点是，只需要一个线程就可以监听N个请求，即Socket，他的优点是能够链接海量的请求，而不是减少性能什么的。后续的业务操作还是得自己开线程。

Java的NIO，指的就是IO Multiplexing。不要搞错了。

## Java NIO tutorial

https://developer.ibm.com/tutorials/j-nio/

## Java NIO FileCopyDemo

```java
public class FileCopyDemo {

    public static void main(String[] args) {
        String sourcePath = "/Users/zero/Downloads/source.txt";
        String targetPath = "/Users/zero/Downloads/target.txt";
        Charset charset = Charset.forName("UTF-8");
        try {
            FileChannel channel = new FileInputStream(sourcePath).getChannel();
            FileChannel out = new FileOutputStream(targetPath).getChannel();
            ByteBuffer allocate = ByteBuffer.allocate(3);
            CharsetDecoder charsetDecoder = charset.newDecoder();
            CharsetEncoder charsetEncoder = charset.newEncoder();
            channel.position(3);
            while (channel.read(allocate) != -1) {
                allocate.flip();
                out.write(allocate);
                allocate.clear();
            }

        } catch (FileNotFoundException e) {
            throw new RuntimeException(e);
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
    }
}
```

## Java NIo EchoServer

```java
public class EchoServer {
    public static void main(String[] args) {
        try {
            Selector selector = Selector.open();

            ServerSocketChannel socketChannel = ServerSocketChannel.open();

            socketChannel.bind(new InetSocketAddress("127.0.0.1", 9900));

            socketChannel.configureBlocking(false);

            socketChannel.register(selector, SelectionKey.OP_ACCEPT);

            while (true) {
                selector.select();
                Set<SelectionKey> selectionKeys = selector.selectedKeys();
                Iterator<SelectionKey> iterator = selectionKeys.iterator();
                while (iterator.hasNext()) {
                    SelectionKey next = iterator.next();
                    if (next.isAcceptable()) {
                        SelectableChannel channel = next.channel();
                        ServerSocketChannel ssc = (ServerSocketChannel) channel;
                        SocketChannel conn = ssc.accept();
                        conn.configureBlocking(false);
                        conn.register(selector, SelectionKey.OP_READ);
                        System.out.println("someone established");

                    } else if (next.isReadable()) {
                        SocketChannel channel = (SocketChannel) next.channel();
                        ByteBuffer buffer = ByteBuffer.allocate(1024);
                        channel.read(buffer);
                        buffer.flip();
                        CharsetDecoder charsetDecoder = Charset.forName("UTF-8").newDecoder();
                        CharBuffer decode = charsetDecoder.decode(buffer);
                        String msg = decode.toString();
                        System.out.println("receive msg" + msg);
                        channel.register(selector, SelectionKey.OP_WRITE,msg);

                    } else if (next.isWritable()) {
                        SocketChannel channel = (SocketChannel) next.channel();
                        String msg = (String) next.attachment();
                        CharsetEncoder encoder = Charset.forName("UTF-8").newEncoder();
                        channel.write(encoder.encode(CharBuffer.wrap(msg)));
                        channel.register(selector, SelectionKey.OP_READ);

                    }
                    iterator.remove();
                }
            }

        } catch (IOException e) {
            throw new RuntimeException(e);
        }
    }
}

```

## Reactor Design Pattern

Reference https://gee.cs.oswego.edu/dl/cpjslides/nio.pdf

- 经典基础模式，Reactor内部完成了请求的accept和worker应该完成的事情，acceptor只是封装了一个类而已。从accpet到worker的事情，都是同一个线程。

  <img src="https://pic-frank.oss-cn-beijing.aliyuncs.com/img/202311141118705.png" alt="image-20231114111835736" style="zoom:50%;" />

- worker线程池模式，在处理Worker的工作时，使用线程池完成。

  <img src="https://pic-frank.oss-cn-beijing.aliyuncs.com/img/202311141120811.png" alt="image-20231114112003778" style="zoom:50%;" />

- Reactor线程池模式，主从Reactor，所谓的主从实际上是，mainReactor只管accept的事情，而subReactor处理read，send的事件，并且对于worker也是依然使用线程池，极大的利用cpu的多核，这里的从和主都可以是多个。

  <img src="https://pic-frank.oss-cn-beijing.aliyuncs.com/img/202311141121592.png" alt="image-20231114112108558" style="zoom:50%;" />

具体实现参考https://github.com/chuondev/reactor

#  Netty

参考https://medium.com/geekculture/a-tour-of-netty-5020ecee5494

<img src="https://pic-frank.oss-cn-beijing.aliyuncs.com/img/202311161634013.png" alt="image-20231116163412858" style="zoom:50%;" />

## NioEventLoopGroup

NioEventLoop的集合，内部维护了一个Children的数组，里面存放NioEventLoop。

默认参数是当前机器核心的2倍线程，可以传参修改。每个Thread对应一个NioEventLoop

## NioEventLoop

- NioEventLoop在构造函数中会直接调用Java的Selector.open()
- 一个NioEventLoop等于==1个Thread+1个Queue+1个Selector

## ServerBootStrap

启动整个服务

### Bind方法

- validate方法会检查childHandler是否为空，空则异常；检查childGroup是否为空，空则使用parentGroup代替

- iniAndRegister方法会完成channel创建和初始化工作。

  - channel创建，由channelFactory完成。默认情况下ChannelFactory使用的是**ReflectiveChannelFactory**，直接通过反射构建channel，参数就是channel的Class对象。因此对于服务端，我们要调用ServerBootstrap的channel方法，指定class是NioServerSocketChannel

    ```java
     .channel(NioServerSocketChannel.class)
    ```

    #### NioServerSocketChannel

    ```java
    public NioServerSocketChannel(ServerSocketChannel channel) {
            super(null, channel, SelectionKey.OP_ACCEPT);
            config = new NioServerSocketChannelConfig(this, javaChannel().socket());
        }
    ```

    这里设置了监听事件OP_ACCEPT

    因为inherit了AbstractChannel，因此在构造方法执行的时候，生成了id，unsafe类，以及pipeline

    ```java
        protected AbstractChannel(Channel parent) {
            this.parent = parent;
            id = newId();
            unsafe = newUnsafe();
            pipeline = newChannelPipeline();
        }
    ```

    newUnsafe是进行真正的java nio层面的实现

    NioServerSocketChannel对应的是**NioMessageUnsafe**

    ```java
        protected AbstractNioUnsafe newUnsafe() {
            return new NioMessageUnsafe();
        }
    ```

    pipeline是一个双向链表，并且这里把channel自身也穿进去了

    ```java
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

    **注意这里的NioServerSocketChannel的interestOp是ON_ACCEPT**

    

  - channel的init

    ```java
        void init(Channel channel) {
            setChannelOptions(channel, newOptionsArray(), logger);
            setAttributes(channel, newAttributesArray());
    
            ChannelPipeline p = channel.pipeline();
    
            final EventLoopGroup currentChildGroup = childGroup;
            final ChannelHandler currentChildHandler = childHandler;
            final Entry<ChannelOption<?>, Object>[] currentChildOptions = newOptionsArray(childOptions);
            final Entry<AttributeKey<?>, Object>[] currentChildAttrs = newAttributesArray(childAttrs);
    
            p.addLast(new ChannelInitializer<Channel>() {
                @Override
                public void initChannel(final Channel ch) {
                    final ChannelPipeline pipeline = ch.pipeline();
                    ChannelHandler handler = config.handler();
                    if (handler != null) {
                        pipeline.addLast(handler);
                    }
    
                    ch.eventLoop().execute(new Runnable() {
                        @Override
                        public void run() {
                            pipeline.addLast(new ServerBootstrapAcceptor(
                                    ch, currentChildGroup, currentChildHandler, currentChildOptions, currentChildAttrs));
                        }
                    });
                }
            });
        }
    
    ```

    设置一些参数,核心是获取channel自己的pipeline，加一个特殊的channelHandler，即ChannelInitalizer

    ```java
    ChannelPipeline addLast(ChannelHandler... handlers);
    ```

    ```java
        public final ChannelPipeline addLast(EventExecutorGroup group, String name, ChannelHandler handler) {
            final AbstractChannelHandlerContext newCtx;
            synchronized (this) {
                checkMultiplicity(handler);
    
                newCtx = newContext(group, filterName(name, handler), handler);
    
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
    
        private AbstractChannelHandlerContext newContext(EventExecutorGroup group, String name, ChannelHandler handler) {
            return new DefaultChannelHandlerContext(this, childExecutor(group), name, handler);
        }
        private void addLast0(AbstractChannelHandlerContext newCtx) {
            AbstractChannelHandlerContext prev = tail.prev;
            newCtx.prev = prev;
            newCtx.next = tail;
            prev.next = newCtx;
            tail.prev = newCtx;
        }
    
    
    ```

    增加到最后，这个增加实际上是增加AbstractHandlerContext，而且这里的addLast实际上是，增加到HEAD和TAIL中间，也就是HEAD和TAIL永远在第一和最后

  - register方法

    ```java
            ChannelFuture regFuture = config().group().register(channel);
    ```

    让channel注册到group上，可以理解对应java nio上的，channel register到selector上，即NioServerSocketChannel注册到NioEventLoopGroup上。

    NioServerSocketChannel inherited SingleThreadEventLoop，我们看下具体实现

    ```java
        @Override
        public ChannelFuture register(final ChannelPromise promise) {
            ObjectUtil.checkNotNull(promise, "promise");
            promise.channel().unsafe().register(this, promise);
            return promise;
        }
    
    ```

    可以看到是调用channel的Unsafe的register方法，并且将自己即NioServerSocketChannel传进去，并且传了一个ChannelFuture的Promise

    之前提到过NioServerSocketChannel的Unsafe是**NioMessageUnsafe**,不过register被父类**AbstractUnsafe**实现

    ```java
            public final void register(EventLoop eventLoop, final ChannelPromise promise) {
                ObjectUtil.checkNotNull(eventLoop, "eventLoop");
                if (isRegistered()) {
                    promise.setFailure(new IllegalStateException("registered to an event loop already"));
                    return;
                }
                if (!isCompatible(eventLoop)) {
                    promise.setFailure(
                            new IllegalStateException("incompatible event loop type: " + eventLoop.getClass().getName()));
                    return;
                }
    
                AbstractChannel.this.eventLoop = eventLoop;
    
                if (eventLoop.inEventLoop()) {
                    register0(promise);
                } else {
                    try {
                        eventLoop.execute(new Runnable() {
                            @Override
                            public void run() {
                                register0(promise);
                            }
                        });
                    } catch (Throwable t) {
                        logger.warn(
                                "Force-closing a channel whose registration task was not accepted by an event loop: {}",
                                AbstractChannel.this, t);
                        closeForcibly();
                        closeFuture.setClosed();
                        safeSetFailure(promise, t);
                    }
                }
            }
    ```

    这里可以看到Unsafe也把eventLoop即NioEventLoopGroup放在自己身上了，并且让eventLoop执行了一个task。

    这里的eventLoop执行一个task，实际上就是选一个NioEventLoop去执行的

    ```java
        private void execute(Runnable task, boolean immediate) {
            boolean inEventLoop = inEventLoop();
            addTask(task);
            if (!inEventLoop) {
                startThread();
                if (isShutdown()) {
                    boolean reject = false;
                    try {
                        if (removeTask(task)) {
                            reject = true;
                        }
                    } catch (UnsupportedOperationException e) {
                        // The task queue does not support removal so the best thing we can do is to just move on and
                        // hope we will be able to pick-up the task before its completely terminated.
                        // In worst case we will log on termination.
                    }
                    if (reject) {
                        reject();
                    }
                }
            }
    
            if (!addTaskWakesUp && immediate) {
                wakeup(inEventLoop);
            }
        }
        private void startThread() {
            if (state == ST_NOT_STARTED) {
                if (STATE_UPDATER.compareAndSet(this, ST_NOT_STARTED, ST_STARTED)) {
                    boolean success = false;
                    try {
                        doStartThread();
                        success = true;
                    } finally {
                        if (!success) {
                            STATE_UPDATER.compareAndSet(this, ST_STARTED, ST_NOT_STARTED);
                        }
                    }
                }
            }
        }
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
                        SingleThreadEventExecutor.this.run();
                        success = true;
                    } catch (Throwable t) {
                        logger.warn("Unexpected exception from an event executor: ", t);
                    } finally {
                        for (;;) {
                            int oldState = state;
                            if (oldState >= ST_SHUTTING_DOWN || STATE_UPDATER.compareAndSet(
                                    SingleThreadEventExecutor.this, oldState, ST_SHUTTING_DOWN)) {
                                break;
                            }
                        }
    
                        // Check if confirmShutdown() was called at the end of the loop.
                        if (success && gracefulShutdownStartTime == 0) {
                            if (logger.isErrorEnabled()) {
                                logger.error("Buggy " + EventExecutor.class.getSimpleName() + " implementation; " +
                                        SingleThreadEventExecutor.class.getSimpleName() + ".confirmShutdown() must " +
                                        "be called before run() implementation terminates.");
                            }
                        }
    
                        try {
                            // Run all remaining tasks and shutdown hooks. At this point the event loop
                            // is in ST_SHUTTING_DOWN state still accepting tasks which is needed for
                            // graceful shutdown with quietPeriod.
                            for (;;) {
                                if (confirmShutdown()) {
                                    break;
                                }
                            }
    
                            // Now we want to make sure no more tasks can be added from this point. This is
                            // achieved by switching the state. Any new tasks beyond this point will be rejected.
                            for (;;) {
                                int oldState = state;
                                if (oldState >= ST_SHUTDOWN || STATE_UPDATER.compareAndSet(
                                        SingleThreadEventExecutor.this, oldState, ST_SHUTDOWN)) {
                                    break;
                                }
                            }
    
                            // We have the final set of tasks in the queue now, no more can be added, run all remaining.
                            // No need to loop here, this is the final pass.
                            confirmShutdown();
                        } finally {
                            try {
                                cleanup();
                            } finally {
                                // Lets remove all FastThreadLocals for the Thread as we are about to terminate and notify
                                // the future. The user may block on the future and once it unblocks the JVM may terminate
                                // and start unloading classes.
                                // See https://github.com/netty/netty/issues/6596.
                                FastThreadLocal.removeAll();
    
                                STATE_UPDATER.set(SingleThreadEventExecutor.this, ST_TERMINATED);
                                threadLock.countDown();
                                int numUserTasks = drainTasks();
                                if (numUserTasks > 0 && logger.isWarnEnabled()) {
                                    logger.warn("An event executor terminated with " +
                                            "non-empty task queue (" + numUserTasks + ')');
                                }
                                terminationFuture.setSuccess(null);
                            }
                        }
                    }
                }
            });
        }
    
    ```

    execute这个方法，首先将这个task扔进了EventLoop自身维护的一个queue里。然后cas 更新loop的状态，开始这个线程。

    这个线程就是咱们之前分析的，NioEventLoopGroup内部维护的NioEventLoop对应的唯一线程。

    ```java
    SingleThreadEventExecutor.this.run();	
    
        protected void run() {
            int selectCnt = 0;
            for (;;) {
                try {
                    int strategy;
                    try {
                        strategy = selectStrategy.calculateStrategy(selectNowSupplier, hasTasks());
                        switch (strategy) {
                        case SelectStrategy.CONTINUE:
                            continue;
    
                        case SelectStrategy.BUSY_WAIT:
                            // fall-through to SELECT since the busy-wait is not supported with NIO
    
                        case SelectStrategy.SELECT:
                            long curDeadlineNanos = nextScheduledTaskDeadlineNanos();
                            if (curDeadlineNanos == -1L) {
                                curDeadlineNanos = NONE; // nothing on the calendar
                            }
                            nextWakeupNanos.set(curDeadlineNanos);
                            try {
                                if (!hasTasks()) {
                                  
                                  //开始调用java nio的select方法
                                    strategy = select(curDeadlineNanos);
                                }
                            } finally {
                                // This update is just to help block unnecessary selector wakeups
                                // so use of lazySet is ok (no race condition)
                                nextWakeupNanos.lazySet(AWAKE);
                            }
                            // fall through
                        default:
                        }
                    } catch (IOException e) {
                        // If we receive an IOException here its because the Selector is messed up. Let's rebuild
                        // the selector and retry. https://github.com/netty/netty/issues/8566
                        rebuildSelector0();
                        selectCnt = 0;
                        handleLoopException(e);
                        continue;
                    }
    
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
                } catch (CancelledKeyException e) {
                    // Harmless exception - log anyway
                    if (logger.isDebugEnabled()) {
                        logger.debug(CancelledKeyException.class.getSimpleName() + " raised by a Selector {} - JDK bug?",
                                selector, e);
                    }
                } catch (Error e) {
                    throw e;
                } catch (Throwable t) {
                    handleLoopException(t);
                } finally {
                    // Always handle shutdown even if the loop processing threw an exception.
                    try {
                        if (isShuttingDown()) {
                            closeAll();
                            if (confirmShutdown()) {
                                return;
                            }
                        }
                    } catch (Error e) {
                        throw e;
                    } catch (Throwable t) {
                        handleLoopException(t);
                    }
                }
            }
        }
    
    ```

    上面的方法就调用了真正的select方法，但我们还没register呢，回到刚刚的execute的task。task就是去register的

    ```java
                        eventLoop.execute(new Runnable() {
                            @Override
                            public void run() {
                                register0(promise);
                            }
                        });
    
     private void register0(ChannelPromise promise) {
                try {
                    // check if the channel is still open as it could be closed in the mean time when the register
                    // call was outside of the eventLoop
                    if (!promise.setUncancellable() || !ensureOpen(promise)) {
                        return;
                    }
                    boolean firstRegistration = neverRegistered;
                    doRegister();
                    neverRegistered = false;
                    registered = true;
    
                    // Ensure we call handlerAdded(...) before we actually notify the promise. This is needed as the
                    // user may already fire events through the pipeline in the ChannelFutureListener.
                    pipeline.invokeHandlerAddedIfNeeded();
    
                    safeSetSuccess(promise); //设置promise是success的
                    pipeline.fireChannelRegistered();
                    // Only fire a channelActive if the channel has never been registered. This prevents firing
                    // multiple channel actives if the channel is deregistered and re-registered.
                    if (isActive()) {
                        if (firstRegistration) {
                            pipeline.fireChannelActive();
                        } else if (config().isAutoRead()) {
                            // This channel was registered before and autoRead() is set. This means we need to begin read
                            // again so that we process inbound data.
                            //
                            // See https://github.com/netty/netty/issues/4805
                            beginRead();
                        }
                    }
                } catch (Throwable t) {
                    // Close the channel directly to avoid FD leak.
                    closeForcibly();
                    closeFuture.setClosed();
                    safeSetFailure(promise, t);
                }
            }
    
        @Override
        protected void doRegister() throws Exception {
            boolean selected = false;
            for (;;) {
                try {
                    selectionKey = javaChannel().register(eventLoop().unwrappedSelector(), 0, this);
                    return;
                } catch (CancelledKeyException e) {
                    if (!selected) {
                        // Force the Selector to select now as the "canceled" SelectionKey may still be
                        // cached and not removed because no Select.select(..) operation was called yet.
                        eventLoop().selectNow();
                        selected = true;
                    } else {
                        // We forced a select operation on the selector before but the SelectionKey is still cached
                        // for whatever reason. JDK bug ?
                        throw e;
                    }
                }
            }
        }
    
    ```

    这里可以看到，调用了java的channel , register到selector上。register完成后，通知pipeline，触发相关方法invokeHandlerAddedIfNeeded，

    **invokeHandlerAddedIfNeeded**最终会触发。Handler的，确切说是在register之前，执行一些initChannel方法，即ChannelInitializer的initChannel方法。

    和咱们之前在init NioEventSocketChannel的时候，提到的，pipeline增加了一个匿名的ChannelInitializer。

    ```java
            p.addLast(new ChannelInitializer<Channel>() {
                @Override
                public void initChannel(final Channel ch) {
                    final ChannelPipeline pipeline = ch.pipeline();
                    ChannelHandler handler = config.handler();
                    if (handler != null) {
                        pipeline.addLast(handler);
                    }
    
                    ch.eventLoop().execute(new Runnable() {
                        @Override
                        public void run() {
                            pipeline.addLast(new ServerBootstrapAcceptor(
                                    ch, currentChildGroup, currentChildHandler, currentChildOptions, currentChildAttrs));
                        }
                    });
                }
            });
    ```

    这里就会触发eventLoop执行一个任务，增加一个channelHandler即**ServerBootStrapAcceptor**，他实现了**channelRead**方法

    ```java
            @Override
            @SuppressWarnings("unchecked")
            public void channelRead(ChannelHandlerContext ctx, Object msg) {
                final Channel child = (Channel) msg;
    
                child.pipeline().addLast(childHandler);
    
                setChannelOptions(child, childOptions, logger);
                setAttributes(child, childAttrs);
    
                try {
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

  - 很明显，在可以read的时候将Channel注册到子的NioSocketEventGroup上去。

  - doBind()方法

    ```java
        private static void doBind0(
                final ChannelFuture regFuture, final Channel channel,
                final SocketAddress localAddress, final ChannelPromise promise) {
    
            // This method is invoked before channelRegistered() is triggered.  Give user handlers a chance to set up
            // the pipeline in its channelRegistered() implementation.
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
        }
    ```

    eventLoop继续执行一个task，即bind操作，最后会最后一个TailContext依次往前执行bind方法。最后会找到HeadContext，这就是为什么要从后往前，因为HeadContext的Bind方法，完成了真正的java层面的bind操作，是为了让Tail到Head中间的Handler有机会在bind的时候做一些事情，典型的责任链模式。例如LoggerHandler

    ```java
    //HeadContext 的bind方法
    public void bind(
                    ChannelHandlerContext ctx, SocketAddress localAddress, ChannelPromise promise) {
                unsafe.bind(localAddress, promise);
    }
    ```

    这里的Unsafe是Pipeline一起绑定的，NioServerSocketChannel的Unsafe，对应着NioMessageUnsafe

    ```java
            public final void bind(final SocketAddress localAddress, final ChannelPromise promise) {
                assertEventLoop();
    
                if (!promise.setUncancellable() || !ensureOpen(promise)) {
                    return;
                }
    
                // See: https://github.com/netty/netty/issues/576
                if (Boolean.TRUE.equals(config().getOption(ChannelOption.SO_BROADCAST)) &&
                    localAddress instanceof InetSocketAddress &&
                    !((InetSocketAddress) localAddress).getAddress().isAnyLocalAddress() &&
                    !PlatformDependent.isWindows() && !PlatformDependent.maybeSuperUser()) {
                    // Warn a user about the fact that a non-root user can't receive a
                    // broadcast packet on *nix if the socket is bound on non-wildcard address.
                    logger.warn(
                            "A non-root user can't receive a broadcast packet if the socket " +
                            "is not bound to a wildcard address; binding to a non-wildcard " +
                            "address (" + localAddress + ") anyway as requested.");
                }
    
                boolean wasActive = isActive();
                try {
                    doBind(localAddress);
                } catch (Throwable t) {
                    safeSetFailure(promise, t);
                    closeIfClosed();
                    return;
                }
    
                if (!wasActive && isActive()) {
                    invokeLater(new Runnable() {
                        @Override
                        public void run() {
                            pipeline.fireChannelActive();
                        }
                    });
                }
    
                safeSetSuccess(promise);
            }
    
    
    //最后被NioServerSocketChannel自己实现了。
    
        @SuppressJava6Requirement(reason = "Usage guarded by java version check")
        @Override
        protected void doBind(SocketAddress localAddress) throws Exception {
            if (PlatformDependent.javaVersion() >= 7) {
                javaChannel().bind(localAddress, config.getBacklog());
            } else {
                javaChannel().socket().bind(localAddress, config.getBacklog());
            }
        }
    ```

## ServerBootstrapAcceptor

在前面我们分析过，在channel register到loop上的时候，会触发channelInitializer的initChannel方法，内部在pipeline上增加了一个handler，即ServerBootstrapAcceptor。

在执行完以上的工作之后，task任务会为空，所以会走到select()代码中，并阻塞，等待请求的到来。

```java
        @Override
        @SuppressWarnings("unchecked")
        public void channelRead(ChannelHandlerContext ctx, Object msg) {
            final Channel child = (Channel) msg;

            child.pipeline().addLast(childHandler);

            setChannelOptions(child, childOptions, logger);
            setAttributes(child, childAttrs);

            try {
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

从这个ChannelRead的方法，即可看出他的意义就是，让有read的event时候，将得到的ChildChannel，即NioSocketChannel，注册到childGroup上。

我们回到之前Select那边，最后会走到Unsafe的read方法，还是调用的java层面的read

```java
    private final class NioMessageUnsafe extends AbstractNioUnsafe {

        private final List<Object> readBuf = new ArrayList<Object>();

        @Override
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
                  //通知读取数据了
                    pipeline.fireChannelRead(readBuf.get(i));
                }
                readBuf.clear();
                allocHandle.readComplete();
              //通知读取完毕
                pipeline.fireChannelReadComplete();

                if (exception != null) {
                    closed = closeOnReadError(exception);

                    pipeline.fireExceptionCaught(exception);
                }

                if (closed) {
                    inputShutdown = true;
                    if (isOpen()) {
                        close(voidPromise());
                    }
                }
            } finally {
                // Check if there is a readPending which was not processed yet.
                // This could be for two reasons:
                // * The user called Channel.read() or ChannelHandlerContext.read() in channelRead(...) method
                // * The user called Channel.read() or ChannelHandlerContext.read() in channelReadComplete(...) method
                //
                // See https://github.com/netty/netty/issues/2254
                if (!readPending && !config.isAutoRead()) {
                    removeReadOp();
                }
            }
        }
    }
```

在fireChannelRead的时候，会通知所有handler执行channelRead的方法。并生成NioSocketChannel，将SocketChannel封装在里面，传到handler那边。

```java
NioServerScoketChannel.java
  
@Override
    protected int doReadMessages(List<Object> buf) throws Exception {
        SocketChannel ch = SocketUtils.accept(javaChannel());

        try {
            if (ch != null) {
                buf.add(new NioSocketChannel(this, ch));
                return 1;
            }
        } catch (Throwable t) {
            logger.warn("Failed to create a new channel from an accepted socket.", t);

            try {
                ch.close();
            } catch (Throwable t2) {
                logger.warn("Failed to close a socket.", t2);
            }
        }

        return 0;
    }

```

可以看到NioServerSocketChannel，传到handler那边的buf的list，里面放的就是NioSocketChannel，实现了所谓的NioSocketChannel注册到SubReactor。

**一旦channel注册到了EventLoop上就会开启一个线程，进行select，是一个for(;;)循环**

## Write

只是将要写入socket的数据进行一个保存，真正执行写入是Flush

## Flush

真正调用Java SocketChannel进行write方法的地方。

<img src="https://pic-frank.oss-cn-beijing.aliyuncs.com/img/202311171828890.png" style="zoom:50%;" />
