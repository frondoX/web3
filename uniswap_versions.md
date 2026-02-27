# Uniswap 全版本深度解析：V1 → V2 → V3 → V4
## 含完整代码实现 + 数学模型推导

> 本文档系统梳理 Uniswap 各版本的设计思想、核心功能、数学模型、代码实现、优缺点，以及版本迭代解决的核心问题。每个关键机制均附带可运行的 Solidity 代码和数学推导。

---

## 目录

1. [Uniswap V1 — AMM 的起点](#uniswap-v1)
2. [Uniswap V2 — 全面进化](#uniswap-v2)
3. [Uniswap V3 — 集中流动性革命](#uniswap-v3)
4. [Uniswap V4 — 可编程流动性](#uniswap-v4)
5. [版本演进全景对比](#版本演进全景对比)
6. [数学模型汇总速查](#数学模型汇总速查)

---

## Uniswap V1

> 发布时间：2018 年 11 月 | 开发语言：Vyper | 部署网络：以太坊主网

---

### 1.1 核心数学模型：恒定乘积公式

#### 数学定义

$$x \cdot y = k$$

其中：
- $x$ = 池中 ETH 储量
- $y$ = 池中 ERC20 代币储量  
- $k$ = 常数（不变量）

**几何意义**：这是一条双曲线。随着 x 增大，y 必然减小，反之亦然。价格永远不会到达 0 或无穷大，池子永远不会被完全耗尽。

```
y
│
│\
│ \
│  \       ← 双曲线：x * y = k
│   \
│    ──\___
└─────────── x
```

#### 即时价格推导

在任意时刻，token0 相对 token1 的边际价格为：

$$P = \frac{y}{x} = \frac{\text{ERC20储量}}{\text{ETH储量}}$$

**直觉理解**：ETH 越少（被买走），每个 ETH 越贵；ERC20 越少，每个 ERC20 越贵。价格由稀缺性决定。

#### swap 输出量推导

设用户输入 $\Delta x$ 个 ETH，求可获得的 ERC20 数量 $\Delta y$：

```
交易前：x * y = k
交易后：(x + Δx) * (y - Δy) = k

因此：
  y - Δy = k / (x + Δx) = x*y / (x + Δx)
  Δy = y - x*y / (x + Δx)
  Δy = y * Δx / (x + Δx)
```

考虑 0.3% 手续费（997/1000 近似）：

$$\Delta y = \frac{y \cdot \Delta x \cdot 997}{x \cdot 1000 + \Delta x \cdot 997}$$

**代码中的体现（V1 Vyper 核心逻辑）：**

```python
# Uniswap V1 核心 swap 函数（Vyper 伪代码）
@public
@payable
def ethToTokenSwapInput(min_tokens: uint256, deadline: timestamp) -> uint256:
    # 数学模型：Δy = y * Δx * 997 / (x * 1000 + Δx * 997)
    
    eth_sold: uint256 = msg.value                    # Δx（用户输入的 ETH）
    eth_reserve: uint256 = self.balance - eth_sold   # x（ETH 储量，减去刚收到的）
    token_reserve: uint256 = self.token.balanceOf(self)  # y（ERC20 储量）
    
    # 应用恒定乘积公式 + 手续费
    # 手续费 = 0.3%，即 997/1000
    numerator: uint256 = eth_sold * 997 * token_reserve
    denominator: uint256 = eth_reserve * 1000 + eth_sold * 997
    tokens_bought: uint256 = numerator / denominator  # Δy
    
    assert tokens_bought >= min_tokens  # 滑点保护
    assert block.timestamp <= deadline  # 时间保护
    
    self.token.transfer(msg.sender, tokens_bought)
    return tokens_bought
```

#### 价格冲击数值示例

```
池子状态：1,000 ETH + 2,000,000 DAI（初始价格 2000 DAI/ETH）

买入 1 ETH（Δx = 1）：
  Δy = 2,000,000 * 1 * 997 / (1,000 * 1000 + 1 * 997)
     = 1,994,000 / 1,000,997
     ≈ 1,993 DAI
  实际价格：1,993 DAI/ETH，滑点 ~0.35%

买入 100 ETH（Δx = 100）：
  Δy = 2,000,000 * 100 * 997 / (1,000 * 1000 + 100 * 997)
     = 199,400,000 / 1,099,700
     ≈ 181,320 DAI
  实际价格：1,813 DAI/ETH，滑点 ~9.35%

结论：交易量越大，价格冲击越大（双曲线曲率导致）
```

---

### 1.2 LP Token 数学模型

#### 首次添加流动性

$$\text{LP} = \sqrt{x_0 \cdot y_0}$$

使用几何平均数的原因：
- 算术平均数依赖计价单位（DAI 计价和 ETH 计价结果不同）
- 几何平均数 $\sqrt{x \cdot y}$ 是无单位的，保证 LP 价值不随计价货币变化

**代码实现（V1 mint 逻辑）：**

```python
@public
def addLiquidity(min_liquidity: uint256, max_tokens: uint256, deadline: timestamp) -> uint256:
    eth_amount: uint256 = msg.value
    total_liquidity: uint256 = self.totalSupply
    
    if total_liquidity > 0:
        # 后续添加：按比例计算，取两者的最小值（严格按比例）
        # 数学：LP = min(Δx/x * S, Δy/y * S)
        eth_reserve: uint256 = self.balance - eth_amount
        token_amount: uint256 = eth_amount * self.token.balanceOf(self) / eth_reserve + 1
        liquidity_minted: uint256 = eth_amount * total_liquidity / eth_reserve
        
        self.token.transferFrom(msg.sender, self, token_amount)
        self.totalSupply = total_liquidity + liquidity_minted
        self.balances[msg.sender] += liquidity_minted
        return liquidity_minted
    else:
        # 首次添加：几何平均数
        # 数学：LP = sqrt(x * y)
        # V1 简化为直接使用 eth_amount（因为首次比例任意）
        token_amount: uint256 = max_tokens
        initial_liquidity: uint256 = self.balance  # = eth_amount
        self.totalSupply = initial_liquidity
        self.balances[msg.sender] = initial_liquidity
        self.token.transferFrom(msg.sender, self, token_amount)
        return initial_liquidity
```

#### 移除流动性

$$\Delta x = \frac{\text{LP\_burned}}{\text{totalSupply}} \cdot x$$
$$\Delta y = \frac{\text{LP\_burned}}{\text{totalSupply}} \cdot y$$

LP 按持有比例等比例取回两种资产。

---

### 1.3 V1 架构与关键限制

#### ETH 中心化架构

```
所有 V1 池子：ETH ←→ ERC20

ERC20 A → ERC20 B 必须两跳：
  ERC20 A → ETH（Pool A，手续费 0.3%）
  ETH → ERC20 B（Pool B，手续费 0.3%）

总手续费 ≈ 0.6%（两次 0.3%，复合计算）
```

**数学上的复合手续费损耗：**

```
单跳手续费因子：997/1000 = 0.997
两跳手续费因子：0.997 * 0.997 = 0.994009
有效手续费：1 - 0.994009 ≈ 0.599%（而非简单相加的 0.6%）
```

#### V1 的根本局限（为 V2 埋下伏笔）

```
问题 1：ERC20/ERC20 必须经过 ETH，手续费翻倍
问题 2：稳定币 DAI/USDC 兑换时承担 ETH 价格风险
问题 3：无价格预言机，外部协议无法可信地获取链上价格
问题 4：无闪电贷，资本利用率低
问题 5：原生 ETH 处理增加合约复杂度
```

---

## Uniswap V2

> 发布时间：2020 年 5 月 | 开发语言：Solidity 0.5.x | 核心改进：ERC20/ERC20、TWAP、Flash Swap

---

### 2.1 核心数学模型：带手续费的恒定乘积

#### 手续费的数学处理

V2 在验证 k 值时对手续费做了精妙的整数运算处理，避免浮点数：

$$\text{balance0Adjusted} \cdot \text{balance1Adjusted} \geq \text{reserve0} \cdot \text{reserve1} \cdot 1000^2$$

其中：
$$\text{balanceAdjusted} = \text{balance} \cdot 1000 - \text{amount\_in} \cdot 3$$

**数学含义**：扣除 0.3% 手续费后的余额，必须满足 k 不减（实际上手续费让 k 缓慢增长）。

**代码实现（UniswapV2Pair.sol swap 函数完整版）：**

```solidity
// UniswapV2Pair.sol
function swap(
    uint amount0Out,
    uint amount1Out,
    address to,
    bytes calldata data
) external lock {
    require(amount0Out > 0 || amount1Out > 0, 'UniswapV2: INSUFFICIENT_OUTPUT_AMOUNT');

    (uint112 _reserve0, uint112 _reserve1,) = getReserves(); // 读取快照储量

    // ① 乐观转出（先给钱，后验证）
    if (amount0Out > 0) _safeTransfer(_token0, to, amount0Out);
    if (amount1Out > 0) _safeTransfer(_token1, to, amount1Out);

    // ② 如果 data 非空，触发闪电贷回调
    // 借款方在此回调中执行任意操作，但必须在回调结束前还款
    if (data.length > 0) IUniswapV2Callee(to).uniswapV2Call(
        msg.sender, amount0Out, amount1Out, data
    );

    // ③ 读取当前真实余额（回调执行后）
    uint balance0 = IERC20(_token0).balanceOf(address(this));
    uint balance1 = IERC20(_token1).balanceOf(address(this));

    // ④ 计算实际转入量（真实余额 - (储量 - 转出量)）
    uint amount0In = balance0 > _reserve0 - amount0Out
        ? balance0 - (_reserve0 - amount0Out) : 0;
    uint amount1In = balance1 > _reserve1 - amount1Out
        ? balance1 - (_reserve1 - amount1Out) : 0;
    require(amount0In > 0 || amount1In > 0, 'UniswapV2: INSUFFICIENT_INPUT_AMOUNT');

    // ⑤ 验证 k 值约束（核心数学模型）
    // 数学：(balance0 * 1000 - amount0In * 3) * (balance1 * 1000 - amount1In * 3) >= reserve0 * reserve1 * 1000^2
    // 等价于：扣除手续费后的余额乘积 >= 原始 k 值
    // 手续费 0.3% = 3/1000，所以扣除因子是 (1000 - 3) / 1000
    {
        uint balance0Adjusted = balance0.mul(1000).sub(amount0In.mul(3));
        uint balance1Adjusted = balance1.mul(1000).sub(amount1In.mul(3));
        require(
            balance0Adjusted.mul(balance1Adjusted) >= uint(_reserve0).mul(_reserve1).mul(1000**2),
            'UniswapV2: K'
        );
    }

    // ⑥ 更新储量快照
    _update(balance0, balance1, _reserve0, _reserve1);
    emit Swap(msg.sender, amount0In, amount1In, amount0Out, amount1Out, to);
}
```

**为什么用 1000 而不是直接用小数？**

```
Solidity 整数运算不支持小数，0.997 无法直接表示。
解决方案：放大 1000 倍来模拟小数运算：

手续费 = 0.3% = 3/1000
扣除手续费后的有效金额 = amount_in * 997/1000

但在 k 验证时：
  balance_adjusted = balance * 1000 - amount_in * 3
  等价于：balance - amount_in * 3/1000
        = balance - amount_in * 0.3%
        = 余额中扣除手续费部分

两侧都乘以 1000，得到：
  balance0Adjusted * balance1Adjusted >= reserve0 * reserve1 * 1000^2
```

---

### 2.2 TWAP 价格预言机的数学模型

#### 为什么不用瞬时价格

**瞬时价格攻击模型：**

```
攻击者目标：操纵借贷协议，用被高估的抵押品借出大量资产

步骤：
  T=0: 用闪电贷借出大量 DAI
  T=0: 用 DAI 买入大量 ETH → ETH 瞬时价格被推高 3x
  T=0: 借贷协议读取瞬时价格（ETH=6000 DAI）
  T=0: 用少量 ETH 作抵押，借出大量 DAI（基于虚高价格）
  T=0: 归还闪电贷
  T=0: 净赚：借出的 DAI - 抵押品真实价值

整个过程在同一个区块（同一笔交易）内完成
攻击成本仅为手续费（< 0.3%）
```

#### TWAP 数学原理

**时间加权平均价格（TWAP）**本质上是价格关于时间的积分：

$$\text{TWAP}_{t_1 \to t_2} = \frac{\int_{t_1}^{t_2} P(t) \, dt}{t_2 - t_1}$$

离散化为累积求和：

$$\text{priceCumulative}(t) = \sum_{i} P_i \cdot \Delta t_i$$

$$\text{TWAP} = \frac{\text{priceCumulative}(t_2) - \text{priceCumulative}(t_1)}{t_2 - t_1}$$

**代码实现（UniswapV2Pair.sol _update 函数）：**

```solidity
// 价格累积器（UQ112x112 定点数格式）
uint public price0CumulativeLast;  // token1/token0 的累积价格
uint public price1CumulativeLast;  // token0/token1 的累积价格
uint32 public blockTimestampLast;

function _update(
    uint balance0,
    uint balance1,
    uint112 _reserve0,
    uint112 _reserve1
) private {
    uint32 blockTimestamp = uint32(block.timestamp % 2**32);
    uint32 timeElapsed = blockTimestamp - blockTimestampLast; // 溢出安全（模运算）

    if (timeElapsed > 0 && _reserve0 != 0 && _reserve1 != 0) {
        // 核心数学：累积 = 上次累积 + 当前价格 × 经过时间
        // UQ112x112 是 Q 格式定点数：高 112 位是整数部分，低 112 位是小数部分
        // encode() 将 uint112 转为 UQ112x112（左移 112 位）
        // uqdiv 执行定点数除法

        // price0 = reserve1 / reserve0（token0 用 token1 计价）
        price0CumulativeLast += uint(UQ112x112.encode(_reserve1).uqdiv(_reserve0)) * timeElapsed;
        // price1 = reserve0 / reserve1（token1 用 token0 计价）
        price1CumulativeLast += uint(UQ112x112.encode(_reserve0).uqdiv(_reserve1)) * timeElapsed;
    }

    reserve0 = uint112(balance0);
    reserve1 = uint112(balance1);
    blockTimestampLast = blockTimestamp;
}
```

**UQ112x112 定点数格式说明：**

```
UQ112x112 = Unsigned Q112.112 定点数
  整数部分：高 112 位
  小数部分：低 112 位
  精度：1 / 2^112 ≈ 2 × 10^{-34}

为什么用 112 位？
  两个 UQ112x112 相乘 = 224 位，刚好能放进 uint256（256 位）
  reserve0 和 reserve1 各用 112 位，合计 224 位，同样放进 uint256

溢出设计：
  uint32 时间戳约 136 年后溢出一次
  溢出后模运算保证差值仍然正确（如果两次快照间隔 < 136 年）
  故意允许累积值溢出，差值运算通过模运算保持正确
```

**外部合约读取 TWAP 的完整示例：**

```solidity
// 链上预言机合约示例
contract TWAPOracle {
    address public immutable pair;
    uint public price0CumulativeLast;
    uint public price1CumulativeLast;
    uint32 public blockTimestampLast;
    uint224 public price0Average;  // UQ112x112 格式的 TWAP
    uint224 public price1Average;

    uint public constant PERIOD = 24 hours;  // TWAP 窗口期

    constructor(address _pair) {
        pair = _pair;
        // 初始化快照
        IUniswapV2Pair p = IUniswapV2Pair(_pair);
        price0CumulativeLast = p.price0CumulativeLast();
        price1CumulativeLast = p.price1CumulativeLast();
        (,, blockTimestampLast) = p.getReserves();
    }

    // 更新 TWAP（任何人都可以调用）
    function update() external {
        IUniswapV2Pair p = IUniswapV2Pair(pair);
        uint32 blockTimestamp = uint32(block.timestamp % 2**32);
        uint32 timeElapsed = blockTimestamp - blockTimestampLast;

        require(timeElapsed >= PERIOD, 'TWAP: PERIOD_NOT_ELAPSED');

        uint price0Cumulative = p.price0CumulativeLast();
        uint price1Cumulative = p.price1CumulativeLast();

        // TWAP 数学：(cumulative_now - cumulative_last) / timeElapsed
        // 得到时间窗口内的平均价格
        price0Average = uint224(
            (price0Cumulative - price0CumulativeLast) / timeElapsed
        );
        price1Average = uint224(
            (price1Cumulative - price1CumulativeLast) / timeElapsed
        );

        // 更新快照
        price0CumulativeLast = price0Cumulative;
        price1CumulativeLast = price1Cumulative;
        blockTimestampLast = blockTimestamp;
    }

    // 根据 TWAP 计算输出金额
    function consult(address token, uint amountIn)
        external view returns (uint amountOut)
    {
        IUniswapV2Pair p = IUniswapV2Pair(pair);
        if (token == p.token0()) {
            // amountOut = amountIn * price0Average / 2^112
            amountOut = uint(price0Average) * amountIn >> 112;
        } else {
            amountOut = uint(price1Average) * amountIn >> 112;
        }
    }
}
```

**TWAP 防操纵的数学分析：**

```
攻击 24 小时 TWAP 所需成本估算：

设正常价格 P₀ = 2000 DAI/ETH
攻击者想把 TWAP 推高到 P₁ = 4000 DAI/ETH（2倍）

需要维持虚假价格的时间比例：
  如果攻击 t 小时，正常价格占 (24 - t) 小时：
  TWAP = [P₀ * (24 - t) + P₁ * t] / 24 = 4000
  → t = 24 * (4000 - 2000) / (4000 - 2000) = 24 小时

  即需要全程维持虚假价格才能使 TWAP 翻倍！
  期间所有套利者会不断对抗，每秒都有成本
  实际上完全不可行
```

---

### 2.3 Flash Swap 完整实现

#### 数学约束

Flash Swap 的安全性依赖同一个 k 值约束：

$$\text{balance0}_{adj} \cdot \text{balance1}_{adj} \geq \text{reserve0} \cdot \text{reserve1} \cdot 1000^2$$

借出后还款时，k 必须不减少（手续费实际上让 k 增大）。

**完整 Flash Swap 借款方实现：**

```solidity
// Flash Swap 使用示例：跨 DEX 无本金套利
contract FlashArbitrage is IUniswapV2Callee {

    address public immutable factory;

    constructor(address _factory) {
        factory = _factory;
    }

    // 发起 Flash Swap
    function startArbitrage(
        address token0,
        address token1,
        uint amount0,   // 借出 token0 数量（0 表示不借）
        uint amount1    // 借出 token1 数量（0 表示不借）
    ) external {
        address pair = IUniswapV2Factory(factory).getPair(token0, token1);
        require(pair != address(0), 'Pair not found');

        // 将目标 token 地址编码到 data 中，传递给回调
        bytes memory data = abi.encode(token0, token1, msg.sender);

        // 调用 swap，data 非空 → 触发 Flash Swap 模式
        // Pair 会先把 token 转出，再调用 uniswapV2Call，最后验证 k
        IUniswapV2Pair(pair).swap(amount0, amount1, address(this), data);
    }

    // Pair 合约回调此函数（Flash Swap 核心逻辑）
    function uniswapV2Call(
        address sender,
        uint amount0,     // 已收到的 token0 数量
        uint amount1,     // 已收到的 token1 数量
        bytes calldata data
    ) external override {
        // 安全验证：确认调用方是合法的 Pair 合约
        (address token0, address token1, address initiator) =
            abi.decode(data, (address, address, address));
        address pair = IUniswapV2Factory(factory).getPair(token0, token1);
        require(msg.sender == pair, 'Unauthorized');
        require(sender == address(this), 'Not initiated by us');

        // ─── 在这里执行套利逻辑 ───
        // 此时 address(this) 已持有借出的代币
        // 例：在另一个 DEX 卖出套利
        uint amountReceived = _sellOnOtherDex(token0, amount0);

        // 计算需要还款金额（原始借款 + 0.3% 手续费）
        // 数学：repay = borrow * 1000 / 997（向上取整）
        uint fee = (amount0 * 3) / 997 + 1;  // 0.3% 手续费
        uint amountToRepay = amount0 + fee;

        require(amountReceived > amountToRepay, 'Arbitrage not profitable');

        // 还款（还回 token0，连同手续费）
        IERC20(token0).transfer(pair, amountToRepay);

        // 利润归还给发起者
        uint profit = amountReceived - amountToRepay;
        IERC20(token0).transfer(initiator, profit);
    }

    function _sellOnOtherDex(address token, uint amount) internal returns (uint) {
        // 在其他 DEX 卖出代币的逻辑（省略实现）
        return amount; // 占位
    }
}
```

---

### 2.4 LP Token 数学：mint 和 burn

#### mint（添加流动性）数学

```solidity
// UniswapV2Pair.sol mint 函数
function mint(address to) external lock returns (uint liquidity) {
    (uint112 _reserve0, uint112 _reserve1,) = getReserves();

    // 读取当前真实余额（用户已提前转入）
    uint balance0 = IERC20(token0).balanceOf(address(this));
    uint balance1 = IERC20(token1).balanceOf(address(this));

    // 计算转入量 = 当前余额 - 上次储量快照
    uint amount0 = balance0.sub(_reserve0);
    uint amount1 = balance1.sub(_reserve1);

    // 收取协议费（如果开启）
    bool feeOn = _mintFee(_reserve0, _reserve1);

    uint _totalSupply = totalSupply;

    if (_totalSupply == 0) {
        // ── 首次添加流动性 ──
        // 数学：LP = sqrt(x * y) - MINIMUM_LIQUIDITY
        // 几何平均数保证 LP 价值与计价单位无关
        liquidity = Math.sqrt(amount0.mul(amount1)).sub(MINIMUM_LIQUIDITY);

        // 永久锁定 MINIMUM_LIQUIDITY = 1000 到零地址
        // 目的：防止"价值操控攻击"（见 2.5 节）
        _mint(address(0), MINIMUM_LIQUIDITY);
    } else {
        // ── 后续添加流动性 ──
        // 数学：LP = min(Δx/x * S, Δy/y * S)
        // 取两者最小值：如果比例不对，多余的部分不计入（但已转入，成为所有 LP 的收益）
        // 这机制鼓励用户按正确比例添加流动性
        liquidity = Math.min(
            amount0.mul(_totalSupply) / _reserve0,  // Δx/x * S
            amount1.mul(_totalSupply) / _reserve1   // Δy/y * S
        );
    }

    require(liquidity > 0, 'UniswapV2: INSUFFICIENT_LIQUIDITY_MINTED');
    _mint(to, liquidity);

    _update(balance0, balance1, _reserve0, _reserve1);

    if (feeOn) kLast = uint(reserve0).mul(reserve1); // 记录 k 值，用于下次协议费计算
    emit Mint(msg.sender, amount0, amount1);
}
```

#### burn（移除流动性）数学

$$\Delta x = \frac{\text{LP}}{\text{totalSupply}} \cdot x$$
$$\Delta y = \frac{\text{LP}}{\text{totalSupply}} \cdot y$$

```solidity
function burn(address to) external lock returns (uint amount0, uint amount1) {
    (uint112 _reserve0, uint112 _reserve1,) = getReserves();

    uint balance0 = IERC20(token0).balanceOf(address(this));
    uint balance1 = IERC20(token1).balanceOf(address(this));

    // 读取本合约持有的 LP Token 数量（用户已提前转入）
    uint liquidity = balanceOf[address(this)];

    bool feeOn = _mintFee(_reserve0, _reserve1);
    uint _totalSupply = totalSupply;

    // 数学：按比例取出
    // amount = liquidity / totalSupply * balance
    amount0 = liquidity.mul(balance0) / _totalSupply;
    amount1 = liquidity.mul(balance1) / _totalSupply;

    require(amount0 > 0 && amount1 > 0, 'UniswapV2: INSUFFICIENT_LIQUIDITY_BURNED');

    _burn(address(this), liquidity);
    _safeTransfer(token0, to, amount0);
    _safeTransfer(token1, to, amount1);

    balance0 = IERC20(token0).balanceOf(address(this));
    balance1 = IERC20(token1).balanceOf(address(this));

    _update(balance0, balance1, _reserve0, _reserve1);
    if (feeOn) kLast = uint(reserve0).mul(reserve1);
    emit Burn(msg.sender, amount0, amount1, to);
}
```

---

### 2.5 MINIMUM_LIQUIDITY 防攻击数学分析

**攻击向量（无保护时）：**

```
设攻击者能让 totalSupply = 1，reserve 极大

后续用户添加流动性时：
  LP = amount * totalSupply / reserve
     = amount * 1 / (极大数)
     = 0（整数截断）

用户存入大量资产，得到 0 个 LP Token
资产被所有（仅攻击者）LP 持有者瓜分
```

**保护数学：**

```
MINIMUM_LIQUIDITY = 1000 被永久锁定到 address(0)

效果：
  totalSupply 最小值 = 1000
  单个 LP Token 最大价值 = 总资产 / 1000
  
后续用户添加时：
  LP = amount * totalSupply / reserve
     ≥ amount * 1000 / reserve

  最坏情况（整数截断损失）：
  损失 ≤ 1 个 LP Token = 总资产 / totalSupply ≤ 总资产 / 1000 = 0.1%
  
  攻击者需要放弃价值 MINIMUM_LIQUIDITY 对应的资产（约数百美元）
  而损失上限只有 0.1%，使攻击无利可图
```

---

### 2.6 EIP-2612 Permit 签名数学

#### 签名流程

```solidity
// UniswapV2ERC20.sol permit 实现
function permit(
    address owner,      // 授权方
    address spender,    // 被授权方（通常是 Router）
    uint value,         // 授权金额
    uint deadline,      // 签名过期时间
    uint8 v, bytes32 r, bytes32 s  // ECDSA 签名三元组
) external {
    require(deadline >= block.timestamp, 'UniswapV2: EXPIRED');

    // EIP-712 结构化数据哈希
    // 防止跨合约、跨链重放攻击
    bytes32 digest = keccak256(
        abi.encodePacked(
            '\x19\x01',          // EIP-191 前缀，表示结构化数据
            DOMAIN_SEPARATOR,    // 绑定到：协议名 + 版本 + chainId + 合约地址
            keccak256(abi.encode(
                PERMIT_TYPEHASH,   // 函数签名的哈希
                owner,
                spender,
                value,
                nonces[owner]++,   // 每次使用后自增，旧签名立即失效
                deadline
            ))
        )
    );

    // ecrecover：从签名和消息哈希恢复签名者地址
    // 数学：给定 (r, s, v) 和 digest，恢复出签名私钥对应的公钥地址
    address recoveredAddress = ecrecover(digest, v, r, s);

    require(
        recoveredAddress != address(0) && recoveredAddress == owner,
        'UniswapV2: INVALID_SIGNATURE'
    );

    _approve(owner, spender, value);
}
```

**DOMAIN_SEPARATOR 构造：**

```solidity
// 在构造函数中计算，绑定到当前合约和链
DOMAIN_SEPARATOR = keccak256(
    abi.encode(
        keccak256('EIP712Domain(string name,string version,uint256 chainId,address verifyingContract)'),
        keccak256(bytes(name)),        // 合约名称
        keccak256(bytes('1')),         // 版本号
        chainId,                       // 链 ID（防跨链重放）
        address(this)                  // 合约地址（防跨合约重放）
    )
);
```

**前端生成签名的代码（ethers.js）：**

```javascript
// 前端生成 Permit 签名
async function generatePermitSignature(
    signer,           // 用户钱包
    lpTokenContract,  // LP Token 合约
    spender,          // Router 地址
    amount,           // 授权金额
    deadline          // 过期时间
) {
    const domain = {
        name: await lpTokenContract.name(),
        version: '1',
        chainId: (await signer.provider.getNetwork()).chainId,
        verifyingContract: lpTokenContract.address
    };

    const types = {
        Permit: [
            { name: 'owner',    type: 'address' },
            { name: 'spender',  type: 'address' },
            { name: 'value',    type: 'uint256' },
            { name: 'nonce',    type: 'uint256' },
            { name: 'deadline', type: 'uint256' }
        ]
    };

    const value = {
        owner:    await signer.getAddress(),
        spender:  spender,
        value:    amount,
        nonce:    await lpTokenContract.nonces(await signer.getAddress()),
        deadline: deadline
    };

    // _signTypedData 在链下生成签名，不消耗 gas
    const signature = await signer._signTypedData(domain, types, value);
    const { v, r, s } = ethers.utils.splitSignature(signature);
    return { v, r, s };
}
```

---

### 2.7 协议费数学模型

协议费在 mint/burn 时通过铸造新 LP Token 实现，不在每笔 swap 时收取。

**数学推导：**

设开启协议费前 $k_1 = \sqrt{\text{reserve0}_1 \cdot \text{reserve1}_1}$，现在 $k_2 = \sqrt{\text{reserve0}_2 \cdot \text{reserve1}_2}$。

LP 价值增长来自手续费积累（$k_2 > k_1$），协议费按 $1/6$ 提取：

$$\text{feeLiquidity} = \frac{\sqrt{k_2} - \sqrt{k_1}}{\frac{5}{\phi} \cdot \sqrt{k_2} + \sqrt{k_1}}$$

其中 $\phi = 1/6$（协议费占比），代入得：

$$\text{feeLiquidity} = \frac{\sqrt{k_2} - \sqrt{k_1}}{5 \cdot \sqrt{k_2} + \sqrt{k_1}} \cdot \text{totalSupply}$$

```solidity
// _mintFee：协议费计算
function _mintFee(uint112 _reserve0, uint112 _reserve1) private returns (bool feeOn) {
    address feeTo = IUniswapV2Factory(factory).feeTo();
    feeOn = feeTo != address(0);
    uint _kLast = kLast;

    if (feeOn) {
        if (_kLast != 0) {
            // 数学：比较两次操作间 k 值的变化
            // k 的增长 = 手续费积累
            uint rootK = Math.sqrt(uint(_reserve0).mul(_reserve1));   // sqrt(k_now)
            uint rootKLast = Math.sqrt(_kLast);                        // sqrt(k_last)

            if (rootK > rootKLast) {
                // 只有 k 增长时才有协议费可收
                uint numerator = totalSupply.mul(rootK.sub(rootKLast));
                // 分母：5 * rootK + rootKLast → 提取 1/6 的增量
                uint denominator = rootK.mul(5).add(rootKLast);
                uint liquidity = numerator / denominator;

                if (liquidity > 0) _mint(feeTo, liquidity);
            }
        }
    } else if (_kLast != 0) {
        kLast = 0;
    }
}
```

**为什么在 mint/burn 时而非 swap 时收费？**

```
每次 swap 收费：
  需要额外的状态存储和计算
  每笔交易多消耗约 5000 gas
  Uniswap 日交易量数十万笔，总 gas 极高

mint/burn 时收费：
  只在流动性变化时计算（频率低得多）
  通过铸造新 LP Token 给 feeTo 实现
  现有 LP 被稀释，但 gas 成本最优
```

---

### 2.8 getAmountOut 完整推导

Router 合约中核心的输出金额计算：

```solidity
// UniswapV2Library.sol
function getAmountOut(
    uint amountIn,
    uint reserveIn,
    uint reserveOut
) internal pure returns (uint amountOut) {
    require(amountIn > 0, 'UniswapV2Library: INSUFFICIENT_INPUT_AMOUNT');
    require(reserveIn > 0 && reserveOut > 0, 'UniswapV2Library: INSUFFICIENT_LIQUIDITY');

    // 数学推导：
    // 恒定乘积：reserveIn * reserveOut = (reserveIn + amountIn_after_fee) * (reserveOut - amountOut)
    // amountIn_after_fee = amountIn * 997 / 1000（扣除 0.3% 手续费）
    //
    // 展开：
    //   reserveIn * reserveOut = (reserveIn + amountIn * 997/1000) * (reserveOut - amountOut)
    //   reserveOut - amountOut = reserveIn * reserveOut / (reserveIn + amountIn * 997/1000)
    //   amountOut = reserveOut - reserveIn * reserveOut / (reserveIn + amountIn * 997/1000)
    //             = reserveOut * amountIn * 997/1000 / (reserveIn + amountIn * 997/1000)
    //
    // 乘以 1000/1000 消除小数：
    //   amountOut = reserveOut * amountIn * 997 / (reserveIn * 1000 + amountIn * 997)

    uint amountInWithFee = amountIn.mul(997);  // 扣除 0.3% 手续费
    uint numerator = amountInWithFee.mul(reserveOut);
    uint denominator = reserveIn.mul(1000).add(amountInWithFee);
    amountOut = numerator / denominator;        // 整数除法（向下截断）
}

// 反向计算：已知输出量，求需要输入多少
function getAmountIn(
    uint amountOut,
    uint reserveIn,
    uint reserveOut
) internal pure returns (uint amountIn) {
    require(amountOut > 0, 'UniswapV2Library: INSUFFICIENT_OUTPUT_AMOUNT');
    require(reserveIn > 0 && reserveOut > 0, 'UniswapV2Library: INSUFFICIENT_LIQUIDITY');

    // 反解 amountIn：
    //   amountOut = reserveOut * amountIn * 997 / (reserveIn * 1000 + amountIn * 997)
    //   amountOut * (reserveIn * 1000 + amountIn * 997) = reserveOut * amountIn * 997
    //   amountOut * reserveIn * 1000 = amountIn * (reserveOut * 997 - amountOut * 997)
    //   amountIn = amountOut * reserveIn * 1000 / ((reserveOut - amountOut) * 997)
    //
    // 注意：向上取整（+1），确保用户不会因整数截断而少付

    uint numerator = reserveIn.mul(amountOut).mul(1000);
    uint denominator = reserveOut.sub(amountOut).mul(997);
    amountIn = (numerator / denominator).add(1);  // 向上取整
}
```

---

## Uniswap V3

> 发布时间：2021 年 5 月 | 核心贡献：集中流动性、多档手续费、改进 Oracle

---

### 3.1 核心数学模型：集中流动性

#### 从全范围到区间流动性

V2 中，LP 的流动性均匀分布在 $(0, +\infty)$，这等价于在每个可能的价格点都提供了少量流动性。

V3 中，LP 将全部流动性集中在 $[P_{lower}, P_{upper}]$ 区间内。

**V3 核心公式：**

$$L = \frac{\Delta y}{\Delta \sqrt{P}} = \frac{\Delta x}{\Delta(1/\sqrt{P})}$$

其中 $L$ 是"流动性"（一个常数，代表提供流动性的密度）。

**推导两种资产数量：**

当当前价格 $P_c$ 在区间 $[P_l, P_u]$ 内时：

$$x = L \cdot \left(\frac{1}{\sqrt{P_c}} - \frac{1}{\sqrt{P_u}}\right)$$

$$y = L \cdot \left(\sqrt{P_c} - \sqrt{P_l}\right)$$

当 $P_c < P_l$（价格低于区间下限）：持仓全为 token0（x），相当于已全部卖出 token1：

$$x = L \cdot \left(\frac{1}{\sqrt{P_l}} - \frac{1}{\sqrt{P_u}}\right), \quad y = 0$$

当 $P_c > P_u$（价格高于区间上限）：持仓全为 token1（y），相当于已全部卖出 token0：

$$x = 0, \quad y = L \cdot \left(\sqrt{P_u} - \sqrt{P_l}\right)$$

**代码中的体现（SqrtPriceMath.sol）：**

```solidity
// SqrtPriceMath.sol
// 计算从 sqrtRatioA 到 sqrtRatioB 价格变化时，token0 数量的变化

function getAmount0Delta(
    uint160 sqrtRatioAX96,   // sqrt(P_lower) * 2^96
    uint160 sqrtRatioBX96,   // sqrt(P_upper) * 2^96
    uint128 liquidity,        // L（流动性）
    bool roundUp
) internal pure returns (uint256 amount0) {
    if (sqrtRatioAX96 > sqrtRatioBX96)
        (sqrtRatioAX96, sqrtRatioBX96) = (sqrtRatioBX96, sqrtRatioAX96);

    uint256 numerator1 = uint256(liquidity) << FixedPoint96.RESOLUTION; // L * 2^96
    uint256 numerator2 = sqrtRatioBX96 - sqrtRatioAX96;                 // Δ(sqrt(P))

    // 数学：Δx = L * (1/sqrt(P_a) - 1/sqrt(P_b))
    //          = L * (sqrt(P_b) - sqrt(P_a)) / (sqrt(P_a) * sqrt(P_b))
    //
    // 用 Q96 定点数表示：
    //   Δx = (L * 2^96) * (sqrt(P_b) - sqrt(P_a)) / (sqrt(P_a) * sqrt(P_b) / 2^96)
    return roundUp
        ? UnsafeMath.divRoundingUp(
            FullMath.mulDivRoundingUp(numerator1, numerator2, sqrtRatioBX96),
            sqrtRatioAX96
        )
        : FullMath.mulDiv(numerator1, numerator2, sqrtRatioBX96) / sqrtRatioAX96;
}

// 计算 token1 数量的变化
function getAmount1Delta(
    uint160 sqrtRatioAX96,
    uint160 sqrtRatioBX96,
    uint128 liquidity,
    bool roundUp
) internal pure returns (uint256 amount1) {
    if (sqrtRatioAX96 > sqrtRatioBX96)
        (sqrtRatioAX96, sqrtRatioBX96) = (sqrtRatioBX96, sqrtRatioAX96);

    // 数学：Δy = L * (sqrt(P_b) - sqrt(P_a))
    return roundUp
        ? FullMath.mulDivRoundingUp(liquidity, sqrtRatioBX96 - sqrtRatioAX96, FixedPoint96.Q96)
        : FullMath.mulDiv(liquidity, sqrtRatioBX96 - sqrtRatioAX96, FixedPoint96.Q96);
}
```

---

### 3.2 sqrtPriceX96 定点数格式

#### 为什么存 sqrt(price) 而非 price

集中流动性的所有计算（添加流动性、swap 输出量）都涉及 $\sqrt{P}$，如果存 $P$ 则每次操作都要开方。存 $\sqrt{P}$ 直接用于计算，节省大量 gas。

**Q64.96 格式定义：**

$$\text{sqrtPriceX96} = \sqrt{P} \cdot 2^{96}$$

```
格式示意：
  uint160（总 160 位）
  整数部分：高 64 位（最大值 2^64 ≈ 1.8 × 10^19）
  小数部分：低 96 位（精度 2^{-96} ≈ 1.3 × 10^{-29}）

从 sqrtPriceX96 转换为实际价格：
  price = (sqrtPriceX96 / 2^96)^2
        = sqrtPriceX96^2 / 2^192

示例：ETH 价格 = 2000 USDC
  sqrt(2000) ≈ 44.721
  sqrtPriceX96 = 44.721 * 2^96
               ≈ 3,543,191,142,285,914,205,700,096
```

**代码：sqrtPriceX96 的使用（UniswapV3Pool.sol swap 片段）：**

```solidity
// UniswapV3Pool.sol 内部 swap 状态结构
struct SwapState {
    int256 amountSpecifiedRemaining;  // 还需处理的 token 数量
    int256 amountCalculated;           // 已计算的输出量
    uint160 sqrtPriceX96;              // 当前价格（sqrt 格式）
    int24 tick;                        // 当前 tick
    uint128 liquidity;                 // 当前有效流动性
    // 手续费相关字段省略...
}

// swap 主循环（简化版）
while (state.amountSpecifiedRemaining != 0 && state.sqrtPriceX96 != sqrtPriceLimitX96) {
    // 找到下一个有流动性的 tick 边界
    (step.tickNext, step.initialized) = tickBitmap.nextInitializedTickWithinOneWord(
        state.tick, tickSpacing, zeroForOne
    );

    // 计算本 tick 区间内 swap 能完成多少
    (state.sqrtPriceX96, step.amountIn, step.amountOut, step.feeAmount) =
        SwapMath.computeSwapStep(
            state.sqrtPriceX96,   // 当前价格
            step.sqrtPriceNextX96, // 下一个 tick 对应价格
            state.liquidity,
            state.amountSpecifiedRemaining,
            fee                   // 手续费档位
        );

    // 更新剩余数量
    state.amountSpecifiedRemaining -= (step.amountIn + step.feeAmount).toInt256();
    state.amountCalculated = state.amountCalculated.sub(step.amountOut.toInt256());

    // 如果价格越过了 tick 边界，更新流动性（穿越 tick）
    if (state.sqrtPriceX96 == step.sqrtPriceNextX96) {
        if (step.initialized) {
            int128 liquidityNet = ticks.cross(step.tickNext, ...);
            // 穿越 tick 时：zeroForOne 方向流动性减少，反之增加
            if (zeroForOne) liquidityNet = -liquidityNet;
            state.liquidity = LiquidityMath.addDelta(state.liquidity, liquidityNet);
        }
        state.tick = zeroForOne ? step.tickNext - 1 : step.tickNext;
    }
}
```

---

### 3.3 Tick 离散化数学

#### 价格 ↔ Tick 转换

$$P(\text{tick}) = 1.0001^{\text{tick}}$$

$$\text{tick} = \lfloor \log_{1.0001} P \rfloor = \lfloor \frac{\ln P}{\ln 1.0001} \rfloor$$

**为什么选 1.0001 作为公比？**

```
相邻两个 tick 的价格差：
  P(i+1) / P(i) = 1.0001^{i+1} / 1.0001^i = 1.0001

即相邻 tick 价格相差 0.01%（1 个基点）
这是金融领域最小的有意义价格单位
足够精细，同时不会产生过多 tick 数量
```

**代码：Tick 到 sqrtPrice 的转换（TickMath.sol）：**

```solidity
// TickMath.sol（简化版，实际使用二进制分解优化）
// 将 tick 转换为 sqrtPriceX96
function getSqrtRatioAtTick(int24 tick) internal pure returns (uint160 sqrtPriceX96) {
    // 数学：sqrtPriceX96 = sqrt(1.0001^tick) * 2^96
    //
    // 实际实现使用预计算的二进制分解：
    // 1.0001^tick = 1.0001^(bit_0 * 1 + bit_1 * 2 + bit_2 * 4 + ...)
    //            = 1.0001^bit_0 * 1.0001^{2*bit_1} * 1.0001^{4*bit_2} * ...
    // 每个因子预先计算好，只需要乘法，避免对数计算

    uint256 absTick = tick < 0 ? uint256(-int256(tick)) : uint256(int256(tick));

    // ratio 初始化为 1（Q128 格式）
    uint256 ratio = absTick & 0x1 != 0
        ? 0xfffcb933bd6fad37aa2d162d1a594001  // 1.0001^(-1) 的 Q128 表示
        : 0x100000000000000000000000000000000;  // 1.0

    // 对 tick 的每个二进制位，乘以对应的预计算因子
    if (absTick & 0x2 != 0) ratio = (ratio * 0xfff97272373d413259a46990580e213a) >> 128;
    if (absTick & 0x4 != 0) ratio = (ratio * 0xfff2e50f5f656932ef12357cf3c7fdcc) >> 128;
    // ... 继续处理每个二进制位（共 20 位）

    // 如果 tick 为正，取倒数
    if (tick > 0) ratio = type(uint256).max / ratio;

    // 从 Q128 转换到 Q96 格式（sqrtPriceX96）
    sqrtPriceX96 = uint160((ratio >> 32) + (ratio % (1 << 32) == 0 ? 0 : 1));
}

// 反向：从 sqrtPriceX96 计算对应的 tick
function getTickAtSqrtRatio(uint160 sqrtPriceX96) internal pure returns (int24 tick) {
    // 数学：tick = floor(log_{1.0001}(sqrtPriceX96 / 2^96)^2)
    //           = floor(2 * log_{1.0001}(sqrtPriceX96 / 2^96))
    //           = floor(log2(sqrtPriceX96 / 2^96)^2 / log2(1.0001))
    // 使用整数 log2 近似实现，避免浮点数
    // 具体实现用位操作计算整数部分 + 牛顿迭代校正小数部分
    // （省略 100+ 行实现）
}
```

---

### 3.4 Position 和手续费数学

#### 手续费全局增长追踪

V3 用"手续费增长"替代直接存储手续费数量，避免对每个 Position 单独更新：

```
feeGrowthGlobal0X128：token0 手续费总量 / 总流动性（全局累积）

每个 position 记录：
  feeGrowthInside0LastX128：上次收取时，区间内的手续费增长值

用户未收取的手续费 = (当前区间内增长 - 上次快照) * position.liquidity
```

**代码：Position 管理（UniswapV3Pool.sol 片段）：**

```solidity
// Position.sol
struct Info {
    uint128 liquidity;
    // 记录上次操作时，区间内的手续费增长累积值
    uint256 feeGrowthInside0LastX128;
    uint256 feeGrowthInside1LastX128;
    // 已累积但未提取的手续费
    uint128 tokensOwed0;
    uint128 tokensOwed1;
}

function update(
    Info storage self,
    int128 liquidityDelta,
    uint256 feeGrowthInside0X128,  // 当前区间内 token0 手续费增长
    uint256 feeGrowthInside1X128
) internal {
    Info memory _self = self;

    if (liquidityDelta != 0) {
        // 更新流动性
        self.liquidity = liquidityDelta < 0
            ? _self.liquidity - uint128(-liquidityDelta)
            : _self.liquidity + uint128(liquidityDelta);
    }

    // 计算新增手续费
    // 数学：新增 = (当前增长 - 上次快照) * 流动性 / 2^128
    // 使用溢出安全减法（手续费增长器允许溢出）
    self.tokensOwed0 += uint128(
        FullMath.mulDiv(
            feeGrowthInside0X128 - _self.feeGrowthInside0LastX128,
            _self.liquidity,
            FixedPoint128.Q128
        )
    );
    self.tokensOwed1 += uint128(
        FullMath.mulDiv(
            feeGrowthInside1X128 - _self.feeGrowthInside1LastX128,
            _self.liquidity,
            FixedPoint128.Q128
        )
    );

    // 更新快照
    self.feeGrowthInside0LastX128 = feeGrowthInside0X128;
    self.feeGrowthInside1LastX128 = feeGrowthInside1X128;
}
```

---

### 3.5 V3 无常损失数学（比 V2 更严重）

**V2 无常损失公式推导：**

设初始价格 $P_0$，变化后价格 $r = P_1/P_0$

LP 持仓价值 $V_{LP}$：

$$V_{LP} = 2\sqrt{r} \cdot V_0$$

持币不动价值 $V_{hold}$：

$$V_{hold} = (1 + r) \cdot \frac{V_0}{2} \cdot \frac{2}{1+r} = V_0 \cdot \frac{r + 1}{2} \cdot \frac{2\sqrt{r}}{\sqrt{r}} \cdot \frac{1}{...}$$

化简后：

$$\text{IL} = \frac{V_{LP}}{V_{hold}} - 1 = \frac{2\sqrt{r}}{1+r} - 1$$

**V3 集中流动性的无常损失放大：**

```
V2（全范围）无常损失：
  ETH 涨 2 倍：IL = 2*sqrt(2)/(1+2) - 1 ≈ -5.72%

V3（集中在 [1800, 2200]，当前价格 2000）：
  同样 ETH 涨 2 倍（4000），价格超出上限 2200
  整个持仓已全部变成 USDC（token0 全部被卖出）
  IL 远大于全范围情况，接近 100% 相对损失

结论：
  区间越窄，资本效率越高，但无常损失越剧烈
  价格超出区间时，LP 等同于在最高/最低点全部卖出
```

---

### 3.6 V3 Oracle：内置观测点数组

```solidity
// Oracle.sol
struct Observation {
    uint32 blockTimestamp;         // 观测时间戳
    int56 tickCumulative;          // 累积 tick（tick × 秒）
    uint160 secondsPerLiquidityCumulativeX128; // 每单位流动性累积秒数
    bool initialized;
}

// 环形缓冲区，最多 65535 个观测点
Observation[65535] public observations;

// 写入新观测点（每个区块最多一次）
function write(
    Observation[65535] storage self,
    uint16 index,          // 当前写入位置
    uint32 blockTimestamp,
    int24 tick,
    uint128 liquidity,
    uint16 cardinality,    // 当前有效观测点数量
    uint16 cardinalityNext // 下一个容量
) internal returns (uint16 indexUpdated, uint16 cardinalityUpdated) {
    Observation memory last = self[index];

    // 同一区块不重复写入
    if (last.blockTimestamp == blockTimestamp) return (index, cardinality);

    // 提升容量（如果 cardinalityNext > cardinality）
    if (cardinalityNext > cardinality && indexUpdated == (cardinality - 1)) {
        cardinalityUpdated = cardinalityNext;
    } else {
        cardinalityUpdated = cardinality;
    }

    // 环形缓冲区：写入位置 = (当前位置 + 1) % 容量
    indexUpdated = (index + 1) % cardinalityUpdated;

    self[indexUpdated] = Observation({
        blockTimestamp: blockTimestamp,
        // 数学：tickCumulative += tick * timeElapsed
        tickCumulative: last.tickCumulative +
            int56(tick) * int56(uint56(blockTimestamp - last.blockTimestamp)),
        secondsPerLiquidityCumulativeX128: last.secondsPerLiquidityCumulativeX128 +
            ((uint160(blockTimestamp - last.blockTimestamp) << 128) / (liquidity > 0 ? liquidity : 1)),
        initialized: true
    });
}

// 查询任意时间点的 TWAP（支持历史区间查询）
function observe(
    Observation[65535] storage self,
    uint32 time,           // 当前时间
    uint32[] memory secondsAgos,  // 要查询的时间点（相对于现在）
    int24 tick,
    uint16 index,
    uint128 liquidity,
    uint16 cardinality
) internal view returns (
    int56[] memory tickCumulatives,
    uint160[] memory secondsPerLiquidityCumulativeX128s
) {
    tickCumulatives = new int56[](secondsAgos.length);
    secondsPerLiquidityCumulativeX128s = new uint160[](secondsAgos.length);

    for (uint256 i = 0; i < secondsAgos.length; i++) {
        // 对每个查询时间点，在环形缓冲区中二分查找最近的观测点
        // 然后线性插值得到精确值
        (tickCumulatives[i], secondsPerLiquidityCumulativeX128s[i]) =
            observeSingle(self, time, secondsAgos[i], tick, index, liquidity, cardinality);
    }
}
```

**使用 V3 Oracle 的示例：**

```solidity
// 查询过去 30 分钟的 TWAP（tick 形式，需转换为价格）
function getTWAP(address pool, uint32 period) external view returns (int24 arithmeticMeanTick) {
    uint32[] memory secondsAgos = new uint32[](2);
    secondsAgos[0] = period;  // period 秒前
    secondsAgos[1] = 0;       // 现在

    (int56[] memory tickCumulatives,) = IUniswapV3Pool(pool).observe(secondsAgos);

    // TWAP tick = (累积值差) / 时间差
    arithmeticMeanTick = int24(
        (tickCumulatives[1] - tickCumulatives[0]) / int56(uint56(period))
    );

    // 将 tick 转换为价格：price = 1.0001^tick
    // uint160 sqrtPriceX96 = TickMath.getSqrtRatioAtTick(arithmeticMeanTick);
}
```

---

## Uniswap V4

> 发布时间：2024 年底 | 核心贡献：Hook 系统、单例合约、Flash Accounting、ERC-6909

---

### 4.1 核心架构：单例 PoolManager

#### V3 vs V4 合约架构对比

```
V3 架构（多合约）：
  每个交易对 = 独立合约
  ETH/USDC 0.05% → 合约地址 A
  ETH/USDC 0.30% → 合约地址 B
  ETH/USDC 1.00% → 合约地址 C
  每次跨池操作 → 多次 ERC20 transfer

  部署新池：合约部署，消耗 ~500 万 gas

V4 架构（单例）：
  所有池子 → 单一 PoolManager 合约
  不同池子用 PoolKey 区分
  资产统一在 PoolManager 内管理

  部署新池：SSTORE 写入，消耗 ~200 gas
```

**PoolKey 结构：**

```solidity
// IPoolManager.sol
struct PoolKey {
    Currency currency0;    // token0 地址（低地址）
    Currency currency1;    // token1 地址（高地址）
    uint24 fee;            // 手续费档位（0~1000000，百万分之几）
    int24 tickSpacing;     // tick 间隔
    IHooks hooks;          // Hook 合约地址（可为零地址）
}

// PoolId = keccak256(PoolKey)
type PoolId is bytes32;
function toId(PoolKey memory poolKey) internal pure returns (PoolId poolId) {
    assembly {
        // 直接对 PoolKey 的内存区域计算哈希
        poolId := keccak256(poolKey, 0xa0) // PoolKey 大小 5 个 slot = 160 字节
    }
}
```

---

### 4.2 Flash Accounting 数学模型

**核心数据结构：**

```solidity
// 使用 EIP-1153 瞬态存储（transient storage）
// 特点：交易内有效，交易结束后自动清零，比普通存储便宜 90%

// PoolManager.sol
mapping(address => int256) public transient currencyDelta;
// currencyDelta[token] > 0：PoolManager 欠用户 token
// currencyDelta[token] < 0：用户欠 PoolManager token
```

**Flash Accounting 执行流程：**

```solidity
// PoolManager.sol
function unlock(bytes calldata data) external returns (bytes memory result) {
    // 开启 Flash Accounting 上下文
    Lock.unlock();

    // 调用者（通常是 Router）在此回调中执行所有操作
    // 所有操作只记录 delta，不实际转账
    result = IUnlockCallback(msg.sender).unlockCallback(data);

    // 交易结束：所有 delta 必须为 0（已结清）
    // 否则 revert
    if (Lock.isUnlocked()) {
        // 验证所有 currency 的 delta 为 0
        revert CurrencyNotSettled();
    }
}

// 用户交互的 Router 合约示例
contract V4Router is IUnlockCallback {
    function swap(PoolKey calldata key, SwapParams calldata params) external {
        // 发起 Flash Accounting 上下文
        poolManager.unlock(abi.encode(key, params, msg.sender));
    }

    function unlockCallback(bytes calldata data) external returns (bytes memory) {
        (PoolKey memory key, SwapParams memory params, address user) =
            abi.decode(data, (PoolKey, SwapParams, address));

        // 执行 swap（只记录 delta，不转账）
        BalanceDelta delta = poolManager.swap(key, params, "");
        // delta.amount0() < 0：pool 收到 token0（用户要付）
        // delta.amount1() > 0：pool 要付 token1（用户要收）

        // 结算：实际完成转账，使 currencyDelta 归零
        if (delta.amount0() < 0) {
            // 用户付 token0
            key.currency0.transferFrom(user, address(poolManager), uint128(-delta.amount0()));
            poolManager.settle(key.currency0);
        }
        if (delta.amount1() > 0) {
            // 用户收 token1
            poolManager.take(key.currency1, user, uint128(delta.amount1()));
        }

        return "";
    }
}
```

**多跳交易 gas 节省分析：**

```
V3 三跳交易（DAI → USDC → ETH → WBTC）：
  ERC20.transferFrom(user → pool1, DAI)   → ~21,000 gas
  ERC20.transfer(pool1 → pool2, USDC)     → ~21,000 gas
  ERC20.transfer(pool2 → pool3, ETH)      → ~21,000 gas
  ERC20.transfer(pool3 → user, WBTC)      → ~21,000 gas
  合计 transfer gas：~84,000 gas

V4 三跳交易（相同路径）：
  记录 DAI delta（-1000）                  → ~200 gas（transient SSTORE）
  记录 USDC delta（+999/-998）自动抵消      → ~200 gas
  记录 ETH delta（+.../-...）自动抵消       → ~200 gas
  记录 WBTC delta（+x）                    → ~200 gas
  最终只需 2 次实际 transfer：
    DAI: user → PoolManager                → ~21,000 gas
    WBTC: PoolManager → user              → ~21,000 gas
  合计 transfer gas：~42,800 gas（节省约 50%）
```

---

### 4.3 Hook 系统完整实现

**Hook 权限通过合约地址编码：**

```solidity
// Hooks.sol
library Hooks {
    uint160 internal constant BEFORE_INITIALIZE_FLAG   = 1 << 13;
    uint160 internal constant AFTER_INITIALIZE_FLAG    = 1 << 12;
    uint160 internal constant BEFORE_ADD_LIQUIDITY_FLAG    = 1 << 11;
    uint160 internal constant AFTER_ADD_LIQUIDITY_FLAG     = 1 << 10;
    uint160 internal constant BEFORE_REMOVE_LIQUIDITY_FLAG = 1 << 9;
    uint160 internal constant AFTER_REMOVE_LIQUIDITY_FLAG  = 1 << 8;
    uint160 internal constant BEFORE_SWAP_FLAG         = 1 << 7;
    uint160 internal constant AFTER_SWAP_FLAG          = 1 << 6;
    uint160 internal constant BEFORE_DONATE_FLAG       = 1 << 5;
    uint160 internal constant AFTER_DONATE_FLAG        = 1 << 4;
    uint160 internal constant BEFORE_SWAP_RETURNS_DELTA_FLAG = 1 << 3;
    uint160 internal constant AFTER_SWAP_RETURNS_DELTA_FLAG  = 1 << 2;
    uint160 internal constant AFTER_ADD_LIQUIDITY_RETURNS_DELTA_FLAG    = 1 << 1;
    uint160 internal constant AFTER_REMOVE_LIQUIDITY_RETURNS_DELTA_FLAG = 1 << 0;

    // 验证 Hook 地址的最低位是否与声明的权限匹配
    function validateHookPermissions(
        IHooks self,
        Permissions memory permissions
    ) internal pure {
        if (
            permissions.beforeInitialize != self.hasPermission(BEFORE_INITIALIZE_FLAG) ||
            permissions.afterInitialize  != self.hasPermission(AFTER_INITIALIZE_FLAG)  ||
            // ... 验证所有权限位
            false
        ) revert HookAddressNotValid(address(self));
    }

    function hasPermission(IHooks self, uint160 flag) internal pure returns (bool) {
        return uint160(address(self)) & flag != 0;
    }
}
```

**动态手续费 Hook 完整示例：**

```solidity
// DynamicFeeHook.sol：根据波动率动态调整手续费
contract DynamicFeeHook is BaseHook {

    // 每个池子的手续费状态
    mapping(PoolId => uint24) public currentFee;
    mapping(PoolId => uint256) public lastUpdateTime;
    mapping(PoolId => uint256) public recentVolume;

    constructor(IPoolManager _poolManager) BaseHook(_poolManager) {}

    // 声明此 Hook 需要的权限
    function getHookPermissions() public pure override returns (Hooks.Permissions memory) {
        return Hooks.Permissions({
            beforeInitialize: false,
            afterInitialize: true,      // 初始化后设置初始费率
            beforeAddLiquidity: false,
            afterAddLiquidity: false,
            beforeRemoveLiquidity: false,
            afterRemoveLiquidity: false,
            beforeSwap: true,           // swap 前更新手续费
            afterSwap: true,            // swap 后记录交易量
            beforeDonate: false,
            afterDonate: false,
            beforeSwapReturnDelta: false,
            afterSwapReturnDelta: false,
            afterAddLiquidityReturnDelta: false,
            afterRemoveLiquidityReturnDelta: false
        });
    }

    // 池子初始化后：设置初始手续费
    function afterInitialize(
        address,
        PoolKey calldata key,
        uint160,
        int24
    ) external override onlyPoolManager returns (bytes4) {
        PoolId id = key.toId();
        currentFee[id] = 3000;  // 初始 0.3%
        lastUpdateTime[id] = block.timestamp;
        return BaseHook.afterInitialize.selector;
    }

    // swap 前：根据近期波动率动态调整手续费
    function beforeSwap(
        address,
        PoolKey calldata key,
        IPoolManager.SwapParams calldata,
        bytes calldata
    ) external override onlyPoolManager returns (bytes4, BeforeSwapDelta, uint24) {
        PoolId id = key.toId();

        // 计算近期波动率（简化版：用近期交易量代替）
        uint24 newFee = _calculateDynamicFee(id);
        currentFee[id] = newFee;

        // 返回新的手续费（覆盖 PoolKey 中的静态费率）
        return (BaseHook.beforeSwap.selector, BeforeSwapDeltaLibrary.ZERO_DELTA, newFee);
    }

    // swap 后：记录交易量用于下次手续费计算
    function afterSwap(
        address,
        PoolKey calldata key,
        IPoolManager.SwapParams calldata params,
        BalanceDelta delta,
        bytes calldata
    ) external override onlyPoolManager returns (bytes4, int128) {
        PoolId id = key.toId();

        // 记录交易量（用绝对值）
        int128 amount = params.zeroForOne ? delta.amount0() : delta.amount1();
        if (amount < 0) recentVolume[id] += uint256(uint128(-amount));

        return (BaseHook.afterSwap.selector, 0);
    }

    // 动态手续费计算逻辑
    function _calculateDynamicFee(PoolId id) internal view returns (uint24) {
        uint256 volume = recentVolume[id];
        uint256 elapsed = block.timestamp - lastUpdateTime[id];

        // 每小时交易量 > 1M USD → 高费率（补偿 LP 风险）
        // 交易量低时降低费率（吸引更多交易）
        if (elapsed > 0 && volume / elapsed > 1_000_000e18 / 3600) {
            return 5000;  // 高波动：0.5%
        } else if (volume / (elapsed + 1) < 10_000e18 / 3600) {
            return 500;   // 低波动：0.05%
        } else {
            return 3000;  // 默认：0.3%
        }
    }
}
```

**TWAMM Hook（时间加权平均做市商）概念示例：**

```solidity
// TWAMMHook.sol：把大额订单拆分成小单持续执行，减少价格冲击
contract TWAMMHook is BaseHook {

    struct Order {
        address owner;
        bool zeroForOne;      // 交易方向
        uint256 amountTotal;  // 总金额
        uint256 amountSold;   // 已执行金额
        uint256 startTime;
        uint256 endTime;      // 订单终止时间
    }

    mapping(PoolId => Order[]) public orders;

    // swap 前：先执行所有到期的 TWAMM 订单
    function beforeSwap(
        address,
        PoolKey calldata key,
        IPoolManager.SwapParams calldata,
        bytes calldata
    ) external override onlyPoolManager returns (bytes4, BeforeSwapDelta, uint24) {
        PoolId id = key.toId();

        // 执行所有待处理的 TWAMM 订单（时间加权拆分）
        _executeTWAMMOrders(key, id);

        return (BaseHook.beforeSwap.selector, BeforeSwapDeltaLibrary.ZERO_DELTA, 0);
    }

    function _executeTWAMMOrders(PoolKey calldata key, PoolId id) internal {
        Order[] storage poolOrders = orders[id];
        for (uint i = 0; i < poolOrders.length; i++) {
            Order storage order = poolOrders[i];
            if (order.amountSold >= order.amountTotal) continue;

            // 计算这个区块应该执行的部分
            // 数学：本次执行量 = 总量 * (当前时间 - 上次执行) / (结束时间 - 开始时间)
            uint256 elapsed = block.timestamp - order.startTime;
            uint256 duration = order.endTime - order.startTime;
            uint256 targetSold = order.amountTotal * elapsed / duration;
            uint256 toSell = targetSold - order.amountSold;

            if (toSell > 0) {
                // 执行这部分交易（调用 PoolManager.swap）
                _executePartialOrder(key, order, toSell);
                order.amountSold += toSell;
            }
        }
    }
}
```

---

### 4.4 ERC-6909 多代币标准

```solidity
// ERC6909.sol（V4 内部余额管理）
// 相比 ERC20：单一合约管理所有代币，省去每次 approve
// 相比 ERC1155：更简洁，无回调，更安全

contract ERC6909 {
    // tokenId → (owner → balance)
    mapping(uint256 => mapping(address => uint256)) public balanceOf;
    // tokenId → (owner → operator → allowance)
    mapping(uint256 => mapping(address => mapping(address => uint256))) public allowance;
    // owner → (operator → isApproved)
    mapping(address => mapping(address => bool)) public isOperator;

    function transfer(
        address receiver,
        uint256 id,     // token 地址转化为 uint256
        uint256 amount
    ) public returns (bool) {
        balanceOf[id][msg.sender] -= amount;
        balanceOf[id][receiver]   += amount;
        emit Transfer(msg.sender, msg.sender, receiver, id, amount);
        return true;
    }

    function transferFrom(
        address sender,
        address receiver,
        uint256 id,
        uint256 amount
    ) public returns (bool) {
        if (msg.sender != sender && !isOperator[sender][msg.sender]) {
            allowance[id][sender][msg.sender] -= amount;  // 检查并扣减 allowance
        }
        balanceOf[id][sender]   -= amount;
        balanceOf[id][receiver] += amount;
        return true;
    }
}

// V4 PoolManager 继承 ERC6909，把余额留存池内
// 用户选择不提取时，余额以 ERC-6909 形式留存
// 后续操作直接从内部余额扣减，无需外部 transfer
// 节省每次操作的 ~21,000 gas
```

---

## 版本演进全景对比

### 功能对比表

| 特性 | V1 | V2 | V3 | V4 |
|------|----|----|----|----|
| 交易对类型 | ETH/ERC20 | 任意 ERC20/ERC20 | 任意 ERC20/ERC20 | 任意（含原生 ETH）|
| 流动性分布 | 全范围均匀 | 全范围均匀 | LP 自定义区间 | LP 自定义区间 + Hook |
| 手续费 | 固定 0.3% | 固定 0.3% | 三档：0.01/0.05/0.3/1% | 完全自定义（动态）|
| LP Token 标准 | ERC20（Vyper） | ERC20 + Permit | ERC721 NFT | ERC-6909 |
| 价格预言机 | ❌ | TWAP（需外部快照）| TWAP（内置 65535 点）| TWAP + Hook 可扩展 |
| 闪电贷 | ❌ | Flash Swap | Flash Swap | Flash Accounting |
| 合约架构 | 单合约 | Core + Router | 多合约 | 单例 PoolManager |
| 自定义逻辑 | ❌ | ❌ | ❌ | ✅ Hook 系统 |
| Gas 效率 | 基准 | 中 | 中高 | 极高 |
| 原生 ETH | ✅ | ❌ WETH | ❌ WETH | ✅ |
| 部署新池 gas | ~500 万 | ~500 万 | ~500 万 | ~200 |
| 价格表示 | reserve 比值 | reserve 比值 | sqrtPriceX96 | sqrtPriceX96 |
| 流动性表示 | totalSupply | totalSupply | per-position L | per-position L |

---

## 数学模型汇总速查

### 1. 恒定乘积（所有版本）

$$x \cdot y = k$$

$$\Delta y = \frac{y \cdot \Delta x \cdot 997}{x \cdot 1000 + \Delta x \cdot 997} \quad \text{（含 0.3\% 手续费）}$$

**代码对应**：`UniswapV2Library.getAmountOut()`

---

### 2. LP Token 铸造（V1/V2）

$$\text{LP}_{初次} = \sqrt{x \cdot y} - \text{MINIMUM\_LIQUIDITY}$$

$$\text{LP}_{后续} = \min\left(\frac{\Delta x}{x}, \frac{\Delta y}{y}\right) \cdot \text{totalSupply}$$

**代码对应**：`UniswapV2Pair.mint()`

---

### 3. TWAP 预言机（V2/V3）

$$\text{priceCumulative}(t) = \sum_i P_i \cdot \Delta t_i$$

$$\text{TWAP}_{[t_1, t_2]} = \frac{\text{priceCumulative}(t_2) - \text{priceCumulative}(t_1)}{t_2 - t_1}$$

**代码对应**：`UniswapV2Pair._update()` / `Oracle.write()`

---

### 4. V3 集中流动性

$$L = \frac{\Delta y}{\Delta \sqrt{P}}$$

$$\Delta x = L \cdot \left(\frac{1}{\sqrt{P_c}} - \frac{1}{\sqrt{P_u}}\right), \quad \Delta y = L \cdot \left(\sqrt{P_c} - \sqrt{P_l}\right)$$

**代码对应**：`SqrtPriceMath.getAmount0Delta()` / `getAmount1Delta()`

---

### 5. Tick 与价格转换（V3/V4）

$$P(\text{tick}) = 1.0001^{\text{tick}}$$

$$\text{sqrtPriceX96} = \sqrt{P} \cdot 2^{96}$$

**代码对应**：`TickMath.getSqrtRatioAtTick()` / `getTickAtSqrtRatio()`

---

### 6. 无常损失（V2 全范围）

$$\text{IL} = \frac{2\sqrt{r}}{1+r} - 1, \quad r = \frac{P_1}{P_0}$$

| 价格变化 | 无常损失 |
|---------|---------|
| ±25% | -0.6% |
| ±50% | -2.0% |
| ×2 或 ÷2 | -5.7% |
| ×4 或 ÷4 | -20.0% |
| ×10 或 ÷10 | -42.5% |

**注**：V3 集中流动性区间越窄，损失越剧烈，区间完全偏移后损失可达 100%。

---

### 7. Flash Accounting delta 守恒（V4）

$$\sum_{\text{所有 token}} \text{delta}[\text{token}] = 0 \quad \text{（交易结束时）}$$

**代码对应**：`PoolManager.unlock()` 结束时的校验

---

*最后更新：2025 年 2 月*
*参考资料：Uniswap V1/V2/V3/V4 官方白皮书、源代码（MIT License）、以太坊 EIP-712、EIP-2612、EIP-6909*
