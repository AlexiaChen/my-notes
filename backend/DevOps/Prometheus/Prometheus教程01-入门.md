
### 什么是Prometheus

Prometheus 是一个系统监控和警报系统。它由 SoundCloud 于 2012 年开源，是继 Kubernetes 之后第二个加入云原生计算基金会并毕业的项目。Prometheus 将所有指标数据(Metrics)存储为时间序列，即metrics信息与记录的时间戳一起存储。KV形式的labels也会与指标一起存储。

### 什么是指标以及为什么它如此重要

 通俗地说，"指标(Metrics) "就是衡量的标准。我们要衡量的标准因应用而异。对于网络服务器来说，可以是请求时间，对于数据库来说，可以是 CPU 使用率或活动连接数等。

衡量标准在了解应用程序以某种方式运行的原因方面发挥着重要作用。如果你运行一个网络应用程序，有人来找你说程序运行速度慢。你需要一些信息来找出应用程序发生了什么问题。例如，当请求数量较多时，应用程序可能会变慢。如果掌握了请求数指标，就能找出原因，并增加服务器数量来处理大负荷。

在为应用程序定义指标时，您必须戴上侦探的帽子，并提出这样一个问题：如果我的应用程序出现任何问题，哪些信息对我的调试非常重要？

### Prometheus的基本架构

主要有以下三个组件：

- Prometheus 服务器（采集和存储度量数据的服务器）。
- 要抓取的目标，例如公开其度量指标的仪器应用程序，或公开其他应用程序度量指标的exporter。
- 报警管理器，可以根据预设规则发出警报

> (注：除此以外，Prometheus 还有 push_gateway，此处未涉及）。

![[Pasted image 20231107092804.png]]


以网络服务器为例，我们希望提取某个指标，如网络服务器处理的 API 调用数。因此，我们使用 Prometheus 客户端库添加了一些仪器代码，并公开了指标信息。既然网络服务器公开了其指标信息，我们就可以配置 Prometheus 对其进行抓取。

现在，Prometheus 被配置为在特定的时间间隔（例如每分钟）从监听 xyz IP 地址端口 7500 的网络服务器获取指标信息。

11:00:00 时，当我将服务器公开供用户使用时，应用程序会计算请求计数并将其公开，Prometheus 会同时抓取计数指标并将值存储为 0。

到 11:01:00 时，处理了一个请求。服务器中的仪表逻辑会将计数递增为 1。当 Prometheus 扫描指标时，count 的值为 1。

到 11:02:00 时，又处理了两个请求，此时请求计数为 1+2 = 3。同样，度量值也会被采集和存储。
用户可以控制 Prometheus 抓取指标的频率。

|Time Stamp|Request Count (metric)|
|---|---|
|11:00:00|0|
|11:01:00|1|
|11:02:00|3|

> (注：此表只是为了便于理解。Prometheus 并不以这种精确格式存储值）

Prometheus 还有一个 API，允许查询通抓取已被存储下来的指标。该 API 可用于查询指标、创建仪表盘/图表等。PromQL 用于查询这些指标。

根据请求次数指标创建的简单折线图如下所示

![[Pasted image 20231107093111.png]]

用户可以抓取多个有用的指标来了解应用程序中发生的情况，并在这些指标上创建多个图表。将这些图表组合成一个仪表盘，用它来了解应用程序的概况。

### 看看怎么做

让我们动手安装 Prometheus。Prometheus 是用 Go 编写的，你需要的只是为你的操作系统编译的二进制文件。从这里下载与你的操作系统对应的二进制文件，并将二进制文件添加到你的路径中。

Prometheus 公开了自己的度量指标，这些指标可由自身或其他 Prometheus 服务器使用。
现在我们已经安装了 Prometheus，下一步就是运行它。我们需要的只是二进制文件和配置文件。Prometheus 使用 yaml 文件进行配置。

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets: ["localhost:9090"]
```

在上述配置文件中，我们提到了 `scrape_interval`，即我们希望 Prometheus 多长时间抓取一次指标。我们还添加了 `scrape_configs`，其中包含名称和目标，以便从这些名称和目标抓取指标。默认情况下，Prometheus 监听端口为 9090。因此将其添加到目标中。

```bash
prometheus --config.file=prometheus.yml
```


<iframe width="560" height="315" src="https://www.youtube.com/embed/ioa0eISf1Q0" title="prometheus intro" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>



现在，我们已经启动并运行 Prometheus，每隔 15 秒扫描一次自己的指标。Prometheus 有可用的标准导出器来导出指标。接下来，我们将运行一个node exporter，这是一个用于导出机器指标的导出器，并使用 Prometheus 抓取同样的指标。( [Download | Prometheus](https://prometheus.io/download/#node_exporter) ）。

在控制台运行node exporter:

```bash
./node_exporter
```

![[Pasted image 20231107093729.png]]

接下来，我们配置`scrape_configs`

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets: ["localhost:9090"]
  - job_name: node_exporter
    static_configs:
      - targets: ["localhost:9100"]
```


<iframe width="560" height="315" src="https://www.youtube.com/embed/hM5bp53C7Y8" title="node exporter demo" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>

