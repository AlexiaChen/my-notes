

> 没有报警的监控是不完美的
## 介绍

[AlertManager]([Alertmanager | Prometheus](https://prometheus.io/docs/alerting/latest/alertmanager/))  是一个二进制文件，用于处理 Prometheus server发送的警报，并通过电子邮件、Slack 或其他工具通知最终用户。本文只讨论 Slack 和邮件通知。

监控有助于预测系统中的潜在问题或通知当前问题，并提供有关问题的详细信息。警报有助于在问题发生时立即发出通知，并允许团队通过通知识别问题。

使用 Prometheus 发出警报的设置步骤如下：

- 设置和配置 AlertManager。
- 配置 Prometheus 上的配置文件，使其能够与 AlertManager 对话。
- 在 Prometheus server配置中定义警报规则。
- 在 AlertManager 中定义警报机制，以便通过 Slack 和邮件发送警报

## 架构

以下有一个AlertManager与Prometheus的基本架构

![[Pasted image 20231127085027.png]]

警报规则在 Prometheus 配置中定义。Prometheus 只是从其客户端应用程序（node exporter）中刮取（pull）指标。但是，如果出现任何警报条件，Prometheus 会将其推送给 AlertManager，后者会通过静音、抑制、分组和发送通知等流程来管理警报。静音是在给定时间内将警报静音。警报会与激活的静音警报进行匹配检查，如果发现匹配，则不会发送通知。抑制是在其他警报已经触发的情况下抑制某些警报的通知。分组是将性质相似的警报分组为一个通知。这有助于防止同时向 Mail 或 Slack 等接收器发送多个通知。

## AlertManager的安装


我们所讲的配置，是基于以下的架构:

![[Pasted image 20231127085315.png]]

安装Node exporter。[[Prometheus教程05-基于指标的报警]]

下载安装最新版的AlertManager二进制:

```bash
sudo su
cd /opt/
wget https://github.com/prometheus/alertmanager/releases/download/v0.11.0/alertmanager-0.11.0.linux-amd64.tar.gz
tar -xvzf alertmanager-0.11.0.linux-amd64.tar.gz
mv alertmanager-0.11.0.linux-amd64/alertmanager /usr/local/bin/
```

### AlertManager的配置

AlertManager 使用名为 `alertmanager.yml` 的配置文件，该文件包含在解压缩目录中。不过，我们用不上它。因此，我们需要创建自己的 `alertmanager.yml`

```bash
mkdir /etc/alertmanager/
sudo nano /etc/alertmanager/alertmanager.yml
```

然后把以下内容放进去:

```yaml
global:
  smtp_from: 
  smtp_smarthost: 
  smtp_auth_username: 
  smtp_auth_password:
templates:
- '/etc/alertmanager/template/*.tmpl'
route:
 group_by: ['alertname']
 group_wait: 3s
 group_interval: 5s
 repeat_interval: 1h
receiver: mail-slack-receiver
receivers:
- name: 'mail-slack-receiver'
  slack_configs:
  - api_url: put your url here
    channel: 'put your channel name here'
    send_resolved: true
    icon_url: https://avatars3.githubusercontent.com/u/3380462
    text: >-
        {{ range .Alerts -}}
        *Alert:* {{ .Annotations.title }}{{ if .Labels.severity }} - `{{ .Labels.severity }}`{{ end }}
*Description:* {{ .Annotations.description }}
*Details:*
          {{ range .Labels.SortedPairs }} • *{{ .Name }}:* `{{ .Value }}`
          {{ end }}
        {{ end }}
email_configs:
  - to: 'emails of the ones that need to be notified'
    send_resolved: true
```

最后，我们就创建AlertManager的systemd服务:

```bash
nano /etc/systemd/system/alertmanager.service
```

内容:

```ini
[Unit]
Description=AlertManager Server Service
Wants=network-online.target
After=network-online.target

[Service]
User=root
Group=root
Type=simple
ExecStart=/usr/local/bin/alertmanager --config.file /etc/alertmanager/alertmanager.yml -web.external-url=http://x.x.x.x:9093


[Install]
WantedBy=multi-user.target
```

> 使用 -web.external-url=http://x.x.x.x:9093 允许将通知 URL 重定向到 prometheus AlertManager Web 界面。x.x.x.x 对应 prometheus 服务器的Public IP。

重新加载Daemon和启动alertmanager

```bash
systemctl daemon-reload
systemctl start alertmanager
systemctl enable alertmanager
systemctl status alertmanager
```

然后，你可以进去 `x.x.x.x:9093` 看看:

![[Pasted image 20231127090344.png]]

到这里，我们就完成AlertManager的配置了。


## AlertManager集成进Grafana

您还可以在 Grafana 中安装 Prometheus Alertmanager 插件。前往安装 grafana 的实例并安装插件：

```bash
grafana-cli plugins install camptocamp-prometheus-alertmanager-datasource
```

一旦插件安装完成，重启Grafana:

```bash
service grafana-server restart
```

访问URL `x.x.x.x:3030` 然后配置AlertManager的Prometheus数据源

![[Pasted image 20231127090739.png]]

然后安装这个仪表盘  [Prometheus AlertManager | Grafana Labs](https://grafana.com/grafana/dashboards/8010-prometheus-alertmanager/)

![[Pasted image 20231127090816.png]]

如果可以看到以下界面，就表示完成了:

![[Pasted image 20231127090847.png]]

## AlertManager集成进Prometheus

### 集成

可以看看Prometheus的Web界面:  `x.x.x.x:9090`

![[Pasted image 20231127091119.png]]

现在，我们需要配置 Prometheus server，使其能够与 AlertManager 服务对话。我们将设置一个警报规则文件，其中定义了触发警报所需的所有规则。

在 `/etc/prometheus/prometheus.yml` 中添加以下内容:

```yaml
rule_files:
  - alert.rules.yml
alerting:
  alertmanagers:
  - static_configs:
    - targets:
      - 'localhost:9093'
```

最后，我们看到了 `etc/prometheus/prometheus.yml` 文件：

```yaml
global:
  scrape_interval: 10s
rule_files:
  - alert.rules.yml
alerting:
  alertmanagers:
  - static_configs:
    - targets:
      - 'localhost:9093'
scrape_configs:
  - job_name: 'prometheus_metrics'
    scrape_interval: 5s
    static_configs:
      - targets: ['localhost:9090']
  - job_name: 'node_exporter_metrics'
    scrape_interval: 5s
    static_configs:
      - targets: ['localhost:9100','prometheus-target-1:9100','prometheus-target-2:9100']
```

Prometheus 服务器将跟踪传入的时间序列数据，一旦符合 `etc/prometheus/alert.rules.yml` 中定义的任何规则，就会向 AlertManager 服务触发警报，并通知 Slack 上的客户端。

以下是`/etc/prometheus/alert.rules.yml`的内容:

```yaml
groups:
- name: alert.rules
  rules:
  - alert: InstanceDown
    expr: up == 0
    for: 1m
    labels:
      severity: "critical"
    annotations:
      summary: "Endpoint {{ $labels.instance }} down"
      description: "{{ $labels.instance }} of job {{ $labels.job }} has been down for more than 1 minutes."
```

上述警报规则检查实例是否停机。如果停机超过 1 分钟，Prometheus 就会触发警报。我们可以使用 "promtool "工具检查警报文件的语法是否正确。

```bash
promtool check rules alert.rules.yml
```

你可以在这里找到更多你感兴趣的警报规则，可以直接拿来用: [samber/awesome-prometheus-alerts: 🚨 Collection of Prometheus alerting rules (github.com)](https://github.com/samber/awesome-prometheus-alerts)

重启:

```bash
sudo systemctl stop node_exporter &&
sudo systemctl start node_exporter &&
sudo systemctl stop prometheus &&
sudo systemctl start prometheus &&
sudo systemctl stop alertmanager &&
sudo systemctl start alertmanager
```

现在，如果我关闭其中一个目标实例（例如之前的图中，基于架构的target 1），我将在我的 AlertManager 和 Slack 上收到警报：

![[Pasted image 20231127091907.png]]

![[Pasted image 20231127091916.png]]


### Prometheus自定义报警规则

在此，我们将定义一组规则，以便在 CPU 负载、内存或磁盘使用量超过特定阈值或任何受监控实例出现故障时发出警报。

访问 `/etc/prometheus/alert.rules.yml` 文件并输入以下内容：

```yaml
groups:
- name: alert.rules
  rules:
  - alert: InstanceDown
    expr: up == 0
    for: 1m
    labels:
      severity: "critical"
    annotations:
      summary: "Endpoint {{ $labels.instance }} down"
      description: "{{ $labels.instance }} of job {{ $labels.job }} has been down for more than 1 minutes."
  
  - alert: HostOutOfMemory
    expr: node_memory_MemAvailable / node_memory_MemTotal * 100 < 25
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Host out of memory (instance {{ $labels.instance }})"
      description: "Node memory is filling up (< 25% left)\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"


  - alert: HostOutOfDiskSpace
    expr: (node_filesystem_avail{mountpoint="/"}  * 100) / node_filesystem_size{mountpoint="/"} < 50
    for: 1s
    labels:
      severity: warning
    annotations:
      summary: "Host out of disk space (instance {{ $labels.instance }})"
      description: "Disk is almost full (< 50% left)\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"


  - alert: HostHighCpuLoad
    expr: (sum by (instance) (irate(node_cpu{job="node_exporter_metrics",mode="idle"}[5m]))) > 80
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Host high CPU load (instance {{ $labels.instance }})"
      description: "CPU load is > 80%\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"
```

在上述配置中，我们定义了 4 个警报。每个警报都在 `expr` 中定义了自己的规则。`expr` 由查询（左侧）和条件（右侧）组成。

例如，`HostHighCpuLoad` 规则检查node exporter的 CPU 负载百分比是否大于 80%。

为了监控基础设施上 CPU、内存和磁盘负载的百分比，您可以导入此 grafana dashboard并将其链接到 prometheus 数据源。

还可以将之前在 `/etc/systemd/system/alertmanager.service` 中定义的 `-web.external-urllink` 改为`-web.external-url=http://x.x.x.x:3000`。这样，Slack 和 Mail 上收到的警报链接将直接重定向到我们自定义的 Grafana 控制面板。

如果你想自定义仪表盘以获取 CPU 负载、内存和磁盘使用情况，可以使用这些查询。

```txt
Grafana Queries: 
1. CPU load : 

sum(irate(node_cpu_seconds_total{mode="idle",instance=~'$node'}[5m])) or sum(irate(node_cpu{mode="idle",instance=~'$node'}[5m]))

2. Memory Usage : 

100-(node_memory_MemAvailable{instance=~'$node'}/node_memory_MemTotal{instance=~'$node'}*100)

3. Disk Space Usage

100-((node_filesystem_avail{instance=~'$node',mountpoint="/"}  * 100) / node_filesystem_size{instance=~'$node',mountpoint="/"})
```


