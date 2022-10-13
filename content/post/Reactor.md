---
title: "Reactor响应式编程"
date: 2022-01-11T14:43:18+08:00
tags: ['乱七八糟']
---

# Reactive Programming 响应式编程

## 为什么要响应式编程

传统的编程模式应对如今高并发，高响应的需求有很多限制。现在的web应用，越来越需要更高的并发，更快的响应。

### 每个请求都需要一个线程

传统的web请求，每个请求都是使用单个线程贯穿整个流程。如果我们用Spring的生态，对于的就是SpringMvc框架。基于servlet容器，例如tomcat。而tomcat内部会维护一个线程池去，每当请求来的时候，便会从池中分配一个thread来处理请求。这意味着，这个web程序的并发能力只能达到线程池的大小。当然为了提高并发能力，可以增加线程池的大小。但是在Java中一个线程的开销是昂贵的，通常是1MB的内存消耗。更大的线程池大小，则意味着更大的内存。

### 阻塞IO操作

IO操作，如果不做任何处理，势必会阻塞，例如查询数据库、RPC调用其他服务等。一旦发起了一个IO请求，持续等待，直到IO结束，这就是最基础的**阻塞IO**

<img src="https://pic-frank.oss-cn-beijing.aliyuncs.com/img/202207101419311.png" alt="image-20220710141917280" style="zoom:50%;" />

#### 响应时间

在实际的场景中，一次请求中可能有不止一个服务调用链。例如A服务除了要调用B、C服务，还需要操作一次数据库。如果什么都不做的话，意味着这次请求花费的时间是下面花费时间之和

- B服务的响应时间（网络延迟+B服务的处理时间）
- C服务的响应时间（网络延迟+C服务的处理时间）
- 数据的处理时间（网络延迟+处理时间）

<img src="https://pic-frank.oss-cn-beijing.aliyuncs.com/img/202207101426444.png" alt="image-20220710142622420" style="zoom:50%;" />

如果没有业务逻辑上的顺序，如果我们在代码中并发操作的话，那么消耗的时间肯定有所减少。虽然Java提供很多方便的异步回调方法，例如CompletableFutures，但是这极大的增加了代码的复杂性。没人愿意每次都这么复杂，并且并不是每个人都会想到用异步的。

### 压倒客户端

微服务中可能会出现这样的场景，A服务需要调用B一些信息，例如查一个月的订单，但是这个数据非常庞大，对于A服务来说，一次性无法处理，甚至可能出现OOM导致服务A直接挂了。

### 小结

总结上面的一些问题能发现响应式编程能带来的好处：

- 不使用一个线程处理一个请求的模式，可以使用需要更少的线程处理更多的请求
- 防止线程在进行IO操作的时候被阻塞
- 能有简洁的并发操作API
- 提供背压机制，backpress，防止客户端直接被压垮

## 什么是响应式编程

>*“In plain terms reactive programming is about non-blocking applications that are asynchronous and event-driven and require a small number of threads to scale. A key aspect of that definition is the concept of backpressure which is a mechanism to ensure producers don’t overwhelm consumers.”*

"响应式"是指响应围绕着变化的一种编程模型——例如网络组件随着IO事件变化、UI显示随着用户的鼠标点击而变化。从这一点上看，非阻塞就是响应式。因为不同于阻塞式，我们可以在数据准备好之后对通知进行响应，而非等待。

### 如何实现

简单点说就是，通过使用异步数据进行编程。还是A服务想要从B服务获取数据，用响应式编程的方式，服务A发请求到B后，立刻返回（即非阻塞和异步）。而请求的数据将作为A的异步数据流提供给A，B服务为每个数据都发送onNext 事件。当所有的数据都发送完毕的时候，则发送一个Completed事件。一旦有错误发生，则发送一个error事件，并且不会再继续发送数据了

<img src="https://pic-frank.oss-cn-beijing.aliyuncs.com/img/202207101458367.png" alt="image-20220710145808338" style="zoom:50%;" />

响应式编程使用函数式编程的格式，就行StreamAPI，这种方式可以在数据传输的过程中就行修改，一个数据流可以作为另一个的输入，数据流可以被合并、修改、和过滤。

### 响应式系统

一个响应式系统的基础就是响应式编程，基于响应式编程的特性，响应式系统得具备以下特点

- 及时响应
- 在失败的场景依然可以响应
- 在不同的负载下，依然保持响应
- 依赖异步消息传递

## 背景

### ReactiveX

>2011年微软为.Net开发了Reactive Extension library。提供了一个可以创建异步、事件驱动的编程。很快Reactive Extension在其他语言平台也都完成了实现。Java、JavaScript、Python、C++、Go。而Java的实现，是由Nefix在2014年的时候发布的1.0版本，这个实现被称为ReactiveX

### 响应式数据流的定义

随着响应式的影响，Java团队提出了响应式的规范。规范定义了带有背压功能的组件之间如何交互的。Java9通过[Flow API](https://docs.oracle.com/javase/9/docs/api/java/util/concurrent/Flow.html).采用了响应式数据流，FlowAPi的目的是提供Java的实现标准，而不是像RxJava这样，完全自己实现。

其中定义了2个接口

**发布者Publisher：**

数据的源头提供一个方法供订阅者去注册。

```java
public interface Publisher<T> {
    public void subscribe(Subscriber<? super T> s);
}
```

**订阅者Subcriber：**

```java
public interface Subscriber<T> {
    public void onSubscribe(Subscription s);
    public void onNext(T t);
    public void onError(Throwable t);
    public void onComplete();
}
```

- onSubscribe表示，Publisher会将订阅Subcription传给Subcriber
- onNext表示数据流中有一个新数据传过来了
- onError表示Publisher出异常了，并且不会再继续发送数据
- onComplete意味着数据全部发完了

**订阅Subscription：**

订阅提供了方法可以控制Publisher传数据，（即提供了背压功能）

```java
public interface Subscription {
    public void request(long n);
    public void cancel();
}
```

- request允许subscriber控制有多少数据将要被发送
- cancel允许subcriber取消publisher继续发送的数据

**过程Processor：**

如果一个实体，把数据接收了，再传给另一个实体。那么这实体既是subcriber也是Publisher

```java
public interface Processor<T, R> extends Subscriber<T>, Publisher<R> {
}
```

### Project Reactor

Spring在5.0发布后，集成了Project Reactor了。

Project Reactor是一个基于Java定义的响应数据流的非阻塞程序。他是Spring响应式生态的基础。Webflux便是基于他而实现的。

Project Reactor包含了一系列的module，最核心的module Reactor Core里提供了**Flux**和**Mono**，分别实现了之前说的JVM定义响应式数据流中的Publisher接口。

#### Flux和Mono

Flux和Mono都是Publisher，Flux定义了一个会发送0或N个数据的Publisher，而Mono定义了会发送0或1个数据的Publisher。他们都会被complete和error终止，并且他们都会调用下游的订阅者的onNext、onComplete、onError方法，除了实现JVM定义个数据流规范，他们还提供了一组可以修改数据的操作符，例如filter、map。

**简单使用**

```java
@Test
public void test(){
  Flux<String> fluxNumber = Flux.just("1", "2", "3");
  fluxNumber.subscribe(System.out::println);
}
```

just方法创建了一个Flux会发送数据1，2，3   直到有人订阅了他，我们触发了subscribe方法，然后才会打印每一个被发送的数据。Mono也是一样，区别是Mono的just方法只允许传一个参数

**操作符**

查看一下 [Flux API](https://projectreactor.io/docs/core/release/api/reactor/core/publisher/Flux.html) ，发现几乎所有的方法返回值都是Flux或者Mono，这意味着操作符是可以链式调用的。每个操作符都是向flux或者mono中新建一个operate，并将上一个Publisher作为参数从而生成的新实例。数据从最原始的Publisher不断的往下传送，逐步经过每个operate，最终一个subscirber结束了整个流程。如果没有订阅者订阅，则不会发生任何事情。

```java
    public void log(){
        Flux<String> fluxNumber = Flux.just("1", "2", "3");
        fluxNumber.log().subscribe(System.out::println);
    }
```

log操作符，会打印响应式数据流背后的操作细节

```java
15:59:42.580 [main] INFO reactor.Flux.Array.1 - | onSubscribe([Synchronous Fuseable] FluxArray.ArraySubscription)
15:59:42.581 [main] INFO reactor.Flux.Array.1 - | request(unbounded)
15:59:42.582 [main] INFO reactor.Flux.Array.1 - | onNext(1)
1
15:59:42.582 [main] INFO reactor.Flux.Array.1 - | onNext(2)
2
15:59:42.582 [main] INFO reactor.Flux.Array.1 - | onNext(3)
3
15:59:42.582 [main] INFO reactor.Flux.Array.1 - | onComplete()
```

- 常见的操作符

  | OPERATOR CATEGORY                          | EXAMPLES                                              |
  | :----------------------------------------- | :---------------------------------------------------- |
  | Creating a new sequence                    | just, fromArray, fromIterable, fromStream             |
  | Transforming an existing sequence          | map, flatMap, startWith, concatWith                   |
  | Peeking into a sequence                    | doOnNext, doOnComplete, doOnError, doOnCancel         |
  | Filtering a sequence                       | filter, ignoreElements, distinct, elementAt, takeLast |
  | Handling errors                            | onErrorReturn, onErrorResume, retry                   |
  | Working with time                          | elapsed, interval, timestamp, timeout                 |
  | Splitting a Flux                           | buffer, groupBy, window                               |
  | Going back to the synchronous world        | block, blockFirst, blockLast, toIterable, toStream    |
  | Multicasting a Flux to several Subscribers | publish, cache, replay                                |

- map

  ```java
  @Test
  void mapExample() {
      Flux<String> fluxColors = Flux.just("red", "green", "blue");
      fluxColors.map(color -> color.charAt(0)).subscribe(System.out::println);
  }
  ```

- zip

  ```java
  @Test
  void zipExample() {
      Flux<String> fluxFruits = Flux.just("apple", "pear", "plum");
      Flux<String> fluxColors = Flux.just("red", "green", "blue");
      Flux<Integer> fluxAmounts = Flux.just(10, 20, 30);
      Flux.zip(fluxFruits, fluxColors, fluxAmounts).subscribe(System.out::println);
  }
  ```

- error处理

  ```java
  @Test
  public void onErrorExample() {
      Flux<String> fluxCalc = Flux.just(-1, 0, 1)
          .map(i -> "10 / " + i + " = " + (10 / i));
      
      fluxCalc.subscribe(value -> System.out.println("Next: " + value),
          error -> System.err.println("Error: " + error));
  }
  ```

  ```java
  Next: 10 / -1 = -10
  Error: java.lang.ArithmeticException: / by zero
  ```

- onErrorReturn可以用来处理，operate中间发生的异常

  ```java
  @Test
  public void onErrorReturnExample() {
      Flux<String> fluxCalc = Flux.just(-1, 0, 1)
  	    .map(i -> "10 / " + i + " = " + (10 / i))
  		.onErrorReturn(ArithmeticException.class, "Division by 0 not allowed");
  
      fluxCalc.subscribe(value -> System.out.println("Next: " + value),
  	    error -> System.err.println("Error: " + error));
  
  }
  ```

  通常可以使用onErrorReturn来处理异常，但是如果你想再发生异常时，再发送其他的数据而非是默认的数据。则可以使用onErrorResume方法

  ```java
      @Test
      public void onErrorResumeExample() {
          Flux<String> fluxCalc = Flux.just(-1, 0, 1)
                  .map(i -> "10 / " + i + " = " + (10 / i))
                  .onErrorResume((err->{
                      if (err instanceof ArithmeticException){
                          return Flux.just("110","112");
                      }
                      return Mono.just("119");
                  }));
  
          fluxCalc.subscribe(value -> System.out.println("Next: " + value),
                  error -> System.err.println("Error: " + error));
  
      }
  ```

  ```java
  Next: 10 / -1 = -10
  Next: 110
  Next: 112
  ```

- 测试

  Project Reactor提供了test的module

  ```java
  @Test
  public void stepVerifierTest() {
      Flux<String> fluxCalc = Flux.just(-1, 0, 1)
          .map(i -> "10 / " + i + " = " + (10 / i));
  
      StepVerifier.create(fluxCalc)
          .expectNextCount(1)
          .expectError(ArithmeticException.class)
          .verify();
  }
  ```

  

# 参考

https://callistaenterprise.se/blogg/teknik/2020/05/24/blog-series-reactive-programming-part-1/
