
这一章最容易被误读的地方是：读者往往把 Saga 当成一个"分布式事务框架"来学，装上 Eventuate Tram 或 Seata 就算完事。其实 Richardson 真正想表达的是**第一性原理——在分布式系统中，"原子性"不是被"实现"出来的，而是被"模拟"出来的**：通过一系列本地事务的可逆设计，加上精心编写的补偿逻辑，让系统在任何时刻崩溃后都能恢复到一致状态。下面我按"原理 → 章节脉络 → 工业洞见"三层展开。

## 一、第一性原理：为什么"每服务私有数据库"逼出了 Saga

要理解这一章，先回到第1章 Richardson 给出的微服务根本约束——**每个服务都拥有自己的数据库**。这个约束是微服务独立部署、独立扩展、故障隔离的基础，但它同时摧毁了传统分布式事务的前提。

在单体应用里，`createOrder()` 涉及订单、库存、支付、账务的联合更新，一个 Spring 的 `@Transactional` 就能保证要么全成要么全败——数据库帮你做了原子性、隔离性、持久性。但在微服务架构里：

- Order Service 有自己的 orders 库
    
- Inventory Service 有自己的 inventory 库
    
- Payment Service 有自己的 payments 库
    
- Kitchen Service 有自己的 tickets 库
    

**没有任何一个数据库能跨服务加锁、跨服务回滚**。经典的 2PC（两阶段提交）理论上能解决，但在微服务场景下性能损耗过大且存在单点阻塞风险，Netflix、Amazon 这类规模的系统根本不会考虑 2PC。

> 💡 我的洞见：Saga 的本质不是"实现分布式事务"，而是**承认分布式事务在微服务架构下"不可能用 ACID 实现"，转而用"本地事务 + 补偿"模拟出业务层面的原子性**。这是一种范式转换：从"依赖数据库保证原子性"，转向"依赖业务逻辑保证可逆性"。

Saga 只满足 ACD，**不满足 I（隔离性）**。这是理解整个第4章的钥匙——Richardson 在 4.3 节花了大量篇幅讲"缺乏隔离导致的问题"，正是因为 Saga 牺牲了隔离性这一 ACID 特性，才引入了三大异常（脏读、丢失更新、模糊读）。

## 二、章节脉络：从问题到两种协调模式到隔离对策

### 第一部分：微服务架构下的事务管理（4.1）

**4.1.1 对分布式事务的需求**

以 FTGO 的 `createOrder()` 为例，这一次操作需要：

- 读取 Consumer Service 的数据
    
- 写入 Order Service、Kitchen Service、Accounting Service 三个服务的数据
    

传统单体里这是一个 DB 事务；微服务架构里它被拆成了跨 4 个服务的分布式操作。**跨服务、跨数据源的业务场景，彻底打破了单库 ACID 事务的适用边界**。

**4.1.2 分布式事务的挑战**

- 2PC 性能差、有单点阻塞
    
- XA 事务在微服务多语言、多数据库背景下难以统一
    
- 网络分区、服务宕机、消息丢失都可能发生在事务进行中
    

**4.1.3 使用 Saga 模式维护数据一致性**

Saga 的核心定义：**将一个全局事务拆分为一系列连续的本地事务。每个本地事务更新其所属服务的数据库并发布事件/消息，触发下一个步骤。如果某步失败，则反向执行已成功步骤的补偿事务**。

FTGO 的 Create Order Saga 包含 6 个本地事务：

|步骤|服务|正向事务|补偿事务|类型|
|---|---|---|---|---|
|1|Order Service|createOrder()|rejectOrder()|可补偿性|
|2|Consumer Service|verifyConsumerDetails()|-|可补偿性|
|3|Kitchen Service|createTicket()|rejectTicket()|可补偿性|
|4|Accounting Service|authorizeCreditCard()|-|**关键性事务**​|
|5|Kitchen Service|approveTicket()|-|可重复性|
|6|Order Service|approveOrder()|-|可重复性|

这里 Richardson 引入了一个非常精妙的概念——**Saga 三种事务类型**：

- **可补偿性事务**：可以使用补偿事务回滚的事务（步骤 1-3）
    
- **关键性事务**：Saga 的关键节点，如果它成功，Saga 将一直运行到完成（步骤 4）
    
- **可重复性事务**：在关键性事务之后，保证成功（步骤 5-6）
    

> 📌 这个分类的工程意义：它告诉你**补偿逻辑只需要写到关键性事务之前**。关键性事务之后的步骤被假定为"必然成功"，不需要补偿。这大大降低了补偿逻辑的复杂度。

### 第二部分：Saga 的协调模式（4.2）——协同式 vs 编排式

这是本章最关键的架构决策。Richardson 给出了两种实现风格：

#### 协同式（Choreography）：事件驱动的舞蹈

**没有中央协调器**。每个服务在完成本地事务后，发布事件触发下游服务；下游服务监听事件并执行自己的事务，再发布新事件。

FTGO Create Order Saga 的协同式流程：

1. Order Service 创建 PENDING 订单 → 发布 `OrderCreated` 事件
    
2. Consumer Service 监听 → 验证消费者 → 发布 `ConsumerVerified` 事件
    
3. Kitchen Service 监听 → 创建后厨工单 → 发布 `TicketCreated` 事件
    
4. Accounting Service 监听 → 授权信用卡 → 发布 `CardAuthorized` 事件
    
5. Kitchen Service 监听 → 审批工单 → 发布 `TicketApproved` 事件
    
6. Order Service 监听 → 审批订单 → 发布 `OrderApproved` 事件
    

失败时反向：PaymentFailed → InventoryService 释放库存 → OrderService 取消订单。

**好处**：

- 服务间松耦合，没有中央单点
    
- 简单、符合直觉
    
- 横向扩展性好
    

**弊端**：

- 难以跟踪整体 Saga 状态
    
- 可能形成循环依赖
    
- 流程逻辑分散在各服务中，复杂流程难以理解和调试
    
- "事件链地狱"——当流程分支多时，无法直观看清全貌
    

#### 编排式（Orchestration）：中心化的指挥

**引入一个中央 Saga 编排器**（Orchestrator），它知道整个流程，按顺序调用各参与服务，并在失败时按反向顺序触发补偿。

FTGO Create Order Saga 的编排器伪代码逻辑：

```
createOrder():
    saga_id = generate_uuid()
    saga_store.create(saga_id, 'order_saga')
    
    try:
        # Step 1
        order = order_service.create_order(PENDING)
        saga_store.update_step(saga_id, 'order_created')
        
        # Step 2
        consumer_service.verify_consumer_details(order.consumer_id)
        saga_store.update_step(saga_id, 'consumer_verified')
        
        # Step 3
        kitchen_service.create_ticket(order)
        saga_store.update_step(saga_id, 'ticket_created')
        
        # Step 4 - 关键性事务
        payment_service.authorize_card(order)
        saga_store.update_step(saga_id, 'card_authorized')
        
        # Step 5
        kitchen_service.approve_ticket()
        saga_store.update_step(saga_id, 'ticket_approved')
        
        # Step 6
        order_service.approve_order()
        saga_store.mark_completed(saga_id)
        
    except PaymentFailed:
        # 反向补偿
        kitchen_service.reject_ticket()
        consumer_service.compensate()  # 如果有
        order_service.reject_order()
        saga_store.mark_failed(saga_id)
```

**好处**：

- 更简单的依赖，不会引入循环依赖
    
- 较少的耦合：每个服务只需实现供编排器调用的 API
    
- 改善关注点隔离：Saga 协调逻辑本地化在编排器中
    
- 流程集中管理、状态清晰、易调试
    
- 有完整的审计追踪（audit trail）
    

**弊端**：

- 编排器可能成为单点故障（SPOF）或瓶颈
    
- 存在集中过多业务逻辑的风险
    
- 服务与编排器之间有较强耦合
    

#### 工业界的决策框架

综合多方资料，我给你一个清晰的选型矩阵：

|维度|协同式|编排式|
|---|---|---|
|服务数量|2-4 个服务跳转|5+ 步骤、复杂分支|
|流程复杂度|简单线性流程|复杂业务流程、需审计|
|可视化需求|低|高（监管要求、故障排查）|
|耦合度|松耦合|与编排器紧耦合|
|典型工具|Kafka、RabbitMQ、EventBridge|Temporal、AWS Step Functions、Netflix Conductor、Camunda|

**🎯 工业界共识**：生产环境推荐编排式——流程可视化和调试能力比松耦合更重要。或者说得更精确：**2-4 步的简单 Saga 用协同式，5 步以上的复杂 Saga 用编排式**。

### 第三部分：解决隔离问题（4.3）——Saga 最深的坑

这是 Richardson 写作的精华，也是 Saga 模式在工程落地时最容易翻车的地方。

**Saga 牺牲隔离性导致的三大异常**：

1. **丢失更新**：Saga A 修改数据而没有考虑 Saga B 所做的更改，导致 B 的更新被覆盖
    
2. **脏读**：Saga B 读取了 Saga A 尚未完成（可能回滚）的更新
    
3. **模糊/不可重复读**：Saga A 在步骤 1 和步骤 4 读取同一数据，得到了不同结果，因为期间 Saga B 修改了它
    

**五大对策**：

**1. 语义锁（Semantic Lock）**

可补偿性事务在其创建或更新的记录中设置标志（如 `*_PENDING`），告诉其他 Saga 该记录正处于处理中。这是 Richardson 在 4.4 节 Create Order Saga 中实际采用的方案——Order 被创建为 `APPROVAL_PENDING` 状态。

**2. 交换式更新（Commutative Updates）**

将更新操作设计成可以按任何顺序执行而不影响最终结果。典型的例子：账户的 `debit()` 和 `credit()` 操作是可交换的——不管先扣款还是先入账，最终余额都正确。

**3. 悲观视图（Pessimistic View）**

重新排序 Saga 的步骤，把最可能失败的步骤放到最前面，最大限度降低补偿开销。比如：如果信用卡授权是最危险的操作，就在预留库存之前先做授权，这样授权失败时就不需要补偿库存预留。

**4. 重读值（Reread Values）**

在更新数据之前重新读取记录，验证它未被并发修改；如果已更改，则中止当前 Saga 步骤并可能重启。这是乐观离线锁模式的一种形式。

**5. 版本文件（Version Files）**

记录对数据执行的操作日志，以便可以对它们重新排序。典型案例：当 Accounting Service 先收到 Cancel Authorization 请求、再收到 Authorize Card 请求时，它会注意到已经收到过取消请求，从而跳过授权。

**6. 业务风险评级（Risk-based Concurrency）**

使用每个请求的业务风险来动态选择并发机制——低风险更新用 Saga，高风险更新用分布式事务。

> ⚠️ 这六大对策不是"择一而用"，而是**组合使用**。Richardson 在 Create Order Saga 中主要使用了语义锁（Order 状态为 `APPROVAL_PENDING`），但实际工业系统通常会叠加版本文件（乐观锁版本号）和交换式更新（支付与库存可交换）。

### 第四部分：Order Service 和 Create Order Saga 的设计（4.4）

这是 Richardson 给出的完整可运行示例，展示了编排式 Saga 的代码结构：

**Order Service 的双重身份**：它既是 Saga 编排器（CreateOrderSaga 类），又是 Saga 参与方（OrderService 类）。

**核心类**：

- `OrderService`：领域服务，负责创建和管理订单
    
- `CreateOrderSaga`：Saga 编排器，编排整个下单流程
    
- `OrderCommandHandlers`：适配器类，通过处理命令消息来调用 Order Service
    
- `OrderServiceConfiguration`：Spring 配置类，装配上述组件
    

**关键技术细节**：

1. **Saga 状态持久化**：`CreateOrderSaga` 的每个步骤执行前后，都需要将 Saga 状态写入持久化存储（saga log / execution journal），以便在进程崩溃后能恢复。
    
2. **命令/回复通道**：编排器通过消息代理（如 Eventuate Tram）向各服务发送命令消息，并订阅回复通道接收结果。
    
3. **补偿事务的幂等性**：每个补偿事务都必须设计成幂等的——网络重试可能导致补偿被多次调用，如果 `refund(120)` 不是幂等的，执行两次就会退款 240 元。标准做法是使用 `idempotency_key` 头（如 Stripe 的做法）。
    

## 三、工业实践视角：Saga 在生产环境的真实样貌

如果 Richardson 写书时（2018年）的 Saga 实现还需要自己写编排器、自己管状态持久化，那么 2024-2026 年的工业界已经有了成熟的"持久化执行平台"来承载 Saga：

### 1. 现代 Saga 执行平台

**Temporal**：将 Saga 写成普通的顺序代码，平台自动处理状态持久化、崩溃恢复、重试和补偿编排。工程师不需要手动写 saga log。

**AWS Step Functions**：用 JSON 定义状态机，内置重试、超时、补偿逻辑。适合 5+ 步骤的复杂 Saga。

**Netflix Conductor**：Netflix 开源的编排引擎，支撑其大规模微服务架构。

**Cadence**：Temporal 的前身，Uber 开源。

这些平台的出现，让"编排式 Saga"从"自己造轮子"变成了"用现成平台"，大大降低了落地门槛。

### 2. 工业界的关键工程经验

**幂等性是底线，不是高级特性**：

- 每个正向步骤和补偿步骤都必须接受重试
    
- 标准做法：分配唯一的 `saga_id + step_id`，在本地 DB 中存储首次执行结果，重复请求直接返回原结果
    
- Stripe 的 `Idempotency-Key` 头是教科书级实现
    

**Saga 状态必须持久化**：

- 内存编排器在进程崩溃时会丢失所有状态
    
- 生产级 Saga 必须在每步执行前将当前位置写入持久化存储
    
- 重启时从日志中恢复，或触发补偿
    

**补偿事务也可能失败**：

- 这是 Saga 模型最难解决的开放性问题
    
- 标准恢复策略：指数退避重试 → 进入死信队列（DLQ）→ 告警人工介入
    
- 对于不可逆操作（如已寄出的实体邮件、已转账到外部银行），应将其放在 Saga 最后执行，最大限度减少需要补偿的步骤
    

**何时不该用 Saga**：

- 单服务单库的简单操作 → 直接用 DB 事务
    
- 不需要回滚的简单异步操作 → 事件驱动即可
    
- 可以通过重新设计避免的分布式写入 → 重新设计领域模型
    

### 3. 真实案例：Saga 的边界

**正向案例**：电商下单（创建订单 → 预留库存 → 扣款 → 调度发货）是 Saga 的经典场景。每个步骤都是可补偿的：取消订单、释放库存、退款、取消发货。

**反向教训**：Amazon Prime Video 在 2023 年的微服务回退案例中，部分原因就是过度使用分布式协调（包括 Saga 类模式），导致系统在 5% 预期负载下就达到基础设施极限。这不是 Saga 的错，而是**在不需要分布式事务的场景强行使用 Saga**​ 的代价。

> 💡 工业界的共识：**能用领域建模避免的分布式事务，就不要使用 Saga**。Saga 是最后手段，不是首选方案。如果通过重新划分服务边界，能把"跨服务事务"变成"单服务事务"，那是更好的解。

### 4. Saga 与 CQRS 的配合

这是 Richardson 埋下的伏笔——第 4 章解决"写操作的一致性"，但 Saga 的"最终一致性"会带来一个副作用：**读取数据时可能看到中间状态**（如订单已创建但还未支付）。这直接导致第 7 章的 CQRS 模式：写服务和读服务分离，读服务通过订阅 Saga 产生的事件来维护自己的物化视图。

换句话说，**Saga 解决"写的一致性"，CQRS 解决"读的一致性"**，两者是微服务数据架构的阴阳两面。

## 四、本章的核心洞见（我自己的提炼）

读完这一章，我认为 Richardson 真正想传递的不是"Saga 怎么用"，而是四个更深层的认知：

**第一，Saga 不是分布式事务的"实现"，而是分布式事务的"模拟"。**​ 传统 ACID 事务依赖数据库的原子性和隔离性；Saga 依赖业务逻辑的"可逆性设计"——每个正向操作都必须有一个语义等价的反向操作。这意味着**补偿事务的质量直接决定业务安全性**。一个没写好的补偿事务，比没有 Saga 更危险。

**第二，协同式和编排式不是"二选一"，而是"按复杂度分级"。**​ 2-4 步的简单流程用协同式（事件驱动、松耦合）；5 步以上的复杂流程用编排式（流程可视化、易调试）。工业界越来越倾向于编排式，因为**调试和审计的价值超过了松耦合的价值**。

**第三，隔离性缺失是 Saga 的"原罪"，必须用应用层对策弥补。**​ 语义锁、交换式更新、重读值、版本文件——这些对策的本质都是**把数据库失去的隔离性，在业务逻辑层重新实现一遍**。这是 Saga 模式最大的隐性成本：你省下了分布式锁，但必须在每个服务里手写隔离逻辑。

**第四，Saga 引入了"新故障模式"。**​ 单体事务要么提交要么回滚，没有中间状态。Saga 引入了"部分完成"、"补偿中"、"补偿失败"等新状态[citation：12]。这意味着**运维团队必须监控 Saga 的执行状态，对卡住的 Saga 进行人工干预**。这是 Richardson 在书中点到但未充分展开的话题，却是工业界生产环境的事实标准。

> 💡 一个反直觉的洞见：**很多时候，"消除分布式事务"比"实现分布式事务"更有价值**。如果通过重新设计领域模型，把"订单+库存+支付"重新划分，让一次业务操作只涉及一个服务（如把库存检查变成订单服务内的本地逻辑），你就完全避开了 Saga 的复杂度。Saga 是当你无法避免跨服务写入时的最后手段，不是微服务的标配。Amazon、Walmart 等公司在实践中都遵循这一原则：**优先通过领域建模消除分布式事务，其次才用 Saga**。

