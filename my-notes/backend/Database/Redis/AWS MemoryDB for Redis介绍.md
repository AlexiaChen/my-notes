

交互式应用程序需要快速处理请求并做出响应，这个要求适用于其架构的所有组件。当您采用微服务并且您的架构由许多小型独立服务相互通信组成时，这一点变得更加重要。

因此，数据库性能对于应用程序的成功至关重要。为了将读取延迟降低到微秒级别，您可以在持久性数据库前面放置一个内存缓存。对于缓存，许多开发人员使用Redis，一个开源的内存数据结构存储库。事实上，在Stack Overflow 2021年度开发者调查中显示，Redis已经连续五年成为最受喜爱的数据库。

在AWS上实施此设置时，您可以使用Amazon ElastiCache for Redis作为完全托管的内存缓存服务，在持久性数据库服务（如Amazon Aurora或Amazon DynamoDB）之前作为低延迟缓存来减少数据丢失。然而，此设置需要您在应用程序中引入自定义代码以保持缓存与数据库同步。同时运行缓存和数据库也会产生费用。

## AWS MemoryDB for Redis介绍

今天，我很高兴地宣布 Amazon MemoryDB for Redis 已正式推出。这是一个全新的与 Redis 兼容、持久性的内存数据库。MemoryDB 可以轻松且经济高效地构建需要微秒级读取和毫秒级写入性能、具备数据持久性和高可用性要求的应用程序。

您现在可以简化架构并将 MemoryDB 作为单一主数据库，而不再使用持久性数据库前面的低延迟缓存。通过 MemoryDB，所有数据都存储在内存中，实现低延迟和高吞吐量的数据访问。MemoryDB 使用分布式事务日志，在多个可用区（AZ）上存储数据，以实现快速故障转移、数据库恢复和节点重启，并具备较高的持久性。

MemoryDB 与开源 Redis 兼容，并支持相同集合类型、参数和命令。这意味着您今天已经使用开源 Redis 的代码、应用程序、驱动程序和工具可以直接与 MemoryDB 配合使用。作为开发人员，您可以立即访问到诸如字符串、哈希表、列表、集合、有范围查询功能的有序集合（sorted sets）、位图（bitmaps）、HyperLogLogs（基数估计算法）、地理空间索引及流等多种数据结构。此外，您还可以使用内置的复制、最近最少使用（LRU）淘汰、事务和自动分区等高级功能。MemoryDB 兼容 Redis 6.2，并将在开源发布新版本时提供支持。

此时，您可能会有一个问题：MemoryDB 与 ElastiCache 相比如何？因为这两个服务都提供对 Redis 数据结构和 API 的访问：

- MemoryDB可以安全地成为您的应用程序的主要数据库，因为它提供了数据持久性和微秒级读取以及单位数毫秒级写入延迟。使用MemoryDB，您无需在数据库前添加缓存来实现交互式应用程序和微服务架构所需的低延迟。
- 另一方面，ElastiCache为读写操作提供微秒级的延迟。它非常适合缓存工作负载，可以加速从现有数据库中访问数据。ElastiCache也可以用作主要数据存储，适用于可能接受数据丢失的使用情况（例如，因为您可以快速从其他来源重建数据库）。

## 创建一个MemoryDB集群

在[MemoryDB控制台](https://console.aws.amazon.com/memorydb/home)中，我按照左侧导航窗格上的链接进入集群部分，并选择创建集群。这将打开集群设置页面，在此处我输入一个名称和描述以供该集群使用。

![[Pasted image 20231017181100.png]]

所有的MemoryDB集群都在[虚拟私有云（VPC）]([How Amazon VPC works - Amazon Virtual Private Cloud](https://docs.aws.amazon.com/vpc/latest/userguide/how-it-works.html#how-it-works-subnet))中运行。在子网组中，我通过选择一个我的VPC并提供一组子网来创建一个子网组，该集群将使用这些子网来分配其节点。

![[Pasted image 20231017181206.png]]

在集群设置中，我可以更改网络端口、控制节点和集群运行时属性的参数组、节点类型、分片数量以及每个分片的副本数量。存储在集群中的数据被分区到各个分片上。分片数量和每个分片的副本数量决定了我的集群中节点的数量。考虑到每个分片都有一个主节点加上副本，我预计这个集群将有八个节点。

为了与Redis版本兼容性，我选择了6.2版本。其他选项均保持默认，并选择下一步。

![[Pasted image 20231017181251.png]]

在高级设置的安全部分，我为子网组使用的VPC添加了默认安全组，并选择了之前创建的访问控制列表（ACL）。MemoryDB ACL基于[Redis ACL](https://redis.io/topics/acl)，并提供用户凭据和连接到集群的权限。

![[Pasted image 20231017181405.png]]

在快照部分，我将默认设置为让MemoryDB自动创建每日快照，并选择保留期为7天。

![[Pasted image 20231017181425.png]]

为了维护，我保留默认设置然后选择创建。在这个部分，我还可以提供一个Amazon Simple Notification Service (Amazon SNS)主题来接收重要的集群事件通知。[Push Notification Service - Amazon Simple Notification Service (SNS) - AWS](https://aws.amazon.com/sns/)

![[Pasted image 20231017181506.png]]

几分钟后，集群正在运行，我可以使用Redis命令行界面或任何Redis客户端进行连接。


## 参考

[Introducing Amazon MemoryDB for Redis – A Redis-Compatible, Durable, In-Memory Database Service | AWS News Blog](https://aws.amazon.com/blogs/aws/introducing-amazon-memorydb-for-redis-a-redis-compatible-durable-in-memory-database-service/)

