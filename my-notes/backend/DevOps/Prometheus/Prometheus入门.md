
### 下载

[Download | Prometheus](https://prometheus.io/download/)

```bash
tar xvfz prometheus-*.tar.gz
cd prometheus-*
```

### 配置

在启动之前，需要配置。先配置Prometheus监控它自己。

Prometheus 通过抓取 HTTP 端点指标从目标收集指标。由于 Prometheus 会以同样的方式公开自身的数据，因此它也可以采集和监控自身的健康状况。

虽然只收集自身数据的 Prometheus 服务器用处不大，但它是一个很好的入门范例。将以下 Prometheus 基本配置保存为名为 `prometheus.yml` 的文件：

```yaml
global:
  scrape_interval:     15s # By default, scrape targets every 15 seconds.

  # Attach these labels to any time series or alerts when communicating with
  # external systems (federation, remote storage, Alertmanager).
  external_labels:
    monitor: 'codelab-monitor'

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'

    # Override the global default and scrape targets from this job every 5 seconds.
    scrape_interval: 5s

    static_configs:
      - targets: ['localhost:9090']
```

完整的配置规范见  [Configuration | Prometheus](https://prometheus.io/docs/prometheus/latest/configuration/configuration/)

### 启动Prometheus

要使用新创建的配置文件启动 Prometheus，请切换到包含 Prometheus 二进制文件的目录并运行：

```bash
# Start Prometheus.
# By default, Prometheus stores its database in ./data (flag --storage.tsdb.path).
./prometheus --config.file=prometheus.yml
```

Prometheus 应该会启动。你也应该能在 `localhost:9090` 浏览到关于自身的状态页面。给它几秒钟时间，让它从自己的 HTTP 指标端点收集有关自身的数据。

您还可以通过导航到其指标端点：`localhost:9090/metrics` 来验证 Prometheus 是否正在提供有关自身的指标。

### 使用表达式浏览器

让我们探索一下 Prometheus 收集到的有关自身的数据。要使用 Prometheus 的内置表达式浏览器，请导航至 http://localhost:9090/graph  然后选择 "Graph（图表）"选项卡中的 "Table（表格）"视图。

您可以从 `localhost:9090/metrics` 中收集到，Prometheus 输出的关于自身的一个指标名为 `prometheus_target_interval_length_seconds`（目标抓取之间的实际时间间隔）。在表达式控制台中输入以下内容，然后点击 "执行"：

```
prometheus_target_interval_length_seconds
```

这将返回多个不同的时间序列（以及每个序列记录的最新值），每个序列的度量名称都是 `prometheus_target_interval_length_seconds`，但labels各不相同。这些标签表示不同的延迟百分位数和目标组间隔。

如果我们只对第 99 个百分位数的延迟感兴趣，可以使用此查询：

```
prometheus_target_interval_length_seconds{quantile="0.99"}
```

要计算返回时间序列的数量，可以这样写

```
count(prometheus_target_interval_length_seconds)
```

对于更多的表达式语言的文档，请看 https://prometheus.io/docs/prometheus/latest/querying/basics/

### 使用Graphing接口

要绘制表达式图表，请访问 http://localhost:9090/graph 并使用 "图表 "选项卡。
例如，输入以下表达式，即可绘制在自抓取 Prometheus 中创建块的每秒速率图：

```
rate(prometheus_tsdb_head_chunks_created_total[1m])
```

尝试使用图形范围参数和其他设置。

### 启动一些采样目标

让我们添加其他目标，以便 Prometheus 进行抓取。
Node Exporter 是一个示例目标，有关使用它的更多信息，请参阅[这些说明]([Monitoring Linux host metrics with the Node Exporter | Prometheus](https://prometheus.io/docs/guides/node-exporter/))。

```bash
tar -xzvf node_exporter-*.*.tar.gz
cd node_exporter-*.*

# Start 3 example targets in separate terminals:
./node_exporter --web.listen-address 127.0.0.1:8080
./node_exporter --web.listen-address 127.0.0.1:8081
./node_exporter --web.listen-address 127.0.0.1:8082
```

现在，您应该在 http://localhost:8080/metrics    http://localhost:8081/metrics 和 http://localhost:8082/metrics 上监听示例目标。

> node exporter会暴露出很多硬件和内核的指标，一般是运行在Unix-like的系统上

### 配置Prometheus监控这些采样目标

现在，我们将配置 Prometheus 来抓取这些新目标。让我们将所有三个端点归入一个名为 node 的任务中。我们假设前两个端点是生产目标，而第三个端点代表一个金丝雀实例。为了在 Prometheus 中建立模型，我们可以在单个作业中添加几组端点，并为每组目标添加额外的标签。在本例中，我们将为第一组目标添加 `group="production"`` 标签，同时为第二组目标添加 `group="canary"` 。

为此，请将以下作业定义添加到 `prometheus.yml` 中的 `scrape_configs` 部分，然后重启 Prometheus 实例：

```yaml
scrape_configs:
  - job_name:       'node'

    # Override the global default and scrape targets from this job every 5 seconds.
    scrape_interval: 5s

    static_configs:
      - targets: ['localhost:8080', 'localhost:8081']
        labels:
          group: 'production'

      - targets: ['localhost:8082']
        labels:
          group: 'canary'
```

转到表达式浏览器，验证 Prometheus 现在是否有这些示例端点暴露的时间序列信息，如 `node_cpu_seconds_total` 。

### 给抓取到的时间序列数据聚合成新的时间序列（配置这些规则）

虽然在我们的示例中问题不大，但在临时计算时，汇总数千个时间序列的查询可能会变得很慢。为了提高效率，Prometheus 可以通过配置的记录规则将表达式预先记录到新的持久化时间序列中。比方说，我们想记录在 5 分钟窗口内测量的每个实例所有 CPU 的平均每秒 CPU 时间率（node_cpu_seconds_total）（但保留job、instance和mode dimensions）。我们可以将其写成

```
avg by (job, instance, mode) (rate(node_cpu_seconds_total[5m]))
```

尝试绘制此表达式的图形。

要将此表达式产生的时间序列记录到名为 `job_instance_mode:node_cpu_seconds:avg_rate5m` 的新指标中，请创建包含以下记录规则的文件，并将其保存为 `prometheus.rules.yml`：

```yaml
groups:
- name: cpu-node
  rules:
  - record: job_instance_mode:node_cpu_seconds:avg_rate5m
    expr: avg by (job, instance, mode) (rate(node_cpu_seconds_total[5m]))
```

要让 Prometheus 采用这条新规则，请在 `prometheus.yml` 中添加 `rule_files` 语句。现在的配置应该是这样的

```yaml
global:
  scrape_interval:     15s # By default, scrape targets every 15 seconds.
  evaluation_interval: 15s # Evaluate rules every 15 seconds.

  # Attach these extra labels to all timeseries collected by this Prometheus instance.
  external_labels:
    monitor: 'codelab-monitor'

rule_files:
  - 'prometheus.rules.yml'

scrape_configs:
  - job_name: 'prometheus'

    # Override the global default and scrape targets from this job every 5 seconds.
    scrape_interval: 5s

    static_configs:
      - targets: ['localhost:9090']

  - job_name:       'node'

    # Override the global default and scrape targets from this job every 5 seconds.
    scrape_interval: 5s

    static_configs:
      - targets: ['localhost:8080', 'localhost:8081']
        labels:
          group: 'production'

      - targets: ['localhost:8082']
        labels:
          group: 'canary'
```

使用新配置重启 Prometheus，并通过表达式浏览器查询或绘制图表，验证指标名称为 `job_instance_mode:node_cpu_seconds:avg_rate5m` 的新时间序列是否可用。

### 重新加载配置

普罗米修斯支持热加载

### 优雅地关闭Prometheus

```bash
kill -s SIGTERM <pid>
```

