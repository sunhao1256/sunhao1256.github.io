---
title: "Mpush"
date: 2024-01-16T09:24:18+08:00
tags: ["Netty"]
---

# Mpush

```shell
.
├── bin
├── conf
├── logs
├── mpush-api
├── mpush-boot
├── mpush-cache
├── mpush-client
├── mpush-common
├── mpush-core
├── mpush-monitor
├── mpush-netty
├── mpush-test
├── mpush-tools
├── mpush-zk
└── target
```

## mpush-api

```shell
mpush-api/src/main/java/com/mpush/api/
├── common
├── connection
├── event
├── message
├── protocol
├── push
├── router
├── service
├── spi
│   ├── client
│   ├── common
│   ├── core
│   ├── handler
│   ├── net
│   ├── push
│   └── router
└── srd
```

### service

```shell
mpush-api/src/main/java/com/mpush/api/service
├── BaseService.java
├── Client.java
├── FutureListener.java
├── Listener.java
├── Server.java
├── Service.java
└── ServiceException.java
```

- Service.java

  ```java
  public interface Listener {
      void onSuccess(Object... args);
  
      void onFailure(Throwable cause);
  }
  public interface Service {
  
      void start(Listener listener);
  
      void stop(Listener listener);
  
      CompletableFuture<Boolean> start();
  
      CompletableFuture<Boolean> stop();
  
      boolean syncStart();
  
      boolean syncStop();
  
      void init();
  
      boolean isRunning();
  
  }
  
  ```

- FutureListener

  主要是利用他isDone的能力，并且提供了一个monitor，去异步检查是否已经start

  利用join来控制是否同步启动service

  ```java
  public class FutureListener extends CompletableFuture<Boolean> implements Listener {
      private final Listener listener;
      private final AtomicBoolean started;
  
      public FutureListener(AtomicBoolean started) {
          this.started = started;
          this.listener = null;
      }
  
      public FutureListener(Listener listener, AtomicBoolean started) {
          this.listener = listener;
          this.started = started;
      }
  
      @Override
      public void onSuccess(Object... args) {
          if (isDone()) return;// 防止Listener被重复执行
          complete(started.get());
          if (listener != null) listener.onSuccess(args);
      }
  
      @Override
      public void onFailure(Throwable cause) {
          if (isDone()) return;// 防止Listener被重复执行
          completeExceptionally(cause);
          if (listener != null) listener.onFailure(cause);
          throw cause instanceof ServiceException
                  ? (ServiceException) cause
                  : new ServiceException(cause);
      }
  
      /**
       * 防止服务长时间卡在某个地方，增加超时监控
       *
       * @param service 服务
       */
      public void monitor(BaseService service) {
          if (isDone()) return;// 防止Listener被重复执行
          runAsync(() -> {
              try {
                  this.get(service.timeoutMillis(), TimeUnit.MILLISECONDS);
              } catch (Exception e) {
                  this.onFailure(new ServiceException(String.format("service %s monitor timeout", service.getClass().getSimpleName())));
              }
          });
      }
  
      @Override
      public boolean cancel(boolean mayInterruptIfRunning) {
          throw new UnsupportedOperationException();
      }
  
  }
  ```

- Server

  service的一种具像，用于区分Server和Service

  ```java
  public interface Server extends Service {
  
  }
  ```

## mpush-netty

