
## 基础

使用 Prometheus 设置警报分为两步：

首先，您需要在 Prometheus 中创建警报规则，并指定您希望在什么条件下收到警报（例如当实例关闭时）。
其次，您需要设置Alertmanager，它接收Prometheus中指定的警报。然后，Alertmanager 将能够执行多种操作，包括：
将相似性质的警报分组到单个通知中
在特定时间消除警报
如果其他指定警报已触发，则将某些警报的通知静音
选择哪些接收者接收特定警报

#### 第一步-在Prometheus中创建报警规则

我们从之前为每个项目设置的四个子文件夹开始：`server、node_exporter、github_exporter 和 prom_middleware`。我在[关于如何使用简单项目探索 Prometheus 的博文]([How to Explore Prometheus with Easy 'Hello World' Projects | Grafana Labs](https://grafana.com/blog/2019/12/04/how-to-explore-prometheus-with-easy-hello-world-projects/))中解释了这一过程。

我们移动到服务器子文件夹，在代码编辑器中打开内容，然后创建一个新的规则文件。在 `rules.yml` 文件中，你将指定需要发出警报的条件。

```bash
cd Prometheus/server
touch rules.yml
```

我相信大家都同意，知道任何实例何时宕机都是非常重要的。因此，我将使用 up 指标作为我们的条件。通过在 Prometheus 用户界面（http://localhost:9090）上评估该指标，你会发现所有正在运行的实例的值都是 1，而所有当前未运行的实例的值都是 0（我们当前只运行我们的 Prometheus 实例）。

![[Pasted image 20231127105500.png]]

确定警报条件后，您需要在 rules.yml 中指定它们。其内容如下：

```yaml
groups:
- name: AllInstances
  rules:
  - alert: InstanceDown
    # Condition for alerting
    expr: up == 0
    for: 1m
    # Annotation - additional informational labels to store more information
    annotations:
      title: 'Instance {{ $labels.instance }} down'
      description: '{{ $labels.instance }} of job {{ $labels.job }} has been down for more than 1 minute.'
    # Labels - additional labels to be attached to the alert
    labels:
      severity: 'critical'
```

概括地说，如果任何实例将在一分钟内宕机（up == 0 ）  那么警报就会触发。我还添加了注释和标签，用于存储有关警报的其他信息。对于这些信息，你可以使用[模板变量]([template package - text/template - Go Packages](https://pkg.go.dev/text/template))，如 {{ $labels.instance }}，然后将其插入特定实例（如 localhost:9100）。

(有关 Prometheus 警报规则的更多信息，[请点击此处]([Alerting rules | Prometheus](https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/))）。

准备好 `rules.yml` 后，需要将该文件链接到 `prometheus.yml` 并添加警报配置。您的 `prometheus.yml` 将如下所示：

```yaml
global:
  # How frequently to scrape targets
  scrape_interval:     10s
  # How frequently to evaluate rules
  evaluation_interval: 10s

# Rules and alerts are read from the specified file(s)
rule_files:
  - rules.yml

# Alerting specifies settings related to the Alertmanager
alerting:
  alertmanagers:
    - static_configs:
      - targets:
        # Alertmanager's default port is 9093
        - localhost:9093

# A list of scrape configurations that specifies a set of
# targets and parameters describing how to scrape them.
scrape_configs:
  - job_name: 'prometheus'
    scrape_interval: 5s
    static_configs:
      - targets:
        - localhost:9090
  - job_name: 'node_exporter'
    scrape_interval: 5s
    static_configs:
      - targets:
        - localhost:9100
  - job_name: 'prom_middleware'
    scrape_interval: 5s
    static_configs:
      - targets:
        - localhost:9091
```

如果使用`--web.enable-lifecycle`标志启动了Prometheus，可以通过向`/-/reload` endpoint `curl -X POST http://localhost:9090/-/reload` 发送POST请求来重新加载配置。然后，启动 prom_middleware 应用程序（prom_middleware 文件夹中的 `node index.js`）和 `node_exporter`（node_exporter 文件夹中的`./node_exporter`）。

这样做之后，无论何时想创建警报，只需停止 `node_exporter` 或 `prom_middleware `应用程序即可。

#### 第二部-设置AlertManager

在 Prometheus 文件夹中创建 `alert_manager` 子文件夹，`mkdir alert_manager`。然后从 Prometheus 网站下载并解压 Alertmanager 到该文件夹，在不修改 `alertmanager.yml` 的情况下运行` ./alertmanager --config.file=alertmanager.yml` 并打开 localhost:9093。

根据您是否有任何活动警报，Alertmanager 应该已正确设置，外观如下图所示。要查看上面步骤中添加的注释，请单击 `+Info` 按钮。

![[Pasted image 20231127110155.png]]

完成所有这些工作后，我们现在可以看看使用 Alertmanager 和发送警报通知的不同方法。

## 如何建立Slack的报警

如果您想通过 Slack 接收通知，您应该是 Slack 工作区(workspace)的一部分。如果您目前不属于任何 Slack 工作区，或者您想在单独的工作区中进行测试，您可以在这里快速创建一个工作区。

要在 Slack 工作区中设置警报，你需要一个 `Slack API URL`。转到 `Slack -> Administration -> Manage apps`。

![[Pasted image 20231127110336.png]]

在管理应用程序目录中，搜索 `Incoming WebHooks` 并将其添加到你的 Slack 工作区。

![[Pasted image 20231127110410.png]]


接下来，指定您希望从 Alertmanager 接收通知的频道（我创建了 `#monitoring-infrastructure` 频道）。(我创建了 `#monitoring-infrastructure` 频道。）确认并添加传入 WebHooks 集成后，会显示 webhook URL（即您的 Slack API URL）。复制它。

![[Pasted image 20231127110502.png]]

然后，您需要修改 `alertmanager.yml` 文件。首先，在代码编辑器中打开 alert_manager 子文件夹，然后根据下面的模板填写 `alertmanager.yml`。将刚才复制的 URL 用作 `slack_api_url`。

```yaml
global:
  resolve_timeout: 1m
  slack_api_url: 'https://hooks.slack.com/services/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'

route:
  receiver: 'slack-notifications'

receivers:
- name: 'slack-notifications'
  slack_configs:
  - channel: '#monitoring-instances'
    send_resolved: true
```

发送 POST 请求到 `/-/reload` endpoint `curl -X POST http://localhost:9093/-/reload`，重新加载配置。几分钟后（在至少停止一个实例后），你应该会通过 Slack 收到警报通知，就像这样：

![[Pasted image 20231127110700.png]]

如果您想改进您的通知，让它们看起来更漂亮，可以使用下面的模板

```yaml
global:
  resolve_timeout: 1m
  slack_api_url: 'https://hooks.slack.com/services/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'

route:
  receiver: 'slack-notifications'

receivers:
- name: 'slack-notifications'
  slack_configs:
  - channel: '#monitoring-instances'
    send_resolved: true
    icon_url: https://avatars3.githubusercontent.com/u/3380462
    title: |-
     [{{ .Status | toUpper }}{{ if eq .Status "firing" }}:{{ .Alerts.Firing | len }}{{ end }}] {{ .CommonLabels.alertname }} for {{ .CommonLabels.job }}
     {{- if gt (len .CommonLabels) (len .GroupLabels) -}}
       {{" "}}(
       {{- with .CommonLabels.Remove .GroupLabels.Names }}
         {{- range $index, $label := .SortedPairs -}}
           {{ if $index }}, {{ end }}
           {{- $label.Name }}="{{ $label.Value -}}"
         {{- end }}
       {{- end -}}
       )
     {{- end }}
    text: >-
     {{ range .Alerts -}}
     *Alert:* {{ .Annotations.title }}{{ if .Labels.severity }} - `{{ .Labels.severity }}`{{ end }}

     *Description:* {{ .Annotations.description }}

     *Details:*
       {{ range .Labels.SortedPairs }} • *{{ .Name }}:* `{{ .Value }}`
       {{ end }}
     {{ end }}
```

这个模板的最终效果如下:

![[Pasted image 20231127110833.png]]

## 如何建立PagerDuty的报警

PagerDuty 是 IT 部门最知名的事件响应平台之一。要通过 PagerDuty 设置警报，需要在那里创建一个账户。登录后，进入`Configuration -> Services -> + New Service`。

![[Pasted image 20231127110934.png]]

从集成类型列表中选择 Prometheus，并为服务命名。我决定将其命名为 Prometheus Alertmanager。(你也可以自定义事件设置，但我使用的是默认设置）。然后点击保存。

![[Pasted image 20231127111003.png]]

将显示集成密钥。复制密钥。

![[Pasted image 20231127111023.png]]

您需要更新 `alertmanager.yml` 的内容。内容应如下所示，但要使用自己的 `service_key`（来自 PagerDuty 的集成密钥）。`pagerduty_url` 应保持不变，并设置为 `https://events.pagerduty.com/v2/enqueue`。保存并重新启动 Alertmanager。


```yaml
global:
  resolve_timeout: 1m
  pagerduty_url: 'https://events.pagerduty.com/v2/enqueue'

route:
  receiver: 'pagerduty-notifications'

receivers:
- name: 'pagerduty-notifications'
  pagerduty_configs:
  - service_key: 0c1cc665a594419b6d215e81f4e38f7
    send_resolved: true
```

停止其中一个实例。几分钟后，PagerDuty 中应显示警报通知。

![[Pasted image 20231127111205.png]]

在 PagerDuty 用户设置中，您可以决定通知方式：电子邮件和/或电话。我选择了这两种方式，结果都成功了。

## 如何建立Gmail的报警

如果您希望直接通过电子邮件服务发送通知，设置就更简单了。Alertmanager 只需将消息传递给提供商（在本例中，我使用的是 Gmail），然后由提供商代为发送。

不建议您使用个人密码，因此您应该创建一个[应用程序密码]([Sign in with app passwords - Google Account Help](https://support.google.com/accounts/answer/185833?hl=en))。为此，请进入 `"Account Settings"->"Security"->"Signing in Google"->"App Password"`（如果你没有看到 "App password "选项，可能是你还没有设置 `"2-Step Verification"`，需要先进行设置）。复制新创建的密码。

![[Pasted image 20231127111420.png]]

您需要再次更新 `alertmanager.yml` 的内容。内容应与下面的示例相似。不要忘记将电子邮件地址替换为您自己的电子邮件地址，密码替换为您的新应用密码。

```yaml
global:
  resolve_timeout: 1m

route:
  receiver: 'gmail-notifications'

receivers:
- name: 'gmail-notifications'
  email_configs:
  - to: monitoringinstances@gmail.com
    from: monitoringinstances@gmail.com
    smarthost: smtp.gmail.com:587
    auth_username: monitoringinstances@gmail.com
    auth_identity: monitoringinstances@gmail.com
    auth_password: password
    send_resolved: true
```

同样，几分钟后（在你停止至少一个实例后），警报通知就会发送到你的 Gmail。

![[Pasted image 20231127111524.png]]

完结。