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

  

