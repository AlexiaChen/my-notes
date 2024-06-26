
今年早些时候，我们关闭了 Notion 进行五分钟的定期维护。虽然我们的公告表明了“提高稳定性和性能”，但在幕后，几个月来专注而紧急的团队合作达到了高潮：将 Notion 的 PostgreSQL 单体分片成一个水平分区的数据库队列。

虽然转换成功了，但我们仍然保持沉默，以防迁移后出现任何问题。令我们高兴的是，用户很快就开始注意到改进。

但是，单个维护窗口并不能说明全部情况。我们的团队花了几个月的时间来构建这种迁移，以使 Notion 在未来几年内更快、更可靠。

让我告诉你我们是如何分片的故事，以及我们在此过程中学到了什么。

## 决定什么时候分片

分片是我们不断提高应用程序性能的一个重要里程碑。在过去的几年里，看到越来越多的人将 Notion 融入生活的方方面面，我感到欣慰和谦卑。不出所料，所有新的公司wiki、项目跟踪器和Pokédexes都意味着要存储数十亿个新的[块、文件和空间]([[Notion灵活性背后的数据模型]])。到 2020 年年中，很明显，产品使用量将超过我们值得信赖的 Postgres 单体机的能力，后者在五年和四个数量级的增长中尽职尽责地为我们服务。值班工程师经常在数据库 CPU 峰值时醒来，简单的 [仅目录迁移](https://medium.com/paypal-tech/postgresql-at-scale-database-schema-changes-without-downtime-20d3749ed680) 变得不安全且不确定。

在分片方面，快速增长的初创公司必须进行微妙的权衡。在此期间，大量博客文章涌入 [阐述](https//www.percona.com/blog/2009/08/06/why-you-don't-want-to-shard/) [the](http://www.37signals.com/svn/posts/1509-mr-moore-gets-to-punt-on-sharding)   [perils](https://www.drdobbs.com/errant-architectures/184414966) [of](https://www.infoworld.com/article/2073449/think-twice-before-sharding.html)过早地分片：维护负担增加、应用程序级代码和架构路径中新发现的约束当然，在我们的规模上，分片是不可避免的。问题只是什么时候。

对我们来说，当 Postgres 的“VACUUM”进程开始持续停滞时，拐点就到了，这阻止了数据库从死元组中回收磁盘空间。虽然磁盘容量可以增加，但更令人担忧的是 [transaction ID （TXID） wraparound](https://blog.sentry.io/2015/07/23/transaction-id-wraparound-in-postgres) 的前景，这是一种安全机制，在这种机制中，Postgres 将停止处理所有写入以避免破坏现有数据。意识到TXID环绕会对产品构成生存威胁，我们的基础设施团队加倍努力并开始工作。

## 设计一个分片方案

如果您以前从未对数据库进行过分片，那么这个想法是：通过跨多个数据库对数据进行分区，而不是使用逐渐增加的实例垂直扩展数据库。现在，您可以轻松地启动其他主机以适应增长。不幸的是，现在您的数据位于多个位置，因此您需要设计一个在分布式环境中最大限度地提高性能和一致性的系统。

> 为什么不继续垂直扩展呢？正如我们发现的那样，使用 RDS“调整实例大小”按钮玩 Cookie Clicker 不是一个可行的长期策略——即使您有预算。查询性能和维护过程通常在表（table）达到最大硬件绑定大小之前就开始下降;我们停滞不前的 Postgres 自动吸尘器就是这种软限制的一个例子。

### 应用级的分片

我们决定实现我们自己的分区方案，并从应用程序逻辑路由查询，这种方法称为应用程序级分片。在我们最初的研究中，我们还考虑了打包的分片/集群解决方案，例如用于 Postgres 的 [Citus](https://www.citusdata.com/) 或用于 MySQL 的 [Vitess](https://vitess.io/)。虽然这些解决方案简单易用，并提供开箱即用的跨分片工具，但实际的聚类逻辑是不透明的，我们希望控制数据的分布。

应用程序级分片要求我们做出以下设计决策：

- **我们应该对哪些数据进行分片？** 我们的数据集之所以独特，部分原因在于“块”表反映了 [用户创建的内容树]([[Notion灵活性背后的数据模型]])，这些树在大小、深度和分支因子方面可能会有很大差异。例如，单个大型企业客户产生的负载比许多普通个人工作区的总和还要多。我们只想对必要的表进行分片，同时保留相关数据的位置性。
    
- 我们应该如何对数据进行分区？好的分区键可以确保元组均匀分布在分片中。分区键的选择还取决于应用程序结构，因为分布式联接的成本很高，并且事务性保证通常仅限于单个主机。
    
- **我们应该创建多少个分片？这些分片应该如何组织？** 此考虑包括每个表的逻辑分片数，以及逻辑分片与物理主机之间的具体映射。
### 决策1-对与块相关的所有数据进行可传递分片

由于 Notion 的 [数据模型]([[Notion灵活性背后的数据模型]]) 围绕着一个块的概念，每个块在我们的数据库中占据一行，因此“块”表是分片的最高优先级。但是，块可能会引用其他表，例如“空间”（工作区）或“讨论”（页面级和内联讨论线程）。反过来，“讨论”可以引用“注释”表中的行，依此类推。

我们决定通过某种外键关系对 **'block'** ** **table** 可访问的所有表进行分片。并非所有这些表都需要分片，但是如果记录存储在主数据库中，而其相关块存储在不同的物理分片上，则在写入不同的数据存储时可能会引入不一致。


> 例如，假设一个块存储在一个数据库中，相关注释存储在另一个数据库中。如果删除了块，则应更新注释，但由于事务性保证仅适用于每个数据存储，因此当注释更新失败时，块删除可能会成功。

### 决策2-按工作区 ID 对块数据进行分区

一旦我们决定了要分片的表，我们就必须将它们分开。选择一个好的分区方案很大程度上取决于数据的分布和连通性;由于 Notion 是一个基于团队的产品，我们的下一个决定是按工作区 ID 对数据进行分区。

每个工作区在创建时都会分配一个 UUID，因此我们可以将 UUID 空间划分为统一的存储桶。由于分片表中的每一行要么是一个块，要么是与一个块相关，并且每个块只属于一个工作区**，因此我们使用工作区 ID 作为_partition key_。由于用户通常一次在单个工作区中查询数据，因此我们避免了大多数跨分片联接。

### 决策3-容量预估

在决定了分区方案后，我们的目标是设计一个分片设置，以处理我们现有的数据和规模，以轻松满足我们两年的使用预测。以下是我们的一些限制：

- 实例类型：** 磁盘 I/O 吞吐量（以 [IOPS](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/CHAP_Storage.html) 为单位量化）受 AWS 实例类型和磁盘卷的限制。我们需要至少 60K 的总 IOPS 来满足现有需求，并在需要时进一步扩展。
- **物理和逻辑分片数量：** 为了保持 Postgres 的正常运行并保留 RDS 复制保证，我们设置了每个表 500 GB 和每个物理数据库 10 TB 的上限。我们需要选择多个逻辑分片和多个物理数据库，以便这些分片可以在数据库之间平均分配。
- **实例数量：** 实例越多，维护成本越高，但系统越健壮。
- **成本：** 我们希望我们的账单与数据库设置呈线性关系，并且我们希望能够灵活地分别扩展计算和磁盘空间。
    
在处理了这些数字之后，我们确定了一个由 480 个逻辑分片组成的架构，这些逻辑分片均匀分布在 32 个物理数据库中。层次结构如下所示：
- 物理数据库（共 32 个）
- 逻辑分片，表示为 Postgres [schema](https://www.postgresql.org/docs/current/ddl-schemas.html) （每个数据库 15 个，总共 480 个）   
- 'block' 表（每个逻辑分片 1 个，总共 480 个）
- “collection”表（每个逻辑分片 1 个，总共 480 个）
- “space”表（每个逻辑分片 1 个，总共 480 个）
- 等等，用于所有分片表

#### 为什么

你可能想知道，“**为什么是 480 个分片？** 我以为所有的计算机科学都是以 2 的幂完成的，这不是我认识的驱动器大小！

选择480的因素有很多：

- 2    
- 3
- 4
- 5
- 6
- 8
- 10, 12, 15, 16, 20, 24, 30, 32, 40, 48, 60, 80, 96, 120, 160, 240
    
关键是，480 可以被很多数字整除，这提供了添加或删除物理主机的灵活性，同时保持均匀的分片分布。例如，将来我们可以从 32 个主机扩展到 40 个主机，再到 48 个主机，每次都进行增量跳转。

相比之下，假设我们有 512 个逻辑分片。512 的因数都是 2 的幂，这意味着如果我们想保持分片均匀，我们将从 32 个主机跳到 64 个主机。任何 2 的幂都需要我们将物理主机的数量_加倍_以升级。选择包含很多因素的值！

![[Pasted image 20240329093609.png]]

> 我们从包含每个表的单个数据库发展到包含 32 个物理数据库的队列，每个物理数据库包含 15 个逻辑分片，每个分片包含每个分片表中的一个。我们总共有 480 个逻辑分片。

我们选择将 'schema001.block'、'schema002.block' 等构造为单独的表，而不是为每个数据库维护一个具有 15 个子表的 [partitioned](https://www.postgresql.org/docs/10/ddl-partitioning.html)'block' 表。本机分区表引入了另一段路由逻辑：

1. 应用代码：物理数据库→工作空间ID。
2. 分区表：工作空间 ID →逻辑架构。

![[Pasted image 20240329093851.png]]

保留单独的表允许我们直接从应用程序路由到特定的数据库和逻辑分片。

我们想要一个从工作区 ID 路由到逻辑分片的单一事实来源，因此我们选择单独构建表并在应用程序中执行所有路由。

## 迁移到分片

一旦我们建立了分片方案，就该实施它了。对于任何迁移，我们的一般框架是这样的：

1. **双重写入：** 传入写入同时应用于旧数据库和新数据库。
2. **回填：** 开始双重写入后，将旧数据迁移到新数据库。
3. **验证：** 保证新数据库中数据的完整性。
4. **切换：** 实际切换到新数据库。这可以以增量方式完成，例如双重读取，然后迁移所有读取。

### 带有审计日志的双写

双重写入阶段可确保新数据同时填充旧数据库和新数据库，即使新数据库尚未使用也是如此。双重写入有几种选择：

- **直接写入两个数据库：** 看似简单，但任一写入的任何问题都可能很快导致数据库之间的不一致，使这种方法对于关键路径生产数据存储来说过于不稳定。
- 逻辑复制：内置 [Postgres 功能](https://www.postgresql.org/docs/10/logical-replication.html)，使用发布/订阅模型将命令广播到多个数据库。在源数据库和目标数据库之间修改数据的能力有限。
- 审计日志和追赶脚本：** 创建审计日志表，用于跟踪迁移中表的所有写入情况。追赶过程循环访问审核日志，并将每个更新应用于新数据库，并根据需要进行任何修改。
    
我们选择了“审核日志”策略而不是逻辑复制，因为逻辑复制在 [初始快照](https://www.postgresql.org/docs/10/logical-replication-architecture.html#LOGICAL-REPLICATION-SNAPSHOT) 步骤中难以跟上“块”表写入量。

> 我们还准备并测试了反向审计日志和脚本，以防我们需要从分片切换回单体。此脚本将捕获对分片数据库的任何传入写入，并允许我们在单体上重放这些编辑。最后，我们不需要恢复，但这是我们应急计划的重要组成部分。

### 回填旧数据

一旦传入的写入成功传播到新数据库，我们就会启动回填过程来迁移所有现有数据。对于我们置备的“m5.24xlarge”实例上的所有 96 个 CPU （！），我们的最终脚本大约需要三天时间来回填生产环境。

任何值得一试的回填物都应该在写入旧数据之前**比较记录版本**，跳过具有最新更新的记录。通过按任意顺序运行追赶脚本和回填，新数据库最终将收敛以复制整体。

### 校验数据完整性

迁移的好坏取决于底层数据的完整性，因此在分片与单体系统保持同步后，我们开始了验证正确性的过程。

- 验证脚本：我们的脚本从给定值开始验证 UUID 空间的连续范围，将单体上的每条记录与相应的分片记录进行比较。由于全表扫描的成本高得令人望而却步，因此我们随机抽取了 UUID 并验证了它们的相邻范围。
- **“暗”读：** 在迁移读取查询之前，我们添加了一个标志，用于从旧数据库和新数据库获取数据（称为 [dark reading](https://slack.engineering/re-architecting-slacks-workspace-preferences-how-to-move-to-an-eav-model-to-support-scalability/)。我们比较了这些记录并丢弃了分片副本，记录了过程中的差异。引入暗读取会增加 API 延迟，但可以确保无缝切换。
    
作为预防措施，**迁移和验证逻辑由不同的人实现**。否则，有人在两个阶段犯相同错误的可能性更大，从而削弱了验证的前提。

## 学到的难点

虽然大部分分片项目都体现了 Notion 工程团队的最佳状态，但事后我们还是会重新考虑许多决定。以下是一些示例：

- **Shard.** 作为一个小团队，我们敏锐地意识到与过早优化相关的权衡。但是，我们一直等到现有数据库严重紧张，这意味着我们必须非常节俭地进行迁移，以免增加更多负载。此限制使我们无法使用 [logical replication](https://www.postgresql.org/docs/10/logical-replication.html) 进行双重写入。工作区 ID（我们的分区键）尚未填充到旧数据库中，并且 **回填此列会加剧单体上的负载**。取而代之的是，我们在写入分片时即时回填每一行，这需要自定义的追赶脚本。
- **以零停机迁移为目标。 双重写入吞吐量是我们最终切换的主要瓶颈：一旦我们关闭服务器，我们就需要让追赶脚本完成对分片的写入。如果我们再花一周时间优化脚本，在切换期间花费 <30 秒来赶上分片，那么就可以在不停机的情况下在负载均衡器级别进行热插拔。
- **引入组合主键，而不是单独的分区键。 现在，分片表中的行使用复合键：“id”，旧数据库中的主键;和 'space_id'，当前排列中的分区键。由于我们无论如何都必须进行全表扫描，因此我们可以将两个键合并到一个新列中，从而消除在整个应用程序中传递“space_ids”的需要。
    
尽管有这些假设，分片还是取得了巨大的成功。对于 Notion 用户来说，几分钟的停机时间使产品的速度明显加快。在内部，我们展示了协调的团队合作和果断的执行力，并设定了一个对时间敏感的目标。