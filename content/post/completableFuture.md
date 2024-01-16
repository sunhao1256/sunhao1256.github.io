---
title: "CompletableFuture"
date: 2024-01-16T09:55:37+08:00
tags: ["java"]
---

# CompletableFuture

提供了快速实现异步任务的能力，带async即内部方法是异步调用的，使用的Executor默认是ForkJoin的

## Static Methods

| Modifier and Type                  | Method and Description                                       |
| :--------------------------------- | :----------------------------------------------------------- |
| `static CompletableFuture<Void>`   | `allOf(CompletableFuture<?>... cfs)`Returns a new CompletableFuture that is completed when all of the given CompletableFutures complete. |
| `static CompletableFuture<Object>` | `anyOf(CompletableFuture<?>... cfs)`Returns a new CompletableFuture that is completed when any of the given CompletableFutures complete, with the same result. |
| `static <U> CompletableFuture<U>`  | `completedFuture(U value)`Returns a new CompletableFuture that is already completed with the given value. |
| `static CompletableFuture<Void>`   | `runAsync(Runnable runnable)`Returns a new CompletableFuture that is asynchronously completed by a task running in the [`ForkJoinPool.commonPool()`](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ForkJoinPool.html#commonPool--) after it runs the given action. |
| `static CompletableFuture<Void>`   | `runAsync(Runnable runnable, Executor executor)`Returns a new CompletableFuture that is asynchronously completed by a task running in the given executor after it runs the given action. |
| `static <U> CompletableFuture<U>`  | `supplyAsync(Supplier<U> supplier)`Returns a new CompletableFuture that is asynchronously completed by a task running in the [`ForkJoinPool.commonPool()`](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ForkJoinPool.html#commonPool--) with the value obtained by calling the given Supplier. |
| `static <U> CompletableFuture<U>`  | `supplyAsync(Supplier<U> supplier, Executor executor)`Returns a new CompletableFuture that is asynchronously completed by a task running in the given executor with the value obtained by calling the given Supplier. |

## RunAfterEither

任何一个结束，执行runner

```java
    public static void main(String[] args) {
        CompletableFuture<Void> finished = CompletableFuture.supplyAsync(() -> {
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
            return "are you";
        }).runAfterEither(CompletableFuture.supplyAsync(() -> {
            try {
                Thread.sleep(2000);
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
            return "ok";
        }), () -> {
            System.out.println("finished");
        });

        try {
            Void unused = finished.get();
            System.out.println(unused);
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        } catch (ExecutionException e) {
            throw new RuntimeException(e);
        }


        LockSupport.park();
    }

```

```shell
Connected to the target VM, address: '127.0.0.1:50111', transport: 'socket'
finished
null
```

## RunAfterBothAsync

两个completableFuture都结束，异步执行runner

```java
        CompletableFuture<Void> finished = CompletableFuture.supplyAsync(() -> {
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
            System.out.println("are you thread: "+Thread.currentThread().getName());
            return "are you";
        }).runAfterBothAsync(CompletableFuture.supplyAsync(() -> {
            try {
                Thread.sleep(2000);
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
            System.out.println("ok: "+Thread.currentThread().getName());
            return "ok";
        }), () -> {
            System.out.println("runner thread: "+Thread.currentThread().getName());
            System.out.println("finished");
        });

        try {
            Void unused = finished.get();
            System.out.println(unused);
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        } catch (ExecutionException e) {
            throw new RuntimeException(e);
        }


        LockSupport.park();

```

```shell
Connected to the target VM, address: '127.0.0.1:51977', transport: 'socket'
are you thread: ForkJoinPool.commonPool-worker-3
ok: ForkJoinPool.commonPool-worker-5
runner thread: ForkJoinPool.commonPool-worker-3
finished
null

```

## [ThenCombine](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html#thenCombine-java.util.concurrent.CompletionStage-java.util.function.BiFunction-)

两个completableFuture都completed，将两个stage的return，作为BiFunction的参数，执行fn

```java
    public static void main(String[] args) {
        CompletableFuture<String> finished = CompletableFuture.supplyAsync(() -> {
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
            System.out.println("are you thread: "+Thread.currentThread().getName());
            return "are you";
        }).thenCombine(CompletableFuture.supplyAsync(() -> {
            try {
                Thread.sleep(2000);
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
            System.out.println("ok: "+Thread.currentThread().getName());
            return 1;
        }), (a,b) -> {
            System.out.println("runner thread: "+Thread.currentThread().getName());
            System.out.println("finished");
            return a+ b;
        });

        try {
            String unused = finished.get();
            System.out.println(unused);
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        } catch (ExecutionException e) {
            throw new RuntimeException(e);
        }


        LockSupport.park();
    }

```

```shell
Connected to the target VM, address: '127.0.0.1:55911', transport: 'socket'
are you thread: ForkJoinPool.commonPool-worker-3
ok: ForkJoinPool.commonPool-worker-5
runner thread: ForkJoinPool.commonPool-worker-5
finished
are you1

```

## [ThenCompose](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html#thenCompose-java.util.function.Function-)和[ThenApply](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html#thenApply-java.util.function.Function-)

很类似，区别是thenApply的参数是Fn的返回值是具体的值，而ThenCompose的参数Fn返回值还是一个CompletableFuture，所以ThenCompose可以继续再链路操作。所以他命名叫ThenCompose(然后创作，即创作新的CompletableFuture)

```java
    public static void main(String[] args) {
        CompletableFuture<String> finished = CompletableFuture.supplyAsync(() -> {
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
            System.out.println("are you thread: "+Thread.currentThread().getName());
            return "are you";
        }).thenApply(a->{
            return a+" ok";
        });

        try {
            String unused = finished.get();
            System.out.println(unused);
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        } catch (ExecutionException e) {
            throw new RuntimeException(e);
        }


        LockSupport.park();
    }

```

```java
    public static void main(String[] args) {
        CompletableFuture<String> finished = CompletableFuture.supplyAsync(() -> {
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
            System.out.println("are you thread: "+Thread.currentThread().getName());
            return "are you";
        }).thenCompose(a->{
            return CompletableFuture.supplyAsync(()->{
                return a+" ok";
            });
        });

        try {
            String unused = finished.get();
            System.out.println(unused);
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        } catch (ExecutionException e) {
            throw new RuntimeException(e);
        }


        LockSupport.park();
    }

```

```shell
are you thread: ForkJoinPool.commonPool-worker-3
are you ok

```

