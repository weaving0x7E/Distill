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
Uniswap V2的核心思想是池（pooling）：LP可以把流动性质押到合约中，质押后的流动性允许以任何人以去中心化的方式进行交易，类似于Uniswap V1交易者要支付一笔小费，它也会存入合约由LP共享。
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
如果你熟悉[[Uniswap V1]]，你也许记得我们实现了`addLiquidity`函数它计量了新加入的流动性并铸造LP-token。Uniswap V2在periphery合约中用`UniswapV2Router`实现了相同的功能。在pair合约中这些功能以一种更底层的方式被实现：流动性管理被简单的视为对LP-token的管理。当你向pair中增加流动性时合约会铸造LP-token。当你移除流动性时LP-token被销毁。就像之前谈论的那样core合约是一个底层合约它只负责最核心的操作。
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
在第一个分支中我们减去`MINIMUM_LIQUIDITY`（1000或1e<sup>-15</sup>)，当提供初始流动性时它避免小LP使得池的份额（1e<sup>-18</sup>, 1 wei）过贵，对于大多数池来说1000wei的LP-token可以忽略不计但如果有人想让池中每share（LP-token）的价格过贵那他必须支付1000倍的成本。
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
从池中移除流动性意味着销毁LP-token，以换取相应数量的底层token，返还的token数量计算方式如下：

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
当提供失衡的流动性后再移除会发生什么？
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
`Router`是一个顶层合约用作大多数用户应用的入口。这个合约使得创建pair，新增或者移除流动性，计算swap价格变化和执行swap变得更方便。`Router`也由factory部署，和pair一样这也是一个通用合约。
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