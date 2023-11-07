
需要Prometheus的 SDK，它提供了各种语言的SDK。在这里，我们用golang举例。

在本教程中，我们将创建一个简单的 Go HTTP 服务器，并通过添加一个计数器指标来计算服务器处理的请求总数。

这里我们有一个简单的 HTTP 服务器，其 /ping 端点返回 pong 作为响应。

```go
package main

import (
   "fmt"
   "net/http"
)

func ping(w http.ResponseWriter, req *http.Request){
   fmt.Fprintf(w,"pong")
}

func main() {
   http.HandleFunc("/ping",ping)

   http.ListenAndServe(":8090", nil)
}
```

现在，让我们为服务器添加一个度量指标，该指标将记录向 ping 端点发出的请求数，计数器度量类型非常适合，因为我们知道请求数不会减少，只会增加。

创建一个Prometheus计数器

```go
var pingCounter = prometheus.NewCounter(
   prometheus.CounterOpts{
       Name: "ping_request_count",
       Help: "No of request handled by Ping handler",
   },
)
```

接下来，让我们更新 ping 处理程序，使用 `pingCounter.Inc()` 增加计数器的计数。

```go
func ping(w http.ResponseWriter, req *http.Request) {
   pingCounter.Inc()
   fmt.Fprintf(w, "pong")
}
```

然后将计数器注册到默认寄存器，并公开度量值。

```
func main() {
   prometheus.MustRegister(pingCounter)
   http.HandleFunc("/ping", ping)
   http.Handle("/metrics", promhttp.Handler())
   http.ListenAndServe(":8090", nil)
}
```

`prometheus.MustRegister` 函数将 `pingCounter` 注册到默认注册表中。为了公开度量指标，Go Prometheus 客户端库提供了 `promhttp` 包。`promhttp.Handler()` 提供了一个 `http.Handler`，用于公开在默认寄存器中注册的度量指标。

示例代码依赖于

```go
package main

import (
   "fmt"
   "net/http"

   "github.com/prometheus/client_golang/prometheus"
   "github.com/prometheus/client_golang/prometheus/promhttp"
)

var pingCounter = prometheus.NewCounter(
   prometheus.CounterOpts{
       Name: "ping_request_count",
       Help: "No of request handled by Ping handler",
   },
)

func ping(w http.ResponseWriter, req *http.Request) {
   pingCounter.Inc()
   fmt.Fprintf(w, "pong")
}

func main() {
   prometheus.MustRegister(pingCounter)

   http.HandleFunc("/ping", ping)
   http.Handle("/metrics", promhttp.Handler())
   http.ListenAndServe(":8090", nil)
}
```

现在点击 `localhost:8090/ping` 端点几次，然后向 `localhost:8090` 发送请求，就能获得指标。

![[Pasted image 20231107095902.png]]

这里的 `ping_request_count` 显示 `/ping` 端点被调用了 3 次。

Default Register 自带 go 运行时指标收集器，这就是我们看到 `go_threads`、`go_goroutines` 等其他指标的原因。

我们已经构建了第一个指标输出器。让我们更新 Prometheus 配置，从服务器上抓取指标。

```go
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets: ["localhost:9090"]
  - job_name: simple_server
    static_configs:
      - targets: ["localhost:8090"]
```

```bash
prometheus --config.file=prometheus.yml
```

<iframe width="560" height="315" src="https://www.youtube.com/embed/yQIWgZoiW0o" title="instrumentation demo" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>

