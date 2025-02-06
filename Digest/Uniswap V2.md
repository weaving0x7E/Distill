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
$$​​OR
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

## Conclusion
今天的内容就这些~ 希望我们这一部分的实现是足够清晰的，下一章我们会为合约增加更多的功能。