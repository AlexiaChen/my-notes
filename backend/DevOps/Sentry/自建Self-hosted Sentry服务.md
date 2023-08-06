
## 关键概念

- 警报(Alert): 警报通过在满足某些警报规则条件时向你发送通知，让你实时了解你的代码的问题。有几种类型的警报可供选择，有可定制的阈值和集成。
- 附件(attachments): 存储的额外文件，如与错误事件有关的配置或日志文件。
- 数据(data): 您发送给Sentry的任何东西。这包括，事件（错误或事务），附件，和事件元数据。
- 数据源名称(DSN): 数据源名称。一个DSN告诉Sentry SDK在哪里发送事件，以便将事件与正确的项目相关联。当你创建一个项目时，Sentry会自动为你指定一个DSN [Data Source Name (DSN) | Sentry Documentation](https://docs.sentry.io/product/sentry-basics/dsn-explainer/)
- 环境(environment): environment是一个Sentry支持的标签，你可以把它添加到你的SDK中，目的是指你的代码部署的命名规则，如开发、测试、暂存或生产。环境可以帮助你更好地过滤问题（issues）和事务以及其他用途。[Environments | Sentry Documentation](https://docs.sentry.io/product/sentry-basics/environments/)
- 错误(error): 什么算作错误因平台而异，但一般来说，如果有一些看起来像异常的东西，就可以在Sentry中捕获为错误。Sentry会自动捕获错误、未捕获的异常和未处理的拒绝，以及其他类型的错误，具体取决于平台。
- 事件(event): 一个错误或一个事务。
- 问题(issues): 一个问题是相似错误或性能问题的一组数据。每个事件都有一组特征，称为其指纹(fingerprint), 这也是Sentry用来对它们进行分组的。例如，当错误事件是由你的代码的同一部分触发时，Sentry会将它们分组。这种将事件分组为问题的做法可以让你看到一个问题发生的频率以及它影响了多少用户。
- 性能监控(performance monitering):  性能监控是跟踪应用程序性能和测量指标的行为，如正在发送多少交易和所有发生的特定交易的平均响应时间。为了做到这一点，Sentry捕获了由事务和跨度组成的分布式跟踪，以衡量单个服务以及这些服务中的操作。
- 项目(project): 一个项目在Sentry中代表你的服务或应用。你可以为你的应用中使用的特定语言或框架创建一个项目。例如，你可能为你的API服务器和前端客户端有单独的项目。欲了解更多信息，请查看我们的创建项目的最佳实践。项目允许你将事件与你的组织中的不同应用联系起来，并将责任和所有权分配给你组织中的特定用户和团队。
- 发布(release): 发布是你的代码的一个版本，部署在一个环境中。当你通知Sentry一个版本时，你可以识别与之相关的新问题和回归，确认一个问题是否在下一个版本中得到解决，并应用源代码图。[Releases | Sentry Documentation](https://docs.sentry.io/product/releases/)
- 版本健康(release health): 版本健康数据提供了对崩溃和错误影响的洞察力，因为它与你的用户体验有关，并揭示了每个新问题的趋势。[Release Health | Sentry Documentation](https://docs.sentry.io/product/releases/health/)
- Sentry SDK: Sentry用于应用监控的编程语言/框架专用库。当您将我们的SDK添加到您的应用程序中时，来自您的应用程序的事件数据会被捕获并发送给Sentry，因此我们可以为您提供错误和性能报告。
- 事务(transaction): 一个事务代表一个被调用的服务的单一实例，以支持你想测量或跟踪的操作，如页面加载、页面导航或异步任务。事务事件是由事务名称分组的。

## 关键功能

sentry.io的每个关键功能都由应用程序中的一个页面（或一组页面）来代表。

- 警报(Alerts): 你可以在这里创建新的警报规则并管理现有的规则  [Alerts | Sentry Documentation](https://docs.sentry.io/product/alerts/)
- 仪表盘(Dashboards): 通过允许你浏览多个项目的错误和性能数据，为你提供一个应用程序健康状况的广泛概述。仪表盘是由一个或多个小部件组成的，每个小部件显示一个或多个发现或问题查询。[Dashboards | Sentry Documentation](https://docs.sentry.io/product/dashboards/)
- 发现(Discover): 允许你跨环境查询事件，为你提供对错误和事务的可见性，并解锁对整个系统健康状况的洞察力。发现页面将查询的结果可视化。[Discover | Sentry Documentation](https://docs.sentry.io/product/discover-queries/)
- 问题(issues): 显示关于你的应用程序中分组问题的信息。从这里，你可以转到问题详情页，以获得任何问题的更精细的视图。[Issues | Sentry Documentation](https://docs.sentry.io/product/issues/)
- 性能(performance): sentry.io的主要视图，你可以在这里搜索或浏览事务数据。该页面显示直观的事务或趋势图，以及一个表格，你可以查看相关的事务并深入了解更多的信息 [Performance Monitoring | Sentry Documentation](https://docs.sentry.io/product/performance/)
- 项目(projects): 按团队列出你所参加的项目，并为你提供你的项目的高级概述。从这里，你可以进入每个项目的项目细节页面，以获得更详细的信息。[Projects | Sentry Documentation](https://docs.sentry.io/product/projects/)
- 发版(releases):  提供每个版本的高层次视图、相关项目、每个版本的采用阶段、每个提交的作者，以及包括无崩溃用户的百分比和无崩溃会话的百分比在内的版本健康数据。你可以直接导航到 "发布 "页面，或者从 "问题详情 "页面中选择 "最后出现 "下的发布ID。[Releases | Sentry Documentation](https://docs.sentry.io/product/releases/)



## 安装步骤

在此之前，需要安装Docker和Docker compose

```bash
# https://github.com/getsentry/self-hosted
git clone git@github.com:getsentry/self-hosted.git
```

或者下载最新的latest tag的tar.gz的包

运行

```bash
./install.sh
```

期间，会让你选择，是否上报你安装的这台目标机器的监控，这个只是辅助sentry官方团队，优化self-hosted的安装流程，可以选择关闭。

期间，还会让你创建admin的管理员账号和密码。

完成后，它会通知让你用docker compose -d up启动项目，如果你看到9000端口被打开了，就可以通过http://ip:9000访问管理端的Web后台了。


之后就是创建项目，项目里面都有step by step的集成SDK的教程。



## 参考

[Self-Hosted Sentry | Sentry Developer Documentation](https://develop.sentry.dev/self-hosted/)

