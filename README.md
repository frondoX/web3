# web3
Some basic info about web3 like solidty, foundry, chainlink, golang in end back.....


1. solidty

   接收和发送eth
   receive和fallback 这两个是特殊函数，可以在接收到eth之后被触发。
   如果发送时候 msg.data是空的，且有receive函数，那么进receive函数，否则直接进fallback函数。
   发送函数 send，transfer和call。 send和transfer都是限定gas，在接收后无法做复杂任务。 调用方为接收地址。

   传入目标合约函数地址， 可强转位目标合约对象，进行目标合约对象的调用。
   function callSetX(address _Address, uint256 x) external{
    OtherContract(_Address).setX(x);
   }

   function callGetX(OtherContract _Address) external view returns(uint x){
    x = _Address.getX();
   }

   function callGetX2(address _Address) external view returns(uint x){
    OtherContract oc = OtherContract(_Address);
    x = oc.getX();
   }

   call方法
   目标合约地址.call(字节码);
   其中字节码利用结构化编码函数abi.encodeWithSignature获得：
   abi.encodeWithSignature("函数签名", 逗号分隔的具体参数)   
   函数签名为"函数名（逗号分隔的参数类型）"。例如abi.encodeWithSignature("f(uint256,address)", _x, _addr)。
   另外call在调用合约时可以指定交易发送的ETH数额和gas数额：
   目标合约地址.call{value:发送数额, gas:gas数额}(字节码);


   目标合约函数如果是payable 就可以在调用函数的时候发起转账。
   如果目标合约的函数是payable的，那么我们可以通过调用它来给合约转账：_Name(_Address).f{value: _Value}()，其中_Name是合约名，_Address是合约地址，f是目标函数名，_Value是要转的ETH数额（以wei为单位）。

   delegateCall, 和call类似， 唯一的区别就是delegate不能指定发送的eth。 delegate用来部署能够升级的合约。 代理合约只需要有逻辑，状态变量的变更依然在主合约中。 但要求代理合约和主合约的状态变量一致。

   智能合约创建智能合约， create和create2.
   create的用法很简单，就是new一个合约，并传入新合约构造函数所需的参数：
   Contract x = new Contract{value: _value}(params)
   新地址 = hash(创建者地址, nonce)

   create2 可以提前计算创建合约的地址， 避免后续跨合约调用获取地址等。
   0xFF：一个常数，避免和CREATE冲突
   CreatorAddress: 调用 CREATE2 的当前合约（创建合约）地址。
   salt（盐）：一个创建者指定的bytes32类型的值，它的主要目的是用来影响新创建的合约的地址。
   initcode: 新合约的初始字节码（合约的Creation Code和构造函数的参数）。
   新地址 = hash("0xFF",创建者地址, salt, initcode)

   Contract x = new Contract{salt: _salt, value: _value}(params)

   (address token0, address token1) = tokenA < tokenB ? (tokenA, tokenB) : (tokenB, tokenA); //将tokenA和tokenB按大小排序
   bytes32 salt = keccak256(abi.encodePacked(token0, token1));
        // 用create2部署新合约
   Pair pair = new Pair{salt: salt}();

   地址的计算方式：
   (address token0, address token1) = tokenA < tokenB ? (tokenA, tokenB) : (tokenB, tokenA); //将tokenA和tokenB按大小排序
    bytes32 salt = keccak256(abi.encodePacked(token0, token1));
    // 计算合约地址方法 hash()
    predictedAddress = address(uint160(uint(keccak256(abi.encodePacked(
        bytes1(0xff),
        address(this),
        salt,
        keccak256(type(Pair).creationCode)
        )))));
   如何构造函数存在参数，计算时，需要将参数和initcode一起进行打包： 加入有参数address：
   keccak256(abi.encodePacked(type(Pair).creationCode, abi.encode(address(this)))）

   合约销毁 selfdestruct(_addr); 目前只能将合约内的eth转出，并不会销毁合约，合约内的功能依然可运行。

   abi编码：
   4个方法：
   abi.encode, abi.encodePacked, abi.encodeWithSignature, abi.encodeWithSelector
   解码只有一个方法： abi.decode， 只能解码 encode编码的数据
   (dx, daddr, dname, darray) = abi.decode(data, (uint, address, string, uint[2]));

   用于call调用
   abi.encodeWithSignature("foo(uint256,address,string,uint256[2])", x, addr, name, array);
   abi.encodeWithSelector(bytes4(keccak256("foo(uint256,address,string,uint256[2])")), x, addr, name, array);

   Keccak256是solidty的hash函数

   调用只能合约，发送的calldata中前4个字节是selector（函数选择器）
   映射类型参数通常有：contract、enum、struct等。在计算method id时，需要将该类型转化成为ABI类型。
   contract为address， struct使用（将属性列出）， enum为uint8

   bytes4(keccak256("mappingParamSelector(address,(uint256,bytes),uint256[],uint8)"))

   trycatch：try-catch只能被用于external函数或public函数或创建合约时constructor（被视为external函数）的调用
   try externalContract.f() returns(returnType){
    // call成功的情况下 运行一些代码
   } catch Error(string memory /*reason*/) {
    // 捕获revert("reasonString") 和 require(false, "reasonString")
   } catch Panic(uint /*errorCode*/) {
    // 捕获Panic导致的错误 例如assert失败 溢出 除零 数组访问越界
   } catch (bytes memory /*lowLevelData*/) {
    // 如果发生了revert且上面2个异常类型匹配都失败了 会进入该分支
    // 例如revert() require(false) revert自定义类型的error
   }

   但是try-catch并不是万能的，有两种情况try-catch也无法捕获
      1. 函数返回值与期望的不一致
      2. 在try中调用的非合约地址的方法，即codesize为0

3. Defi项目
   1）稳定币类型
     1》基于法币锚定的稳定币 USDT USDC
     2〉基于质押锚定的稳定币 DAI Maker Dao
     3）基于算法的稳定币
   2）ERC4626
   3）去中心化交易所
      uniswap V1 V2 V3 V4
      V2的手续费， TWAP（基于时间的价格预言机）， FlashSwap， 无偿损失
     
