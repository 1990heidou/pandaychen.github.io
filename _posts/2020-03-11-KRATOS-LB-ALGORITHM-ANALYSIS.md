---
layout:     post
title:      gRPC 微服务构建之 LB 算法实现（未完）
subtitle:   Kratos 中的 LB 算法分析
date:       2020-03-11
author:     pandaychen
header-img:
catalog: true
category:   false
tags:
    - Kratos
    - WRR
---

##	0x00	前言

回顾下在先前文章中分析过实现自定义 gRPC Balancer 算法的三个步骤：

1.	注册 Balancer 的名字
2.	实现 PickerBuild 及 Builder() 方法，用于当 resolver 解析器发生解析变更时（后端节点增删）时，更新 Picker 使用的 LB-Pool
3.	实现 Picker 及 Picker() 方法，用于客户端每次 RPC 调用时，进行 LoadBalance 算法的 Choice


##  0x01	Kratos 的 WRR 算法
Kratos 在传统的 [Nginx-WRR 算法](https://tenfy.cn/2018/11/12/smooth-weighted-round-robin/) 的基础上，为加权轮询算法增加了 **动态调节权重值**，用户可以在为每一个 Backend 先配置一个初始的权重分，之后算法会根据 Backend 节点 CPU、延迟、服务端错误率、客户端错误率动态打分，（每一次 RPC 调用后）在将打分乘用户自定义的初始权重分得到最后的权重值。

##  0x02	WRR 的实现

####	结构

定义 Backend 服务的结构：
```go
type serverInfo struct {
	cpu     int64
	success uint64 // float64 bits
}
```

封装的 balancer.Subconn 结构，加入了对此连接的属性
```go
type subConn struct {
	conn balancer.SubConn       // 一个 conn 代表到一个 backend 的长连接
	addr resolver.Address
	meta wmeta.MD

	err     metric.RollingCounter
	latency metric.RollingGauge
	si      serverInfo
	// effective weight
	ewt int64
	// current weight
	cwt int64
	// last score
	score float64
}
```


####  LB 算法
自定义 LB 实现的组件，wrrPickerBuilder 以及 wrrPickerBuilder.Build() 方法，wrrPicker 以及 wrrPicker.Pick() 方法：
```go
type wrrPickerBuilder struct{}

type wrrPicker struct {
	// subConns is the snapshot of the weighted-roundrobin balancer when this picker was
	// created. The slice is immutable. Each Get() will do a round robin
	// selection from it and return the selected SubConn.
    subConns []*subConn     // 使用自定义的 subConn，里面封装了 gRPC 库的 balancer.SubConn，方便加入更多属性
                            //subConns 标识所有活动连接数组
	colors   map[string]*wrrPicker      // 保存 color 的 picker 逻辑（颜色筛选）
	updateAt int64

	mu sync.Mutex
}
```

再看下 wrrPickerBuilder.Build()，该方法由参数的 readySCs，根据权重，构造出给 wrrPicker 选择的初始化连接集合：
```go
func (*wrrPickerBuilder) Build(readySCs map[resolver.Address]balancer.SubConn) balancer.Picker {
    //readySCs 是从 gRPC 的 conn-pool 拿到的最新的可用连接池（每次 watcher 触发都会调用，如果 readyScs 后端无改动，则不会）
    // 初始化需要返回的结构（balancer.Picker）
    //map[{127.0.0.1:11111  <nil> 0 0xc00000fed0}:0xc0001cf270 {127.0.0.1:11112  <nil> 0 0xc00021e2f0}:0xc000214a90]
	p := &wrrPicker{
		colors: make(map[string]*wrrPicker),
    }

	for addr, sc := range readySCs {
		meta, ok := addr.Metadata.(wmeta.MD)
		if !ok {
            // 如果未定义权重，初始化权重为 10
			meta = wmeta.MD{
				Weight: 10,
			}
		}
		subc := &subConn{
			conn: sc,   //save
			addr: addr, //save

			meta:  meta,//save
			ewt:   int64(meta.Weight),
			score: -1,

			err: metric.NewRollingCounter(metric.RollingCounterOpts{
				Size:           10,
				BucketDuration: time.Millisecond * 100,
			}),
			latency: metric.NewRollingGauge(metric.RollingGaugeOpts{
				Size:           10,
				BucketDuration: time.Millisecond * 100,
			}),

            // 初始化 cpu 和成功率的值
			si: serverInfo{cpu: 500, success: math.Float64bits(1)},
		}
		if meta.Color == "" {
			p.subConns = append(p.subConns, subc)
			continue
		}
        // if color not empty, use color picker
        // 构建 color 选择
		cp, ok := p.colors[meta.Color]
		if !ok {
			cp = &wrrPicker{}
			p.colors[meta.Color] = cp
		}
		cp.subConns = append(cp.subConns, subc)
	}
	return p
}
```


#### Picker实现
真正实现 wrr 的算法在下面这个方法中：
```go
func (p *wrrPicker) Pick(ctx context.Context, opts balancer.PickOptions) (balancer.SubConn, func(balancer.DoneInfo), error) {
	// FIXME refactor to unify the color logic，每次客户端 RPC-CALL 都会调用
	color := nmd.String(ctx, nmd.Color)
	if color == ""&& env.Color !="" {
		color = env.Color
	}
	if color != "" {
		if cp, ok := p.colors[color]; ok {
            // 如果定义了 color，走 color 的筛选逻辑
			return cp.pick(ctx, opts)
		}
    }
    // 走默认的 picker 逻辑
	return p.pick(ctx, opts)
}
```


```go
func (p *wrrPicker) pick(ctx context.Context, opts balancer.PickOptions) (balancer.SubConn, func(balancer.DoneInfo), error) {
	var (
		conn        *subConn
		totalWeight int64
	)
	if len(p.subConns) <= 0 {
		return nil, nil, balancer.ErrNoSubConnAvailable
    }

    // 下面是 nginx 的 Weight-rr 算法实现
	p.mu.Lock()
	for _, sc := range p.subConns {
		totalWeight += sc.ewt
		sc.cwt += sc.ewt
		if conn == nil || conn.cwt < sc.cwt {
			conn = sc
		}
    }
    // 按照 WRR 算法的要求
	conn.cwt -= totalWeight
    p.mu.Unlock()

    // 计算本次 RPC 请求的延迟
	start := time.Now()
	if cmd, ok := nmd.FromContext(ctx); ok {
		cmd["conn"] = conn
	}

	return conn.conn, func(di balancer.DoneInfo) {
        // 请求失败计数
		ev := int64(0) // error value ,if error set 1
		if di.Err != nil {
			if st, ok := status.FromError(di.Err); ok {
				// only counter the local grpc error, ignore any business error
				if st.Code() != codes.Unknown && st.Code() != codes.OK {
					ev = 1
				}
			}
        }
        // 失败计数 + 1，成功不加
		conn.err.Add(ev)

        // 计算本次 RPC 请求的延迟
		now := time.Now()
		conn.latency.Add(now.Sub(start).Nanoseconds() / 1e5)
		u := atomic.LoadInt64(&p.updateAt)
		if now.UnixNano()-u < int64(time.Second) {
			return
		}
		if !atomic.CompareAndSwapInt64(&p.updateAt, u, now.UnixNano()) {
			return
		}
		var (
			stats = make([]statistics, len(p.subConns))
			count int
			total float64
		)
		for i, conn := range p.subConns {
            //p.subConns 里面保存了当前连接池中所有和 backend 的连接及其属下（权重）
			cpu := float64(atomic.LoadInt64(&conn.si.cpu))
			ss := math.Float64frombits(atomic.LoadUint64(&conn.si.success))
			errc, req := conn.errSummary()
			lagv, lagc := conn.latencySummary()

			if req > 0 && lagc > 0 && lagv > 0 {
				// client-side success ratio
				cs := 1 - (float64(errc) / float64(req))
				if cs <= 0 {
					cs = 0.1
				} else if cs <= 0.2 && req <= 5 {
					cs = 0.2
				}
				conn.score = math.Sqrt((cs * ss * ss * 1e9) / (lagv * cpu))
				stats[i] = statistics{cs: cs, ss: ss, lantency: lagv, cpu: cpu, req: req}
			}
			stats[i].addr = conn.addr.Addr

			if conn.score > 0 {
				total += conn.score
				count++
			}
		}
		// count must be greater than 1,otherwise will lead ewt to 0
		if count < 2 {
			return
		}
		avgscore := total / float64(count)
		p.mu.Lock()
		for i, conn := range p.subConns {
			if conn.score <= 0 {
                // 更新节点得分
				conn.score = avgscore
			}
			conn.ewt = int64(conn.score * float64(conn.meta.Weight))
			stats[i].ewt = conn.ewt
		}
		p.mu.Unlock()
		log.Info("warden wrr(%s): %+v", conn.addr.ServerName, stats)
	}, nil

}
```

##	0x03	总结

##  0x04	参考
-	[Upstream: smooth weighted round-robin balancing.](https://github.com/phusion/nginx/commit/27e94984486058d73157038f7950a0a36ecc6e35)