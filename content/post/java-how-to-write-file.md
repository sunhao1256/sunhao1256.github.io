---
title: "Java How to Write File"
date: 2022-10-20T11:13:56+08:00
tags: ["java"]
---

if you need to read huge file one line by one line , how will you to achieve the program ?

**Read all bytes to memory**

```java
package io_stream;

import org.apache.tomcat.util.http.fileupload.IOUtils;

import java.io.*;

public class BufferStreamTest {
    public static void main(String[] args) throws IOException {
        // try copy large file in 30MB memory
        InputStream inputStream = new FileInputStream("/Users/zero/Downloads/googlechrome.dmg");

        long start = System.currentTimeMillis();

        // will fill heap memory
        OutputStream output = new ByteArrayOutputStream();

        inputStream.close();
        output.close();
        System.out.println("finished cost:" + (System.currentTimeMillis() - start) + "ms");
    }
}

```

you wiil get **OutofMemoryException** heap , if set JVM option -Xmx100m 

```java
/Users/zero/software/zulu8.64.0.19-ca-jdk8.0.345-macosx_aarch64/bin/java -
Connected to the target VM, address: '127.0.0.1:62181', transport: 'socket'
Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
	at java.util.Arrays.copyOf(Arrays.java:3236)
	at java.io.ByteArrayOutputStream.grow(ByteArrayOutputStream.java:118)
	at java.io.ByteArrayOutputStream.ensureCapacity(ByteArrayOutputStream.java:93)
	at java.io.ByteArrayOutputStream.write(ByteArrayOutputStream.java:153)
	at org.apache.tomcat.util.http.fileupload.IOUtils.copyLarge(IOUtils.java:171)
	at org.apache.tomcat.util.http.fileupload.IOUtils.copy(IOUtils.java:141)
	at io_stream.BufferStreamTest.main(BufferStreamTest.java:18)
Disconnected from the target VM, address: '127.0.0.1:62181', transport: 'socket'

Process finished with exit code 1

```

you need **buffer**, create a byte[] as buffer, or use **BufferedInputStream**

```java
package io_stream;

import org.apache.tomcat.util.http.fileupload.IOUtils;

import java.io.*;

public class BufferStreamTest {
    public static void main(String[] args) throws IOException {
        // try copy large file in 30MB memory
        InputStream inputStream = new FileInputStream("/Users/zero/Downloads/googlechrome.dmg");

        long start = System.currentTimeMillis();

        OutputStream output = new FileOutputStream("/Users/zero/Projects/Personal/javaDemo/tmp.txt");

        byte[] b = new byte[1024];
        while (inputStream.read(b) != -1) {
            output.write(b);
        }
        inputStream.close();
        output.close();
        System.out.println("finished cost:" + (System.currentTimeMillis() - start) + "ms");
    }
}
```

so，you must know how to do when need download hugo file in web program

by the way , Apach Common util  **IOUtils** in same way

```java
   public static int copy(InputStream input, OutputStream output) throws IOException {
        long count = copyLarge(input, output);
        return count > 2147483647L ? -1 : (int)count;
    }

    public static long copyLarge(InputStream input, OutputStream output) throws IOException {
        byte[] buffer = new byte[4096];
        long count = 0L;

        int n;
        for(int n = false; -1 != (n = input.read(buffer)); count += (long)n) {
            output.write(buffer, 0, n);
        }

        return count;
    }
```

