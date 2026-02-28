## UniswapV3白皮书 
关于集中流动性数学理解
核心还是 x*y=k， 虽然真实的提供的流动性是一个区间，但是会引入虚拟流动性来间接扩大真实流动性，从而增加资金利用率。

在区间边界处，一方的token会变为0.

### 公式推导

因为是区间，就有 高低点，我们这里简单成为 upper和lower

Xu * Yu = K

Xl * Yl = k

然后各自点的价格
Pu = Yu / Xu
Pl = Yl / Xl;

因为核心公式为 （x + Xu） * (y + Yl) = k;

所以推导
Xu = L / squrt(Pu);
Yl = L * squrt(Pl);

带入核心公式就是
(x + (L/squrt(Pu))) * (y + (L*squrt(Pl))) = L * L = k;

这里 x和y就是真实提供的币对

然后依据价格区间和提供的币对数量，就能算出L。 

在边界点的时候
(0 + (L/squrt(Pu))) * (ymax + (L*squrt(Pl))) = L * L
所以 Ymax = L * (squrt(Pu) - squrt(Pl));
同理 Xmax = L * (1/(squrt(Pl)) - 1 / (squrt(Pu)));

那么L就可以通过区间加提供的单一币种计算出来， 同样可以反推出另一个币需要多少。
L = Yreal / (squrt(Pc) - squrt(Pl)) = Xreal / (1/(squrt(Pc)) - 1 / (squrt(Pu))); 为什么是Pc，是因为当前你存的时候价格就是Pc

## 存取操作， 单区间
前置约定

token0 = ETH，数量用 a 表示

token1 = USDT，数量用 b 表示

Q   = USDT/ETH（习惯价格，每个 ETH 值多少 USDT）

Qc  = 2000（当前价格）

Ql  = 1500（区间下限）

Qh  = 2500（区间上限）


价格越高 → ETH 越贵 → 池子里 ETH 越少，USDT 越多

价格越低 → ETH 越便宜 → 池子里 ETH 越多，USDT 越少

核心公式来源

V3 把连续价格区间的流动性用积分推导出两个公式：

ETH 数量（从当前价格到上限，ETH 逐渐卖完）：

a=L * ((1 / squrt(Qc) − (1 / squrt(Qh)) ⋯(1)

USDT 数量（从下限到当前价格，USDT 逐渐积累）：

b=L * (squrt(Qc) - squrt(Ql))  ...(2)

关键：两个公式里 L 的数值相同，可以互推。

### 第一部分：存入 10 ETH，求需要多少 USDT

Step 1：计算各价格的平方根

squrt(Qc) = squrt(2000) = 44.7214

squrt(Qh) = squrt(2500) = 50.0000

squrt(Ql) = squrt(1500) = 38.7298

Step 2：用 ETH 数量反推 L

由公式 (1)

已知 a=10

知道Qc 和 Qh 求出L

L​=4236.5

Step 3：用 L 求 USDT 数量

由公式 (2)

知道L， Qc, Ql

b=25381 USDT​

### 第二部分：从池子买入 5 个 ETH，需要付多少 USDT

买入 ETH = 用户拿走 ETH，付出 USDT

池子变化：ETH 减少 5，USDT 增加

价格变化：ETH 变稀缺 → 价格上涨

Step 1：确认初始状态

a0=10,b0=25381,Q0=2000,L=4236.5

Step 2：求买入后新价格 Q₁

买入 5 ETH 后，池子 ETH 数量：a1=10−5=5

代入公式 (1)

已知 a1， L, Qh 求Q1

Q1 = 2229.2
​
价格从 2000 涨到 2229.2

Step 3：求新状态下 USDT 数量 b₁

代入公式 (2)

已知L, Q1, Ql

b1 = 35940 USDT

Step 4：计算付出的 USDT

Δb=b1−b0=35940−25381=10559 USDT​

### 第三部分：再卖出 5 个 ETH，收回多少 USDT

卖出 ETH = 用户付出 ETH，收到 USDT

当前状态：a₁ = 5 ETH，b₁ = 35940 USDT，Q₁ = 2229.2

池子变化：ETH 增加 5，USDT 减少

价格变化：ETH 变充裕 → 价格下跌

Step 1：求卖出后新价格 Q₂

卖出 5 ETH 后，池子 ETH：a2=5+5=10

a2​=5+5=10

代入公式 (1)

Q2=2000

Step 2：求新状态下 USDT 数量 b₂

b2=25381 USDT

Step 3：计算收到的 USDT

Δb=b1−b2=35940−25381=10559 USDT​

这里没有手续费的原因所以价格精确返回

### 跨区间交易
#### 先理解 TickBitmap

-887272 到 887272 共大概177w个

mapping(int16 => uint256) public bitmap;

Tick -887272 到 +887272，总共 1,774,545 个 Tick

每 256 个 Tick 打包进一个 uint256

共需要 ceil(1,774,545 / 256) = 6,932 个 uint256

worldposition/ bitposition

worldposition = tick >> 8

bitposition = tick % 256;

tickbitmap记录的是区间边界， 而不是区间内所有的tick。 

操作的时候， 生成掩码，然后按位或运算， 移除就按位与运算。


