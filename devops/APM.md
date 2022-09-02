## APM

### 链路日志

遵循OpenTracing规范，对分布式应用日志进行链路追踪，以图表的方式展示业务接口的响应时间，并树状显示单条链路的调用层级详情，实现了应用性能管理(APM)。

### 埋点

Jaeger 的客户端，实现了 OpenTracing 的 API，客户端直接集成在目标 Application 中，其作用是记录和发送 Span 到 Jaeger-Sender。在 Application 中调用 Jaeger Client Library 记录 Span 的过程通常被称为埋点。