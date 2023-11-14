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



