# Prometheus

## Prom 简介

Prometheus 是一个基于 Metrics 的监控系统，与 K8s 同属 CNCF，已经成为 K8s 生态圈的核心监控系统。

Prom 提供了通用的**数据模型**，便捷的数据采集、存储和查询接口。同时基于 Go 实现也大大降低了服务端的运维成本，可以借助一些优秀的图形化工具 (Grafana) 实现友好的图形化和报警。

## Prom 架构

![image-20210319143835598](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210319143835598.png)

Prom 的数据采集方式：

- Jobs/Exporters: Prom 可以从 Jobs 或 Exporters 中拉取 (**pull**) 监控数据。Exporter 以 Web API 的形式对外暴露数据采集接口。
- Pushgateway: :对于一些以临时性 Job 运行的组件，Prometheus 可能还没有来得及从中 pull 监控数据的情况下，这些 Job 已经结束了，Job 运行时可以在运行时将监控数据推送 (**push**) 到 Pushgateway 中，Prometheus 从 Pushgateway 中拉取数据，防止监控数据丢失。

Prom 的核心组件：

- Retrieval: 负责定时去暴露的目标上抓取采样指标数据。
- Storage: 负责将采样数据写入指定的时序数据库 (TSDB) 存储。
- Service discovery: Prom 可以动态的发现一些服务，拉取数据进行监控。如从 K8s 中发现， file_sd 为静态配置文件。
- AlertManager: 独立于 Prom 的外部组件，用于发出警告，通过配置文件可以设置一些警告， Prom 会把警告推送到 AlertManager。

Prom 工作流程：

1. Prometheus server 定期从静态配置文件或者服务发现的 targets 拉取要监控的 metrics。
2. Prometheus server 在本地存储收集到的 metrics，并运行定义好的 alert.rules，记录新的时间序列或者向 AlertManager 推送警告。
3. AlertManager 可以根据配置处理数据，最后发送警告。
4. client 还可以通过 API，Prom Web UI 和 Grafana 查询和聚合 metrics。

## Prom 数据

### 时间序列数据模型

Prom 所采集的监控数据均以时间序列的形式保存在 TSDB 中。

在时间序列中的每一个点成为一个 sample，由三部分构成：

- 指标(metric): metric 的名称和 labels
- 时间戳(timestamp) : 精确到毫秒的时间戳
- 样本值(value): float64 类型的值

![image-20210319151812533](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210319151812533.png)

### metric 类型

Prom 提供了四种核心的指标类型。但是这些类型只是在客户端库中，Prom server 不会对指标进行区分，而是视他们为无类型的时间序列。

#### Counter

表示**单调递增**的指标，除非系统重置，否则只增不减。

可以用来统计服务的请求数，已完成的任务数，错误发生次数等。Counter 可以使用户方便的了解事件产生的速率变化，如下：

```go
//通过rate()函数获取HTTP请求量的增长率
rate(http_requests_total[5m])
//查询当前系统中，访问量前10的HTTP地址
topk(10, http_requests_total)
```

#### Gauge

表示一种样本数据可以**任意变化**的指标，可增可减。

常用于内存使用率，温度，并发请求数量等。可以使用 PromQL 内置函数 `delta()` 获取样本在一段时间内的变化情况。

```go
dalta(cpu_temp_celsius{host="zeus"}[2h])
```

#### Histogram

为了区分是平均的慢还是长尾的慢，最简单的方式就是按照请求延迟的范围进行分组。例如，统计延迟在 0~10ms 之间的请求数有多少而 10~20ms 之间的请求数又有多少。通过这种方式可以快速分析系统慢的原因。Histogram 和 Summary 都是为了能够解决这样问题的存在，通过 Histogram 和 Summary 类型的监控指标，我们可以快速了解监控**样本的分布情况**。

Histogram 在**一段时间范围内对数据进行采样**（通常为请求持续时间或响应时间），并将其计入可配置的存储桶 (bucket) 中，后续可以指定区间筛选样本，也可以统计样本总数、

Histogram 提供三种 metrics：

- `basename_bucket{le="xxx"}`，样本的值分布在 bucket 中的数量
- `basename_sum`，所有样本值大小总和。
- `basename_count`，含义与  `basename_bucket{le="xxx"}` 相同。

#### Summary

与 Histogram 类似，用于表示**一段时间内的数据采样结果**（通常是请求持续时间或响应大小等），但它直接存储了分位数（通过客户端计算，然后展示出来），而不是通过区间来计算。

Summary 也提供三种指标：

- `basename{quantile="<φ>"}`，样本值的分位数分布情况。

```go
 // 含义：这 12 次 http 请求中有 50% 的请求响应时间是 3.052404983s
  io_namespace_http_requests_latency_seconds_summary{path="/",method="GET",code="200",quantile="0.5",} 3.052404983
  // 含义：这 12 次 http 请求中有 90% 的请求响应时间是 8.003261666s
  io_namespace_http_requests_latency_seconds_summary{path="/",method="GET",code="200",quantile="0.9",} 8.003261666
```

- `basename_sum`
- `basename_count`

Summary 与 Histogram 的异同

- 都包含了 sum 和 count 指标。
- Histogram 需要通过 `basename_bucket` 计算分位数，而 Summary 直接存储了分位数的值。

## Prom 存储

Prom 2.x 默认将时间序列数据保存在本地磁盘，同时也可以配置将数据保存到第三方存储服务中。

### 本地存储

Prom 使用自定义的存储格式将样本保存在本地磁盘。如果你的本地存储出现故障，最好的办法是停止运行 Prometheus 并删除整个存储目录。因为 Prometheus 的本地存储不支持非 POSIX 兼容的文件系统，一旦发生损坏，将无法恢复。NFS 只有部分兼容 POSIX，大部分实现都不兼容 POSIX。

除了删除整个目录之外，也可以尝试删除个别块目录来解决问题。删除每个块目录将会丢失大约两个小时时间窗口的样本数据。所以，**Prometheus 的本地存储并不能实现长期的持久化存储。**

Prom 按照两小时为一个时间窗口，将这个时间窗口产生的数据存储在一个块 (Block) 。

Block 下包含

- chunks: 该时间窗口内所有样本数据
- meta.json: 元数据文件
- index: 索引文件，索引 Metric_name 和 Labels 在样本数据的位置。
- tombstone: 删除的记录保存的位置。

Block 首先会保存在内存中。为了确保 Prom 能恢复数据，启动时通过 WAL 重新记录操作，从而恢复数据。WAL 保存在 wal 目录下，每个文件 128MB。

```bash
./data 
   |- 01BKGV7JBM69T2G1BGBGM6KB12 # 块
      |- meta.json  # 元数据
      |- wal        # 写入日志
        |- 000002
        |- 000001
   |- 01BKGTZQ1SYQJTR4PB43C8PD98  # 块
      |- meta.json  #元数据
      |- index   # 索引文件
      |- chunks  # 样本数据
        |- 000001
      |- tombstones # 逻辑数据
   |- 01BKGTZQ1HHWHV8FBJXW1Y3W0K
      |- meta.json
      |- wal
        |-000001
```

### 远程存储

Prom 的本地存储无法提供持久化，为了保持简单性，Prom 没有在本地存储上解决，而是通过定义两个标准接口 (remote_wirte/remote_read) 。

**Remote Write**

用户可以在 Prometheus 配置文件中指定 Remote Write（远程写）的 URL 地址，一旦设置了该配置项，Prometheus 将采集到的样本数据通过 HTTP 的形式发送给适配器（Adaptor）。而用户则可以在适配器中对接外部任意的服务。外部服务可以是真正的存储系统，公有云的存储服务，也可以是消息队列等任意形式。

![image-20210319161451535](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210319161451535.png)

**Remote Read**

如下图所示，Promthues 的 Remote Read（远程读）也通过了一个适配器实现。在远程读的流程当中，当用户发起查询请求后，Promthues 将向 remote_read 中配置的 URL 发起查询请求（matchers,ranges），`Adaptor` 根据请求条件从第三方存储服务中获取响应的数据。同时将数据转换为 Promthues 的原始样本数据返回给 Prometheus Server。

当获取到样本数据后，Promthues 在本地使用 PromQL 对样本数据进行二次处理。

![image-20210319162041229](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210319162041229.png)

远程读和远程写协议都使用了基于 HTTP 的 snappy 压缩协议的缓冲区编码，目前还不稳定，在以后的版本中可能会被替换成基于 HTTP/2 的 `gRPC` 协议，前提是 Prometheus 和远程存储之间的所有通信都支持 HTTP/2。

