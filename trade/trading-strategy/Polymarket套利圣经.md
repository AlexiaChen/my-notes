
原文标题: The Math Needed for Trading on Polymarket (Complete Roadmap)

原文作者：Roan，加密分析师

翻译、注释：MrRyanChi，insiders.bot


## 序言

在创立 @insidersdotbot 的过程中，我跟不少高频做市团队和套利团队深有过深度的交流，其中，最大的一个需求，就是怎么做套利策略。

我们的用户，朋友，合作伙伴，都在探索着 Polymarket 套利这一条复杂且多维度的交易路线。如果你是一个推特的活跃用户，那么我相信你也曾经刷到过「我通过 XX 套利策略，从预测市场上赚了多少钱」这样的推文。

然而，大部分文章都过度简化了套利的底层逻辑，让套利变成了「我上我也行」，「用 Clawdbot 就能解决」的交易模式，而并没有去详细解释怎样系统性地去理解并开发属于自己的套利系统。

如果你想理解 Polymarket 上的套利工具是怎么赚到钱，这篇文章，是我目前看到的最完整的解读。

由于英文原文有很多过于技术性，需要进行进一步研究的部分，我帮大家进行了重构和补充，方便大家只需要这一篇文章，不需要停下来查资料，就可以理解全部重点内容。

### Polymarket套利并不是简单的数学问题

你在 Polymarket 上看到一个市场：

YES 价格 $0.62，NO 价格 $0.33。

你心想：0.62 + 0.33 = 0.95，不到 1 块钱，有套利空间！同时买 YES 和 NO，花 $0.95，无论结果如何都能拿回 $1.00，净赚 $0.05。

你是对的。

**但问题是——当你还在手动算这道加法题的时候，量化系统已经在做一件完全不同的事。**

它们在同时扫描 17,218 个条件，跨越 2^63 种可能的结果组合，在毫秒级别内找到所有定价矛盾。**等你下完两笔订单，价差已经消失了**。系统早就在几十个相关市场里找到了同样的漏洞，算好了考虑订单簿深度和手续费之后的最优仓位大小，并行执行了所有交易，然后把资金转向了下一个机会。[1]

**差距不只是速度。是数学基础设施。**

## 第一章-为什么"加法"不够用-----边际多面体问题

单一市场谬误

先看一个简单的例子。

**市场 A：「特朗普会赢下宾夕法尼亚洲的选举吗？」**

YES 价格 $0.48，NO 价格 $0.52。加起来正好 $1.00。

看起来完美，没有套利空间，对吧？

> 错，加一个市场，问题就来了

**再看市场 B：「共和党会在宾夕法尼亚洲超越对手 5 个百分点以上吗？」**

YES 价格 $0.32，NO 价格 $0.68。加起来也是 $1.00。

两个市场各自都「正常」。**但这里有一个逻辑依赖关系**：

美国总统大选不是全国一起数票，而是按州计票。每个州是一个独立的「战场」，谁在这个州拿到更多选票，谁就赢走这个州所有的选举人票（赢者通吃）。特朗普是共和党候选人。所以「共和党在宾夕法尼亚赢」和「特朗普在宾夕法尼亚赢」——是同一件事。如果共和党赢了对手 5 个百分点以上，那不仅意味着特朗普赢了宾夕法尼亚，而且赢得很大。

> **换句话说，市场 B 的 YES（共和党大胜）是市场 A 的 YES（特朗普获胜）的一个子集——大胜一定意味着获胜，但获胜不一定意味着大胜。**

而这种逻辑依赖，就创造了套利机会。

> 这就像是你在赌两件事——「明天会下雨吗」和「明天会有雷暴吗」。

> 如果有雷暴，那一定在下雨（雷暴是下雨的子集）。所以「雷暴 YES」的价格不可能比「下雨 YES」的价格高。如果市场定价违反了这个逻辑，你就可以同时买低卖高，**赚到「无风险利润」，这就是套利。**

指数爆炸：为什么暴力搜索行不通

对于任何有 n 个条件的市场，理论上有 2^n 种可能的价格组合。

听起来还行？来看一个真实案例。

**2010 年 NCAA 锦标赛市场 [2]：63 场比赛，每场有赢/输两种结果。可能的结果组合数是 2^63 = 9,223,372,036,854,775,808——超过 9 百亿亿种。市场上有 5000 多个盘口**。

2^63 这个数字有多大？如果你每秒检查 10 亿种组合，需要大约 292 年 才能全部检查完。这就是为什么「暴力搜索」在这里完全行不通。

> **逐一检查每种组合？计算上不可能。**

再看 2024 年美国大选。研究团队发现了 1,576 对可能存在依赖关系的市场对。如果每对市场各有 10 个条件，那每对需要检查 2^20 = 1,048,576 种组合。乘以 1,576 对。**你的笔记本电脑算完的时候，选举结果早就出来了**。

### 整数规划-用约束代替枚举

量化系统的解决方案不是「更快地枚举」，而是根本不枚举。

**它们用整数规划（Integer Programming）来描述「哪些结果是合法的」。**

来看一个真实例子。Duke 对 Cornell 的比赛市场：每支球队有 7 个盘口（0 到 6 场胜利），总共 14 个条件，2^14 = 16,384 种可能组合。

但有一个约束：它们不可能都赢 5 场以上，因为那样它们会在半决赛相遇（只有一个能晋级）。

**整数规划怎么处理？三条约束就够了：**

> 约束一： Duke 的 7 个盘口里，恰好有一个为真（Duke 只能有一个最终胜场数）。

> · 约束二： Cornell 的 7 个盘口里，恰好有一个为真。

> · 约束三： Duke 赢 5 场 + Duke 赢 6 场 + Cornell 赢 5 场 + Cornell 赢 6 场 ≤ 1（它们不能同时赢那么多）。

**三条线性约束，替代了 16,384 次暴力检查。**

![[Pasted image 20260318094205.png]]

暴力搜索 vs 整数规划

换言之，暴力搜索就像是把字典里的每个单词都读一遍来找一个词。整数规划就像是直接翻到那个字母开头的页面。你不需要检查所有可能性，**你只需要描述「合法答案长什么样」，然后让算法去找违反规则的定价。**

真实数据：41% 的市场存在套利 [2]

原文中提到，研究团队分析了 2024 年 4 月到 2025 年 4 月的数据：

• 检查了 17,218 个条件

• 其中 7,051 个条件存在单一市场套利（占 41%）

• 中位数定价偏差：$0.60（应该是 $1.00）

• 13 对确认的跨市场可利用套利

中位数偏差 $0.60 意味着市场经常性地偏离 40%。这不是「接近有效」，这是「大规模可利用」。

## 第二章-Bregman投影-怎么算出最优套利交易

发现套利是一个问题。算出最优的套利交易是另一个问题。

你不能简单地「取个平均」或者「微调一下价格」。**你需要把当前的市场状态投影到无套利的合法空间上，同时保留价格里的信息结构。**

为什么「直线距离」不行

最直觉的想法是：找到离当前价格最近的「合法价格」，然后交易差价。

用数学语言说，就是最小化欧几里得距离：||μ - θ||²

但这有一个致命问题：它把所有价格变动当成一样的。

从 $0.50 涨到 $0.60，和从 $0.05 涨到 $0.15，都是涨了 10 美分。**但它们的信息含量完全不同。**

> 为什么？**因为价格代表的是隐含概率。**从 50% 变到 60%，是一个温和的观点调整。从 5% 变到 15%，是一个巨大的信念翻转——一个几乎不可能的事件突然变成了「有点可能」。

> 想象你在称体重。从 70 公斤变到 80 公斤，你会说「胖了一点」。但从 30 公斤变到 40 公斤（如果你是成年人），那就是「从濒死变成了严重营养不良」。同样是 10 公斤的变化，**意义完全不同**。**价格也是一样——越接近 0 或 1 的价格变动，信息量越大。**

### Bregmain散度-正确的“距离”

**Polymarket 的做市商使用的是 LMSR（对数市场评分规则）[4]，价格本质上代表概率分布。**

在这种结构下，正确的距离度量不是欧几里得距离，而是 Bregman 散度。[5]

对于 LMSR，Bregman 散度就变成了 KL 散度（Kullback-Leibler 散度)[6]——一个衡量两个概率分布之间「信息论距离」的指标。

**你不需要记住公式。你只需要理解一件事：**

KL 散度会自动给「极端价格附近的变动」更高的权重。从 $0.05 到 $0.15 的变动，在 KL 散度下比从 $0.50 到 $0.60 的变动「更远」。**这正好符合我们的直觉——极端价格的变动意味着更大的信息冲击。**

一个比较好的例子，就是上次 @zachxbt 的预测市场中，Axiom 在最后关头反超 Meteora，也是以极端价格变动，作为一切变化的。

![[Pasted image 20260318094618.png]]

### 套利利润 = Bregman投影的距离

这是原文作者参考整篇论文最核心的结论之一：

> 任何交易能获得的最大保证利润，等于当前市场状态到无套利空间的 Bregman 投影距离。

换人话说：市场价格偏离”合法空间”越远，能赚的钱越多。而 Bregman 投影会告诉你：

> 1. 该买卖什么（投影方向告诉你交易方向）

> 2. 该买卖多少（考虑订单簿深度）

> 3. 能赚多少（投影距离就是最大利润）

排名第一的套利者一年赚了 $2,009,631.76 [2]。他的策略就是比所有人更快、更准地解这道优化题。

![[Pasted image 20260318094840.png]]

> 打个比方来说，想象你站在一座山上，山脚下有一条河（无套利空间）。你现在的位置（当前市场价格）离河有一段距离。

> Bregman 投影就是帮你找到”从你的位置到河边的最短路径”——但不是直线距离，而是考虑了地形（市场结构）之后的最短路径。这条路径的长度，就是你能赚到的最大利润。

## 第三章-Frank-Wolfe算法----让理论变成可执行的代码

好，现在你知道了：要算最优套利，就要做 Bregman 投影。

但问题是——直接计算 Bregman 投影是不可行的。

为什么？因为无套利空间（边际多面体 M）有指数级多的顶点。标准的凸优化方法需要访问完整的约束集，也就是枚举每一个合法结果。我们刚才说了，这在规模化场景下是不可能的。

### Frank-Wolfe的核心思想

Frank-Wolfe 算法 [7] 的天才之处在于：它不试图一次性搞定整个问题，而是一步一步逼近答案。

它的工作方式是这样的：

>  第一步： 从一个小的已知合法结果集合开始。

> 第二步： 在这个小集合上做优化，找到当前最优解。

> 第三步： 用整数规划找到一个新的合法结果，加入集合。

> 第四步： 检查是否足够接近最优解。如果不够，回到第二步。

每一轮迭代，集合只增加一个顶点。即使跑了 100 轮，你也只需要追踪 100 个顶点——而不是 2^63 个。

![[Pasted image 20260318095134.png]]

想象你在一个巨大的迷宫里找出口。

暴力方法是把每条路都走一遍。Frank-Wolfe 的方法是：先随便走一条路，然后在每个岔路口问一个”向导”（整数规划求解器）：“从这里开始，哪个方向最可能通向出口？”然后朝那个方向走一步。你不需要探索整个迷宫，只需要在每个关键节点做出正确的选择。

### 整数规划求解器-每一步的"向导"

Frank-Wolfe 的每一轮迭代都需要解一个整数线性规划问题。这在理论上是 NP 困难的（也就是”没有已知的快速通用算法”）。

但现代求解器，比如 Gurobi [8]，对于结构良好的问题可以高效求解。

研究团队用的是 Gurobi 5.5。实际求解时间 [2]：

• 早期迭代（少量比赛已结束）：不到 1 秒

• 中期（30-40 场比赛已结束）：10-30 秒

• 后期（50+ 场比赛已结束）：不到 5 秒

为什么后期反而更快？ 因为随着比赛结果确定，可行解空间在缩小。变量更少，约束更紧，求解更快。

![[Pasted image 20260318095249.png]]

### 梯度爆炸问题和Barrier Frank-Wolfe

标准的 Frank-Wolfe 有一个技术问题：当价格接近 0 的时候，LMSR 的梯度会趋向负无穷。这会导致算法不稳定。

解决方案是 Barrier Frank-Wolfe：不在完整的多面体 M 上优化，而是在一个稍微”收缩”的版本 M’ 上优化。收缩参数 ε 会随着迭代自适应地减小——开始时离边界远一点（稳定），后来逐渐逼近真实边界（精确）。

研究表明，实际操作中 50 到 150 轮迭代就足够收敛 [2]。

### 真实表现

论文里有一个关键发现 [2]：

在 NCAA 锦标赛的前 16 场比赛中，Frank-Wolfe 做市商（FWMM）和简单的线性约束做市商（LCMM）表现差不多——因为整数规划求解器还太慢。

> 但在 45 场比赛结束后，第一次成功的 30 分钟投影完成了。

> 从那以后，FWMM 在盘口定价上比 LCMM 好了 38%。

转折点就是：当结果空间缩小到整数规划能在交易时间窗口内完成求解的时候。

FWMM 就像一个学生，考试前半段还在热身，但一旦进入状态，就开始碾压。LCMM 是那个一直稳定发挥但天花板有限的学生。关键区别是：FWMM 有更强的”武器”（Bregman 投影），只是需要时间来”装弹”（等求解器跑完）。

## 第四章-执行-----为什么算出来了还可能亏钱

你检测到了套利。你用 Bregman 投影算出了最优交易。

现在你需要执行。

这是大多数策略失败的地方。

### 非原子执行问题

Polymarket 使用的是 CLOB（中央限价订单簿） [9]。跟去中心化交易所不同，CLOB 上的交易是顺序执行的——你不能保证所有订单同时成交。

你的套利计划：

> 买 YES，价格 $0.30。买 NO，价格 $0.30。总成本 $0.60。无论结果如何，回收 $1.00。利润 $0.40。

现实：

> 提交 YES 订单 → 成交价 $0.30 ✓

> 你的订单改变了市场价格。

> 提交 NO 订单 → 成交价 $0.78 ✗

> 总成本：$1.08。回收：$1.00。实际结果：亏 $0.08。

> 一条腿成交了，另一条没有。你暴露了。

这就是为什么论文只统计利润空间超过 $0.05 的机会 [2]。更小的价差会被执行风险吃掉。

![[Pasted image 20260318095712.png]]

### VWAP-真实的成交价格

不要假设你能以报价成交。要计算成交量加权平均价格（VWAP） [10]。

研究团队的方法是：对 Polygon 链上的每个区块（大约 2 秒），计算该区块内所有 YES 交易的 VWAP 和所有 NO 交易的 VWAP。如果 |VWAP_yes + VWAP_no - 1.0| > 0.02，就记录为一次套利机会 [2]。

> VWAP 就是”你实际付的平均价格”。如果你想买 10,000 个代币，但订单簿上 $0.30 只有 2,000 个，$0.32 有 3,000 个，$0.35 有 5,000 个——你的 VWAP 就是 (2000×0.30 + 3000×0.32 + 5000×0.35) / 10000 = $0.326。比你看到的”最优价格” $0.30 贵了不少。

### 流动性约束-能赚多少取决于订单薄深度

即使价格确实有偏差，你能赚到的利润也受限于可用流动性。

真实例子 [2]：

市场显示套利：YES 价格之和 = $0.85。潜在利润：每美元 $0.15。但这些价格上的订单簿深度只有 $234。最大可提取利润：$234 × 0.15 = $35.10。

对于跨市场套利，你需要在所有仓位上同时有流动性。最小的那个决定了你的上限。

这也是为什么，在现有量化平台中，展现出订单价格对成交价格对影响至关重要。

![[Pasted image 20260318095838.png]]

## 第五章-完整系统-------实际部署了什么

理论是干净的。生产环境是混乱的。

这是一个真正能跑的套利系统长什么样 [2]。

### 数据管道

实时数据： WebSocket 连接 Polymarket 上的 API [9]，接收订单簿更新（价格/数量变化）、成交推送、市场创建/结算事件。

历史数据： 通过 Alchemy Polygon 节点 API 查询合约事件——OrderFilled（交易执行）、PositionSplit（新代币铸造）、PositionsMerge（代币销毁）。

研究团队分析了 8600 万笔交易 [2]。这个量级需要基础设施，不是脚本能搞定的。

> P.S 目前我们 [insiders.bot](https://insiders.bot/) 目前也有计划开源高速交易API，如果有计划使用类似的交易模型，希望能够第一时间体验API，也可以随时DM我。

### 依赖关系检测层

对于 305 个美国大选市场，有 46,360 对可能的组合需要检查。手动分析不可能。

研究团队用了一个聪明的方法：用 DeepSeek-R1-Distill-Qwen-32B 大语言模型来做初筛 [2]。

输入：两个市场的条件描述。输出：合法结果组合的 JSON。

然后做三层验证：每个市场是否恰好有一个条件为真？合法组合数是否少于 n × m（存在依赖）？依赖子集是否满足套利条件？

结果：40,057 对独立（无套利）→ 1,576 对有依赖（潜在套利）→ 374 对满足严格条件 → 13 对经人工验证可利用 [2]。

LLM 在复杂多条件市场上的准确率是 81.45%。足够用来做初筛，但执行前需要人工验证。

### 三层优化引擎

第一层：简单线性约束（LCMM）。 快速检查基本规则——“概率之和等于 1”、“如果 A 蕴含 B，那 P(A) 不能超过 P(B)”。毫秒级完成，去除明显的定价错误。

第二层：整数规划投影（Frank-Wolfe + Gurobi）。 这是核心。参数：Alpha = 0.9（提取至少 90% 的可用套利），初始 ε = 0.1（10% 收缩），收敛阈值 = 1e-6，时间限制 = 30 分钟。典型迭代次数：50-150 次。每次迭代求解时间：1-30 秒 [2]。

第三层：执行验证。 在提交订单之前，模拟当前订单簿上的成交。检查：流动性是否充足？预期滑点是多少？扣除滑点后的保证利润是多少？利润是否超过最低门槛（$0.05）？只有全部通过才执行。

### 仓位管理-改良版Kelly公式

标准的 Kelly 公式 [11] 告诉你该把多少比例的资金投入一笔交易。但在套利场景下，需要加入执行风险的调整：

f = (b×p - q) / b × √p

其中 b 是套利利润百分比，p 是完全执行的概率（根据订单簿深度估算），q = 1 - p。

上限：订单簿深度的 50%。 超过这个比例，你的订单本身就会大幅移动市场。

### 最终结果

2024 年 4 月到 2025 年 4 月，总提取利润 [2]：

单一条件套利： 低买两边 $5,899,287 + 高卖两边 $4,682,075 = $10,581,362

市场再平衡： 低买所有 YES $11,092,286 + 高卖所有 YES $612,189 + 买所有 NO $17,307,114 = $29,011,589

跨市场组合套利： $95,634

总计：$39,688,585

前 10 名套利者拿走了 $8,127,849（总额的 20.5%）。排名第一的套利者：$2,009,632，来自 4,049 笔交易，平均每笔 $496 [2]。

不是彩票。不是运气。是数学精度的系统化执行。

### 最后的现实

当交易者还在读”预测市场 10 个技巧”的时候，量化系统在做什么？

它们在用整数规划检测 17,218 个条件之间的依赖关系。在用 Bregman 投影计算最优套利交易。在运行 Frank-Wolfe 算法处理梯度爆炸。在用 VWAP 估算滑点并行执行订单。在系统性地提取 4000 万美元的保证利润。

差距不是运气。是数学基础设施。

论文是公开的 [1]。算法是已知的。利润是真实的。

问题是：在下一个 4000 万被提取之前，你能建出来吗？


## 可视化图文

套利是什么？为什么 Polymarket 有这么多

![[Pasted image 20260318100359.png]]

> 中位数偏差 $0.60 意味着：市场经常性地偏离正确价格 40%。这不是"接近有效"，这是"大规模可利用"。

这是 Polymarket 套利系统全貌。

![[Pasted image 20260318100423.png]]

展示了整个四层管道：数据采集 → 依赖检测 → 三级优化 → 执行，最终产出近 $4000 万的年化利润。

跨市场套利：逻辑依赖关系

接下来看最核心的数学问题——为什么"加法"不够用。

![[Pasted image 20260318100458.png]]

说明两件事：

单市场层面，逻辑依赖关系创造了隐性矛盾；

两个市场，各自"正常"，合起来有矛盾

![[Pasted image 20260318100515.png]]

> 逻辑关系： "大胜 5%" 是 "获胜" 的子集。就像"打雷"一定意味着"下雨"，但"下雨"不一定有雷。 所以 P(大胜) 永远不能超过 P(获胜)。 若市场 B YES 报价 $0.55 > 市场 A YES 报价 $0.48 → 逻辑矛盾 → 套利！

![[Pasted image 20260318100533.png]]

Bregman 投影：算出最优套利量

![[Pasted image 20260318100557.png]]

Bregman 投影 + Frank-Wolfe 是数学核心。

KL 散度比欧几里得距离更适合概率定价——越极端的价格变动权重越高。

![[Pasted image 20260318100614.png]]

KL 散度（Bregman 散度的一种）会自动给"变动 B"更高的权重。这正是 Polymarket 价格定价机制（LMSR）的正确度量方式。

![[Pasted image 20260318100646.png]]

Frank-Wolfe 的聪明之处在于每轮只加一个新顶点，50–150 轮就能收敛，完全绕开了枚举指数级顶点的问题。

山谷比喻：投影距离 = 可赚利润

![[Pasted image 20260318100708.png]]

Frank-Wolfe 算法：一步一步逼近答案

![[Pasted image 20260318100726.png]]

后期反而更快：随着比赛结果确定，可行解空间缩小，变量更少，约束更紧。

执行风险：算对了，下单也可能亏,理论上能赚钱，但执行不好照样亏钱。

![[Pasted image 20260318100757.png]]


Polymarket 用 CLOB 顺序撮合，第一条腿改变了市场价格，第二条腿就以更差的价格成交。

门槛 $0.05 的利润空间 + 改良 Kelly 公式是防御执行风险的关键机制。

![[Pasted image 20260318100811.png]]

订单簿有深度限制，你的大单会以不同价格成交。

最终结果：$3,968 万的去向

![[Pasted image 20260318100832.png]]

2024 年 4 月到 2025 年 4 月，总提取利润：

单一条件套利： 低买两边 $5,899,287 + 高卖两边 $4,682,075 = $10,581,362

市场再平衡： 低买所有 YES $11,092,286 + 高卖所有 YES $612,189 + 买所有 NO $17,307,114 = $29,011,589

跨市场组合套利： $95,634

总计：$39,688,585

前 10 名套利者拿走了 $8,127,849（总额的 20.5%）。排名第一的套利者：$2,009,632，来自 4,049 笔交易，平均每笔 $496 [2]。

不是彩票。不是运气。是数学精度的系统化执行。


## 概念速查

• 边际多面体（Marginal Polytope） → 所有”合法价格”组成的空间。价格必须在这个空间内才是无套利的。可以理解为”价格的合法区域”

• 整数规划（Integer Programming） → 用线性约束描述合法结果，避免暴力枚举。把 2^63 次检查压缩成几条约束 [3]

• Bregman 散度 / KL 散度 → 衡量两个概率分布之间”距离”的方法，比欧几里得距离更适合价格/概率场景。极端价格附近的变动权重更高 [5] [6]

• LMSR（对数市场评分规则） → Polymarket 做市商使用的定价机制，价格代表隐含概率 [4]

• Frank-Wolfe 算法 → 一种迭代优化算法，每轮只加一个新顶点，避免了枚举指数级多的合法结果 [7]

• Gurobi → 业界领先的整数规划求解器，Frank-Wolfe 每轮迭代的”向导” [8]

• CLOB（中央限价订单簿） → Polymarket 的交易撮合机制，订单顺序执行，不能保证原子性 [9]

• VWAP（成交量加权平均价格） → 你实际付的平均价格，考虑了订单簿深度。比”最优报价”更真实 [10]

• Kelly 公式 → 告诉你该把多少比例的资金投入一笔交易，平衡收益和风险 [11]

• 非原子执行 → 多笔订单不能保证同时成交的问题。一条腿成交另一条没成交 = 暴露风险

• DeepSeek → 用来做市场依赖关系初筛的大语言模型，准确率 81.45%


## 英文原文

I'm going to break down the essential math you need for trading on Polymarket. I'll also share the exact roadmap and resources that helped me personally.

Let's get straight to it

A recent research paper just exposed the reality. Sophisticated traders extracted $40 million in guaranteed arbitrage profits from Polymarket in one year. The top trader alone made $2,009,631.76. These aren't lucky gamblers. They're running Bregman projections, Frank-Wolfe algorithms, and solving optimization problems that would make most computer science PhDs uncomfortable.

> Bookmark This - I’m Roan, a backend developer working on system design, HFT-style execution, and quantitative trading systems. My work focuses on how prediction markets actually behave under load.

When you see a market where YES is $0.62 and NO is $0.33, you think “that adds up to $0.95, there’s arbitrage.” You’re right. What most people never realize is that while they’re manually checking whether YES plus NO equals $1, quantitative systems are solving integer programs that scan 17,218 conditions across 2^63 possible outcomes in milliseconds. By the time a human places both orders, the spread is gone. The systems have already found the same violation across dozens of correlated markets, calculated optimal position sizes accounting for order book depth and fees, executed parallel non-atomic trades, and rotated capital into the next opportunity.

The difference isn't just speed. It's mathematical infrastructure.

By the end of this article, you will understand the exact optimization frameworks that extracted $40 million from Polymarket. You'll know why simple addition fails, how integer programming compresses exponential search spaces, and what Bregman divergence actually means for pricing efficiency. More importantly, you'll see the specific code patterns and algorithmic strategies that separate hobby projects from production systems running millions in capital. Note: This isn’t a skim. If you’re serious about building systems that can scale to seven figures, read it end to end. If you’re here for quick wins or vibe coding, this isn’t for you.

### Part I: The Marginal Polytope Problem (Why Simple Math Fails)

The Reality of Multi-Condition Markets

Single condition market: "Will Trump win Pennsylvania?"

- YES: $0.48
    
- NO: $0.52
    
- Sum: $1.00
    

Looks perfect. No arbitrage, right?

Wrong.

Now add another market: "Will Republicans win Pennsylvania by 5+ points?"

- YES: $0.32
    
- NO: $0.68
    

Still both sum to $1. Still looks fine.

But there's a logical dependency. If Republicans win by 5+ points, Trump must win Pennsylvania. These markets aren't independent. And that creates arbitrage. The Mathematical Framework

For any market with n conditions, there are 2^n possible price combinations. But only n valid outcomes because exactly one condition must resolve to TRUE.

Define the set of valid payoff vectors:

> Z = {φ(ω) : ω ∈ Ω}

Where φ(ω) is a binary vector showing which condition is TRUE in outcome ω.

The marginal polytope is the convex hull of these valid vectors:

> M = conv(Z)

Arbitrage-free prices must lie in M. Anything outside M is exploitable.

For the Pennsylvania example:

1. Market A has 2 conditions, 2 valid outcomes
    
2. Market B has 2 conditions, 2 valid outcomes
    
3. Combined naive check: 2 × 2 = 4 possible outcomes
    
4. Actual valid outcomes: 3 (dependency eliminates one)
    

When prices assume 4 independent outcomes but only 3 exist, the mispricing creates guaranteed profit.

Why Brute Force Dies

NCAA 2010 tournament market had:

- 63 games (win/loss each)
    
- 2^63 = 9,223,372,036,854,775,808 possible outcomes
    
- 5,000+ securities
    

Checking every combination is computationally impossible.

The research paper found 1,576 potentially dependent market pairs in the 2024 US election alone. Naive pairwise verification would require checking 2^(n+m) combinations for each pair.

At just 10 conditions per market, that's 2^20 = 1,048,576 checks per pair. Multiply by 1,576 pairs. Your laptop will still be computing when the election results are already known.

The Integer Programming Solution

Instead of enumerating outcomes, describe the valid set with linear constraints.

> Z = {z ∈ {0,1}^I : A^T × z ≥ b}

Real example from Duke vs Cornell market:

Each team has 7 securities (0 to 6 wins). That's 14 conditions, 2^14 = 16,384 possible combinations.

But they can't both win 5+ games because they'd meet in the semifinals.

Integer programming constraints:

> Sum of z(duke, 0 to 6) = 1 Sum of z(cornell, 0 to 6) = 1 z(duke,5) + z(duke,6) + z(cornell,5) + z(cornell,6) ≤ 1

Three linear constraints replace 16,384 brute force checks.

This is how quantitative systems handle exponential complexity. They don't enumerate. They constrain.

Detection Results from Real Data

The research team analyzed markets from April 2024 to April 2025:

- 17,218 total conditions examined
    
- 7,051 conditions showed single-market arbitrage (41%)
    
- Median mispricing: $0.60 per dollar (should be $1.00)
    
- 13 confirmed dependent market pairs with exploitable arbitrage
    

The median mispricing of $0.60 means markets were regularly wrong by 40%. Not close to efficient. Massively exploitable.

Key takeaway: Arbitrage detection isn't about checking if numbers add up. It's about solving constraint satisfaction problems over exponentially large outcome spaces using compact linear representations.

### Part II: Bregman Projection (How to Actually Remove Arbitrage)

Finding arbitrage is one problem. Calculating the optimal exploiting trade is another.

You can't just "fix" prices by averaging or nudging numbers. You need to project the current market state onto the arbitrage-free manifold while preserving the information structure.

Why Standard Distance Fails

Euclidean projection would minimize:

> ||μ - θ||^2

This treats all price movements equally. But markets use cost functions. A price move from $0.50 to $0.60 has different information content than a move from $0.05 to $0.15, even though both are 10 cent changes.

Market makers use logarithmic cost functions (LMSR) where prices represent implied probabilities. The right distance metric must respect this structure.

The Bregman Divergence

For any convex function R with gradient ∇R, the Bregman divergence is:

> D(μ||θ) = R(μ) + C(θ) - θ·μ

Where:

- R(μ) is the convex conjugate of the cost function C
    
- θ is the current market state
    
- μ is the target price vector
    
- C(θ) is the market maker's cost function
    

For LMSR, R(μ) is negative entropy:

> R(μ) = Sum of μ_i × ln(μ_i)

This makes D(μ||θ) the Kullback-Leibler divergence, measuring information-theoretic distance between probability distributions.

The Arbitrage Profit Formula

The maximum guaranteed profit from any trade equals:

> max over all trades δ of [min over outcomes ω of (δ·φ(ω) - C(θ+δ) + C(θ))] = D(μ||θ)*

Where μ* is the Bregman projection of θ onto M.

This is not obvious. The proof requires convex duality theory. But the implication is clear: finding the optimal arbitrage trade is equivalent to computing the Bregman projection.

Real Numbers

The top arbitrageur extracted $2,009,631.76 over one year. Their strategy was solving this optimization problem faster and more accurately than everyone else:

> μ = argmin over μ in M of D(μ||θ)*

Every profitable trade was finding μ* before prices moved.

Why This Matters for Execution

When you detect arbitrage, you need to know:

1. What positions to take (which conditions to buy/sell)
    
2. What size (accounting for order book depth)
    
3. What profit to expect (accounting for execution risk)
    

Bregman projection gives you all three.

The projection μ* tells you the arbitrage-free price vector. The divergence D(μ*||θ) tells you the maximum extractable profit. The gradient ∇D tells you the trading direction.

Without this framework, you're guessing. With it, you're optimizing.

Key takeaway: Arbitrage isn't about spotting mispriced assets. It's about solving constrained convex optimization problems in spaces defined by market microstructure. The math determines profitability. You can't just "fix" prices by averaging or nudging numbers. You need to project the current market state onto the arbitrage-free manifold while preserving the information structure.

### Part III: The Frank-Wolfe Algorithm (Making It Computationally Tractable)

Computing the Bregman projection directly is intractable. The marginal polytope M has exponentially many vertices.

Standard convex optimization requires access to the full constraint set. For prediction markets, that means enumerating every valid outcome. Impossible at scale.

The Frank-Wolfe algorithm solves this by reducing projection to a sequence of linear programs.

The Core Insight

Instead of optimizing over all of M at once, Frank-Wolfe builds it iteratively.

Algorithm:

> 1. Start with a small set of known vertices $Z_0$ 2. For iteration t: a. Solve convex optimization over $conv(Z_{t-1}) μ_t$ = argmin over μ in $conv(Z_{t-1})$ of F(μ) b. Find new descent vertex by solving IP: $z_t = argmin$ over z in Z of $∇F(μ_t)·z$ c. Add to active set: $Z_t = Z_{t-1} ∪ {z_t}$ d. Compute convergence gap: $g(μ_t) = ∇F(μ_t)·(μ_t - z_t)$ e. Stop if $g(μ_t) ≤ ε$

The active set $Z_t$ grows by one vertex per iteration. Even after 100 iterations, you're only tracking 100 vertices instead of 2^63.

The Integer Programming Oracle

Step 2b is the expensive part. Each iteration requires solving:

min over z in Z of c·z

Where c = ∇F(μ_t) is the current gradient and Z is the set of valid payoff vectors defined by integer constraints.

This is an integer linear program. NP-hard in general. But modern IP solvers like Gurobi handle these efficiently for well-structured problems.

The research team used Gurobi 5.5. Typical solve times:

- Early iterations (small partial outcomes): under 1 second
    
- Mid-tournament (30-40 games settled): 10-30 seconds
    
- Late tournament (50+ games settled): under 5 seconds
    

Why does it get faster later? Because as outcomes settle, the feasible set shrinks. Fewer variables, tighter constraints, faster solves.

The Controlled Growth Problem

Standard Frank-Wolfe assumes the gradient ∇F is Lipschitz continuous with bounded constant.

For LMSR, ∇R(μ) = ln(μ) + 1. As μ approaches 0, the gradient explodes to negative infinity.

This violates standard convergence proofs.

The solution is Barrier Frank-Wolfe. Instead of optimizing over M, optimize over a contracted polytope:

> M' = (1-ε)M + εu

Where u is an interior point with all coordinates strictly between 0 and 1, and ε in (0,1) is the contraction parameter.

For any ε greater than 0, the gradient is bounded on M'. The Lipschitz constant is O(1/ε).

The algorithm adaptively decreases ε as iterations progress:

> If $g(μ_t) / (-4g_u) < ε_{t-1}: ε_t = min{g(μ_t)/(-4g_u), ε_{t-1}/2} Else: ε_t = ε_{t-1}$

This ensures ε goes to 0 asymptotically, so the contracted problem converges to the true projection.

Convergence Rate

Frank-Wolfe converges at rate O(L × diam(M) / t) where L is the Lipschitz constant and diam(M) is the diameter of M.

For LMSR with adaptive contraction, this becomes O(1/(ε×t)). As ε shrinks adaptively, convergence slows but remains polynomial.

The research showed that in practice, 50 to 150 iterations were sufficient for convergence on markets with thousands of conditions.

Production Performance

From the paper: "Once projections become practically fast, FWMM achieves superior accuracy to LCMM."

Timeline:

- First 16 games: LCMM and FWMM perform similarly (IP solver too slow)
    
- After 45 games settled: First successful 30-minute projection completes
    
- Remaining tournament: FWMM outperforms LCMM by 38% median improvement on security prices
    

The crossover point is when the outcome space shrinks enough for IP solves to complete within trading timeframes.

Key takeaway: Theoretical elegance means nothing without computational tractability. Frank-Wolfe with integer programming oracles makes Bregman projection practical on markets with trillions of outcomes. This is how $40 million in arbitrage actually got computed and executed.

### Part IV: Execution Under Non-Atomic Constraints (Why Order Books Change Everything)

You've detected arbitrage. You've computed the optimal trade via Bregman projection. Now you need to execute. This is where most strategies fail.

The Non-Atomic Problem

Polymarket uses a Central Limit Order Book (CLOB). Unlike decentralized exchanges where arbitrage can be atomic (all trades succeed or all fail), CLOB execution is sequential.

Your arbitrage plan:

> Buy YES at $0.30 Buy NO at $0.30 Total cost: $0.60 Guaranteed payout: $1.00 Expected profit: $0.40

Reality:

> Submit YES order → Fills at $0.30 ✓ Price updates due to your order Submit NO order → Fills at $0.78 ✗ Total cost: $1.08 Payout: $1.00 Actual result: -$0.08 loss

One leg fills. The other doesn't. You're exposed.

This is why the research paper only counted opportunities with at least $0.05 profit margin. Smaller edges get eaten by execution risk.

Volume-Weighted Average Price (VWAP) Analysis

Instead of assuming instant fills at quoted prices, calculate expected execution price:

> VWAP = Sum of (price_i × volume_i) / Sum of (volume_i)

The research methodology:

> For each block on Polygon (approximately 2 seconds): Calculate VWAP_yes from all YES trades in that block Calculate VWAP_no from all NO trades in that block If abs(VWAP_yes + VWAP_no - 1.0) > 0.02: Record arbitrage opportunity Profit = abs(VWAP_yes + VWAP_no - 1.0)

Blocks are the atomic time unit. Analyzing per-block VWAP captures the actual achievable prices, not the fantasy of instant execution.

The Liquidity Constraint

Even if prices are mispriced, you can only capture profit up to available liquidity.

Real example from the data:

- Market shows arbitrage: sum of YES prices = $0.85
    
- Potential profit: $0.15 per dollar
    
- Order book depth at these prices: $234 total volume
    
- Maximum extractable profit: $234 × 0.15 = $35.10
    

The research calculated maximum profit per opportunity as:

> profit = (price deviation) × min(volume across all required positions)

For multi-condition markets, you need liquidity in ALL positions simultaneously. The minimum determines your cap.

Time Window Analysis

The research used a 950-block window (approximately 1 hour) to group related trades.

Why 1 hour? Because 75% of matched orders on Polymarket fill within this timeframe. Orders submitted, matched, and executed on-chain typically complete within 60 minutes.

For each trader address, all bids within a 950-block window were grouped as a single strategy execution. Profit was calculated as the guaranteed minimum payout across all possible outcomes minus total cost.

Execution Success Rate

Of the detected arbitrage opportunities:

- Single condition arbitrage: 41% of conditions had opportunities, most were exploited
    
- Market rebalancing: 42% of multi-condition markets had opportunities
    
- Combinatorial arbitrage: 13 valid pairs identified, 5 showed execution
    

The gap between detection and execution is execution risk.

Latency Layers: The Speed Hierarchy

Retail trader execution:

> Polymarket API call: ~50ms Matching engine: ~100ms Polygon block time: ~2,000ms Block propagation: ~500ms Total: ~2,650ms

Sophisticated arbitrage system:

> WebSocket price feed: <5ms (real-time push) Decision computation: <10ms (pre-calculated) Direct RPC submission: ~15ms (bypass API) Parallel execution: ~10ms (all legs at once) Polygon block inclusion: ~2,000ms (unavoidable) Total: ~2,040ms

The 20-30ms you see on-chain is decision-to-mempool time. Fast wallets submit all positions within 30ms, eliminating sequential execution risk by confirming everything in the same block.

The compounding advantage:

By the time you see their transaction confirmed on-chain (Block N), they detected the opportunity 2+ seconds earlier (Block N-1), submitted all legs in 30ms, and the market already rebalanced. When you copy at Block N+1, you're 4 seconds behind a sub-second opportunity.

Why Copytrading Fast Wallets Fails

What actually happens: Block N-1: Fast system detects mispricing, submits 4 transactions in 30ms Block N: All transactions confirm, arbitrage captured, you see this Block N+1: You copy their trade, but price is now $0.78 (was $0.30)

You're not arbitraging. You're providing exit liquidity.

Order book depth kills you:

Fast wallet buys 50,000 tokens:

- VWAP: $0.322 across multiple price levels
    
- Market moves
    

You buy 5,000 tokens after:

- VWAP: $0.344 (market already shifted)
    
- They paid $0.322, you paid $0.344
    
- Their 10 cent edge became your 2.2 cent loss
    

The Capital Efficiency Problem

Top arbitrageur operated with $500K+ capital. With $5K capital, the same strategy breaks because:

- Slippage eats larger percentage of smaller positions
    
- Cannot diversify across enough opportunities
    
- Single failed execution wipes out days of profit
    
- Fixed costs (gas) consume more of profit margin
    

Gas fees on 4-leg strategy: ~$0.02

- $0.08 profit → 25% goes to gas
    
- $0.03 profit → 67% goes to gas
    

This is why $0.05 minimum threshold exists.

Real Execution Data

Single condition arbitrage:

- Detected: 7,051 conditions
    
- Executed: 87% success rate
    
- Failed due to: liquidity (48%), price movement (31%), competition (21%)
    

Combinatorial arbitrage:

- Detected: 13 pairs
    
- Executed: 45% success rate
    
- Failed due to: insufficient simultaneous liquidity (71%), speed competition (18%)
    

Key takeaway: Mathematical correctness is necessary but not sufficient. Execution speed, order book depth, and non-atomic fill risk determine actual profitability. The research showed $40 million extracted because sophisticated actors solved execution problems, not just math problems.

### Part V: The Complete System (What Actually Got Deployed)

Theory is clean. Production is messy. Here's what a working arbitrage system actually looks like based on the research findings and practical requirements.

The Data Pipeline

> Real-time requirements:

> WebSocket connection to Polymarket CLOB API └─ Order book updates (price/volume changes) └─ Trade execution feed (fills happening) └─ Market creation/settlement events Historical analysis: Alchemy Polygon node API └─ Query events from contract 0x4D97DCd97eC945f40cF65F87097ACe5EA0476045 └─ OrderFilled events (trades executed) └─ PositionSplit events (new tokens minted) └─ PositionsMerge events (tokens burned)

The research analyzed 86 million transactions. That volume requires infrastructure, not scripts.

The Dependency Detection Layer

For 305 US election markets, there are 46,360 possible pairs to check.

Manual analysis is impossible. The research used DeepSeek-R1-Distill-Qwen-32B with prompt engineering:

> Input: Two markets with their condition descriptions Output: JSON of valid outcome combinations Validation checks: 1. Does each market have exactly one TRUE condition per outcome? 2. Are there fewer valid combinations than n × m (dependency exists)? 3. Do dependent subsets satisfy arbitrage conditions? Results on election markets: 40,057 independent pairs (no arbitrage possible) 1,576 dependent pairs (potential arbitrage) 374 satisfied strict combinatorial conditions 13 manually verified as exploitable

81.45% accuracy on complex multi-condition markets. Good enough for filtering. Requires manual verification for execution.

The Optimization Engine

Three-layer arbitrage removal:

Layer 1: Simple LCMM constraints Fast linear programming relaxations. Check basic constraints like "sum of probabilities equals 1" and "if A implies B, then P(A) cannot exceed P(B)."

Runs in milliseconds. Removes obvious mispricing.

Layer 2: Integer programming projection Frank-Wolfe algorithm with Gurobi IP solver.

Parameters from research:

- Alpha = 0.9 (extract at least 90% of available arbitrage)
    
- Initial epsilon = 0.1 (10% contraction)
    
- Convergence threshold = 1e-6
    
- Time limit = 30 minutes (reduced as markets shrink)
    

Typical iterations: 50 to 150. Typical solve time per iteration: 1 to 30 seconds depending on market size.

Layer 3: Execution validation Before submitting orders, simulate fills against current order book.

Check:

- Is liquidity sufficient at these prices?
    
- What is expected slippage?
    
- What is guaranteed profit after slippage?
    
- Does profit exceed minimum threshold (research used $0.05)?
    

Only execute if all checks pass.

Position Sizing Logic

Modified Kelly criterion accounting for execution risk:

> f = (b×p - q) / b × sqrt(p)*

Where:

- b = arbitrage profit percentage
    
- p = probability of full execution (estimated from order book depth)
    
- q = 1 - p
    

Cap at 50% of order book depth to avoid moving the market.

The Monitoring Dashboard

Track in real-time:

> Opportunities detected per minute Opportunities executed per minute Execution success rate Total profit (running sum) Current drawdown percentage Average latency (detection to submission) Alerts: Drawdown exceeds 15% Execution rate drops below 30% IP solver timeouts increase Order fill failures spike

The research identified the top arbitrageur made 4,049 transactions. That's approximately 11 trades per day over one year. Not high-frequency in the traditional sense, but systematic and consistent.

The Actual Results

Total extracted April 2024 to April 2025:

> Single condition arbitrage: Buy both < $1: $5,899,287 Sell both > $1: $4,682,075 Subtotal: $10,581,362 Market rebalancing: Buy all YES < $1: $11,092,286 Sell all YES > $1: $612,189 Buy all NO: $17,307,114 Subtotal: $29,011,589 Combinatorial arbitrage: Cross-market execution: $95,634 Total: $39,688,585

Top 10 extractors took $8,127,849 (20.5% of total).

Top single extractor: $2,009,632 from 4,049 trades.

Average profit per trade for top player: $496.

Not lottery wins. Not lucky timing. Mathematical precision executed systematically.

What Separates Winners from Losers

The research makes it clear:

Retail approach:

- Check prices every 30 seconds
    
- See if YES + NO roughly equals $1
    
- Maybe use a spreadsheet
    
- Manual order submission
    
- Hope for the best
    

Quantitative approach:

- Real-time WebSocket feeds
    
- Integer programming for dependency detection
    
- Frank-Wolfe with Bregman projection for optimal trades
    
- Parallel order execution with VWAP estimation
    
- Systematic position sizing under execution constraints
    
- 2.65 second latency vs. 30 second polling
    

One group extracted $40 million. The other group provided the liquidity.

Key takeaway: Production systems require mathematical rigor AND engineering sophistication. Optimization theory, distributed systems, real-time data processing, risk management, execution algorithms. All of it. The math is the foundation. The infrastructure is what makes it profitable.

### The Final Reality

While traders were reading "10 tips for prediction markets," quantitative systems were:

1. Solving integer programs to detect dependencies across 17,218 conditions
    
2. Computing Bregman projections to find optimal arbitrage trades
    
3. Running Frank-Wolfe algorithms with controlled gradient growth
    
4. Executing parallel orders with VWAP-based slippage estimation
    
5. Systematically extracting $40 million in guaranteed profits
    

The difference is not luck. It's mathematical infrastructure. The research paper is public. The algorithms are known. The profits are real.

The question is: can you build it before the next $40 million is extracted?

Resources:

- Research paper: "Unravelling the Probabilistic Forest: Arbitrage in Prediction Markets" (arXiv:2508.03474v1)
- Theory foundation: "Arbitrage-Free Combinatorial Market Making via Integer Programming" (arXiv:1606.02825v2)
- IP solver: Gurobi Optimizer
- LLM for dependencies: DeepSeek-R1-Distill-Qwen-32B
- Data source: Alchemy Polygon node API
    
The math works. The infrastructure exists. The only question is execution.

## 参考链接

[1] 原文：

[https://x.com/RohOnChain/status/2017314080395296995](https://x.com/RohOnChain/status/2017314080395296995)

[2] 研究论文 “Unravelling the Probabilistic Forest: Arbitrage in Prediction Markets”：

[https://arxiv.org/abs/2508.03474](https://arxiv.org/abs/2508.03474)

[3] 理论基础论文 “Arbitrage-Free Combinatorial Market Making via Integer Programming”：

[https://arxiv.org/abs/1606.02825](https://arxiv.org/abs/1606.02825)

[4] LMSR 对数市场评分规则解释：

[https://www.cultivatelabs.com/crowdsourced-forecasting-guide/how-does-logarithmic-market-scoring-rule-lmsr-work](https://www.cultivatelabs.com/crowdsourced-forecasting-guide/how-does-logarithmic-market-scoring-rule-lmsr-work)

[5] Bregman 散度入门：

[https://mark.reid.name/blog/meet-the-bregman-divergences.html](https://mark.reid.name/blog/meet-the-bregman-divergences.html)

[6] KL 散度 - Wikipedia：

[https://en.wikipedia.org/wiki/Kullback%E2%80%93Leibler_divergence](https://en.wikipedia.org/wiki/Kullback%E2%80%93Leibler_divergence)

[7] Frank-Wolfe 算法 - Wikipedia：

[https://en.wikipedia.org/wiki/Frank%E2%80%93Wolfe_algorithm](https://en.wikipedia.org/wiki/Frank%E2%80%93Wolfe_algorithm)

[8] Gurobi 优化器：

[https://www.gurobi.com/](https://www.gurobi.com/)

[9] Polymarket CLOB API 文档：

[https://docs.polymarket.com/](https://docs.polymarket.com/)

[10] VWAP 解释 - Investopedia：

[https://www.investopedia.com/terms/v/vwap.asp](https://www.investopedia.com/terms/v/vwap.asp)

[11] Kelly 公式 - Investopedia：

[https://www.investopedia.com/articles/trading/04/091504.asp](https://www.investopedia.com/articles/trading/04/091504.asp)

[12] Decrypt 报道 “The $40 Million Free Money Glitch”：

[https://decrypt.co/339958/40-million-free-money-glitch-crypto-prediction-markets](https://decrypt.co/339958/40-million-free-money-glitch-crypto-prediction-markets)