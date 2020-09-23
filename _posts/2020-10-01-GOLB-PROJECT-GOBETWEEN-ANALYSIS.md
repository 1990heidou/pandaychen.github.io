---
layout:     post
title:      基于 Golang 实现的负载均衡网关：gobetween 分析
subtitle:   分析一款 Golang 实现的四层代理 CLB
date:       2020-10-01
header-img: img/super-mario.jpg
author:     pandaychen
catalog:    true
tags:
    - Gateway
---


##  0x00    前言
前面分析过一款简单的反向代理实现：[一个 Http(s) 网关的实现分析](https://pandaychen.github.io/2020/03/22/A-GOLANG-HTTP-GATEWAY-ANALYSIS/)，这篇文章分析一款商用的 LB 开源项目：gobetween。它是一款 Pure-Golang 实现的四层代理网关，[文档在此](http://gobetween.io/documentation.html)，本文来探索下其实现及核心的源码分析。

> For a long time all of us have been using "traditional" load balancers / proxies like nginx, haproxy, and others.
> But in modern world balancing become more and more flexible because of environment changes are made more often. Nodes behind load balancer are spawning and disappearing according to load and/or other requirements. Auto scaling and containerization became almost a "silver bullet" in modern IT infrastructure architectures.
> In the IP-telephony world DNS SRV records are main mechanism to find out nearest and less loaded call router.
> Same situation is in modern microservices world, but unfortunately, there are almost no lb / proxy that has flexible and complete automatic discovery feature. There are lot's of tricks and workarounds like this.
> gobetween is aiming to fill this gap and provide fast, flexible and full-featured load balancing solution for modern microservice architectures.

gobetween 的架构图如下：
![image](https://camo.githubusercontent.com/252e2764756a51aea3ae4783fccef0a760eecb86/687474703a2f2f692e70696363792e696e666f2f69392f38623932313534343335626533326632316561613366663762336463366431632f313436363234343333322f37343435372f313034333438372f676f672e706e67)

## 0x01 特性
官方提供的特性，如下：
* [Fast L4 Load Balancing](https://github.com/yyyar/gobetween/wiki) -- 支持代理的
  * **TCP** - with optional [The PROXY Protocol](https://github.com/yyyar/gobetween/wiki/Proxy-Protocol) support
  * **TLS** - [TLS Termination](https://github.com/yyyar/gobetween/wiki/Protocols#tls) + [ACME](https://github.com/yyyar/gobetween/wiki/Protocols#tls) & [TLS Proxy](https://github.com/yyyar/gobetween/wiki/Tls-Proxying)
  * **UDP** - with optional virtual sessions and transparent mode


* [Clear & Flexible Configuration](https://github.com/yyyar/gobetween/wiki/Configuration) with [TOML](config/gobetween.toml) or [JSON](config/gobetween.json) -- 提供本地配置或远程配置
  * **File** - read configuration from the file
  * **URL** - query URL by HTTP and get configuration from the response body
  * **Consul** - query Consul key-value storage API for configuration

* [Management REST API](https://github.com/yyyar/gobetween/wiki/REST-API) -- 管理端的 API 设置及基础监控、后端节点管理等
  * **System Information** - general server info
  * **Configuration** - dump current config
  * **Servers** - list, create & delete
  * **Stats & Metrics** - for servers and backends including rx/tx, status, active connections & etc.

* [Discovery](https://github.com/yyyar/gobetween/wiki/Discovery) -- 后端服务发现的方式
  * **Static** - hardcode backends list in the config file
  * **Docker** - query backends from Docker / Swarm API filtered by label
  * **Exec** - execute an arbitrary program and get backends from its stdout
  * **JSON** - query arbitrary http url and pick backends from response json (of any structure)
  * **Plaintext** - query arbitrary http and parse backends from response text with customized regexp
  * **SRV** - query DNS server and get backends from SRV records
  * **Consul** - query Consul Services API for backends
  * **LXD** - query backends from LXD

* [Healthchecks](https://github.com/yyyar/gobetween/wiki/Healthchecks)  -- 支持健康检查的方式
  * **Ping** - simple TCP ping healthcheck
  * **Exec** - execute arbitrary program passing host & port as options, and read healthcheck status from the stdout
  * **Probe** - send specific bytes to backend (udp, tcp or tls) and expect a correct answer (bytes or regexp)

* [Balancing Strategies](https://github.com/yyyar/gobetween/wiki/Balancing) (with [SNI](https://github.com/yyyar/gobetween/wiki/Server-Name-Indication) support) -- 后端节点负载均衡的策略
  * **Weight** - select backend from pool based relative weights of backends
  * **Roundrobin** - simple elect backend from pool in circular order
  * **Iphash** - route client to the same backend based on client ip hash
  * **Iphash1** - same as iphash but backend removal consistent (clients remain connecting to the same backend, even if some other backends down)
  * **Leastconn** - select backend with least active connections
  * **Leastbandwidth** -  backends with least bandwidth


##  0x02   分析路线
个人比较感兴趣的点有如下几块：
1.  Gateway 实现的模型，各个子模块之间的联动策略及通信方式
2.  和 `Consul` 的结合做服务发现
3.  Metrics 指标及采集方法
4.  配置热重启
5.  Gateway 的扩展能力及高可用实现



##  0x03    代码分析

####    配置 Config
gobetween 的 [主要配置结构如下](https://github.com/yyyar/gobetween/blob/master/src/config/config.go)：
`Config` 是全局配置，不难了解各参数的意义，注意 `Servers  map[string]Server`，一个 `Server`（Key）名字代表了一个 LB 负载均衡器：
```golang
type Config struct {
	Logging  LoggingConfig     `toml:"logging" json:"logging"`
	Api      ApiConfig         `toml:"api" json:"api"`
	Metrics  MetricsConfig     `toml:"metrics" json:"metrics"`
	Defaults ConnectionOptions `toml:"defaults" json:"defaults"`
	Acme     *AcmeConfig       `toml:"acme" json:"acme"`
	Profiler *ProfilerConfig   `toml:"profiler" json:"profiler"`
	Servers  map[string]Server `toml:"servers" json:"servers"`
}

type Server struct {
	ConnectionOptions

	// hostname:port
	Bind string `toml:"bind" json:"bind"`

	// tcp | udp | tls
	Protocol string `toml:"protocol" json:"protocol"`

	// weight | leastconn | roundrobin
	Balance string `toml:"balance" json:"balance"`

	// Optional configuration for server name indication
	Sni *Sni `toml:"sni" json:"sni"`

	// Optional configuration for protocol = tls
	Tls *Tls `toml:"tls" json:"tls"`

	// Optional configuration for backend_tls_enabled = true
	BackendsTls *BackendsTls `toml:"backends_tls" json:"backends_tls"`

	// Optional configuration for protocol = udp
	Udp *Udp `toml:"udp" json:"udp"`

	// Access configuration
	Access *AccessConfig `toml:"access" json:"access"`

	// ProxyProtocol configuration
	ProxyProtocol *ProxyProtocol `toml:"proxy_protocol" json:"proxy_protocol"`

	// Discovery configuration
	Discovery *DiscoveryConfig `toml:"discovery" json:"discovery"`

	// Healthcheck configuration
	Healthcheck *HealthcheckConfig `toml:"healthcheck" json:"healthcheck"`
}
```




####    主流程

我们从 [main.go](https://github.com/yyyar/gobetween/blob/master/main.go) 开始，`main` 方法中独立启动了 `3` 个子逻辑，传入参数为 `cfg` 配置：
1.  `manager`：核心逻辑
2.  `metrics`：启动 metrics 服务
3.  `api`：使用 `gin` 框架构建的管理端 CGI 服务

```golang
func main(){
    ...
    // Process flags and start
	cmd.Execute(func(cfg *config.Config) {
		// Configure logging
		logging.Configure(cfg.Logging.Output, cfg.Logging.Level, cfg.Logging.Format)
		// Start manager
		manager.Initialize(*cfg)
		/* setup metrics */
		metrics.Start((*cfg).Metrics)
		// Start API
		api.Start((*cfg).Api)
		// block forever
		<-(chan string)(nil)
    })
}
```

##  核心结构
[`src/core`](https://github.com/yyyar/gobetween/tree/master/src/core) 下面定义了 gobetween 的核心结构的抽象，这里列出来一下：
1、[`Balancer` 结构](https://github.com/yyyar/gobetween/blob/master/src/core/balancer.go)，负载均衡算法的抽象，需要实现 `Elect` 方法：
```golang
/**
 * Balancer interface
 */
type Balancer interface {
	/**
	 * Elect backend based on Balancer implementation
	 */
	Elect(Context, []*Backend) (*Backend, error)
}
```
2、[Server 结构](https://github.com/yyyar/gobetween/blob/master/src/core/server.go)：抽象 LB 负载均衡器的公共接口，对应 [于此](https://github.com/yyyar/gobetween/blob/master/src/server/server.go)
```golang
type Server interface {
	/**
	 * Start server
	 */
	Start() error

	/**
	 * Stop server and wait until it stop
	 */
	Stop()

	/**
	 * Get server configuration
	 */
	Cfg() config.Server
}
```
3、`ReadWriteCount` 结构：
```golang
type ReadWriteCount struct {
	/* Read bytes count */
	CountRead uint

	/* Write bytes count */
	CountWrite uint

	Target Target
}
```
4、[`Context` 及 `TcpContext`](https://github.com/yyyar/gobetween/blob/master/src/core/context.go)：抽象了 TCP 连接的属性
```golang
type Context interface {
	String() string
	Ip() net.IP
	Port() int
	Sni() string
}

/**
 * Proxy tcp context
 */
type TcpContext struct {
	Hostname string
	/**
	 * Current client connection
	 */
	Conn net.Conn
}
```
5、[`Service`结构](https://github.com/yyyar/gobetween/blob/master/src/core/service.go)
```golang
/**
 * Service is a global facility that could be Enabled or Disabled for a number
 * of core.Server instances, depending on their configration. See services/registry
 * for exact examples.
 */
type Service interface {
	/**
	 * Enable service for Server
	 */
	Enable(Server) error

	/**
	 * Disable service for Server
	 */
	Disable(Server) error
}
```
6、[`Backend`(https://github.com/yyyar/gobetween/blob/master/src/core/backend.go)：定义了后端节点及统计信息的通用结构
```golang
/**
 * Backend means upstream server
 * with all needed associate information
 */
type Backend struct {
	Target
	Priority int          `json:"priority"`
	Weight   int          `json:"weight"`
	Sni      string       `json:"sni,omitempty"`
	Stats    BackendStats `json:"stats"`
}

/**
 * Backend status
 */
type BackendStats struct {
	Live               bool   `json:"live"`
	Discovered         bool   `json:"discovered"`
	TotalConnections   int64  `json:"total_connections"`
	ActiveConnections  uint   `json:"active_connections"`
	RefusedConnections uint64 `json:"refused_connections"`
	RxBytes            uint64 `json:"rx"`
	TxBytes            uint64 `json:"tx"`
	RxSecond           uint   `json:"rx_second"`
	TxSecond           uint   `json:"tx_second"`
}
```

##  参考
-   [golb](https://github.com/onestraw/golb)
-   [vulcand](https://github.com/vulcand/vulcand)
-   [gobetween](https://github.com/yyyar/gobetween)

