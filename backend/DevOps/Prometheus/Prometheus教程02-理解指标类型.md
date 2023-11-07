
Prometheus 支持四种类型的指标，分别是 - 计数器(Counter) - 仪表(Gauge) - 直方图(Histogram) - 摘要(Summary)

### Counter

计数器是一种只能增加或重置的度量值，也就是说，其值不能低于前一个值。它可用于请求数、错误数等指标。

在查询栏中键入以下查询，然后点击执行。

`go_gc_duration_seconds_count`

![[Pasted image 20231107094318.png]]

PromQL 中的 rate() 函数获取一段时间内的历史度量值，并计算该值每秒的增长速度。速率仅适用于计数器值。

`rate(go_gc_duration_seconds_count[5m])`

![[Pasted image 20231107094409.png]]

### Gauge

Gauge 是一个可升可降的数字。它可用于衡量集群中 pod 的数量、队列中事件的数量等。

`go_memstats_heap_alloc_bytes`

![[Pasted image 20231107094520.png]]

`max_over_time`, `min_over_time` 和 `avg_over_time` 等 PromQL 函数可用于测量指标

### Histogram

与前两种度量类型相比，直方图是一种更为复杂的度量类型。直方图可用于根据桶值计算的任何计算值。桶的边界可由开发人员配置。一个常见的例子是回复请求所需的时间，称为延迟。

举例说明： 假设我们想观察处理 API 请求所需的时间。直方图允许我们以桶为单位存储请求时间，而不是存储每个请求的请求时间。我们为所花费的时间定义桶，例如低于或等于 0.3、0.5、0.7、1 和 1.2。一旦计算出一个请求的耗时，就会将其添加到所有桶的计数中，这些桶的边界高于测量值。

假设端点"/ping "的请求 1 耗时 0.25 秒。那么桶的计数值为:

|Bucket|Count|
|---|---|
|0 - 0.3|1|
|0 - 0.5|1|
|0 - 0.7|1|
|0 - 1|1|
|0 - 1.2|1|
|0 - +Inf|1|

注：默认添加 +Inf 桶。

(由于直方图是一个累积频率，所有大于该值的桶都会加上 1）。

端点"/ping "的请求 2 耗时 0.4 秒 桶的计数值将如下。

|Bucket|Count|
|---|---|
|0 - 0.3|1|
|0 - 0.5|2|
|0 - 0.7|2|
|0 - 1|2|
|0 - 1.2|2|
|0 - +Inf|2|

由于 0.4 低于 0.5，因此在该界限以内的所有桶的计数都会增加。
让我们探索一下普罗米修斯用户界面中的直方图度量，并应用一些函数。

`prometheus_http_request_duration_seconds_bucket{handler="/graph"}`

![[Pasted image 20231107094945.png]]

`histogram_quantile()` 函数可用于从直方图中计算量化值

`histogram_quantile(0.9,prometheus_http_request_duration_seconds_bucket{handler="/graph"})`

![[Pasted image 20231107095032.png]]

图中显示第 90 个百分位数是 0.09，要找到过去 5m 的直方图_四分位数，可以使用 rate() 和时间框架。

``histogram_quantile(0.9, rate(prometheus_http_request_duration_seconds_bucket{handler="/graph"}[5m]))`

![[Pasted image 20231107095118.png]]

### Summary


汇总表也可以测量事件，是直方图的替代品。它们的成本更低，但丢失的数据更多。它们是在应用程序级别计算的，因此无法汇总同一流程多个实例的指标。当事先不知道度量值的桶时，可以使用直方图，但强烈建议尽可能使用直方图而不是汇总。

在本教程中，我们详细介绍了度量类型和一些 PromQL 操作，如速率、直方图_四分位数等。
