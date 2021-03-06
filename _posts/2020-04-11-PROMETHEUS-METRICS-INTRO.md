---
layout: post
title: 理解 Prometheus 的基本数据类型
subtitle: Prometheus 介绍及基础应用
date: 2020-04-11
author: pandaychen
header-img: golang-horse-fly.png
catalog: true
category: false
tags:
  - Prometheus
  - Metrics
---

## 0x00 前言

这篇文章来分析下 Prometheus 中常用的 Metrics，Metrics 是一种采样数据的总称，是一种对于度量计算单位的抽象。Prometheus 中常见的 Metrics，如 Counters、Counters、Histograms。

## 0x01 Gauges

Gauges 理解为（待监控的）瞬时状态，如当前时刻 CPU 的使用率、内存的使用量、硬盘的容量等等。因为此类型的特点是随着时间的推移不断，值（相对而言）没有规则的变化。在 Kratos 框架中，针对 RPC 每次请求的延迟（latency）就是一个 Gauges，一段时间内的 Gauges 就组合成了一个 [RollingGauges](https://github.com/go-kratos/kratos/blob/master/pkg/stat/metric/rolling_gauge.go#L10)

![image](https://s1.ax1x.com/2020/04/12/GLSUx0.jpg)

以时间为横坐标、数值为纵坐标，如下面的内存的使用图，就是典型的采集使用 Gauge 形式的 Metrics：

![image](https://s1.ax1x.com/2020/04/12/GLpym8.png)

## 0x02 Counters

Counter 就是计数器，从数据量 `0` 开始累计计算，只能增加，或者保持不变（增加 `0`），典型对应的场景是：持续增加的访问量采样数据。Counter 一般从 `0` 开始，一直不断的累加，但有可能保持不变（在图中以一条水平线表示）

以时间为横坐标、数值为纵坐标，可以画出用 Counters 形式的 Metrics：

![image](https://s1.ax1x.com/2020/04/12/GLilND.png)

同样的，在 Kratos 中，每个连接上 RPC 请求的总次数和错误次数，就是一个 [Counters](https://github.com/go-kratos/kratos/blob/master/pkg/stat/metric/rolling_counter.go)。

<font color="#dd0000">一定要注意的是：不要使用计数器来监控可能减少的值</font>。例如，不要使用计数器来处理当前正在运行的进程数，而应该用 Gauge。Counter 主要有两个方法：

```golang
// 将 counter 值加 1
Inc()
// 将指定值加到 counter 值上，如果指定值 < 0 会 panic.
Add(float64)
```

## 0x03 Histograms && Summary

Histograms 意为直方图，Histogram 会在一段时间范围内对数据进行采样（通常是请求持续时间或响应大小等），并将其计入可配置的存储桶（Bucket）中。

![histogram]()

#### Histograms 的应用意义

在大多数情况下人们都倾向于使用某些量化指标的平均值，例如 CPU 的平均使用率、页面的平均响应时间。这种方式的问题很明显，以系统 API 调用的平均响应时间为例：如果大多数 API 请求都维持在 100ms 的响应时间范围内，而个别请求的响应时间需要 `5s`，那么就会导致某些 WEB 页面的响应时间落到中位数的情况，而这种现象被称为长尾问题。为了区分是平均的慢还是长尾的慢，最简单的方式就是按照请求延迟的范围进行分组。例如，统计延迟在 `0~10ms` 之间的请求数、延迟在 `10~20ms` 之间的请求数等等。通过这种方式可以快速分析系统慢的原因。Histogram 和 Summary 都是为了能够解决这样问题的存在，通过 Histogram 和 Summary 类型的监控指标，可以快速了解监控样本的分布情况。

## 0x04 参考

- [一文搞懂 Prometheus 的直方图](https://juejin.im/post/5d492d1d5188251dff55b0b5)
- [如何区分 prometheus 中 Histogram 和 Summary 类型的 metrics？](https://www.cnblogs.com/aguncn/p/9920545.html)
- [Metrics 设计：teleport](https://goteleport.com/teleport/docs/metrics-logs-reference/)
- [Lock-free Observations for Prometheus Histograms](https://grafana.com/blog/2020/01/08/lock-free-observations-for-prometheus-histograms/)

转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权
