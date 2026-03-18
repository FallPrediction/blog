+++
title = 'Prometheus Client Golang'
date = 2026-03-19T07:25:14+08:00
draft = false
description = '筆記 Prometheus Client Golang 基本用法'
toc = true
tags = ['GO', 'Prometheus']
categories = ['GO', 'Prometheus']
+++

## 簡介
本篇為 [Prometheus GO client](https://pkg.go.dev/github.com/prometheus/client_golang/prometheus) package 的用法筆記

### metric endpoint
```go
http.Handle("/metrics", promhttp.Handler())
```

### 自定義 metric
加上自定義 metric，[文件](https://pkg.go.dev/github.com/prometheus/client_golang/prometheus#hdr-Advanced_Uses_of_the_Registry)建議用 `MustRegister()`確保 collector 和 metric 一致，如果不相容或不一致，例如有相同 name 的 collector，會在註冊時就引發 panic，不會等到實際抓 metric 才 panic
```go
import (
	"net/http"

	"github.com/prometheus/client_golang/prometheus"
	"github.com/prometheus/client_golang/prometheus/collectors"
	"github.com/prometheus/client_golang/prometheus/promhttp"
)

var (
    RequestsTotal = promauto.NewCounterVec(
		prometheus.CounterOpts{
			Name: "http_requests_total",
			Help: "Tracks the number of HTTP requests.",
		}, []string{"method", "code", "uri"},
	)
	RequestDuration = promauto.NewHistogramVec(
		prometheus.HistogramOpts{
			Name:    "http_request_duration_seconds",
			Help:    "Tracks the latencies for HTTP requests.",
			Buckets: prometheus.ExponentialBuckets(0.1, 5, 5),
		},
		[]string{"uri"},
	)
)

func main() {
	registry := prometheus.NewRegistry()
	registry.MustRegister(
		collectors.NewGoCollector(),
		collectors.NewProcessCollector(collectors.ProcessCollectorOpts{}),
		helper.RequestDuration,
		helper.RequestsTotal,
	)

	http.Handle("/metrics", promhttp.HandlerFor(registry, promhttp.HandlerOpts{}))
	if err := http.ListenAndServe(":8080", nil); err != nil {
		panic(err)
	}
}
```

### Collector
常規的 Counter、Gauge、Histogram、Summary\
metric 新增 value（還有其他方式見文件）
```go
counter.Inc()
gauge.Inc()
gauge.Dec()
histogram.Observe(1.23)
summary.Observe(1.23)
```
還各自有向量 `Vec` 版本，可以定義一組一樣的資料但不同的 label\
範例
```go
requestsTotal := promauto.NewCounterVec(
    prometheus.CounterOpts{
        Name: "http_requests_total",
        Help: "Tracks the number of HTTP requests.",
    }, []string{"method", "code", "uri"},
)
requestsTotal.WithLabelValues("POST", 200, "user").Inc()
```

#### histogram bucket
```go
func ExponentialBuckets(start, factor float64, count int) []float64
```
`start` 為第一個上邊界，每個 bucket 邊界為前一個邊界乘上 `factor`，共有 `count` + 1 個 bucket\
例 `ExponentialBuckets(0.1, 5, 5)`，其 6 個 bucket 的 le 分別為
- 0.1
- 0.5
- 2.5
- 12.5
- 62.5
- +Inf