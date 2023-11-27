主要分开源版，企业版，云服务版本。后面两个提供了增强型和付费的功能。暂时不介绍。只主要介绍开源版本。

## 关于Grafana

Grafana 开源软件使您能够查询、可视化、报警和探索您的指标、日志和跟踪，无论它们存储在何处。

Grafana OSS 为您提供了将时间序列数据库 (TSDB) 数据转化为具有洞察力的图表和可视化数据的工具。Grafana OSS 插件框架还能让您连接其他数据源，如 NoSQL/SQL 数据库、Jira 或 ServiceNow 等票据工具以及 GitLab 等 CI/CD 工具。

安装 Grafana 并使用 "开始使用 Grafana "中的说明设置第一个仪表盘后，您将有很多选项可根据自己的要求进行选择。例如，如果您想查看天气数据和智能家居的统计数据，那么您可以创建一个播放列表。如果您是一家企业的管理员，要为多个团队管理 Grafana，那么您可以设置配置和身份验证。

以下部分概述了 Grafana 的功能，并提供了产品文档链接以帮助您了解更多信息。有关更多指导和想法，请访问我们的 Grafana 社区论坛。

### 探索指标，日志和跟踪(traces)

通过临时查询和动态下拉来探索数据。拆分查看并并排比较不同的时间范围、查询和数据源。更多信息，请参阅 ["探索"]([Explore | Grafana documentation](https://grafana.com/docs/grafana/latest/explore/))。

### 报警(Alerts)

如果您使用的是 Grafana Alerting，那么您可以通过多种不同的警报通知器发送警报，包括 PagerDuty、SMS、电子邮件、VictorOps、OpsGenie 或 Slack。

如果您偏爱其他通信渠道，警报钩子允许您用代码创建不同的通知器。可视化定义最重要指标的警报规则。[Configure Alerting | Grafana documentation](https://grafana.com/docs/grafana/latest/alerting/alerting-rules/)

### 注解(Annotations)

用来自不同数据源的丰富事件为图表添加注释。将鼠标悬停在事件上可查看完整的事件元数据和标签。

该功能在 Grafana 中显示为图形标记，可用于关联数据，以防出错。您可以手动创建注释--只需按住 Control 键单击图形并输入一些文本--也可以从任何数据源获取数据。有关详细信息，请参阅注释。[Annotate visualizations | Grafana documentation](https://grafana.com/docs/grafana/latest/dashboards/build-dashboards/annotate-visualizations/)


### 仪表盘变量

通过[模板变量]([Variables | Grafana documentation](https://grafana.com/docs/grafana/latest/dashboards/variables/))，您可以创建可重复用于多种不同用例的仪表盘。这些模板的值不是硬编码的，因此，例如，如果你有一个生产服务器和一个测试服务器，你可以在这两个服务器上使用相同的仪表盘。

使用模板可以深入研究数据，例如，从所有数据到北美数据，再到德克萨斯州数据，等等。您还可以在组织内跨团队共享这些仪表盘，或者如果您为某个常用数据源创建了出色的仪表盘模板，也可以将其贡献给整个社区，供大家定制和使用。

### 配置Grafana

如果您是 Grafana 管理员，那么您需要彻底熟悉 [Grafana 配置选项]([Configure Grafana | Grafana documentation](https://grafana.com/docs/grafana/latest/setup-grafana/configure-grafana/))和 [Grafana CLI]([Grafana CLI | Grafana documentation](https://grafana.com/docs/grafana/latest/cli/))。

配置包括配置文件和环境变量。您可以设置默认端口、日志级别、电子邮件 IP 地址、安全性等。

### 导入仪表盘和插件

在官方资料库中发现数以百计的[仪表盘]([Dashboards | Grafana Labs](https://grafana.com/grafana/dashboards/))和[插件]([Grafana Plugins - extend and customize your Grafana](https://grafana.com/grafana/plugins/))。得益于社区成员的热情和动力，每周都会有新的插件加入。

### 认证

Grafana 支持 LDAP 和 OAuth 等不同的身份验证方法，并允许您将用户映射到组织。有关详细信息，请参阅[用户身份验证概述]([Configure authentication | Grafana documentation](https://grafana.com/docs/grafana/latest/setup-grafana/configure-security/configure-authentication/))。

在 Grafana 企业版中，您还可以将用户映射到团队： 如果贵公司有自己的身份验证系统，Grafana 允许您将内部系统中的团队映射到 Grafana 中的团队。这样，您就可以自动让员工访问为其团队指定的仪表板。有关详细信息，请参阅 [Grafana Enterprise]([Grafana Enterprise | Grafana documentation](https://grafana.com/docs/grafana/latest/introduction/grafana-enterprise/))。


### 调配(Provisioning)

点击、拖拽和下拉即可轻松创建单个仪表盘，但需要许多仪表盘的高级用户则希望使用脚本自动进行设置。您可以在 Grafana 中编写任何脚本。

例如，如果你要启动一个新的 Kubernetes 集群，你也可以用脚本自动启动一个 Grafana，该脚本会预设并锁定正确的服务器、IP 地址和数据源，这样用户就无法更改它们。这也是一种控制大量仪表盘的方法。有关详细信息，请参阅 ["调配"]([Provision Grafana | Grafana documentation](https://grafana.com/docs/grafana/latest/administration/provisioning/))。

### 权限

当企业拥有一个 Grafana 和多个团队时，他们往往希望既能保持独立，又能共享仪表盘。您可以创建一个用户团队，然后设置[文件夹和仪表盘的权限]([Manage dashboard permissions | Grafana documentation](https://grafana.com/docs/grafana/latest/administration/user-management/manage-dashboard-permissions/))，如果您使用的是 Grafana Enterprise，还可以设置到[数据源级别的权限]([Data source management | Grafana documentation](https://grafana.com/docs/grafana/latest/administration/data-source-management/#data-source-permissions))。


### 其他Grafana实验室的OSS项目

除 Grafana 外，Grafana Labs 还提供以下开源项目：

- Grafana Loki： Grafana Loki 是一套开源组件，可组成功能齐全的日志栈。有关详细信息，请参阅 Grafana Loki 文档。 [Grafana Loki documentation | Grafana Loki documentation](https://grafana.com/docs/loki/latest/)
- Grafana Tempo：Grafana Tempo 是一个开源、易用且容量大的分布式跟踪后端。有关详细信息，请参阅 Grafana Tempo 文档。[Tempo documentation | Grafana Tempo documentation](https://grafana.com/docs/tempo/latest/?pg=oss-tempo&plcmt=hero-txt/)
- Grafana Mimir：Grafana Mimir 是一个开源软件项目，可为 Prometheus 提供可扩展的长期存储。有关 Grafana Mimir 的更多信息，请参阅 Grafana Mimir 文档。[Grafana Mimir documentation | Grafana Mimir documentation](https://grafana.com/docs/mimir/latest/)
- Grafana Phlare： Grafana Phlare 是一个用于聚合连续剖析数据的开源软件项目。连续剖析是一种可观察性信号，可让您了解工作负载的资源（CPU、内存等）使用情况，并精确到行数。有关使用 Grafana Phlare 的更多信息，请参阅 Grafana Phlare 文档。[Grafana Pyroscope documentation | Grafana Pyroscope documentation](https://grafana.com/docs/pyroscope/latest/)

## 什么是Prometheus

Grafana大量的实践与Prometheus结合，所以需要了解它。  请看 [[Prometheus入门]]


## 仪表盘概述

您想过仪表盘是什么吗？在可观察性领域，这个词经常被使用，但它到底是什么意思呢？这个概念是从汽车中借用过来的，在汽车中，仪表盘为驾驶员提供了操作车辆所需的控制功能。同样，数字仪表盘也能帮助我们理解和管理系统。本主题将解释 Grafana 仪表盘的功能，让您更轻松地创建自己的仪表盘。

下图展示了 Grafana 仪表盘示例：

![[Pasted image 20231107113217.png]]

Grafana 仪表板由以精美的图表和其他可视化方式显示数据的面板(Panel)组成。这些面板是使用将数据源的原始数据转换为可视化的组件创建的。这一过程包括通过三个关口传递数据：插件、查询和可选转换。

下面的图片显示了所有的门，随后详细解释了它们的目的、用法和意义。

![[Pasted image 20231107113317.png]]


### 数据源

数据源是指由数据组成的任何实体。它可以是 SQL 数据库、Grafana Loki、Grafana Mimir 或基于 JSON 的 API。它甚至可以是一个基本的 CSV 文件。创建仪表盘可视化的第一步是选择包含所需数据的数据源。

要理解不同数据源之间的区别可能比较困难，因为每个数据源都有自己的结构，需要不同的查询方法。不过，在仪表盘中，您可以在一个视图中看到不同数据源的可视化效果，从而更容易理解整体数据。

### 插件

Grafana 插件是为 Grafana 添加新功能的软件。它们有多种类型，但现在我们要讨论的是数据源插件。Grafana 数据源插件的工作是接收您想要回答的查询，从数据源检索数据，并协调数据源的数据模型和 Grafana 面板的数据模型之间的差异。插件使用一种称为[数据帧]([Data frames | Grafana Plugin Tools](https://grafana.com/developers/plugin-tools/introduction/data-frames))的统一数据结构来实现这一功能。

从数据源进入插件的数据可能有多种不同的格式（如 JSON、行和列或 CSV），但当数据离开插件并通过其余的门移动到可视化时，它总是以数据帧的形式存在。

目前，Grafana 提供了 155 种不同的数据源供您使用。最常用的选项已经预装并可以访问。在探索其他选项之前，请查找符合您要求的现有数据源。Grafana 会不断更新列表，如果找不到合适的数据源，可以浏览[插件目录]([Data source plugins for Grafana | Grafana Labs](https://grafana.com/grafana/plugins/data-source-plugins/?type=datasource))或[创建插件]([Get started | Grafana Plugin Tools](https://grafana.com/developers/plugin-tools))。


### 查询

通过查询，您可以将全部数据简化为特定的数据集，从而提供更易于管理的可视化效果。查询有助于回答有关系统和操作流程的问题。例如，一家拥有在线商店的公司可能希望确定将产品添加到购物车的客户数量。这可以通过查询来实现，查询可以汇总购物车服务的访问指标，显示每秒访问该服务的用户数量。

在使用数据源时，必须认识到每个数据源都有自己独特的查询语言。例如，Prometheus 数据源使用 [PromQL]([Introduction to PromQL, the Prometheus query language | Grafana Labs](https://grafana.com/blog/2020/02/04/introduction-to-promql-the-prometheus-query-language/))，而 [LogQL]([LogQL: Log query language | Grafana Loki documentation](https://grafana.com/docs/loki/latest/query/)) 用于日志，特定数据库则使用 SQL。查询是 Grafana 中每个可视化的基础，一个仪表盘可能会使用一系列查询语言。

下图显示了与 Prometheus 数据源相关联的查询编辑器。`node_cpu_seconds_total` 查询使用 PromQL 编写，只请求一个指标：

![[Pasted image 20231107113827.png]]

### 转换

当可视化中的数据格式不符合你的要求时，你可以应用[转换]([Transform data | Grafana documentation](https://grafana.com/docs/grafana/latest/panels-visualizations/query-transform-data/transform-data/))来处理查询返回的数据。刚开始使用时，你可能不需要转换数据，但它们功能强大，值得一提。

在以下几种情况下，转换数据非常有用：

- 您想将两个字段合并在一起，例如，将 "给名 "和 "姓氏 "连接成一个 "全名 "字段。
- 您有 CSV 数据（全文本），但想转换字段类型（例如从字符串中解析出日期或数字）。
- 你想过滤、连接、合并或执行其他类似 SQL 的操作，而底层数据源或查询语言可能不支持这些操作。

转换位于面板编辑对话框的转换选项卡中。选择需要的转换并定义转换。下图显示，你可以像查询一样拥有任意数量的转换。例如，可以将一系列转换串联起来，对数据类型进行更改、过滤结果、组织列并将结果排序到一个数据管道中。每次刷新仪表盘时，转换都会应用到数据源的最新数据。

下图显示了转换对话框：

![[Pasted image 20231107114025.png]]

### 面板(Panels)

在对数据进行寻源、查询和转换后，数据会被传送到面板，这是通往 Grafana 可视化的最后一道关卡。面板是一个容器，用于显示可视化并为您提供各种控制来对其进行操作。面板配置是您指定如何查看数据的地方。例如，你可以使用面板右上方的下拉菜单来指定你想看到的可视化类型，如柱状图、饼图或直方图。

面板选项可以让你自定义可视化的许多方面，而且根据你选择的可视化类型，选项也有所不同。面板还包含指定面板可视化数据的查询。

下图显示了一个正在编辑的表格面板，面板设置底部显示查询，右侧显示面板选项。在这张图片中，你可以看到数据源、插件、查询和面板是如何结合在一起的。

![[Pasted image 20231107114134.png]]

选择最佳的可视化方式取决于数据和您希望的展示方式。要在一个地方查看可以浏览和检查的仪表盘示例，请参阅 [Grafana Play]([Grafana](https://play.grafana.org/))，其中有功能展示和各种示例。

### 总结

有了数据源、插件、查询、转换和面板模型，您现在就可以直接查看遇到的任何 Grafana 面板，并想象如何构建自己的面板。

构建 Grafana 仪表盘的过程首先是确定您的仪表盘需求，然后确定哪些数据源支持这些需求。如果要将专用数据库与 Grafana 面板集成，则必须确保安装了正确的插件，以便添加数据源与该插件配合使用。

确定数据源并安装插件后，您就可以编写查询、转换数据和格式化可视化，以满足您的需求。
这种组件式架构是 Grafana 如此强大和通用的原因之一。有了数据源插件和数据帧抽象，您可以访问的任何数据源都可以使用相同的通用方法。

## 时间序列介绍

想象一下，您想知道一天中室外温度的变化情况。每隔一小时，你就会查看一次温度计，然后写下时间和当前温度。一段时间后，你就会得到这样的结果：

|Time|Value|
|---|---|
|09:00|24°C|
|10:00|26°C|
|11:00|27°C|

这样的温度数据就是我们所说的时间序列的一个例子--按时间顺序排列的一系列测量数据。表格中的每一行都代表一个特定时间的单个测量值。

当您想确定单个测量值时，表格很有用，但它很难让您看到全局。更常见的时间序列可视化方式是图表，它将每个测量值沿着时间轴排列。像图表这样的可视化表现形式更容易发现数据中的模式和特征，否则就很难看到这些模式和特征。

![[Pasted image 20231107114606.png]]

温度数据（如示例中的数据）远非时间序列的唯一示例。时间序列的其他例子包括
- CPU 和内存使用率
- 传感器数据
- 股票市场指数

虽然这些例子都是按时间顺序排列的测量数据序列，但它们也有其他共同属性：
- 新数据会以固定的时间间隔添加到序列末尾，例如每小时 9:00、10:00、11:00 等。
- 测量数据在添加后很少更新，例如，昨天的温度不会改变。

时间序列功能强大。通过分析任意时间点的系统状态，它们可以帮助你了解过去。时间序列可以告诉你，在可用磁盘空间降为零后，服务器瞬间就崩溃了。

时间序列还能通过揭示数据趋势帮助你预测未来。如果注册用户数量在过去几个月里每月增长 4%，你就可以预测年底的用户基数有多大。

有些时间序列具有在已知时间段内重复出现的模式。例如，气温通常在白天较高，晚上才会下降。通过识别这些周期性或季节性的时间序列，您就可以对下一个时间段做出有把握的预测。如果你知道系统负载在每天 18:00 左右达到峰值，你就可以在此之前增加更多的机器。

### 聚合时间序列

根据测量内容的不同，数据会有很大差异。如果您想比较的时间段长于测量时间间隔呢？如果每小时测量一次温度，那么每天就会有 24 个数据点。要比较历年八月的温度，就必须将 31 乘以 24 的数据点合并成一个。

将一系列测量数据合并起来就叫做聚合。汇总时间序列数据有几种方法。下面是一些常见的方法：

- 平均值返回所有数值的总和除以数值的总数。
- 最小值和最大值返回集合中的最小值和最大值。
- Sum 返回集合中所有值的总和。
- Count 返回集合中数值的个数。

例如，通过汇总一个月的数据，您可以确定 2017 年 8 月的平均气温高于前一年。相反，要查看哪个月的气温最高，您可以比较每个月的最高气温。

如何聚合时间序列数据是一个重要的决定，取决于您想用数据讲述什么故事。使用不同的聚合方式以不同的方式可视化相同的时间序列数据是很常见的。

### 时间序列与监控

在 IT 行业，收集时间序列数据通常是为了监控基础设施、硬件或应用程序事件等。机器生成的时间序列数据通常以较短的时间间隔收集，这使您能够在任何意外变化发生后立即做出反应。因此，数据积累的速度非常快，因此必须有一种高效存储和查询数据的方法。因此，针对时间序列数据进行优化的数据库近年来越来越受欢迎。

#### 时序数据库

时间序列数据库（TSDB）是专门为时间序列数据设计的数据库。虽然可以使用任何常规数据库来存储测量数据，但 TSDB 具有一些有用的优化功能。

现代时间序列数据库利用了这样一个事实，即测量数据只被添加，很少被更新或删除。例如，每个测量值的时间戳随时间的变化很小，这就导致了冗余数据的存储。

请看这一串 Unix 时间戳：

1572524345, 1572524375, 1572524404, 1572524434, 1572524464

查看这些时间戳，它们都以 1572524 开始，导致磁盘空间利用率低下。相反，我们可以将每个后续时间戳存储为与第一个时间戳的差值或 delta：

1572524345, +30, +29, +30, +30

我们甚至可以更进一步，计算这些变化量的变化值：

1572524345, +30, -1, +1, +0

由于进行了类似的优化，TSDB 比其他数据库占用的空间要少得多。

TSDB 的另一个特点是可以使用标签过滤测量结果。每个数据点都标有一个标签(label)，以添加上下文信息，如测量地点。下面是 [InfluxDB 数据格式](https://docs.influxdata.com/influxdb/v1.7/write_protocols/line_protocol_tutorial/#syntax)的一个示例，展示了每个测量点的存储方式。Prometheus也有label的概念。

![[Pasted image 20231107115529.png]]

当然，以下时序数据库Grafana都支持: Graphite InfluxDB Prometheus

#### 采集时序数据

既然我们已经有了存储时间序列的地方，那么如何才能真正收集到测量数据呢？要收集时间序列数据，通常需要在要监控的设备、机器或实例上安装收集器。有些收集器是针对特定数据库制作的，有些则支持不同的输出目的地。

下面是一些收集器的示例：

- collectd https://collectd.org/
- statsd [github.com/statsd/statsd](https://github.com/statsd/statsd)
- Prmetheus exporter [Exporters and integrations | Prometheus](https://prometheus.io/docs/instrumenting/exporters/)
- Telegraf [influxdata/telegraf: The plugin-driven server agent for collecting & reporting metrics. (github.com)](https://github.com/influxdata/telegraf)

收集器要么向数据库推送数据，要么让数据库从中提取数据。也就是推和拉模式。这两种方法各有利弊：

|   |  Pros | Cons  |
|---|---|---|
|Push|Easier to replicate data to multiple destinations.|The TSDB has no control over how much data gets sent.|
|Pull|Better control of how much data that gets ingested, and its authenticity.|Firewalls, VPNs or load balancers can make it hard to access the agents.|

由于将每次测量数据写入数据库的效率很低，因此采集器会预先汇总数据，然后定期写入时间序列数据库。
## 时间序列维度

## 直方图和热力图

## 样例

## 名词