---
title: 利用 Grafana 以及 node_exporter 中的 tcpstat 收集器监控 TCP 连接各个状态的数量
date: 2020-04-21 23:27:46
permalink: grafana-tcp-status
category:
  - grafana
tags:
  - grafana
  - node_exporter
  - tcp
toc:
---

在经历了 CLOSE_WAIT 增多而导致的服务器性能下降的问题之后，为了能实时监控 TCP 各个状态的数量，我选择了 Grafana 以及 node_exporter 的 tcpstat 收集器。

关于 node_exporter 就不多做介绍了，想详细了解的同学可以访问 Github 上的仓库 👉 [Node exporter](https://github.com/prometheus/node_exporter) 。

<!-- more -->

简单介绍一下 **tcpstat** 收集器，是 node_exporter 自带的收集器，适用的环境是 Linux，本质是从 `/proc/net/tcp` 以及 `/proc/net/tcp6` 获取 TCP 连接状态的信息。需要注意的是当前版本（0.18.1）在高负载下会出现性能问题，具体表现就是采集地址无法访问导致 Prometheus 无法获取数据。另外，tcpstat 收集器是**默认不开启**，所以启动的时候需要带上 `--collector.tcpstat` 参数（可以通过参数 `-h` 了解）。

![tcpstat](https://i.loli.net/2020/04/21/uGJ3iLVBTrMt7Nc.png)

启动好 node_exporter 并开启 tcpstat 收集器之后，我们就可以前往 Grafana 设置监控面板了。

在新建 panel 中添加一个 Query，查询语句：`node_tcp_connection_states{instance="localhost:9100"`

- 其中 `node_tcp_connection_states` 是 TCP 连接状态的 metric 的名称，这个怎么来的呢，~~我是瞎蒙的~~，因为 Grafana 自带语法提示，所以输入 `tcp` 就可以直接找到（）。而 `9100` 就是 node_exporter 的默认端口号。

![Metrics](https://i.loli.net/2020/04/21/H1TsKtjidBymIwl.png)

完整的 Query 就是下面这个样子。

![Query](https://i.loli.net/2020/04/21/7ctZ5BUDhCRE8wP.png)

最后查询出来的效果如下：

![panel](https://i.loli.net/2020/04/21/1p34hfakFzLo5DN.png)

tcpstat 收集器相关的实现在这里 👉 [tcpstat_linux.go](https://github.com/prometheus/node_exporter/blob/master/collector/tcpstat_linux.go) 。下面贴一下主要代码片段。

首先 tcpstat 定义了 TCP 连接各个状态的常量，即我们可以收集（监控）到的信息。

```go
const (
    // TCP_ESTABLISHED
    tcpEstablished tcpConnectionState = iota + 1
    // TCP_SYN_SENT
    tcpSynSent
    // TCP_SYN_RECV
    tcpSynRecv
    // TCP_FIN_WAIT1
    tcpFinWait1
    // TCP_FIN_WAIT2
    tcpFinWait2
    // TCP_TIME_WAIT
    tcpTimeWait
    // TCP_CLOSE
    tcpClose
    // TCP_CLOSE_WAIT
    tcpCloseWait
    // TCP_LAST_ACK
    tcpLastAck
    // TCP_LISTEN
    tcpListen
    // TCP_CLOSING
    tcpClosing
    // TCP_RX_BUFFER
    tcpRxQueuedBytes
    // TCP_TX_BUFFER
    tcpTxQueuedBytes
)
```

接着定义了收集器的相关信息，比如上边提到的 metric 的名称 `node_tcp_connection_states` 以及提供的模板变量 `state`。

```go
func NewTCPStatCollector(logger log.Logger) (Collector, error) {
    return &tcpStatCollector{
        desc: typedDesc{prometheus.NewDesc(
            prometheus.BuildFQName(namespace, "tcp", "connection_states"),
            "Number of connection states.",
            []string{"state"}, nil,
        ), prometheus.GaugeValue},
        logger: logger,
    }, nil
}
```

下面两个方法就是从 `/proc/net/tcp` 以及 `/proc/net/tcp6` 这两个文件获取 TCP 连接状态的信息并转化为 metric。

```go
func (c *tcpStatCollector) Update(ch chan<- prometheus.Metric) error {
    tcpStats, err := getTCPStats(procFilePath("net/tcp"))
    if err != nil {
        return fmt.Errorf("couldn't get tcpstats: %s", err)
    }

    // if enabled ipv6 system
    tcp6File := procFilePath("net/tcp6")
    if _, hasIPv6 := os.Stat(tcp6File); hasIPv6 == nil {
        tcp6Stats, err := getTCPStats(tcp6File)
        if err != nil {
            return fmt.Errorf("couldn't get tcp6stats: %s", err)
        }

        for st, value := range tcp6Stats {
            tcpStats[st] += value
        }
    }

    for st, value := range tcpStats {
        ch <- c.desc.mustNewConstMetric(value, st.String())
    }
    return nil
}

func getTCPStats(statsFile string) (map[tcpConnectionState]float64, error) {
    file, err := os.Open(statsFile)
    if err != nil {
        return nil, err
    }
    defer file.Close()

    return parseTCPStats(file)
}
```

本文到这里结束啦，主要介绍了如何利用 Grafana + node_exporter.tcpstat 的方式实时监控 TCP 连接状态信息，如果大家有其他的方式，欢迎下方留言讨论。
