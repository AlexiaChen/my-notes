
在本教程中，我们将使用 [Grafana]([github.com/grafana/grafana](https://github.com/grafana/grafana)) 创建一个简单的仪表盘，以可视化我们在[[Prometheus教程03-以http server的形式暴露应用自定义的指标]] 中仪表化的 `ping_request_count` 指标。

如果您想知道，既然可以使用 Prometheus 查询和查看图表，为什么还要使用 Grafana 这样的工具，答案是我们在 Prometheus 上运行查询时看到的图表是用来运行临时查询的。

Grafana 和[控制台模板]([Console templates | Prometheus](https://prometheus.io/docs/visualization/consoles/))是创建图表的两种推荐方式。

### 安装和设置Grafana

根据您的操作系统，按照[此处的步骤]([Install Grafana | Grafana documentation](https://grafana.com/docs/grafana/latest/setup-grafana/installation/#supported-operating-systems))安装并运行 Grafana。

安装并运行 Grafana 后，在浏览器中导航至 http://localhost:3000   使用默认凭据（用户名为 admin，密码为 admin）登录并设置新凭据。

### 在Grafana中把Prometheus添加成为一个数据源

点击侧边栏的齿轮图标，选择数据源，为 Grafana 添加数据源(Data Source)。

⚙ > Data Sources

在 "数据源 "页面中，您可以看到 Grafana 支持多种数据源，如 Graphite、PostgreSQL 等。选择 Prometheus 进行设置。

在 HTTP 部分输入 Prometheus 的URL（http://localhost:9090），然后单击 "保存并测试"。

<iframe width="560" height="315" src="https://www.youtube.com/embed/QT66dU_h9lo" title="Add Prometheus as Data Source to Grafana" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>


### 创建我们的第一个仪表盘(Dashboard)


现在我们已成功将 Prometheus 添加为数据源，接下来我们将为上一教程中的 ping_request_count 指标创建第一个仪表盘。

- 单击侧边栏中的 + 图标并选择仪表盘。
- 在下一个屏幕中，点击添加新面板按钮(Add new pannel)。
- 在 "查询 "选项卡中键入 PromQL 查询，本例中只需键入 `ping_request_count`。
- 访问 ping 端点几次并刷新图表，以验证其是否按预期运行。
- 在面板选项下的右侧部分，将标题设为 `Ping 请求计数`。
- 单击右角的 "保存 "图标保存仪表板。

<iframe width="560" height="315" src="https://www.youtube.com/embed/giVZHO6akRA" title="Create Dashboard" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>


