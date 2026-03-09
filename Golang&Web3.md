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
   
   1） 多节点轮询 + 自动切换。 维护一个节点池。 主节点断了立即切换备用节点。 节点池后台定时ping，标记节点是否可用，用来做后续的备用节点。 重连成功后，立即用FilterLogs补拉断连期间的节点数据。
   
   2） 区块高度记录始终记录， 为重连后获取历史数据准备。  然后收到log时，不能直接入库， 需要先放进待确认队列，等区块的确认数达到安全阈值（eth12块）再入库。同时检查log.removed==true。幂等写入DB 以（blockHash + texHash + logindex）作为联合唯一键

   3） 建立有界的worker pool。 按 address % N固定分配到同一个worker。 如果需要按顺序处理的话。 监控lgs channel和worker pool的性能， 要么扩worker，要么优化单个执行的性能。

   4） 精确的FilterQuery Topics， 减少无用log的发送。 批量入库。 热点数据缓存，避免每次rpc或者查库。

2. 快速扫链
 
   基础就是 将要扫的链分割给多个worker， 每个worker并发去拉数据。

	1） 动态分片大小， 节点对FilterLogs有限制， 需要每次依据结果动态调整log数量。

    2） 断点续扫，扫描数据入库，重启后从断点继续

    3） 拉取和处理逻辑分离，各干各的。


## 针对Go高并发的工程设计
   1） 背压控制。 不同功能层处理速度不一致， 慢的会被撑爆， 使用channel buffer是最简单，最粗糙的背压手段。 使用令牌桶 + 反向通知，下游性能快到瓶颈的时候，告诉上游降速，而不是让buffer打满后阻塞。

   2） 事件溯源。 将原始event入库，业务状态由event重放派生出来。 好处就是链重组不需要回滚，直接从正确的event重放就能得到正确的状态。 任何时间点都可以还原。 

   3） 使用状态机管理每一笔交易，并将状态入库。 即使服务器中断，也不会影响交易丢失。

   4） 使用幂等性设计，整个链路都需要。 (chainId + blockHash + texHash + logIndex) 就是天然的幂等键。

   5） 读写分离

   6） 熔断器， 对每一个外部依赖都要由熔断。 RPC节点， DB写入超时，熔断， 下游服务不可用熔断。 熔断后需要怎么处理就看业务需求。 

   7） 数据一致性分级， 有些是一定要强一致，同步， 像充值提币。 有些可用异步，最终一致就可以， 交易历史，持仓快照。 还有一些允许短暂不一致（缓存） 像价格，TVL统计。

   8） 死信队列。 Worker处理失败的log，不能直接丢弃。 入死信队列，人工参与分析。

## goeth库
   1） 节点相关 

```
// HTTP
client, err := ethclient.Dial("https://mainnet.infura.io/v3/KEY")

// WebSocket（订阅用）
client, err := ethclient.Dial("wss://mainnet.infura.io/ws/v3/KEY")

// IPC（本地节点，延迟最低）
client, err := ethclient.Dial("/var/run/geth.ipc")

// 带超时的连接
client, err := ethclient.DialContext(ctx, rpcURL)

// 底层 rpc.Client（需要 batch 调用时用）
rpcClient, err := rpc.DialContext(ctx, rpcURL)

// 关闭连接
client.Close()

// 获取 ChainID
chainID, err := client.ChainID(ctx)

// 获取网络 ID
networkID, err := client.NetworkID(ctx)
```

  2) 查区块数据

```
// 最新块高
blockNum, err := client.BlockNumber(ctx)

// 按块号查区块（含完整交易）
block, err := client.BlockByNumber(ctx, big.NewInt(18000000))
block, err := client.BlockByNumber(ctx, nil)              // nil = 最新块

// 按 Hash 查区块
block, err := client.BlockByHash(ctx, blockHash)

// 只查块头（不含交易，更轻量）
header, err := client.HeaderByNumber(ctx, nil)
header, err := client.HeaderByHash(ctx, blockHash)
header.Number    // 块高
header.Time      // 时间戳
header.BaseFee   // EIP-1559 baseFee

// 查块内交易数量
count, err := client.TransactionCount(ctx, blockHash)

// 按块内索引查交易
tx, err := client.TransactionInBlock(ctx, blockHash, index)
```

   3) 查交易数据

```
// 按 Hash 查交易
tx, isPending, err := client.TransactionByHash(ctx, txHash)
isPending        // true = 还在 mempool，未打包

// 查交易收据（已上链才有）
receipt, err := client.TransactionReceipt(ctx, txHash)
receipt.Status        // 1=成功 0=revert
receipt.GasUsed       // 实际消耗 Gas
receipt.Logs          // 产生的所有 Log
receipt.BlockNumber   // 打包进哪个块
receipt.ContractAddress // 如果是部署合约，这里有合约地址
```

   4) 查庄户数据

```
// 查 ETH 余额
balance, err := client.BalanceAt(ctx, address, nil)           // 最新块
balance, err := client.BalanceAt(ctx, address, blockNumber)   // 指定块（归档节点）

// 查已确认 Nonce
nonce, err := client.NonceAt(ctx, address, nil)

// 查 Pending Nonce（发交易用这个）
nonce, err := client.PendingNonceAt(ctx, address)

// 查合约字节码
code, err := client.CodeAt(ctx, address, nil)
isContract := len(code) > 0

// 查合约存储槽（读原始 storage）
value, err := client.StorageAt(ctx, address, slotHash, nil)

// 查 Pending 余额
balance, err := client.PendingBalanceAt(ctx, address)
```

   5) 查eventlog

```
// 查历史 Log（HTTP）
logs, err := client.FilterLogs(ctx, ethereum.FilterQuery{
    FromBlock: big.NewInt(18000000),
    ToBlock:   big.NewInt(18001000),
    Addresses: []common.Address{contractAddr},
    Topics:    [][]common.Hash{{eventSig}},
})

// 订阅实时 Log（WebSocket）
logsCh := make(chan types.Log, 1000)
sub, err := client.SubscribeFilterLogs(ctx, ethereum.FilterQuery{
    Addresses: []common.Address{contractAddr},
    Topics:    [][]common.Hash{{eventSig}},
}, logsCh)
sub.Err()        // 监听订阅错误
sub.Unsubscribe()// 取消订阅

// 订阅新块头（WebSocket）
headersCh := make(chan *types.Header, 100)
sub, err := client.SubscribeNewHead(ctx, headersCh)
```

   6) gas相关

```
// 估算 Gas Limit
gasLimit, err := client.EstimateGas(ctx, ethereum.CallMsg{
    From: fromAddr,
    To:   &toAddr,
    Data: calldata,
})

// 建议 Gas Price（Legacy，适合 BSC 等）
gasPrice, err := client.SuggestGasPrice(ctx)

// 建议 Priority Fee（EIP-1559，适合 ETH 主网）
gasTipCap, err := client.SuggestGasTipCap(ctx)

// baseFee 从 Header 里取
header, _ := client.HeaderByNumber(ctx, nil)
baseFee := header.BaseFee

// maxFeePerGas = 2 * baseFee + priorityFee
gasFeeCap := new(big.Int).Add(
    new(big.Int).Mul(baseFee, big.NewInt(2)),
    gasTipCap,
)
```

  7) 构建交易

```
// Legacy 交易（Type 0，BSC/旧链用）
tx := types.NewTransaction(
    nonce,
    toAddr,
    value,
    gasLimit,
    gasPrice,
    data,
)

// EIP-1559 交易（Type 2，ETH 主网推荐）
tx := types.NewTx(&types.DynamicFeeTx{
    ChainID:   chainID,
    Nonce:     nonce,
    To:        &toAddr,
    Value:     value,
    Gas:       gasLimit,
    GasTipCap: gasTipCap,   // priority fee
    GasFeeCap: gasFeeCap,   // max fee
    Data:      data,
})

// 部署合约（To 为 nil）
tx := types.NewTx(&types.DynamicFeeTx{
    ChainID:  chainID,
    Nonce:    nonce,
    To:       nil,           // nil = 部署合约
    Value:    big.NewInt(0),
    Gas:      gasLimit,
    Data:     contractBytecode,
})
```

   8) 签名
```
// 从私钥构建签名器
signer := types.LatestSignerForChainID(chainID)

// 签名交易
signedTx, err := types.SignTx(tx, signer, privateKey)

// 从私钥获取地址
fromAddr := crypto.PubkeyToAddress(privateKey.PublicKey)

// 签名任意消息（EIP-191 personal_sign）
msg := "Login to MyApp: 1234567890"
msgHash := crypto.Keccak256([]byte(
    "\x19Ethereum Signed Message:\n" +
    strconv.Itoa(len(msg)) + msg,
))
sig, err := crypto.Sign(msgHash, privateKey)
sig[64] += 27  // 调整 v 值

// 从签名恢复公钥
pubKey, err := crypto.SigToPub(msgHash, sig)

// 从签名恢复地址
recoveredAddr := crypto.PubkeyToAddress(*pubKey)

// 验证签名（对比恢复地址和声明地址）
isValid := recoveredAddr == claimedAddr

// 生成新私钥
privateKey, err := crypto.GenerateKey()

// 从 hex 加载私钥
privateKey, err := crypto.HexToECDSA("私钥hex字符串")

// 计算任意数据的 Keccak256 哈希
hash := crypto.Keccak256Hash([]byte("Transfer(address,address,uint256)"))
```

   9) 发送交易

```
// 广播已签名交易
err := client.SendTransaction(ctx, signedTx)

// 获取 txHash
txHash := signedTx.Hash()

// 等待交易被打包（轮询 Receipt）
func waitMined(client *ethclient.Client, txHash common.Hash) (*types.Receipt, error) {
    for {
        receipt, err := client.TransactionReceipt(ctx, txHash)
        if err == nil {
            return receipt, nil  // 已打包
        }
        time.Sleep(2 * time.Second)
    }
}

// Batch 发送多个 RPC 请求（底层 rpc.Client）
batch := []rpc.BatchElem{
    {Method: "eth_getBalance", Args: []any{addr1, "latest"}, Result: &bal1},
    {Method: "eth_getBalance", Args: []any{addr2, "latest"}, Result: &bal2},
}
err := rpcClient.BatchCallContext(ctx, batch)
```

   10) ABI编码解码

```
// 解析 ABI
contractABI, err := abi.JSON(strings.NewReader(abiJSON))

// 编码函数调用（生成 calldata）
data, err := contractABI.Pack("transfer", toAddr, amount)

// 解码返回值
var result *big.Int
err = contractABI.UnpackIntoInterface(&result, "balanceOf", returnData)

// 解码到 map
output := make(map[string]interface{})
err = contractABI.UnpackIntoMap(output, "transfer", returnData)

// 解析 Log（非 indexed 参数从 Data 解码）
event := make(map[string]interface{})
err = contractABI.UnpackIntoMap(event, "Transfer", log.Data)

// 计算事件签名 hash（Topics[0]）
eventSig := contractABI.Events["Transfer"].ID
// 或手动计算
eventSig := crypto.Keccak256Hash([]byte("Transfer(address,address,uint256)"))
```

   11) 合约调用（bind 层）

```
// ── abigen 生成的 binding ────────────────────────────

// 实例化合约
token, err := erc20.NewErc20(contractAddr, client)

// 读操作（CallOpts）
opts := &bind.CallOpts{
    Context:     ctx,
    BlockNumber: nil,      // nil = 最新块
    From:        fromAddr, // 模拟 msg.sender
    Pending:     false,    // 是否查 pending 状态
}
balance, err := token.BalanceOf(opts, userAddr)

// 写操作（TransactOpts）
auth, err := bind.NewKeyedTransactorWithChainID(privateKey, chainID)
auth.Context  = ctx
auth.Nonce    = big.NewInt(int64(nonce))
auth.GasLimit = uint64(100000)
auth.GasPrice = gasPrice    // Legacy
auth.GasTipCap= gasTipCap   // EIP-1559
auth.GasFeeCap= gasFeeCap   // EIP-1559
auth.Value    = big.NewInt(0)// 附带的 ETH 数量

tx, err := token.Transfer(auth, toAddr, amount)

// 模拟调用（不上链，只看结果）
auth.NoSend = true  // 设置 NoSend，只构建不发送
tx, err := token.Transfer(auth, toAddr, amount)

// ── 手动 CallContract ────────────────────────────────
result, err := client.CallContract(ctx, ethereum.CallMsg{
    From: fromAddr,
    To:   &contractAddr,
    Data: calldata,
}, nil)
```

  12) Keystore（本地密钥管理）

```
// 创建 keystore
ks := keystore.NewKeyStore("./keystore", keystore.StandardScryptN, keystore.StandardScryptP)

// 创建新账户
account, err := ks.NewAccount("密码")

// 导入私钥
account, err := ks.ImportECDSA(privateKey, "密码")

// 解锁账户
err := ks.Unlock(account, "密码")

// 用 keystore 签名交易
signedTx, err := ks.SignTx(account, tx, chainID)
```

   13) 常用的

```
查余额          → BalanceAt
查交易状态      → TransactionReceipt
读合约数据      → token.BalanceOf(CallOpts)  或  CallContract
写合约          → token.Transfer(TransactOpts)
发 ETH          → 构建 tx + SignTx + SendTransaction
订阅事件        → SubscribeFilterLogs
历史事件        → FilterLogs
签名验证        → crypto.Sign + crypto.SigToPub
等待确认        → 轮询 TransactionReceipt
批量查询        → rpcClient.BatchCallContext
```
