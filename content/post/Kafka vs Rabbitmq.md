# Kafka vs Rabbitmq

## difference

> reference https://tunghatbh.medium.com/kafka-vs-rabbitmq-a0ad83d0a820

- rabbtimq used to delivery message between different application, it can make sure message recevied. just like post office
- kafka use pull mechanism , so it doesn't guarantee the message whether consumed. but it supply durable ability , so consumer can re receive message via offset . hence, kafa just like book store.

- Kafka is made of topics, not queues (unlike RabbitMQ, ActiveMQ, SQS). Queues are meant to scale to millions of consumers and to delete messages once processed. In Kafka data is not deleted once processed and consumers cannot scale beyond the number of partitions in a topic.