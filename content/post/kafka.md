---
title: "Kafka"
date: 2024-07-04T10:18:22+08:00
tags: []
---

- difference between kafka and rabbitmq

  > reference https://tunghatbh.medium.com/kafka-vs-rabbitmq-a0ad83d0a820

  - rabbtimq used to delivery message between different application, it can make sure message recevied. just like post office
  - kafka use pull mechanism , so it doesn't guarantee the message whether consumed. but it supply durable ability , so consumer can re receive message via offset . hence, kafa just like book store.

  - Kafka is made of topics, not queues (unlike RabbitMQ, ActiveMQ, SQS). Queues are meant to scale to millions of consumers and to delete messages once processed. In Kafka data is not deleted once processed and consumers cannot scale beyond the number of partitions in a topic.

- kafka的核心组件相互直接的关系与作用

- kafka的consumer group是什么？怎么实现的？

- kafka如何保证消息顺序？消息不丢失？消息不重复消费 ？

- kafka为什么吞吐高延迟又低

- kafka是怎么实现的事务

- zookeeper在kafka中扮演什么角色，做了哪些事情，与kRaft模式的区别

- kafka说的这个日志是什么，随机写入和顺序写入的区别？java的RandomAccessFile 干什么的，java的NIO包又是干什么的？

- kafka的集群架构是什么样子的？

