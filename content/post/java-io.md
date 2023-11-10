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
