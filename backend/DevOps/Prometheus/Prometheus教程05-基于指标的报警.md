
在本教程中，我们将针对 ping_request_count 指标创建警报，我们在之前用 Go 编写的  [[Prometheus教程03-以http server的形式暴露应用自定义的指标]] 教程中对该指标进行了检测。


在本教程中，我们将在 `ping_request_count` 指标大于 5 时发出警报。当然，在生产环境中，肯定不是这么简单粗暴，报警不报警需要遵守最佳实践 [Alerting | Prometheus](https://prometheus.io/docs/practices/alerting/) 

从[此处]([Releases · prometheus/alertmanager (github.com)](https://github.com/prometheus/alertmanager/releases))为你的操作系统下载最新版本的 Alertmanager


Alertmanager 支持多种接收器，如电子邮件、webhook、pagerduty、slack 等，当警报触发时，它可以通过这些接收器发出通知。您可以在[这里]([Configuration | Prometheus](https://prometheus.io/docs/alerting/latest/configuration/)) 找到接收器列表和配置方法。

本教程将使用 webhook 作为接收器，请访问 [webhook.site]([Webhook.site - Test, process and transform emails and HTTP requests](https://webhook.site/)) 并复制 webhook URL，稍后我们将使用该 URL 配置 Alertmanager。

首先，让我们用 webhook 接收器设置 Alertmanager 以下的文件名叫`alertmanager.yml`

```yaml
global:
  resolve_timeout: 5m
route:
  receiver: webhook_receiver
receivers:
    - name: webhook_receiver
      webhook_configs:
        - url: '<INSERT-YOUR-WEBHOOK>'
          send_resolved: false
```

将 \<INSERT-YOUR-WEBHOOK\> 替换为我们之前在 alertmanager.yml 文件中复制的webhook，然后使用以下命令运行 Alertmanager。

`alertmanager --config.file=alertmanager.yml`

一旦 Alertmanager 启动并运行，请导航至 http://localhost:9093 然后就可以访问它了。

<iframe width="560" height="315" src="https://www.youtube.com/embed/RKXwHhQZ5RE" title="Set up Alertmanager" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>

既然我们已经用webhook接收器配置了 Alertmanager，那就把规则添加到 Prometheus 配置中吧。

```yaml
global:
 scrape_interval: 15s
 evaluation_interval: 10s
rule_files:
  - rules.yml
alerting:
  alertmanagers:
  - static_configs:
    - targets:
       - localhost:9093
scrape_configs:
 - job_name: prometheus
   static_configs:
       - targets: ["localhost:9090"]
 - job_name: simple_server
   static_configs:
       - targets: ["localhost:8090"]
```

如果你注意到在 Prometheus 配置中添加了 `evaluation_interval`、`rule_files` 和 `alerting` 部分，其中 `evaluation_interval` 定义了评估规则的时间间隔，`rule_files` 接受定义规则的 yaml 文件数组，而 `alerting` 部分则定义了 Alertmanager 配置。

正如本教程开头提到的，我们将创建一个基本规则，当 `ping_request_count` 值大于 5 时发出警报。

以下是`rules.yml`文件

```yaml
groups:
 - name: Count greater than 5
   rules:
   - alert: CountGreaterThan5
     expr: ping_request_count > 5
     for: 10s
```

运行prometheus：`prometheus --config.file=./prometheus.yml`

在浏览器中打开Prometheus的 http://localhost:9090/rules 查看规则。然后运行之前的 ping 服务器，访问 http://localhost:8090/ping 端点并刷新页面至少 6 次。您可以通过导航到ping服务器的 http://localhost:8090/metrics 端点检查 ping 计数。要查看警报状态，请访问Prometheus http://localhost:9090/alerts 。 一旦 ping_request_count > 5 为真超过 10 秒，状态将变为 FIRING。现在，如果您回到 `webhook.site` URL，就会看到警报信息。

<iframe width="560" height="315" src="https://www.youtube.com/embed/xaMXVrle98M" title="Alertmanager webhook example" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>

同样，Alertmanager 也可与其他接收器一起配置，以便在警报触发时发出通知。

