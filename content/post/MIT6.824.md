# MIT6.824

# lecture 1

## 扩展性

我们在写一个web程序的时候，背后可能有个数据库。当我的web节点不够支持流量时，我可以增加web节点。然而数以千万的web节点满足了流量后，我的数据库只有一个node，因此database便成为了bottleneck。因此我们想通过增加机器来实现拓展，这点在现实中很难的。

## 可用性

容错，Fault Tolerance，即FT

容错有很多办法，例如写入磁盘或者replicate复制。磁盘在目前的场景下，相较于flash闪存来说效率太低。而replicate是目前的主要方案，当然是相当复杂的。

## 一致性

由于cluster带来的的多节点，可能会导致不同的client同时访问各自的server，这样会导致查询的数据不一致。也就是说这里要考虑是**Strong consistency**还是**Weak consistency**

强一致性，要求client发送一个put请求后，server需要确保其他的replicator也写入成功，这期间需要相当大的网络通信，也导致效率降低。

甚至是现在为了保证多副本的时候，考虑物理因素，会将server放在不同的地区甚至国家，在网络传输中也需要几十ms的延迟。因此目前人们通常使用弱一致系统，我们只需要更新最近的副本数据，并且只从最近的副本获取数据。

# GFS

# Vmare FT

## Raft





