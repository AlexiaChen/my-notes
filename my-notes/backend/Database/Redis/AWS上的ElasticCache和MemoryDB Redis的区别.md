
## ElasticCache for Redis

其实就是一个云上托管版的Redis，主要是给缓存场景用的，内存数据存储，一般放在数据库的前面，加速读操作之类的。

## MemoryDB for Redis

目标是实现缓存和数据库的一体化，也就是即作为缓存，也提供数据库的持久化的功能。

## 功能对比

#### 性能

MemoryDB支持每秒390K个读请求和100K个写请求。也就是1.3GB/s的读和100MB/s写的吞吐量(每个节点)，


#### 延迟

ElasticCache Redis 是微妙级的读写延迟。
MemoryDB Redis是微妙级的读和毫秒级的写延迟。

#### 可扩展性

都支持纵向和横向扩展。允许数据[分片]([MemoryDB nodes and shards - Amazon MemoryDB for Redis](https://docs.aws.amazon.com/memorydb/latest/devguide/nodes.nodegroups.html))  和[副本读]([Understanding MemoryDB replication - Amazon MemoryDB for Redis](https://docs.aws.amazon.com/memorydb/latest/devguide/replication.html))


#### 可用性

都支持多可用区域这样的跨区域副本。

#### 数据一致性

ElasticCache Redis是弱一致性与无界不一致窗口。  

Redis允许在每个分片的主节点上进行写入和强一致性读取，并从读取副本上进行最终一致性读取。如果主节点发生故障，这些一致性属性将无法保证，因为在故障转移期间写入可能会丢失，从而违反一致性模型。

MemoryDB Redis是主节点上的强一致性，副本节点上的最终一致性读取。  

MemoryDB的一致性模型与Redis的ElastiCache类似。然而，在MemoryDB中，数据在故障转移过程中不会丢失，使得客户端可以从主节点读取自己的写入操作，而不受节点故障的影响。只有成功持久化到多AZ事务日志中的数据才可见。副本节点仍然是最终一致性的。

#### 故障转移到读副本

自动检测和恢复缓存节点故障。在主节点故障的情况下，ElastiCache for Redis将自动将现有的读取副本提升为主节点。Redis复制系统是异步的：所有对主节点的更新在提交后进行复制。在主节点故障的情况下，已确认的更新可能会丢失。

MemoryDB支持无损故障转移。  
当主节点发生故障时，MemoryDB将自动进行故障转移，并将其中一个副本提升为新的主节点，将写入流量引导到该节点。此外，MemoryDB利用分布式事务日志确保副本上的数据保持最新，即使主节点发生故障也是如此。对于非计划中断，故障转移通常在不到10秒的时间内完成，对于计划中断，通常在不到200毫秒的时间内完成。

一句话： 就是ElasticCache在主节点出现故障时，会丢数据，不能保证主从节点的数据一致性。MmeoryDB则不会丢失数据。

#### 数据持久性

ElastiCache for Redis包括一个可选的追加日志文件（AOF）功能，它将数据持久化到主节点磁盘上的文件中，以实现持久性。当启用此功能时，节点将所有修改缓存数据的命令写入追加日志文件中。当节点重新启动并启动缓存引擎时，AOF文件会被“回放”。结果是一个包含完整数据的热Redis缓存。

AOF不能保护所有故障场景。例如，如果一个节点因底层物理服务器的硬件故障而失败，ElastiCache会在另一个服务器上预配一个新节点。在这种情况下，AOF文件将不再可用，无法用于恢复数据。


MemoryDB将您的整个数据集存储在内存中，并使用分布式多可用区域事务日志来提供数据的持久性。

一句话：就持久性保证，也是MemoryDB更强。

#### 备份和恢复

自动和手动快照



## 参考

[Amazon ElastiCache for Redis vs Amazon MemoryDB for Redis - Cloud well served](https://cloudwellserved.com/amazon-elasticache-for-redis-vs-amazon-memorydb-for-redis/)