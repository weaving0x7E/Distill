译自：
* https://jeiwan.net/posts/programming-defi-uniswapv2-1/
* https://jeiwan.net/posts/programming-defi-uniswapv2-2/
* https://jeiwan.net/posts/programming-defi-uniswapv2-3/
* https://jeiwan.net/posts/programming-defi-uniswapv2-4/

Author: Ivan Kuznetsov 

Content of this article is licensed under a [Creative Commons Attribution-NonCommercial-ShareAlike 4.0 International License](http://creativecommons.org/licenses/by-nc-sa/4.0/)
# Part 1
## Introduction
Uniswap是一个运行于以太坊的去中心化交易所。它是完全自动化、不受人工干预、去中心化的。它经历了多次开发迭代：第一个版本于2018年十一月上线，第二个版本于2020年五月上线，第三个版本也是最后一个版本于2021年三月上线。
这篇文章将着重介绍Uniswap V2，和之前一样我们将从零开始写一个Uniswap V2并学习去中心化交易所的一些核心理念，但这次我们不会详细介绍恒定乘积公式和与其相关的核心机制，如果想了解那些内容可以阅读这篇[[Uniswap V1|文章]]。

## Tooling
本文中我们将使用Foundry作为开发与测试的工具。Foundry是一个由Georgios Konstantopoulos用Rust编写的现代以太坊工具包。它的运行速度远胜于Hardhat并且它允许使用Solidity来编写测试，相比于使用JS来编写测试这显然更加方便也清晰多了。
我们将使用[solmate](https://github.com/transmissions11/solmate)实现的ERC20而不是用Openzeppelin的因为它不允许将token转至零地址，而solmate没这些限制。
值得注意的是，2020年Uniswap V2发布以后，许多机制都发生了变化。例如Solidity 0.8引入溢出检查`SafeMath`就显得没那么重要了。所以我们要构建的是一个更现代化的Uniswap。

## Architecture of Uniswap V2
Uniswap V2的核心思想是池化（pooling）：LP可以把流动性质押到合约中，质押后的流动性允许以任何人以去中心化的方式进行交易，类似于Uniswap V1交易者要支付一笔小费，它也会存入合约由LP共享。
Uniswap V2中的核心合约是[UniswapV2Pair](https://github.com/Uniswap/v2-core/blob/master/contracts/UniswapV2Pair.sol)。“Pair”和“Pool”以一对同义的术语，它们都代表`UniswapV2Pair`合约。这个合约用于接收用户的token，并使用累积的token提供交换服务。这就是为什么称它为pool。每个`UniswapV2Pair`合约只能池化一对token，并允许在这两个token间互换，这就是为什么它也可以称为pair。
Uniswap V2的代码库被分成两个repo
* [core](https://github.com/Uniswap/v2-core)
* [periphery](https://github.com/Uniswap/v2-periphery)

core中包含这些合约：
1. `UniswapV2ERC20`: 为LP-tokens提供扩展的ERC20实现，它还实现了EIP-2612来支持离链转账
2. `UniswapV2Factory`: 类似于V1的factory合约它负责创建pair合约并注册。注册器使用`create2`来生成pair地址，我们接下来会展示相应实现细节
3. `UniswapV2Pair`: 负责核心逻辑的主合约。要注意的是factory不允许重复创建pair以免稀释流动性
periphery仓库包含多个合约，使Uniswap的使用更加方便。其中包括`UniswapV2Router`。它是Uniswap UI以及其他网络和在Uniswap上运行的去中心化应用的入口。它的接口和Uniswap V1中的exchange合约类似。
periphery中的另一个重要合约是`UniswapV2Library`，它是实现重要计算的辅助函数集合。我们将实现这两个合约。
我们将从core合约开始关注其最重要的机制。我们将会看到这些合约是非常普适的，它们的实现依赖一些底层结构来执行这使得受攻击的可能性下降，也让整个架构更加精细。

## Pooling liquidity
任何交易都需要流动性，因此我们首先实现的功能就是流动性池。
池合约存储着token的流动性并允许用这种流动性来进行swap。每一个合约都拥有自己的存储，ERC20 token自然也不例外，它们用`mapping`来存储地址和对应的余额。我们的池也有自己的ERC20余额，但这对于承担起提供流动性的功能来说还不够。
主要原因是只依赖ERC20会使得价格易被操纵：例如有人可以通过发送了大量的token到池中以进行不公平的swap，最后套现离场。为了避免这种情况我们需要跟踪池储备，同时控制它们何时被更新。
我们使用`reserve0`和`reserve1`两个变量来跟踪池的储备。
```solidity
contract ZuniswapV2Pair is ERC20, Math {
  ...

  uint256 private reserve0;
  uint256 private reserve1;

  ...
}
```
如果你熟悉[[Uniswap V1]]，你也许记得我们实现了`addLiquidity`函数它计量了新加入的流动性并铸造LP-token。Uniswap V2在periphery合约中用`UniswapV2Router`实现了相同的功能。在pair合约中这些功能以一种更底层的方式被实现：流动性管理被简单的视为对LP-token的管理。当你向pair中增加流动性时合约会铸造LP-token。当你收回流动性时LP-token被销毁。就像之前讨论的那样core合约是一个底层合约它只负责最核心的操作。
以下是存入流动性的底层函数：
```solidity
function mint() public {
   uint256 balance0 = IERC20(token0).balanceOf(address(this));
   uint256 balance1 = IERC20(token1).balanceOf(address(this));
   uint256 amount0 = balance0 - reserve0;
   uint256 amount1 = balance1 - reserve1;

   uint256 liquidity;

   if (totalSupply == 0) {
      liquidity = ???
      _mint(address(0), MINIMUM_LIQUIDITY);
   } else {
      liquidity = ???
   }

   if (liquidity <= 0) revert InsufficientLiquidityMinted();

   _mint(msg.sender, liquidity);

   _update(balance0, balance1);

   emit Mint(msg.sender, amount0, amount1);
}
```
我们首先计算新存入的金额然后计算需要发送给LP的LP-token数量最后发送token并更新储备(`_update`把balance更新到reserve变量中）。
从代码中可以看出池的状态不同时流动性的计算方法也是不同的(`totalSupply == 0`)。那么问题来了，当池中没有流动性时得铸造多少LP-token呢？在Uniswap V1中这取决于存入的ether数量，它使得初始LP-token数量取决于初始存入的资产比例，也就是说没有一个在一开始就可以纠偏价格的机制。此外Uniswap V2现在支持任意ERC20 pair，这意味着单纯的依赖ETH来计价已不可能。
对于初始的LP-token数量Uniswap V2使用存入资产的几何平均数来计算：

$$
Liquidity_{minted}=\sqrt{Amount_{0}​∗Amount_{1}}​​
$$

这样的好处在于初始的资产比例不会影响池中份额的价值。
现在让我们看看对于已有流动性的池LP-token该如何计算，显然这种算法要满足两个条件：
1. 与存入的资产成比例
2. 与已发行的LP-token成比例

回忆一下V1中的公式：

$$
Liquidity_{minted​}=TotalSupply_{LP}​∗\frac{Amount_{deposited}}{Reserve}​​
$$

新LPtoken的数量和存入的ether成比例但是在V2中有两个底层token，我们该把哪个用到公式中呢？
我们可以选择任意一个，一个值得注意的现象是：存入资产的比例与储备资产的比例越接近，所引起的价格变更就越小，因此如果存入的两种资产的比例失衡，那么根据某一资产算得LP-token的数量也会不同，如果选择占比大的这将通过提供流动性的方法激励价格变更从而导致可能的价格操纵，如果选择占比小的意味着惩罚失衡的流动性（LP将获得更少的LP-token）。显然选择占比小的资产有助于价格的公允性。
```solidity
if (totalSupply == 0) {
   liquidity = Math.sqrt(amount0 * amount1) - MINIMUM_LIQUIDITY;
   _mint(address(0), MINIMUM_LIQUIDITY);
} else {
   liquidity = Math.min(
      (amount0 * totalSupply) / _reserve0,
      (amount1 * totalSupply) / _reserve1
   );
}
```
在第一个分支中我们减去`MINIMUM_LIQUIDITY`（$1000$或$1e^{-15}$)，当提供初始流动性时它避免小LP使得池的份额（$1e^{-18}$, 1 wei）过贵，对于大多数池来说1000wei的LP-token可以忽略不计但如果有人想让池中每share（LP-token）的价格过贵那他必须支付1000倍的成本。
## Writing tests in Solidity
```solidity
contract ZuniswapV2PairTest is Test {
  ERC20Mintable token0;
  ERC20Mintable token1;
  ZuniswapV2Pair pair;

  function setUp() public {
    token0 = new ERC20Mintable("Token A", "TKNA");
    token1 = new ERC20Mintable("Token B", "TKNB");
    pair = new ZuniswapV2Pair(address(token0), address(token1));

    token0.mint(10 ether);
    token1.mint(10 ether);
  }

  // Any function starting with "test" is a test case.
}
```
初始化pair（提供流动性）
```solidity
function testMintBootstrap() public {
  token0.transfer(address(pair), 1 ether);
  token1.transfer(address(pair), 1 ether);

  pair.mint();

  assertEq(pair.balanceOf(address(this)), 1 ether - 1000);
  assertReserves(1 ether, 1 ether);
  assertEq(pair.totalSupply(), 1 ether);
}
```
1 ether的`token 0`和1 ether的`token 1`被放入test pool。结果是1 ether的LP-token被发行我们得到了1 ether - 1000。pool的储备和总供应也相应改变。
此时继续增加流动性会发生什么呢？
```solidity
function testMintWhenTheresLiquidity() public {
  token0.transfer(address(pair), 1 ether);
  token1.transfer(address(pair), 1 ether);

  pair.mint(); // + 1 LP

  token0.transfer(address(pair), 2 ether);
  token1.transfer(address(pair), 2 ether);

  pair.mint(); // + 2 LP

  assertEq(pair.balanceOf(address(this)), 3 ether - 1000);
  assertEq(pair.totalSupply(), 3 ether);
  assertReserves(3 ether, 3 ether);
}
```
目前一切正常，如果提供失衡的流动性呢？
```solidity
function testMintUnbalanced() public {
  token0.transfer(address(pair), 1 ether);
  token1.transfer(address(pair), 1 ether);

  pair.mint(); // + 1 LP
  assertEq(pair.balanceOf(address(this)), 1 ether - 1000);
  assertReserves(1 ether, 1 ether);

  token0.transfer(address(pair), 2 ether);
  token1.transfer(address(pair), 1 ether);

  pair.mint(); // + 1 LP
  assertEq(pair.balanceOf(address(this)), 2 ether - 1000);
  assertReserves(3 ether, 2 ether);
}
```
和之前谈到的一样即便提供多余的`token0`也只能得到1 LP-token
## Removing liquidity
从池中收回流动性意味着销毁LP-token，以换取相应数量的底层token，返还的token数量计算方式如下：

$$
Amount_{token}​=Reserve_{token}​∗\frac{​Balance_{LP}}{TotalSupply_{LP}}​​
$$

返还的token数量与持有的LP-token占比呈对应比例。持有的LP-token越多自然在销毁返还的token越多。
burn function：
```solidity
function burn(address to) public returns (uint256 amount0, uint256 amount1) {
    uint256 balance0 = IERC20(token0).balanceOf(address(this));
    uint256 balance1 = IERC20(token1).balanceOf(address(this));
    uint256 liquidity = balanceOf[address(this)];

    amount0 = (liquidity * balance0) / totalSupply;
    amount1 = (liquidity * balance1) / totalSupply;

    if (amount0 == 0 || amount1 == 0) revert InsufficientLiquidityBurned();

    _burn(address(this), liquidity);

    _safeTransfer(token0, to, amount0);
    _safeTransfer(token1, to, amount1);

    balance0 = IERC20(token0).balanceOf(address(this));
    balance1 = IERC20(token1).balanceOf(address(this));

    (uint112 reserve0_, uint112 reserve1_, ) = getReserves();
    _update(balance0, balance1, reserve0_, reserve1_);

    emit Burn(msg.sender, amount0, amount1, to);
}
```

```solidity
function testBurn() public {
  token0.transfer(address(pair), 1 ether);
  token1.transfer(address(pair), 1 ether);

  pair.mint();
  pair.burn();

  assertEq(pair.balanceOf(address(this)), 0);
  assertReserves(1000, 1000);
  assertEq(pair.totalSupply(), 1000);
  assertEq(token0.balanceOf(address(this)), 10 ether - 1000);
  assertEq(token1.balanceOf(address(this)), 10 ether - 1000);
}
```
可以看到池基本回到了初始状态，除了0地址里的那1000 wei
当提供失衡的流动性后再收回会发生什么？
```solidity
function testBurnUnbalanced() public {
  token0.transfer(address(pair), 1 ether);
  token1.transfer(address(pair), 1 ether);

  pair.mint();

  token0.transfer(address(pair), 2 ether);
  token1.transfer(address(pair), 1 ether);

  pair.mint(); // + 1 LP

  pair.burn();

  assertEq(pair.balanceOf(address(this)), 0);
  assertReserves(1500, 1000);
  assertEq(pair.totalSupply(), 1000);
  assertEq(token0.balanceOf(address(this)), 10 ether - 1500);
  assertEq(token1.balanceOf(address(this)), 10 ether - 1000);
}
```
我们损失了token0中的500 wei，这是对价格操纵的惩罚。不过这种惩罚看起来太轻微了，这是因为我们是目前唯一的LP如果我们向一个由其他用户初始化的池提供失衡流动性呢？
```solidity
function testBurnUnbalancedDifferentUsers() public {
  testUser.provideLiquidity(
    address(pair),
    address(token0),
    address(token1),
    1 ether,
    1 ether
  );

  assertEq(pair.balanceOf(address(this)), 0);
  assertEq(pair.balanceOf(address(testUser)), 1 ether - 1000);
  assertEq(pair.totalSupply(), 1 ether);

  token0.transfer(address(pair), 2 ether);
  token1.transfer(address(pair), 1 ether);

  pair.mint(); // + 1 LP

  assertEq(pair.balanceOf(address(this)), 1);

  pair.burn();

  assertEq(pair.balanceOf(address(this)), 0);
  assertReserves(1.5 ether, 1 ether);
  assertEq(pair.totalSupply(), 1 ether);
  assertEq(token0.balanceOf(address(this)), 10 ether - 0.5 ether);
  assertEq(token1.balanceOf(address(this)), 10 ether);
}
```
现在这会损失0.5 ether的token0，这占到我们存入金额的25%。同时推高了每一LP-token的内在价值让每个场内的LP受益。

# Part 2
今天我们要实现Uniswap V2的另一个核心功能——swapping。去中心化的进行token交换是Uniwsap要实现的目标，今天我们就要来看一看该如何实现这一点。我们还是在pair合约中进行开发这意味着我们的实现依旧是底层且精简的，没有方便的interface甚至没有价格计算功能。
此外们将实现一个价格预言机：pair合约的设计使得只要几行代码就实现这一功能。
最后我们将阐述pair合约背后的一些细节和思考。

## Tokens swapping
交换意味着用Token A交换Token B但我们还需要某种媒介它可以：
1. 提供实际汇率
2. 保证所有的交易是全额支付的，即所有的交易都在正确的汇率下进行
我们已经学过DEX的价格机制受池中流动性的影响，在[[Uniswap V1]]中我解释过恒定乘积公式和成功交换的条件，即储备的乘积在交换后需要和之前相同或者增加。无论池中的储备是多少，恒定乘积都必须保持不变这是我们的底线，这也令我们无需计算swap价格。
像我在介绍中提到的pair合约是核心合约，它必须尽可能的底层且精简。它也影响着我们如何向合约中发送token，转移token给某人有两种方式：
1. 通过token的`transfer`方法，指定接收方和要发送的金额
2. 通过`approve`方法允许其他用户或合约来接收你账户中的token到他的地址，另一种方式是调用`transferFrom`来获得你的token，由另一方为转账付费，你只需要批准数量。

这种审批模式在ETH应用中非常常见：为了避免一次又一次的调用`approve`，dapps往往向用户申请最大的token额度。但这个不是我们现在要关注的，现在我们依然采用手工转账到pair合约的方式。😂
这个函数接受两个输出金额，每token对应一个，代表调用方想要换得的token数量。之所以这样做是因为我们不想强制指定交换的方向，调用方可以指定其中一个或者两个的数量，我们只执行必要的检查。
```solidity
function swap(
    uint256 amount0Out,
    uint256 amount1Out,
    address to
) public {
    if (amount0Out == 0 && amount1Out == 0)
        revert InsufficientOutputAmount();

    ...
```
现在我们需要确保有足够的储备
```solidity
  ...

    (uint112 reserve0_, uint112 reserve1_, ) = getReserves();

    if (amount0Out > reserve0_ || amount1Out > reserve1_)
        revert InsufficientLiquidity();

    ...
```
接下来我们用余额减去待发送的token数量，此时我们假设调用方已经把他们想交易的token发到合约里了，所以token的余额要大于要交易的金额。
```solidity
   ...
    uint256 balance0 = IERC20(token0).balanceOf(address(this)) - amount0Out;
    uint256 balance1 = IERC20(token1).balanceOf(address(this)) - amount1Out;
    ...
```
接下来进行恒定乘积检查，我们期待合约token的余额和储备不同（余额后续会更新到储备中）并且我们需要确保它们的乘积大于等于当前的储备，则意味着：
1. 调用方正确的计算了汇率（包含滑点）
2. 输出金额是正确的
3. 被转入合约的金额是正确的
```solidity
    ...
    if (balance0 * balance1 < uint256(reserve0_) * uint256(reserve1_))
        revert InvalidK();
    ...
```
现在可以安全的转出token并更新储备了
```solidity
    _update(balance0, balance1, reserve0_, reserve1_);

    if (amount0Out > 0) _safeTransfer(token0, to, amount0Out);
    if (amount1Out > 0) _safeTransfer(token1, to, amount1Out);

    emit Swap(msg.sender, amount0Out, amount1Out, to);
}
```
	译者注：这一小节的表述可能有些模糊，因为原文中有些关键问题没做出解释。比如
	1.为什么`swap`接受两个参数`amount0Out`和`amount1Out`就保证了交易方向？
	我们假设现在想卖Token0以换取Token1，那么就有swap(0, amount1Out, to);也就是amount1Out=0意味着不取出Token0，此时合约会交易者池中的Token0数量进行检查，以换出对应的Token1。反之亦然。
	2.如何计算用户需要为此次swap支付的成本也即对应amountOut的amountIn？
	这里是把amountOut带入恒定乘积公式计算出的，在官方实现中这个过程的公式如下

$$
	amountIn=\frac{amountOut×reserveIn}{reserveOut−amountOut}​×\frac{1000}{997}​
$$
	
	也就是说交易者需要在调用`swap`前至少存入amountIn这么多的token
## Re-entrancy attacks and protection
可重入攻击是ETH智能合约中最常见的攻击方法，当合约发起外部调用但没做必要的检查或更新状态时可能会给攻击者留下机会。使得攻击者可以在合约处于错误的状态时再次进入合约导致资金损失。
在pair合约中`swap`调用`safeTransfer`来把token发送给调用方，可重入攻击正是针对这样的调用。它简单的假设调用`transfer`方法按照期待执行，但事实token合约没有被强制实现任何ERC20函数，它背后的开发者可以让它做任何事。
这有两个常用的防止可重入攻击的方法。
1. 使用re-entrancy guard
	具体的方法当函数被调用时设置一个标志位，当标志位存在时不允许调用该函数，当调用完成时取消标志位。这种机制不允许函数在调用过程中又被调用。
2. 遵循CEI原则
	这个原则强制执行时遵循一个严格的操作顺序：首先，所有必要的检查要在函数真正执行前进行，第二，函数根据其逻辑正常更新其状态。最后函数调用外部函数。这样操作顺序保证了每一个函数调用都是在函数状态最终确定且正确的状态下进行的。
	
我们的`swap`有漏洞吗？能否让它把所有的储备都发给调用方？理论上可以，因为它依赖第三方合约（token）任何一个token合约都可以向它提供错误的余额，以欺骗它将所有储备发送给调用者。当然如果token合约本身就是恶意的话，那可重入攻击相比之下就显得微不足道了。
## Price oracle
预言机是连接区块链和线下服务的桥梁，让智能合约可以查询真实世界的数据，这也想法已经存在了很长时间，Chainlink是最大的预言机网络之一，于2017年建立也是很多DeFi程序的关键依赖。
Uniswap既是一个链上应用也可以作为一个预言机。每个Uniswap pair合约的广泛使用也吸引着套利者通过其与交易所之间的差价来获利。套利者使得Uniswap的价格和中心化交易所的价格尽可能的接近，这也可以被看做中心化交易所的价格反馈给区块链。现在这个桥梁已经建立，来看看Uniswap V2是怎么利用这个现象的吧。
Uniswap V2中价格预言机提供的价格被称为时间加权平均价（time-weighted average price）或者TWAP。它提供两个时刻间的平均价格。为了做到这一点，合约存储了累积的价格：在每次swap之前它计算当前边际价格（不包含费用），乘以自上次交换以来的秒数，并将结果与之前的价格做和。
边际价格指的是两种资产的储备比值：

$$
price_0​=\frac{​reserve_1}{reserve_0}
$$

​​OR

$$
price_1​=\frac{​reserve_0}{reserve_1}
$$

对于预言机功能Uniswap V2使用的边际价格它不包含滑点和交换费用，也不依赖交换的数量。
由于Solidity不支持浮点数除法，计算这种价格有点麻烦：比如说储量比是$\frac{2}{3}$，那价格就是0。所以我们需要在计算边际价格时增加小数点，Uniswap V2使用[UQ112.112 number](https://en.wikipedia.org/wiki/Q_%28number_format%29)来处理这种情况。
UQ112.112简单讲就是用112个bit存储小数部分112个bit存储整数部分。之所以使用`uint112`是为了优化存储效率（这将在下一节详细介绍）。储备量被保存在UQ112.112的整数部分，这就是为什么要在计算价格前乘以`2**112`，具体细节可以看`UQ112x112.sol`。
为了让代码易读我们只新增一个状态变量：
```solidity
uint32 private blockTimestampLast;
```
它用来存储上一次swap（准确的说是储备更新）的时间戳。另外我们需要修改储备更新函数：
```solidity
function _update(
    uint256 balance0,
    uint256 balance1,
    uint112 reserve0_,
    uint112 reserve1_
) private {
    ...
    unchecked {
        uint32 timeElapsed = uint32(block.timestamp) - blockTimestampLast;

        if (timeElapsed > 0 && reserve0_ > 0 && reserve1_ > 0) {
            price0CumulativeLast +=
                uint256(UQ112x112.encode(reserve1_).uqdiv(reserve0_)) *
                timeElapsed;
            price1CumulativeLast +=
                uint256(UQ112x112.encode(reserve0_).uqdiv(reserve1_)) *
                timeElapsed;
        }
    }

    reserve0 = uint112(balance0);
    reserve1 = uint112(balance1);
    blockTimestampLast = uint32(block.timestamp);

    ...
}
```
`UQ112x112.encode`用`2**112`乘以`uint112`类型的值使得它变为`uint224`类型。然后它除以另一个资产的储备量再乘以`timeElapsed`所得的结果再加到现在的值中。关于`unchecked`中的内容我们稍后会再详细讨论。
## Storage optimization
`uint112`到底是什么类型？为什么不用`uint256`？——为了优化gas。
每个EVM操作消耗一定量的gas，简单操作比如算术运算消耗少量gas，IO操作比如`SSTORE`存储值到合约中和对应的取回操作`SLOAD`gas消耗是很大的。而`uuint112`正是为了降低IO消耗的gas而作。
先看一看变量的布局。
```solidity
address public token0;
address public token1;

uint112 private reserve0;
uint112 private reserve1;
uint32 private blockTimestampLast;

uint256 public price0CumulativeLast;
uint256 public price1CumulativeLast;
```
这一点很重要——它们必须完全按照这个顺序排列，因为每个状态变量对应一个特定的EVM存储槽（32 byte）。当你读取状态变量时实际是读的指针，每个`SLOAD`会取回32byte，`SSTORE`写入32byte，因为这些操作比较贵我们当然想减少IO操作量，而这正是对状态变量进行适当布局可能有所帮助的地方。
如果有几个连续的状态变量占用的空间少于32byte我们需要分别读取它们吗？其实不需要。EVM对小于32byte的邻接变量进行了打包。
详细解释一下这个变量布局：
1. 前两个是`address`类型。`address`占用20byte，两个就是40，这意味着它们位于不同的存储槽。
2. 两个`uint112`变量和一个`uint32`，也就是112+112+32=256，这意味着它们可以在一个存储槽中！这就是为什么选择`uint112`来作为记录储备量的类型，因为和储备相关的这几个变量都是一起被取回的，这当然相比于分别读取节约gas。这节约了一次`SLOAD`，对于储备这种常用的操作而言这意味着巨大的gas优化。
3. 最后是两个`uint256`它们自身没什么特殊的，这两个不能被一起打包。但是它在前一个完整的存储槽后面也占据着完整的存储槽，这避免之前的变量会错误i的打包进后面的存储槽，使得之前的优化白做。
## Integer overflow and underflow
现在再来审视`unchecked`中的代码。
另一个在合约中常见的漏洞是整数溢出或下溢。`uint256`的最大值是2<sup>256</sup>-1最小值是0.整数溢出意味着在`uint256`的最大值上继续增加导致它从0开始重新计数。

$$
uint256(2^{256}−1)+1=0
$$

类似的从零减去减一也会导致从最大值重新计数

$$
uint256(0)−1=2^{256}−1
$$

在Soldity 0.8以前不会对溢出或下溢进行检查，开发者一般使用SafeMath来应对这些问题，但现在Solidity原生支持在溢出或下溢时抛出异常。
0.8版本也引入了`uncheched`它意味着关闭对应代码的溢出/下溢检查。
我们在使用`timeElapsed`来计算累积价格时使用`unchecked`，这看起来似乎降低了安全性，但是即使我们的计算结果溢出时这也不会破坏合约的正常运行，所以在此处使用`unchecked`是安全的。
但我们现在的这种情况其实很罕见，在绝大多数情况下溢出/下溢检查不应被关闭。
## Safe transfer
你也许意识到在发送token时我们调用了一个陌生的方法。
```solidity
function _safeTransfer(
  address token,
  address to,
  uint256 value
) private {
  (bool success, bytes memory data) = token.call(
    abi.encodeWithSignature("transfer(address,uint256)", to, value)
  );
  if (!success || (data.length != 0 && !abi.decode(data, (bool))))
    revert TransferFailed();
}
```
为什么不直接使用`transfer`呢？
在pair合约中转移token时我们自然希望交易成功，但在`ERC20`中`transfer`方法返回的是一个布尔值，大多数token都正确的实现了这一方法，但是有些token的这个方法没有返回值。当然我们不能检查每一个token的合约也不能确保token转移事实上成功，但是我们至少可以检查转移结果。使得在转账失败的情况下revert。
`call`在这里是一个`address`[方法](https://docs.soliditylang.org/en/latest/types.html#members-of-addresses)这是一个底层函数它使得我们能更细粒度的控制合约调用。它使我们获得return的结果无论`transfer`是否返回恰当的值。
## Conclusion
今天的内容就这些~ 希望我们这一部分的实现是足够清晰的，下一章我们会为合约增加更多的功能。

# Part 3
今天我们要实现factory合约，它为所有pair合约提供一个注册器；此外还要开始实现一些顶层合约以降低用户使用时的心智成本。
## Factory contract
factory合约在Uniswap中起到重要作用，因为它避免了出现相同的pair而导致的流动性降低，同时提也提供了一套简单的pair部署机制。
Uniswap团队部署了一套factory合约作为pair的官方注册器，当然我们也可以绕开注册器直接手工部署pair合约。
```solidity
contract ZuniswapV2Factory {
    error IdenticalAddresses();
    error PairExists();
    error ZeroAddress();

    event PairCreated(
        address indexed token0,
        address indexed token1,
        address pair,
        uint256
    );

    mapping(address => mapping(address => address)) public pairs;
    address[] public allPairs;
...
```
factory合约非常简洁，它只在创建pair时触发PairCreated事件并存储以注册的pair。
而创建一个pair就显得有些复杂了：
```solidity
function createPair(address tokenA, address tokenB)
  public
  returns (address pair)
{
  if (tokenA == tokenB) revert IdenticalAddresses();

  (address token0, address token1) = tokenA < tokenB
    ? (tokenA, tokenB)
    : (tokenB, tokenA);

  if (token0 == address(0)) revert ZeroAddress();

  if (pairs[token0][token1] != address(0)) revert PairExists();

  bytes memory bytecode = type(ZuniswapV2Pair).creationCode;
  bytes32 salt = keccak256(abi.encodePacked(token0, token1));
  assembly {
    pair := create2(0, add(bytecode, 32), mload(bytecode), salt)
  }

  IZuniswapV2Pair(pair).initialize(token0, token1);

  pairs[token0][token1] = pair;
  pairs[token1][token0] = pair;
  allPairs.push(pair);

  emit PairCreated(token0, token1, pair, allPairs.length);
}
```
首先我们不允许pairs中存在相同的token，注意我们没有检查token合约是否真的存在，我们不关心这一点因为这取决于用户是否提供了合法的ERC20
接下来，我们对token地址进行排序以避免重复注册，用token地址来生成pair的地址。之后就是最需关注的部署pair了。
## Contracts deployment via CREATE2 opcode
在以太坊中，合约可以由合约部署。
在EVM中有两个opcode负责部署合约：
1. [CREATE](https://www.evm.codes/#f0)：这个opcode创建新账户（以太坊地址）并部署合约代码到对应地址。新地址基于部署者的合约nonce来生成，这和手工部署时的生成方法相同。nonce是账户的交易计数器，当发起交易时nonce会增加，而基于nonce来生成地址使得这一过程不具有确定性。
2. [CREATE2](https://www.evm.codes/#f5)：这个opcode在[EIP-1014](https://eips.ethereum.org/EIPS/eip-1014)中被添加。这个opcode很像`CREATE`但它允许生成具有确定地址的合约。部署者只需要关心要部署的合约代码，和提供一个生成地址的盐。
```solidity
...
bytes memory bytecode = type(ZuniswapV2Pair).creationCode;
bytes32 salt = keccak256(abi.encodePacked(token0, token1));
assembly {
    pair := create2(0, add(bytecode, 32), mload(bytecode), salt)
}
...
```
在第一行我们得到了`ZuniswapV2Pair`的字节码，这包括：
1. 不会上链的构造函数，这部分负责初始化智能合约并部署它
2. 运行时字节码，这包含合约的实际逻辑这一部分会上链
下一行通过hash pair的token地址来生成盐，由它来确定性的生成合约地址。
最后一行调用create2来：
1. 用`bytecode`+`salt`来创建新的确定性地址
2. 部署新的`ZuniswapV2Pair`合约
3. 得到pair地址
剩余的`createPair`应该就比较清晰了：
1. 初始化
```solidity
function initialize(address token0_, address token1_) public {
  if (token0 != address(0) || token1 != address(0))
    revert AlreadyInitialized();

  token0 = token0_;
  token1 = token1_;
}
```
2. 存储新的pair到存储在`pairs`和`allPairs`中
3. 发出`PairCreated`事件
## Router contract
现在是时候打开新篇章了，我们将要实现`Router`合约。
`Router`是一个顶层合约用作大多数用户应用的入口。这个合约使得创建pair，新增或者收回流动性，计算swap价格变化和执行swap变得更方便。`Router`也由factory部署，和pair一样这也是一个通用合约。
	这也是一个工程量比较大的合约，我们不实现它的所有功能，因为它们大多都是swap的变种。
和`Router`一起实现的还有`Library`合约它实现了所有基础且核心的函数，其中大部分是对swap数量进行的交换。
```solidity
contract ZuniswapV2Router {
    error InsufficientAAmount();
    error InsufficientBAmount();
    error SafeTransferFailed();

    IZuniswapV2Factory factory;

    constructor(address factoryAddress) {
        factory = IZuniswapV2Factory(factoryAddress);
    }
    ...
```
今天我们只实现流动性管理，下回再实现完整的合约。
```solidity
function addLiquidity(
    address tokenA,
    address tokenB,
    uint256 amountADesired,
    uint256 amountBDesired,
    uint256 amountAMin,
    uint256 amountBMin,
    address to
)
    public
    returns (
        uint256 amountA,
        uint256 amountB,
        uint256 liquidity
    )
    ...
```
1. `tokenA`、`tokenB`用于去寻找（或创造）我们想增加流动性的pair
2. `amountADesired`和`amountBDesired`是我们想存进pair的数量（上限）。
3. `amountAMin`、`amountBMin`是我们想存入的最小数量。还记得在注入失衡流动性时`pair`会发行更少数量的LP-token吗？而`min`参数正是用于指定我们愿意失去多少流动性。
4. `to`指向LP-token的接收地址。
```solidity
...
if (factory.pairs(tokenA, tokenB) == address(0)) {
    factory.createPair(tokenA, tokenB);
}
...
```
如果没有特定token的pair合约则会由`Router`来创建。
```solidity
...
(amountA, amountB) = _calculateLiquidity(
    tokenA,
    tokenB,
    amountADesired,
    amountBDesired,
    amountAMin,
    amountBMin
);
...
```
下一步，我们将计算将存入的金额。稍后我们将回到这个函数。
```solidity
...
address pairAddress = ZuniswapV2Library.pairFor(
    address(factory),
    tokenA,
    tokenB
);
_safeTransferFrom(tokenA, msg.sender, pairAddress, amountA);
_safeTransferFrom(tokenB, msg.sender, pairAddress, amountB);
liquidity = IZuniswapV2Pair(pairAddress).mint(to);
...
```
计算完流动性数量后我们终于可以从用户那里接收token并铸造LP-token作为交换。除了`pairFor`外代码里的大多数应该是比较熟悉的，这个函数我们会在实现`_calculateLiquidity`后再详细解释。另外要注意的是，该合约并不期望用户手动转移token——它使用ERC20 transferFrom函数从用户的余额中转移token。
```solidity
function _calculateLiquidity(
    address tokenA,
    address tokenB,
    uint256 amountADesired,
    uint256 amountBDesired,
    uint256 amountAMin,
    uint256 amountBMin
) internal returns (uint256 amountA, uint256 amountB) {
    (uint256 reserveA, uint256 reserveB) = ZuniswapV2Library.getReserves(
        address(factory),
        tokenA,
        tokenB
    );

    ...
```
在这个函数中，我们想要找到符合我们期待的最小金额的流动性金额，由于我们在UI中选择流动性金额和交易的实际执行有一定的延迟，实际的储备比例可能发生变化，它会使得我们损失一些LP-token。通过选择期望最小金额，我们可以尽可能把损失降低、
首先用`library`来得到池的储备，通过这个我们能计算最佳的流动性数量
```solidity
...
if (reserveA == 0 && reserveB == 0) {
    (amountA, amountB) = (amountADesired, amountBDesired);
...
```
如果储备是空也就是说这就是一个新的pair。那么我们的流动性将定义储备比例，这意味着我们不必因失衡流动性而被惩罚。因此我们允许存入期望的所有token
```solidity
...
} else {
    uint256 amountBOptimal = ZuniswapV2Library.quote(
        amountADesired,
        reserveA,
        reserveB
    );
    if (amountBOptimal <= amountBDesired) {
        if (amountBOptimal <= amountBMin) revert InsufficientBAmount();
        (amountA, amountB) = (amountADesired, amountBOptimal);
...
```
否则我们需要对token数量进行优化，`quote`是`library`提供的另一个函数：通过输入的量和pair储备来计算输出量即tokenA的价格乘以tokenB的输入量。
如果`amountBOptimal`小于等于我们期望的数量并且大于`amountBMin`那就采用。期望值和最小值的差异减轻滑点的影响。
然而`amountBOptimal`大于期望值时则不能采用，我们就要再从tokenA的角度再搜索一下，期待能找到合适的数量。
```solidity
...
} else {
    uint256 amountAOptimal = ZuniswapV2Library.quote(
        amountBDesired,
        reserveB,
        reserveA
    );
    assert(amountAOptimal <= amountADesired);

    if (amountAOptimal <= amountAMin) revert InsufficientAAmount();
    (amountA, amountB) = (amountAOptimal, amountBDesired);
}
```
使用相同的逻辑，我们可以找到`amountAOptimal`：它自然也必须在我们的最小期望范围内。
## Library contract
```solidity
library ZuniswapV2Library {
    error InsufficientAmount();
    error InsufficientLiquidity();

    function getReserves(
        address factoryAddress,
        address tokenA,
        address tokenB
    ) public returns (uint256 reserveA, uint256 reserveB) {
        (address token0, address token1) = _sortTokens(tokenA, tokenB);
        (uint256 reserve0, uint256 reserve1, ) = IZuniswapV2Pair(
            pairFor(factoryAddress, token0, token1)
        ).getReserves();
        (reserveA, reserveB) = tokenA == token0
            ? (reserve0, reserve1)
            : (reserve1, reserve0);
    }
    ...
```
这也是一个顶层函数，它可以获得任意pair的储备。
先对token地址进行排序这也是通过token找pair的必须步骤，再用factory来获得对应pair的储备。
```solidity
function pairFor(
    address factoryAddress,
    address tokenA,
    address tokenB
) internal pure returns (address pairAddress) {
```
这个函数被用于通过token和factory来找到pair地址。这个函数签名非常直球就不多解释了
```solidity
ZuniswapV2Factory(factoryAddress).pairs(address(token0), address(token1))
```
如果直接像上面的代码这样操作的话会产生外部调用，使运行的成本增加，所以Uniswap使用了一种更灵巧的方法，也就是利用`CREATE2`的机制直接得到pair地址省去一步查factory的调用。
```solidity
(address token0, address token1) = sortTokens(tokenA, tokenB);
pairAddress = address(
    uint160(
        uint256(
            keccak256(
                abi.encodePacked(
                    hex"ff",
                    factoryAddress,
                    keccak256(abi.encodePacked(token0, token1)),
                    keccak256(type(ZuniswapV2Pair).creationCode)
                )
            )
        )
    )
);
```
1. 先给token排序一下，还记得`createPair`函数吗？我们用排序后的token地址作为盐
2. 接下来构建字节序列 
* `0xff`用来避免和`CREATE`冲突
>引用自EIP-1014：`CREATE2`用endowment, memory_start, memory_length, salt这4个栈参数来执行，除了生成地址的方式为`keccak256( 0xff ++ address ++ salt ++ keccak256(init_code))[12:]`外其余行为与`CREATE(0xf0)`相同。
* `factoryAddress`用于部署pair的factory地址
* `salt`排序并哈希后的token地址
* `pair.createCode`的字节码哈希
3. 最后这个字节序列被`keccak256`哈希并转换成`address` (`bytes`->`uint256`->`uint160`->`address`)
终于到`quote`函数了
```solidity
function quote(
  uint256 amountIn,
  uint256 reserveIn,
  uint256 reserveOut
) public pure returns (uint256 amountOut) {
  if (amountIn == 0) revert InsufficientAmount();
  if (reserveIn == 0 || reserveOut == 0) revert InsufficientLiquidity();

  return (amountIn * reserveOut) / reserveIn;
}
```
如前所述，该函数根据输入量和pair的储备计算输出量。这样，我们就可以知道用特定数量的tokenA换取多少tokenB。这个方式仅用于流动性计算，在swap中还是使用恒定乘积公式。
今天就到这里！

# Part 4
终于到本文的最后一部分了，没错我们已经几乎从零实现了Uniswap V2，不过今天我们还要继续完善一些细节。
## LP-tokens burning bug
为了理解这个bug我们先看一下这个测试
```solidity
function testBurn() public {
    token0.transfer(address(pair), 1 ether);
    token1.transfer(address(pair), 1 ether);

    pair.mint(address(this));
    pair.burn();

    assertEq(pair.balanceOf(address(this)), 0);
}
```
当用户提供了流动性到pool时他们得到了LP-token作为回报。当流动性收回时LP-token用于换回流动性并销毁。在测试中你可以看到我们没有把LP-token放回池中，但是仍然能被销毁。如果仔细查看`burn`能发现这样的一行代码。
```solidity
_burn(msg.sender, liquidity);
```
合约在未经明确许可的前提下销毁sender的token！这显然不正确。用户需要一个方法来明确告知合约应当销毁多少token：
1. 让用户发送一定量的LP-token到合约中
2. 让合约只能销毁自身的LP-token
你可以通过这个[commit](https://github.com/Jeiwan/zuniswapv2/commit/babf8509b8be96796e2d944710bfcb22cc1fe77d#diff-835d3f34100b5508951336ba5a961932492eaa6923e3c5299f77007019bf2b6fR84)找到对应的更正实现
现在我们该实现`Router`中收回流动性的方法了。
## Liquidity removal
`Router`合约是一个顶层合约用于让Uniswap更易用。由此它的函数往往需要执行多个操作。现在我们需要这样一个函数：
1. 给用户提供用token操作的接口，而不是pair
2. 用户需要能指定转入pair合约的LP-token数量
3. 让LP能从pair中收回流动性
4. 收回流动性时减轻滑点的影响
```solidity
function removeLiquidity(
    address tokenA,
    address tokenB,
    uint256 liquidity,
    uint256 amountAMin,
    uint256 amountBMin,
    address to
) public returns (uint256 amountA, uint256 amountB) {
  ...
```
* `tokenA`, `tokenB`是pair中的token的地址，由于用户使用token操作自然无需pair的地址
* `liquidity`是要销毁的LP-token数量
* `amountAMin`, `amountBMin`是要收回的tokenA和tokenB的最低数量，这个参数用来避免滑点
* `to`接收token的地址
```solidity
address pair = ZuniswapV2Library.pairFor(
    address(factory),
    tokenA,
    tokenB
);
```
首先找到pair
```solidity
IZuniswapV2Pair(pair).transferFrom(msg.sender, pair, liquidity);
(amountA, amountB) = IZuniswapV2Pair(pair).burn(to);
```
再发送LP-token到pair并精确销毁这么多数量的token
```solidity
if (amountA < amountAMin) revert InsufficientAAmount();
if (amountB < amountBMin) revert InsufficientBAmount();
```
最后检查返回的token数量是否满足用户能接受的滑点范围
就这么简单~
## Output amount calculation
现在我们几乎可以实现高级swap了，包含chained swaping(通过TokenB来用TokenA换得TokenC)。在实现之前我们得先了解Uniswap如何计算输出量。首先看一下数量和价格的相关性。
什么是价格？朴素的定义是换得一单位某物所要付出的成本。在恒定乘积交易所中价格是储备之间的一种关系。我们在`quote`中实现了价格计算，然而当真正swap时这个价格是错误的，因为它只代表储备之间在那个时刻的关系。但当swap完成时储备会发生改变，我们真正期望的是在储备发生变化后价格会下降。
为了解释上面的结论，我们先回忆一下恒定乘积公式：

$$
x∗y=k
$$

$x$与$y$代表pair的储备（`reserve0`与`reserve1`）
当swap时$x$和$y$会改变但$k$恒定（由于交易费用的原因它实际上是缓慢增加的），我们可以写成另一个式子：

$$
(x+rΔx)(y−Δy)=xy
$$

`r=1-swap fee（1-0.3%=0.997`，$Δx$是为$Δy$付出的成本。
这个简洁的公式指明swap前后的储备乘积应当相同，我们也能借此计算Δy也就是我们能得到的token数量：

$$
Δy=\frac{yrΔx}{x+rΔx}​
$$

也就是说$Δy$受到用户的付出的$(rΔx)$影响。
现在来看看怎么实现吧
```solidity
function getAmountOut(
    uint256 amountIn,
    uint256 reserveIn,
    uint256 reserveOut
) public pure returns (uint256) {
  if (amountIn == 0) revert InsufficientAmount();
  if (reserveIn == 0 || reserveOut == 0) revert InsufficientLiquidity();
  ...
```
`amountIn`指代$Δx$, `reserveIn`指代$x$, `reserveOut`指代$y$
```solidity
uint256 amountInWithFee = amountIn * 997;
uint256 numerator = amountInWithFee * reserveOut;
uint256 denominator = (reserveIn * 1000) + amountInWithFee;

return numerator / denominator;
```
和以前一样还是把分母乘1000，分子乘(1000-3)，算得$Δy$
## swapExactTokensForTokens
```solidity
function swapExactTokensForTokens(
    uint256 amountIn,
    uint256 amountOutMin,
    address[] calldata path,
    address to
) public returns (uint256[] memory amounts) {
  ...
```
这个函数通过明确的成本(`amountIn`)来换得不少于`amountOutMin`的token。`path`则指定chained swap的swap顺序。
`path`参数也许看起来比较复杂，这是一个数组形式的token地址。如果我们想要用TokenA换TokenB，那`path`中自然只包含TokenA和TokenB的地址。如果我们像通过TokenB来用TokenA换TokenC，那`path`中则包含TokenA、TokenB、TokenC的地址，合约将会用TokenA换得TokenB然后再用TokenB换得TokenC，这个过程我们将会在接下来的测试中看到。
在这个函数中我们使用path预计算所有token的产生的交易量
```solidity
amounts = ZuniswapV2Library.getAmountsOut(
    address(factory),
    amountIn,
    path
);
```
`getAmountsOut`（注意这里是复数amounts）是一个我们还未实现的新函数，在这里我不会解释它的具体实现只对它做出简要定义，具体的实现你可以自行查阅对应代码。这个函数从path中提取pair并迭代的调用`getAmountOut`并为它们构建一个数组形式的输出量结果。

```solidity
if (amounts[amounts.length - 1] < amountOutMin)
    revert InsufficientOutputAmount();
```
在获得输出量后我们立刻可以验证最终输出量
```solidity
_safeTransferFrom(
    path[0],
    msg.sender,
    ZuniswapV2Library.pairFor(address(factory), path[0], path[1]),
    amounts[0]
);
```
如果最后的数量没问题，那合约将通过发送输入的token到第一个pair来启动swap
```solidity
_swap(amounts, path, to);
```
然后执行chained swap
```solidity
function _swap(
    uint256[] memory amounts,
    address[] memory path,
    address to_
) internal {
    for (uint256 i; i < path.length - 1; i++) {
      ...
```
开始迭代path进行交换
```solidity
(address input, address output) = (path[i], path[i + 1]);
(address token0, ) = ZuniswapV2Library.sortTokens(input, output);
```
从path中取回当前和下一个token地址并排序，之所以这么做是应为在pair中token按地址升序存储，但在path中这些token是按交易顺序排列的，输入token在前输出token在后，中间是零或多个媒介token。
```solidity
uint256 amountOut = amounts[i + 1];
(uint256 amount0Out, uint256 amount1Out) = input == token0
    ? (uint256(0), amountOut)
    : (amountOut, uint256(0));
```
如果`input==token0`则表示输入的是`token0`那自然输出的就是`token1`反之亦然。
处理完输出量后我们需要找到第一个swap目的地址，我们有两个选项：
1. 如果现在的pair不包含最后的path元素，那我们想直接发送token到下一个pair中以节约gas
2. 如果现在的pair包含最后的path元素，那我们要发送token到`to_`中。
```solidity
address to = i < path.length - 2
    ? ZuniswapV2Library.pairFor(
        address(factory),
        output,
        path[i + 2]
    )
    : to_;
```
在获得所有swap参数后开始实际执行swap
```solidity
IZuniswapV2Pair(
    ZuniswapV2Library.pairFor(address(factory), input, output)
).swap(amount0Out, amount1Out, to, "");
```
我们刚刚实现的就是Uniswap的核心功能！这其实并不是很艰难，对吧？
## swapTokensForExactTokens
原始的`Router`合约实现了[很多不同的swap方式](https://github.com/Uniswap/v2-periphery/blob/master/contracts/UniswapV2Router02.sol#L224-L400)，我们不打算把它们全实现一遍，我只想向你演示如何实现反向swap：用未知数量的输入token来获得特定数量的输出token。这是一个虽然不常见但很有趣的用例。
我们首先看一下swap公式：

$$
(x+rΔx)(y−Δy)=xy
$$

求解$Δx$的过程就是反向swap的过程

$$
Δx=\frac{xΔy}{(y−Δy)r}​
$$

根据这个公式写出代码
```solidity
function getAmountIn(
    uint256 amountOut,
    uint256 reserveIn,
    uint256 reserveOut
) public pure returns (uint256) {
    if (amountOut == 0) revert InsufficientAmount();
    if (reserveIn == 0 || reserveOut == 0) revert InsufficientLiquidity();

    uint256 numerator = reserveIn * amountOut * 1000;
    uint256 denominator = (reserveOut - amountOut) * 997;

    return (numerator / denominator) + 1;
}
```
一切看起来都比较熟悉，除了最后结果中的`+1`，这是由于在Solidity中的整数除法是向下取整的，我们希望计算出的输出量能达到要求的`amountOut`，如果最后结果被取整了，那输出量自然会略小一些。
接下来是`getAmountsIn`函数
```solidity
function getAmountsIn(
    address factory,
    uint256 amountOut,
    address[] memory path
) public returns (uint256[] memory) {
    if (path.length < 2) revert InvalidPath();
    uint256[] memory amounts = new uint256[](path.length);
    amounts[amounts.length - 1] = amountOut;

    for (uint256 i = path.length - 1; i > 0; i--) {
        (uint256 reserve0, uint256 reserve1) = getReserves(
            factory,
            path[i - 1],
            path[i]
        );
        amounts[i - 1] = getAmountIn(amounts[i], reserve0, reserve1);
    }

    return amounts;
}
```
相比于`getAmountsOut`这个函数的改动在于：path现在是反向遍历的。由于我们已知输出量想求解输入量，我们从path的末尾开始反向填充`amount`数组。
顶层的swap函数看起来也很熟悉：
```solidity
function swapTokensForExactTokens(
    uint256 amountOut,
    uint256 amountInMax,
    address[] calldata path,
    address to
) public returns (uint256[] memory amounts) {
    amounts = ZuniswapV2Library.getAmountsIn(
        address(factory),
        amountOut,
        path
    );
    if (amounts[amounts.length - 1] > amountInMax)
        revert ExcessiveInputAmount();
    _safeTransferFrom(
        path[0],
        msg.sender,
        ZuniswapV2Library.pairFor(address(factory), path[0], path[1]),
        amounts[0]
    );
    _swap(amounts, path, to);
}
```
这几乎与我们之前实现的那个相同，不过这个调用的是`getAmountsIn`。另一值得关注的是即使现在用的是输入量我们依然可以使用`_swap`。
## Fixing swap fee bug
在pair合约中暗藏了一个bug，现在来仔细检查这些代码：
```solidity
uint256 balance0 = IERC20(token0).balanceOf(address(this)) - amount0Out;
uint256 balance1 = IERC20(token1).balanceOf(address(this)) - amount1Out;

if (balance0 * balance1 < uint256(reserve0_) * uint256(reserve1_))
    revert InvalidK();
```
这个检查保障了swap的常量不被打破，但是它没考虑swap费用！😅
先来仔细分析一下这些代码吧：
1. 首先我们得到现在pair中的token余额
2. 从中减去输出量因为我们得把它们发给用户
3. 最终余额包含输入量（由用户提供）并减去了输出量。但没包含swap费用
4. 最后计算`k`是否因此而减小
对`k`的计算只考虑了输入输出量没考虑费用这显然不对。
为了修复这个问题我们得重写这个函数。
首先我们对储备进行预检查后，第一件事是把token转给用户。转账完成后，我们将计算输入金额：
```solidity
if (amount0Out > 0) _safeTransfer(token0, to, amount0Out);
if (amount1Out > 0) _safeTransfer(token1, to, amount1Out);

uint256 balance0 = IERC20(token0).balanceOf(address(this));
uint256 balance1 = IERC20(token1).balanceOf(address(this));

uint256 amount0In = balance0 > reserve0 - amount0Out
    ? balance0 - (reserve0 - amount0Out)
    : 0;
uint256 amount1In = balance1 > reserve1 - amount1Out
    ? balance1 - (reserve1 - amount1Out)
    : 0;

if (amount0In == 0 && amount1In == 0) revert InsufficientInputAmount();
```
为了方便描述，我们可以认为`reserve0`和`reserve1`是旧余额（swap之前合约的余额）。
当swap时我们通常提供`amount0Out`或`amount1Out`，由此产生`amount0In`或`amount1In`（另一个将为零）。但是，这部分代码（以及`swap`函数）允许我们同时设置 `amount0Out`和`amount1Out`，因此也有可能`amount0In`和`amount1In`都大于零。但如果它们都为零，则意味着用户没有向合约发送任何代币，这是不被允许的。
所以接下来的代码会找到新余额：它不包含输出量但包含输入量。
```solidity
uint256 balance0Adjusted = (balance0 * 1000) - (amount0In * 3);
uint256 balance1Adjusted = (balance1 * 1000) - (amount1In * 3);

if (
    balance0Adjusted * balance1Adjusted <
    uint256(reserve0_) * uint256(reserve1_) * (1000**2)
) revert InvalidK();
```
首先计算调整后的余额：当前余额减去输入量乘swap费用，然后为调整后的余额计算新的`k`，看`k`的变化是否符合规则。
```solidity
function testSwapUnpaidFee() public {
    token0.transfer(address(pair), 1 ether);
    token1.transfer(address(pair), 2 ether);
    pair.mint(address(this));

    token0.transfer(address(pair), 0.1 ether);

    vm.expectRevert(encodeError("InvalidK()"));
    pair.swap(0, 0.181322178776029827 ether, address(this), "");
}
```
## Flash loans
flash loans是一个有力的金融工具，在传统金融中我们看不到类似的东西。它是一种无限额且无抵押并必须在同一笔交易中偿还的贷款。Uniswap是提供这种贷款的平台之一，来看看我们怎么在我们的实现中增加flash loan功能。
flash loan按照以下规则运行：
1. 智能合约从其他合约中借出flash loan
2. 出借方合约发送token到期望借款的合约中并调用一个特殊函数
3. 在特殊函数中借方合约对贷款进行一些操作然后将贷款转回出借方
4. 出借方合约确保整个贷款完整归还、费用足额支付
5. 控制权移交给借方合约
为了实现flash loan我们得对`swap`做出一些改动：
```solidity
function swap(
    uint256 amount0Out,
    uint256 amount1Out,
    address to,
    bytes calldata data
) public {
```
新增一个byte数组类型的`data`参数。下一步是发行贷款：
```solidity
...
if (amount0Out > 0) _safeTransfer(token0, to, amount0Out);
if (amount1Out > 0) _safeTransfer(token1, to, amount1Out);
...
```
这意味着我们已经支付了由用户索要的任意金额，但没向用户索要任何抵押。我们要做出的唯一改变是让用户能归还贷款。我们通过调用一个用户合约的特殊函数完成这一点：
```solidity
...
if (amount0Out > 0) _safeTransfer(token0, to, amount0Out);
if (amount1Out > 0) _safeTransfer(token1, to, amount1Out);
if (data.length > 0) IZuniswapV2Callee(to).zuniswapV2Call(msg.sender, amount0Out, amount1Out, data);
...
```
根据约定，我们期望用户合约实现`zuniswapV2Call`，它接收`msg.sender, amount0Out, amount1Out, data`合约中的其余部分无需更改。
基本上搞定了，事实证明，我们已经实现了检查贷款是否偿还的逻辑，这与检查新k是否有效的逻辑是相同的！
现在测试一下flash loan我们希望整体流程在测试后可以变得更清晰。
如上所述为了使用flash loan我们需要写一个新合约（Flashloaner）。
```solidity
contract Flashloaner {
    error InsufficientFlashLoanAmount();

    uint256 expectedLoanAmount;

    ...
}
```
借出flash loan和进行swap一样简单
```solidity
function flashloan(
    address pairAddress,
    uint256 amount0Out,
    uint256 amount1Out,
    address tokenAddress
) public {
    if (amount0Out > 0) {
        expectedLoanAmount = amount0Out;
    }
    if (amount1Out > 0) {
        expectedLoanAmount = amount1Out;
    }

    ZuniswapV2Pair(pairAddress).swap(
        amount0Out,
        amount1Out,
        address(this),
        abi.encode(tokenAddress)
    );
}
```
在swap之前我们要设置`expectedLoanAmount`，这样我们就可以在稍后检查所要求的token是否已发放给我们。
在调用swap时可以看到我们把`tokenAddress`作为`data`参数。我们稍后会用这个来偿还贷款。另外我们应当把地址存储为状态变量。由于`data`是字节数组我们需要一种将地址转换为字节的方法，而abi.encode是一种常用的方法。
```solidity
function zuniswapV2Call(
    address sender,
    uint256 amount0Out,
    uint256 amount1Out,
    bytes calldata data
) public {
    address tokenAddress = abi.decode(data, (address));
    uint256 balance = ERC20(tokenAddress).balanceOf(address(this));

    if (balance < expectedLoanAmount) revert InsufficientFlashLoanAmount();

    ERC20(tokenAddress).transfer(msg.sender, balance);
}
```
这个函数将被pair合约在swap函数中调用，`zuniswapV2Call`中我们要确保确实获得了所请求的贷款并偿还。当然偿还前我们可以利用它来做一些事情，比如杠杆、套利或利用智能合约中的漏洞。flash loans是一个有力的工具我们可以用它来实现好的目的当然也能拿来作恶。
最后，让我们添加一个测试来获取贷款并确保正确的偿还：
```solidity
function testFlashloan() public {
    token0.transfer(address(pair), 1 ether);
    token1.transfer(address(pair), 2 ether);
    pair.mint(address(this));

    uint256 flashloanAmount = 0.1 ether;
    uint256 flashloanFee = (flashloanAmount * 1000) / 997 - flashloanAmount + 1;

    Flashloaner fl = new Flashloaner();

    token1.transfer(address(fl), flashloanFee);

    fl.flashloan(address(pair), 0, flashloanAmount, address(token1));

    assertEq(token1.balanceOf(address(fl)), 0);
    assertEq(token1.balanceOf(address(pair)), 2 ether + flashloanFee);
}
```
回想一下，我们没有实现任何额外的检查是否偿还了flash loan，我们只是使用了新的k来检查。同样的，当我们归还flash loan时，我们必须支付我们所取的金额+ 0.3%（实际上略高于这个数字：0.3009027%）。
`Flashloaner`计算``flashloanFee``并偿还了全部金额，在偿还后`flashloanFee`的余额是0，而pair合约得到了利息。
## Fixing re-entrancy vulnerability
最后的最后，随着pair合约的改进，我们引入了一个可重入漏洞。我们在此之前曾讨论过：当实现可以外部调用的函数时要非常谨慎可重入的问题，也讨论了CEI是避免被攻击的方法之一。然而在重写`swap`时我们不能使用这种模式因为这个实现迫使我们在外部调用（转移token）前应用状态变更（更新储备）。我们想要乐观转账并且想保持实现的简洁性！所以得运用一种新的保护手段。
当无法CEI时我们使用[Guard Check](https://fravoll.github.io/solidity-patterns/guard_check.html):我们可以简单的在swap被调用时增加一个标志位，当标志位存在时swap拒绝再次被调用。
首先添加一个`isEntered`来存储标志位：
```solidity
contract ZuniswapV2Pair is ERC20, Math {
    ...
    bool private isEntered;
    ...
}
```
这样做增加了gas消耗，这也是为什么更推荐CEI
接下来增加一个modifier：
```solidity
  modifier nonReentrant() {
      require(!isEntered);
      isEntered = true;

      _;

      isEntered = false;
  }
```
* 确保标志位不存在
* 添加标志位
* 执行函数体
* 完成后取消标志位
最后，我们需要将这个modifier应用到swap中
```solidity
function swap(
    uint256 amount0Out,
    uint256 amount1Out,
    address to,
    bytes calldata data
) public nonReentrant {
    ...
}
```
## Conclusion
我们的旅程到此结束。我衷心希望你享受这段旅程，并在其中学到很多东西。Uniswap V2 是一个集简洁、优雅和独特于一身的奇妙项目。它的代码是给我们的礼物，它让我们看到，一个真正的去中心化平台和一个完整的 DeFi 解决方案可以通过一套简单而优雅的智能合约来实现——这是每一个Solidity开发者的榜样！