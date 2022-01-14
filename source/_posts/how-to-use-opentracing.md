---
title: how-to-use-opentracing
date: 2020-05-05 09:24:37
desc: 使用 opentracing 追踪请求
tags: tracing-go
---

## golang中使用jaeger实现链路追踪。

在分布式系统中，问题的定位非常困难，有了jaeger, 一切变的简单了。 举个简单好理解的例子，追踪就像孙悟空(rootSpan)进了盘丝洞,  发现有很多的洞，每个洞都让他的小猴子(childSpan)进入探路，最终所有的猴子探好了路把信息发给一个收集(jaeger-collector), 最后通过jeagerUI查询(jaeger-query)展示

<!-- more -->

1. 安装jaegertracing/all-in-one

```bash
  docker run  --rm   -p 6831:6831/udp   -p 6832:6832/udp   -p 5778:5778   -p 16686:16686   -p 14268:14268   -p 14250:14250   -p 9411:9411   jaegertracing/all-in-one:latest
```

2. 创建opentracing对象

```golang
type Tracing struct {
	opentracing.Tracer
}

func NewTracing() (*Tracing, func(), error) {
	var (
		err    error
		closer io.Closer
		cfg    *config.Configuration
		tracer opentracing.Tracer
	)
	cfg = &config.Configuration{
		Sampler: &config.SamplerConfig{
			Type:  "const", // 全量
			Param: 1,
		},
		Reporter: &config.ReporterConfig{
			LogSpans:           true,
			LocalAgentHostPort: "192.168.165.21:6831", // 运行 all-in-one 的服务IP, 
		},
	}
	tracer, closer, err = cfg.New(
        "test-tracing", // 你的服务名
		config.Logger(jaeger.StdLogger), // 日志, 可以使用自己的logger
	)
	if err != nil {
		log.Fatalf("ERROR: cannot init Jaeger: %v", err)
	}
	closeFunc := func() {
		_ = closer.Close()
	}
	opentracing.SetGlobalTracer(tracer) // 设置全局
	tracing := &Tracing{
		Tracer:  tracer,
	}
	return tracing, closeFunc, err
}
````

3. 使用, 一般的 gateway/webhttp 做为追踪的起点， 跟据客户端请求traceid, 服务端接收设置tag标记, 使用gin测试

```golang 

func main () {
	g := gin.New()
	g.GET("/login", func(ctx *gin.Context) {
		option := opentracing.Tag{Key: "traceId", Value: ctx.GetHeader("traceId")}
		span, childctx := opentracing.StartSpanFromContext(ctx, "login", option) // 设置tag
		defer span.Finish()
		span.LogKV("loginRequest", ctx.Request.UserAgent())
		selectdb := func() {
			dbspan, _ := opentracing.StartSpanFromContext(childctx, "selectFromDb") // 从父span派生child span
			defer dbspan.Finish()
			dbspan.LogKV("selectFromDb", "id:100")
		}
		selectdb()
        c.JSON(200, gin.H({"message": "ok"}))
	})
	g.Run(":8080")
}

```

4. 访问http://localhost:8080/login 后再去jaegerUI查看 http://192.168.165.21:16686/, 能看到记录的span。

PS: 这个tracing个入感觉对代码的侵入很大。可以针对有必要的进行追踪.

[更好的学习资料](https://github.com/yurishkuro/opentracing-tutorial)
