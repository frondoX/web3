# Golang 在 web3 下的使用

## Gin 框架

1. 路由算法： 基数树。 放弃之前的正则匹配，是因为为每一个路由存储一个正则对象耗内存，还有只能通过遍历来获取路由，性能和内存都不占优。 

基数树通过压缩公共前缀来增加遍历性能。  和以太坊中的默克尔帕特里夏树的逻辑一样 （MPT Merkle Patricia Tree）。  公共前缀，及确定性（这一点在以太坊中至关重要，因为不同的EVM环境一定要跑出确定性的RootHash）。 这两的区别就是MPT在此基础上又添加了哈希检验。

在Gin中为每一种http方法都维护一个独立的基数树。 Get， Post， Put等。 减少树的深度，且不同的方法业务逻辑一般是不同，逻辑解耦。

节点类型： static 静态节点， param 参数节点， catchAll 通配符节点。

统一层级， 不能同时存在带有不同参数名的节点。

Get /user/:id 和 Get /user/:name 无法区分。

2. 中间件

中间件是在请求到达具体的业务逻辑之前或之后，针对特定路由节点或者整颗路由树进行验证，记录和处理。

全局中间件
路由组中间件（group）
单个插件中间件

核心原理 c.Next() 洋葱模型。 内部结构是一个函数切片。
c.Abort() 当前函数还会执行，只是阻止后续函数执行，所以要跟一个return返回。 

如果在中间件种开了 go func()异步处理， 不能直接用 gin.Context， 因为Context被复用，需要拷贝一个出来使用。

然后注意 Context的使用， 如果直接的层级关系。 如果不想依赖上层的context，使用context.Background()或者context.TODO() 这两底层一样。

只有完全路由完了，才会建立中间件和handler的函数切片。 开始执行。

```
r := gin.New() // 使用 gin.New() 而非 gin.Default()，手动控制中间件

	// ── ① 全局中间件 ─────────────────────────────────────────
	// 对所有路由生效
	r.Use(GlobalLogger())

	// ── 公开路由（仅受全局中间件保护）────────────────────────
	r.GET("/", handleHome)

	// ── ② 路由组中间件 ───────────────────────────────────────
	// 仅对 /api/* 路由生效
	api := r.Group("/api")
	api.Use(GroupAuthMiddleware())
	{
		api.GET("/profile", handleProfile)
		api.GET("/settings", handleSettings)

		// ── ③ 单路由中间件 ───────────────────────────────────
		// 仅对 GET /api/resource 生效，叠加在组中间件之上
		api.GET("/resource", SingleRateLimiter(), handleLimitedResource)
	}
```

## Gin与合约的深度交互

1. 订阅链上实时的Event log
   基础就是通过带重连的websocket 和 节点进行连接， 然后建立缓冲通道读取数据，再通过select监听通道及连接处理重连还是信息。  重连采用退避抖动重连，并设置最大时间。
   1） 优化， 使用go func处理消息，

