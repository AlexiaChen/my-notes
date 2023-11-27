
## 报警概览

Prometheus 的警报分为两个部分。Prometheus 服务器中的警报规则向警报管理器发送警报。然后 Alertmanager 管理这些警报，包括静音、抑制、聚合以及通过电子邮件、on-call通知系统和聊天平台等方式发送通知。

设置警报和通知的主要步骤是

- 设置和 [[#配置]] Alertmanager
- 配置 Prometheus 与 Alertmanager 对话
- 在 Prometheus 中创建[警报规则]([Alerting rules | Prometheus](https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/))
## Alertmanager

[Alertmanager]([prometheus/alertmanager: Prometheus Alertmanager (github.com)](https://github.com/prometheus/alertmanager)) 处理客户端应用程序（例如 Prometheus 服务器）发送的警报。它负责重复数据删除、分组，并将它们路由到正确的接收器（receiver）集成，例如电子邮件、PagerDuty 或 OpsGenie。它还负责警报的静音和抑制。

下面介绍Alertmanager实现的核心概念。请参阅配置文档  [[#配置]] 以了解如何更详细地使用它们。

### 分组(Grouping)

分组将性质相似的警报归类到一个通知中。当许多系统同时发生故障，成百上千个警报可能同时响起时，这在较大的故障期间尤其有用。

例如： 当网络分区发生时，群集中正在运行数十或数百个服务实例。一半的服务实例无法再访问数据库。Prometheus 中的警报规则被配置为在每个服务实例无法与数据库通信时为其发送警报。结果，数百条警报被发送到 Alertmanager。

作为用户，我们只希望获得一个页面，同时还能看到受影响的服务实例。因此，可以对 Alertmanager 进行配置，按群集和警报名称对警报进行分组，这样就能发送一个简洁的通知。

警报分组、分组通知的时间以及这些通知的接收者都是通过配置文件中的路由树来配置的。

### 抑制(Inhibition)

抑制的概念是，如果某些其他警报已经触发，则抑制某些警报的通知。

例如： 一个警报正在发出，通知整个群集无法访问。可以对 Alertmanager 进行配置，使其在该特定警报触发时静音处理与该群集有关的所有其他警报。这样就可以防止数百或数千个与实际问题无关的警报触发通知。

抑制功能可通过 Alertmanager 的配置文件进行配置。

### 静音(Silences)

静音是将警报静音一段时间的直接方法。静音是根据匹配器配置的，就像路由树一样。会检查传入的警报是否与活动静默的所有等价或正则表达式匹配器匹配。如果匹配，则不会为该警报发送通知。

静默在 Alertmanager 的 Web 界面中进行配置。

### 客户端行为

Alertmanager 对其客户端的行为有特殊要求  [[#客户端]] 。这些要求只适用于不使用 Prometheus 发送警报的高级用例。

### 高可用

Alertmanager 支持为高可用性创建集群的配置。可以使用 `--cluster-*` flag进行配置。
重要的是，不要在 Prometheus 及其 Alertmanager 之间进行负载均衡，而是将 Prometheus 指向所有 Alertmanager 的列表。

## 配置

Alertmanager 通过命令行标志和配置文件进行配置。命令行标志用于配置不可更改的系统参数，而配置文件则用于定义抑制规则、通知路由和通知接收器。

[可视化编辑器]([Routing tree editor | Prometheus](https://www.prometheus.io/webtools/alerting/routing-tree-editor/))可帮助建立路由树。

要查看所有可用的命令行标志，请运行 `alertmanager -h`。

Alertmanager 可以在运行时重新加载配置。如果新的配置不完善，更改将无法应用，并记录错误。向进程发送 `SIGHUP` 或向 `/-/reload` 端点发送 HTTP POST 请求都会触发配置重载。

### 配置文件介绍

要指定要加载的配置文件，请使用 `--config.file` 标志。

```bash
./alertmanager --config.file=alertmanager.yml
```

文件以 YAML 格式写入，由下面描述的方案定义。括号表示参数为可选参数。对于非列表参数，其值将设置为指定的默认值。

通用占位符定义如下：

![[Pasted image 20231127095638.png]]

其他占位符单独指定。

这个是一个简单的使用例子: [alertmanager/doc/examples/simple.yml at main · prometheus/alertmanager (github.com)](https://github.com/prometheus/alertmanager/blob/main/doc/examples/simple.yml) 

### 文件布局和全局设置

全局配置指定了在所有其他配置上下文中都有效的参数。它们也是其他配置部分的默认值。本页下面将介绍其他顶级配置部分。

```yaml
global:
  # The default SMTP From header field.
  [ smtp_from: <tmpl_string> ]
  # The default SMTP smarthost used for sending emails, including port number.
  # Port number usually is 25, or 587 for SMTP over TLS (sometimes referred to as STARTTLS).
  # Example: smtp.example.org:587
  [ smtp_smarthost: <string> ]
  # The default hostname to identify to the SMTP server.
  [ smtp_hello: <string> | default = "localhost" ]
  # SMTP Auth using CRAM-MD5, LOGIN and PLAIN. If empty, Alertmanager doesn't authenticate to the SMTP server.
  [ smtp_auth_username: <string> ]
  # SMTP Auth using LOGIN and PLAIN.
  [ smtp_auth_password: <secret> ]
  # SMTP Auth using LOGIN and PLAIN.
  [ smtp_auth_password_file: <string> ]
  # SMTP Auth using PLAIN.
  [ smtp_auth_identity: <string> ]
  # SMTP Auth using CRAM-MD5.
  [ smtp_auth_secret: <secret> ]
  # The default SMTP TLS requirement.
  # Note that Go does not support unencrypted connections to remote SMTP endpoints.
  [ smtp_require_tls: <bool> | default = true ]

  # The API URL to use for Slack notifications.
  [ slack_api_url: <secret> ]
  [ slack_api_url_file: <filepath> ]
  [ victorops_api_key: <secret> ]
  [ victorops_api_key_file: <filepath> ]
  [ victorops_api_url: <string> | default = "https://alert.victorops.com/integrations/generic/20131114/alert/" ]
  [ pagerduty_url: <string> | default = "https://events.pagerduty.com/v2/enqueue" ]
  [ opsgenie_api_key: <secret> ]
  [ opsgenie_api_key_file: <filepath> ]
  [ opsgenie_api_url: <string> | default = "https://api.opsgenie.com/" ]
  [ wechat_api_url: <string> | default = "https://qyapi.weixin.qq.com/cgi-bin/" ]
  [ wechat_api_secret: <secret> ]
  [ wechat_api_corp_id: <string> ]
  [ telegram_api_url: <string> | default = "https://api.telegram.org" ]
  [ webex_api_url: <string> | default = "https://webexapis.com/v1/messages" ]
  # The default HTTP client configuration
  [ http_config: <http_config> ]

  # ResolveTimeout is the default value used by alertmanager if the alert does
  # not include EndsAt, after this time passes it can declare the alert as resolved if it has not been updated.
  # This has no impact on alerts from Prometheus, as they always include EndsAt.
  [ resolve_timeout: <duration> | default = 5m ]

# Files from which custom notification template definitions are read.
# The last component may use a wildcard matcher, e.g. 'templates/*.tmpl'.
templates:
  [ - <filepath> ... ]

# The root node of the routing tree.
route: <route>

# A list of notification receivers.
receivers:
  - <receiver> ...

# A list of inhibition rules.
inhibit_rules:
  [ - <inhibit_rule> ... ]

# DEPRECATED: use time_intervals below.
# A list of mute time intervals for muting routes.
mute_time_intervals:
  [ - <mute_time_interval> ... ]

# A list of time intervals for muting/activating routes.
time_intervals:
  [ - <time_interval> ... ]
```

### 路由相关的设置

路由相关设置允许配置警报如何根据时间路由、聚合、限制和静音。

\<route\>

路由块定义路由树中的一个节点及其子节点。如果未设置路由块的可选配置参数，则从其父节点继承。

每个警报都从配置的顶级路由进入路由树，该路由必须与所有警报匹配（即没有任何配置的匹配器）。然后它会遍历子节点。如果 `continue` 设置为 false，则会在第一个匹配的子节点后停止。如果匹配节点上的 `continue` 设置为 true，则警报将继续匹配后续的同级节点。如果警报不匹配节点的任何子节点（没有匹配的子节点或不存在匹配的子节点），则会根据当前节点的配置参数处理警报。

```yaml
[ receiver: <string> ]
# The labels by which incoming alerts are grouped together. For example,
# multiple alerts coming in for cluster=A and alertname=LatencyHigh would
# be batched into a single group.
#
# To aggregate by all possible labels use the special value '...' as the sole label name, for example:
# group_by: ['...']
# This effectively disables aggregation entirely, passing through all
# alerts as-is. This is unlikely to be what you want, unless you have
# a very low alert volume or your upstream notification system performs
# its own grouping.
[ group_by: '[' <labelname>, ... ']' ]

# Whether an alert should continue matching subsequent sibling nodes.
[ continue: <boolean> | default = false ]

# DEPRECATED: Use matchers below.
# A set of equality matchers an alert has to fulfill to match the node.
match:
  [ <labelname>: <labelvalue>, ... ]

# DEPRECATED: Use matchers below.
# A set of regex-matchers an alert has to fulfill to match the node.
match_re:
  [ <labelname>: <regex>, ... ]

# A list of matchers that an alert has to fulfill to match the node.
matchers:
  [ - <matcher> ... ]

# How long to initially wait to send a notification for a group
# of alerts. Allows to wait for an inhibiting alert to arrive or collect
# more initial alerts for the same group. (Usually ~0s to few minutes.)
[ group_wait: <duration> | default = 30s ]

# How long to wait before sending a notification about new alerts that
# are added to a group of alerts for which an initial notification has
# already been sent. (Usually ~5m or more.)
[ group_interval: <duration> | default = 5m ]

# How long to wait before sending a notification again if it has already
# been sent successfully for an alert. (Usually ~3h or more).
# Note that this parameter is implicitly bound by Alertmanager's
# `--data.retention` configuration flag. Notifications will be resent after either
# repeat_interval or the data retention period have passed, whichever
# occurs first. `repeat_interval` should not be less than `group_interval`.
[ repeat_interval: <duration> | default = 4h ]

# Times when the route should be muted. These must match the name of a
# mute time interval defined in the mute_time_intervals section.
# Additionally, the root node cannot have any mute times.
# When a route is muted it will not send any notifications, but
# otherwise acts normally (including ending the route-matching process
# if the `continue` option is not set.)
mute_time_intervals:
  [ - <string> ...]

# Times when the route should be active. These must match the name of a
# time interval defined in the time_intervals section. An empty value
# means that the route is always active.
# Additionally, the root node cannot have any active times.
# The route will send notifications only when active, but otherwise
# acts normally (including ending the route-matching process
# if the `continue` option is not set).
active_time_intervals:
  [ - <string> ...]

# Zero or more child routes.
routes:
  [ - <route> ... ]
```

例子:

```yaml
# The root route with all parameters, which are inherited by the child
# routes if they are not overwritten.
route:
  receiver: 'default-receiver'
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  group_by: [cluster, alertname]
  # All alerts that do not match the following child routes
  # will remain at the root node and be dispatched to 'default-receiver'.
  routes:
  # All alerts with service=mysql or service=cassandra
  # are dispatched to the database pager.
  - receiver: 'database-pager'
    group_wait: 10s
    matchers:
    - service=~"mysql|cassandra"
  # All alerts with the team=frontend label match this sub-route.
  # They are grouped by product and environment rather than cluster
  # and alertname.
  - receiver: 'frontend-pager'
    group_by: [product, environment]
    matchers:
    - team="frontend"

  # All alerts with the service=inhouse-service label match this sub-route.
  # the route will be muted during offhours and holidays time intervals.
  # even if it matches, it will continue to the next sub-route
  - receiver: 'dev-pager'
    matchers:
      - service="inhouse-service"
    mute_time_intervals:
      - offhours
      - holidays
    continue: true

    # All alerts with the service=inhouse-service label match this sub-route
    # the route will be active only during offhours and holidays time intervals.
  - receiver: 'on-call-pager'
    matchers:
      - service="inhouse-service"
    active_time_intervals:
      - offhours
      - holidays
```

\<time_interval\>

`time_interval` 指定了一个指定的时间间隔，可在路由树中引用，以便在一天中的特定时间静音/激活特定路由。

\<time_interval_spec>\

`time_interval_spec` 包含时间间隔的实际定义。该语法支持以下字段：

```yaml
- times:
  [ - <time_range> ...]
  weekdays:
  [ - <weekday_range> ...]
  days_of_month:
  [ - <days_of_month_range> ...]
  months:
  [ - <month_range> ...]
  years:
  [ - <year_range> ...]
  location: <string>
```

所有字段都是列表。在每个非空列表中，必须至少满足一个元素才能匹配字段。如果字段未指定，则任何值都会匹配该字段。要使一个时间瞬间匹配一个完整的时间间隔，所有字段都必须匹配。某些字段支持范围和负指数，详情如下。如果未指定时区，则以 UTC 时间为准。

时间范围： 范围包括起始时间，不包括结束时间，以便于表示以小时为界限的起始/结束时间。例如，`start_time: "17:00 "`和 `end_time: "24:00 "`将从 17:00 开始，在 24:00 之前结束。它们是这样指定的

```yaml
 times:
    - start_time: HH:MM
      end_time: HH:MM
```

`weekday_range`： 一周的天数列表，一周从周日开始，到周六结束。天数应按名称指定（如 "星期日"）。为方便起见，也接受 `<start_day>:<end_day>`` 形式的范围，且两端都包含在内。例如 `\[周一：周三"、"周六"、"周日\］`
`days_of_month_range`： 每月天数的数字列表。也接受从月末开始的负值，例如，1 月份的 -1 代表 1 月 31 日。例如 `['1:5', '-3:-1']`. 超过月初或月末的值将被箝位。例如，在二月份指定`['1:31']`，根据闰年的不同，实际结束日期将被箝位在 28 或 29 日。两端均包含。
`month_range`： 不区分大小写的名称（如 "1 月"）或数字标识的日历月份列表，其中 1 月 = 1。也接受范围。例如，`['1:3'、'5 月:8 月'、'12 月']`。两端都包含。
`year_range`： 年份的数字列表。接受范围。例如，`['2020:2022', '2030']`。两端均包含。
`location`：位置： 与 IANA 时区数据库中的位置相匹配的字符串。例如，`"Ausrralia/Sydbey"`。位置为时间间隔提供时区。例如，位置为 "Ausrralia/Sydbey "的时间间隔包含以下内容：

```yaml
 times:
    - start_time: 09:00
      end_time: 17:00
    weekdays: ['monday:friday']
```

将包括星期一至星期五上午 9:00 至下午 5:00 之间的任何时间，使用澳大利亚悉尼的当地时间。
您也可以使用 `"Local "`作为位置，使用 Alertmanager 运行所在机器的当地时间，或使用 "UTC "作为UTC 时间。注意：在 Windows 系统中，除非使用 `ZONEINFO` 环境变量提供自定义时区数据库，否则只支持本地时间或 UTC 时间。

### 抑制相关的设置

当存在与另一组匹配器匹配的警报（源）时，抑制规则会抑制与一组匹配器匹配的警报（目标）。目标警报和源警报在相等列表中的标签名称必须具有相同的标签值。

从语义上讲，缺失的标签和空值的标签是一回事。因此，如果源警报和目标警报中都缺少等号中列出的所有标签名称，则抑制规则将适用。

为防止警报抑制自身，与规则的目标方和源方都匹配的警报不能被相同情况的警报（包括自身）抑制。不过，我们建议在选择目标和源匹配器时，不要让警报同时匹配两边。这样更容易推理，也不会触发这种特殊情况。

```yaml
# DEPRECATED: Use target_matchers below.
# Matchers that have to be fulfilled in the alerts to be muted.
target_match:
  [ <labelname>: <labelvalue>, ... ]
# DEPRECATED: Use target_matchers below.
target_match_re:
  [ <labelname>: <regex>, ... ]

# A list of matchers that have to be fulfilled by the target
# alerts to be muted.
target_matchers:
  [ - <matcher> ... ]

# DEPRECATED: Use source_matchers below.
# Matchers for which one or more alerts have to exist for the
# inhibition to take effect.
source_match:
  [ <labelname>: <labelvalue>, ... ]
# DEPRECATED: Use source_matchers below.
source_match_re:
  [ <labelname>: <regex>, ... ]

# A list of matchers for which one or more alerts have
# to exist for the inhibition to take effect.
source_matchers:
  [ - <matcher> ... ]

# Labels that must have an equal value in the source and target
# alert for the inhibition to take effect.
[ equal: '[' <labelname>, ... ']' ]
```

### Label matchers

https://prometheus.io/docs/alerting/latest/configuration/#label-matchers

### 通用的receiver相关的设置

这些接收器设置允许为基于 HTTP 的接收器配置通知目的地（接收器）和 HTTP 客户端选项。

\<receiver\>

接收器是一个或多个通知集成的命名配置。

注：作为取消过去暂停使用新接收器规定的一部分，除现有要求外，新的通知集成还必须有一个具有推送权限的承诺维护者。

```yaml
# The unique name of the receiver.
name: <string>

# Configurations for several notification integrations.
discord_configs:
  [ - <discord_config>, ... ]
email_configs:
  [ - <email_config>, ... ]
msteams_configs:
  [ - <msteams_config>, ... ]
opsgenie_configs:
  [ - <opsgenie_config>, ... ]
pagerduty_configs:
  [ - <pagerduty_config>, ... ]
pushover_configs:
  [ - <pushover_config>, ... ]
slack_configs:
  [ - <slack_config>, ... ]
sns_configs:
  [ - <sns_config>, ... ]
telegram_configs:
  [ - <telegram_config>, ... ]
victorops_configs:
  [ - <victorops_config>, ... ]
webex_configs:
  [ - <webex_config>, ... ]
webhook_configs:
  [ - <webhook_config>, ... ]
wechat_configs:
  [ - <wechat_config>, ... ]
```

\<http_config\>

`http_config` 允许配置接收方用于与基于 HTTP 的 API 服务进行通信的 HTTP 客户端。

```yaml
# Note that `basic_auth` and `authorization` options are mutually exclusive.

# Sets the `Authorization` header with the configured username and password.
# password and password_file are mutually exclusive.
basic_auth:
  [ username: <string> ]
  [ password: <secret> ]
  [ password_file: <string> ]

# Optional the `Authorization` header configuration.
authorization:
  # Sets the authentication type.
  [ type: <string> | default: Bearer ]
  # Sets the credentials. It is mutually exclusive with
  # `credentials_file`.
  [ credentials: <secret> ]
  # Sets the credentials with the credentials read from the configured file.
  # It is mutually exclusive with `credentials`.
  [ credentials_file: <filename> ]

# Optional OAuth 2.0 configuration.
# Cannot be used at the same time as basic_auth or authorization.
oauth2:
  [ <oauth2> ]

# Whether to enable HTTP2.
[ enable_http2: <bool> | default: true ]

# Optional proxy URL.
[ proxy_url: <string> ]
# Comma-separated string that can contain IPs, CIDR notation, domain names
# that should be excluded from proxying. IP and domain names can
# contain port numbers.
[ no_proxy: <string> ]
# Use proxy URL indicated by environment variables (HTTP_PROXY, https_proxy, HTTPs_PROXY, https_proxy, and no_proxy)
[ proxy_from_environment: <boolean> | default: false ]
# Specifies headers to send to proxies during CONNECT requests.
[ proxy_connect_header:
  [ <string>: [<secret>, ...] ] ]

# Configure whether HTTP requests follow HTTP 3xx redirects.
[ follow_redirects: <bool> | default = true ]

# Configures the TLS settings.
tls_config:
  [ <tls_config> ]
```

\<oauth2\>

使用客户端凭证授予类型进行 OAuth 2.0 身份验证。Alertmanager 使用给定的客户端访问密钥和秘密密钥从指定端点获取访问令牌。

```yaml
client_id: <string>
[ client_secret: <secret> ]

# Read the client secret from a file.
# It is mutually exclusive with `client_secret`.
[ client_secret_file: <filename> ]

# Scopes for the token request.
scopes:
  [ - <string> ... ]

# The URL to fetch the token from.
token_url: <string>

# Optional parameters to append to the token URL.
endpoint_params:
  [ <string>: <string> ... ]

# Configures the token request's TLS settings.
tls_config:
  [ <tls_config> ]

# Optional proxy URL.
[ proxy_url: <string> ]
# Comma-separated string that can contain IPs, CIDR notation, domain names
# that should be excluded from proxying. IP and domain names can
# contain port numbers.
[ no_proxy: <string> ]
# Use proxy URL indicated by environment variables (HTTP_PROXY, https_proxy, HTTPs_PROXY, https_proxy, and no_proxy)
[ proxy_from_environment: <boolean> | default: false ]
# Specifies headers to send to proxies during CONNECT requests.
[ proxy_connect_header:
  [ <string>: [<secret>, ...] ] ]
```

\<tls_config\>

`tls_config` 允许配置 TLS 连接。

```yaml
# CA certificate to validate the server certificate with.
[ ca_file: <filepath> ]

# Certificate and key files for client cert authentication to the server.
[ cert_file: <filepath> ]
[ key_file: <filepath> ]

# ServerName extension to indicate the name of the server.
# http://tools.ietf.org/html/rfc4366#section-3.1
[ server_name: <string> ]

# Disable validation of the server certificate.
[ insecure_skip_verify: <boolean> | default = false]

# Minimum acceptable TLS version. Accepted values: TLS10 (TLS 1.0), TLS11 (TLS
# 1.1), TLS12 (TLS 1.2), TLS13 (TLS 1.3).
# If unset, Prometheus will use Go default minimum version, which is TLS 1.2.
# See MinVersion in https://pkg.go.dev/crypto/tls#Config.
[ min_version: <string> ]
# Maximum acceptable TLS version. Accepted values: TLS10 (TLS 1.0), TLS11 (TLS
# 1.1), TLS12 (TLS 1.2), TLS13 (TLS 1.3).
# If unset, Prometheus will use Go default maximum version, which is TLS 1.3.
# See MaxVersion in https://pkg.go.dev/crypto/tls#Config.
[ max_version: <string> ]
```

### Receiver集成相关的设置

这些设置允许配置特定的接收器集成。比如Discord， email， wechat， https://prometheus.io/docs/alerting/latest/configuration/#receiver-integration-settings

## 客户端

其实客户端也就是发送报警的机制。就是向AlertManager发送报警的地方都算”客户端“。

> 免责声明：Prometheus 会自动发送由其配置的警报规则生成的警报。强烈建议根据时间序列数据在 Prometheus 中配置警报规则，而不是直接使用客户端,。 也就是直接调用alertmanager的Http API

Alertmanager 有两个APIs，即 v1 和 v2，两者都监听警报。下面的代码片段描述了 v1 的方案。v2 的方案被指定为 OpenAPI 规范，可在 [Alertmanager 存储库中找到]([alertmanager/api/v2/openapi.yaml at main · prometheus/alertmanager (github.com)](https://github.com/prometheus/alertmanager/blob/main/api/v2/openapi.yaml))。只要警报仍处于活动状态（通常为 30 秒至 3 分钟），客户端就会不断重新发送警报。客户端可通过 POST 请求向 Alertmanager 推送警报列表。

每个警报的标签用于识别警报的相同实例并执行重复数据删除。注释总是设置为最近收到的注释，而不是识别警报的注释。

`startsAt` 和 `endsAt` 时间戳均为可选项。如果省略 `startsAt`，警报管理器就会指定当前时间。否则，它将被设置为可配置的超时时间，从上次收到警报的时间算起。

`GeneratorURL` 字段是一个唯一的反向链接，用于在客户端中识别导致此警报的实体。


```JSON
[
  {
    "labels": {
      "alertname": "<requiredAlertName>",
      "<labelname>": "<labelvalue>",
      ...
    },
    "annotations": {
      "<labelname>": "<labelvalue>",
    },
    "startsAt": "<rfc3339>",
    "endsAt": "<rfc3339>",
    "generatorURL": "<generator_url>"
  },
  ...
]
```


## 通知模板引用

Prometheus 创建并向 Alertmanager 发送警报，然后 Alertmanager 根据标签向不同的receivers发送通知。接收器可以是多种集成之一，包括 Slack、PagerDuty、电子邮件或通过通用 Webhook 接口的自定义集成。

发送给receivers的通知是通过模板构建的。Alertmanager 自带默认模板，但也可以自定义。为避免混淆，需要注意的是 Alertmanager 模板与 [Prometheus 中的模板]([Template reference | Prometheus](https://prometheus.io/docs/prometheus/latest/configuration/template_reference/))不同，但 Prometheus 模板也包括警报规则标签/注释中的模板。

Alertmanager 的通知模板基于 [Go 模板系统]([template package - text/template - Go Packages](https://pkg.go.dev/text/template))。请注意，有些字段以文本形式评估，有些则以 HTML 形式评估，这将影响转义。

更多的请参考 [Notification template reference | Prometheus](https://prometheus.io/docs/alerting/latest/notifications/) 

## 通知模板样例

以下是所有不同的警报示例和相应的 Alertmanager 配置文件设置（alertmanager.yml）。每个示例都使用 Go 模板系统。[template package - text/template - Go Packages](https://pkg.go.dev/text/template)

### 自定义Slack的通知

在本例中，我们自定义了 Slack 通知，以发送一个 URL 到我们组织的维基站点，说明如何处理已发送的特定警报。

```yaml
global:
  # Also possible to place this URL in a file.
  # Ex: `slack_api_url_file: '/etc/alertmanager/slack_url'`
  slack_api_url: '<slack_webhook_url>'

route:
  receiver: 'slack-notifications'
  group_by: [alertname, datacenter, app]

receivers:
- name: 'slack-notifications'
  slack_configs:
  - channel: '#alerts'
    text: 'https://internal.myorg.net/wiki/alerts/{{ .GroupLabels.app }}/{{ .GroupLabels.alertname }}'
```

### 在CommonAnnotations中访问annotations

在本例中，我们再次通过访问 Alertmanager 发送的数据的 `CommonAnnotations` 中存储的`summary`和`description`，自定义发送到 Slack receiver的文本。

Alert

```yaml
groups:
- name: Instances
  rules:
  - alert: InstanceDown
    expr: up == 0
    for: 5m
    labels:
      severity: page
    # Prometheus templates apply here in the annotation and label fields of the alert.
    annotations:
      description: '{{ $labels.instance }} of job {{ $labels.job }} has been down for more than 5 minutes.'
      summary: 'Instance {{ $labels.instance }} down'
```

Receiver

```yaml
- name: 'team-x'
  slack_configs:
  - channel: '#alerts'
    # Alertmanager templates apply here.
    text: "<!channel> \nsummary: {{ .CommonAnnotations.summary }}\ndescription: {{ .CommonAnnotations.description }}"
```

### 涵盖所有收到的警报

最后，假定警报与上一个例子相同，我们自定义接收器，以覆盖从警报管理器收到的所有警报，并在新行中打印各自的注释摘要和说明。

Receiver

```yaml
- name: 'default-receiver'
  slack_configs:
  - channel: '#alerts'
    title: "{{ range .Alerts }}{{ .Annotations.summary }}\n{{ end }}"
    text: "{{ range .Alerts }}{{ .Annotations.description }}\n{{ end }}"
```

### 定义可复用的模板

回到第一个例子，我们也可以提供一个包含命名模板的文件，然后由 Alertmanager 加载，以避免跨行的复杂模板。在 `/alertmanager/template/myorg.tmpl` 下创建一个文件，并在其中创建一个名为 `"slack.myorg.txt "`的模板：

```txt
{{ define "slack.myorg.text" }}https://internal.myorg.net/wiki/alerts/{{ .GroupLabels.app }}/{{ .GroupLabels.alertname }}{{ end}}
```

现在，配置会为 "文本 "字段加载带有指定名称的模板，我们还提供了自定义模板文件的路径：

```yaml
global:
  slack_api_url: '<slack_webhook_url>'

route:
  receiver: 'slack-notifications'
  group_by: [alertname, datacenter, app]

receivers:
- name: 'slack-notifications'
  slack_configs:
  - channel: '#alerts'
    text: '{{ template "slack.myorg.text" . }}'

templates:
- '/etc/alertmanager/templates/myorg.tmpl'
```

## 管理API

Alertmanager 提供一套管理API，便于自动化和集成。

[Management API | Prometheus](https://prometheus.io/docs/alerting/latest/management_api/)

## HTTPS和认证

Alertmanager 支持基本身份验证和 TLS。这只是试验性的，将来可能会改变。
目前，HTTP 流量和流言流量支持 TLS。

[HTTPS and authentication | Prometheus](https://prometheus.io/docs/alerting/latest/https/)

