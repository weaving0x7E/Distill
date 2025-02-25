译自：
* https://jeiwan.net/posts/programming-defi-uniswap-1/
* https://jeiwan.net/posts/programming-defi-uniswap-2/
* https://jeiwan.net/posts/programming-defi-uniswap-3/

Author: Ivan Kuznetsov 

Content of this article is licensed under a [Creative Commons Attribution-NonCommercial-ShareAlike 4.0 International License](http://creativecommons.org/licenses/by-nc-sa/4.0/)

# Part 1
## Introduction
截至2021年6月共推出过三个版本的Uniswap。第一个版本于2018年推出允许换ether和其他token，还可以进行chained swap，以实现token与token之间的交换。V2于2020年三月推出相比于V1它允许ERC20之间直接交换，当然也允许用chained swap交换任意token。V3于2021年五月推出本次升级极大的提升了资本利用率它允许LP从资金池中取出更大比例的流动性，同时仍能获得同样的回报。在本系列中我们将深入每个版本并从零开始构建一个简单版本的Uniswap。

## What is Uniswap
简单的说Uniswap是一个去中心化交易所（DEX）它的目的是提供一个中心化交易所的替代方案。它在Ethereum上运行并且是完全依赖代码本身来运转的，没有任何人能在这套机制中有任何特权。
它底层依赖一套建立pool或token pairs的机制以为其提供流动性，使得用户可以利用这种流动性来交换token，这种算法被称为自动做市商(automated market maker)。
做市商是提供流动性的实体，有了流动性才有交易的可能，如果你想售出商品但没人买，那交易自然无法达成。有些交易对有高流动性比如BTC-USDT但是也有些币对没啥流动性。
一个DEX必须有足够的流动新以实现中心化交易所的职能，一个获得流动性的方法是把DEX自己的资金投入其中成为做市商。但这不太现实因为这得为每个币对提供巨量的流动性，此外这将让DEX变得中心化，如果市场上只有一个做市商那没人能保证它不作恶。
一个更好的方案是允许任意人成为做市商，这就是Uniswap在做的事，任何用户可以存入他们自己的资金到交易对中并从中获益。
Uniswap提供的另一个重要机制是price oracle，它从中心化交易所拉取交易对价格并提供给智能合约，这个价格一般来说无法被操纵因为中心化交易所的资金通常十分巨大。
Uniswap可以当作二级市场来通过不同交易所间的币对价差来套利，这也是一种自纠正机制能让DEX的价格和大交易所的价格不会有过大的背离。

## Constant product market maker
自动做市商是一个总称，包含了不同的去中心化做市商算法。最受欢迎的（和那些产生了这个词的）是与预测市场有关的——市场允许从预测中获利。Uniswap和其他链上交易所是这些算法的延续。
Uniswap的核心恒等式是

$$
x∗y=k
$$

$x$是ether储备$y$是token储备$k$是常量。Uniswap需要$k$保持恒定无论$x$与$y$的储备如何。当你用ether交易其他token时你，存入ether到合约中并得到一定数量的token。Uniswap确保在交换后$k$不变（但这也不绝对）。这个公式也负责价格计算，下文将详述这一机制。

## Token contract
```solidity
pragma solidity ^0.8.0;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";

contract Token is ERC20 {
  constructor(
    string memory name,
    string memory symbol,
    uint256 initialSupply
  ) ERC20(name, symbol) {
    _mint(msg.sender, initialSupply);
  }
}
```
## Exchange contract
Uniswap V1只有两个合约：Factory和Exchange。
Factory提供合约注册功能，用于创建exchange并持续追踪已部署的exchange，允许通过token地址搜索exchange地址，exchange合约通常定义了交换逻辑，每一个交易对由exchange合约部署。
下面是一个空合约
```solidity
pragma solidity ^0.8.0;

contract Exchange {}
```
由于每个exchange需要一种进行交易的token，所以我们把exchange和token地址进行绑定
```solidity
contract Exchange {
  address public tokenAddress;

  constructor(address _token) {
    require(_token != address(0), "invalid token address");

    tokenAddress = _token;
  }
}
```
token地址有public修饰，使得用户可以看到这个token和哪个exchange关联，constructor中我们检查提供的token是否合法（是否是零地址）。
### Providing liquidity
现在让我们来给exchange合约提供流动性：
```solidity
import "@openzeppelin/contracts/token/ERC20/IERC20.sol";

contract Exchange {
    ...

    function addLiquidity(uint256 _tokenAmount) public payable {
        IERC20 token = IERC20(tokenAddress);
        token.transferFrom(msg.sender, address(this), _tokenAmount);
    }
}
```
通常合约不能直接接收ether，所以得添加`payable`来修饰使得这个函数，任何通过这个函数发送的ether都将计入合约的余额中。
由于token余额存储在token合约中（ERC20）所以我们使用`transerFrom`函数来给转账，另外交易发起方需要调用`approve`函数以允许我们的exchange合约能获得他们的token。
顺手再写一个返回token余额的功能
```solidity
function getReserve() public view returns (uint256) {
  return IERC20(tokenAddress).balanceOf(address(this));
}
```
现在我们需要测试`addLiquidity`来确保一切正常。
```javascript
describe("addLiquidity", async () => {
  it("adds liquidity", async () => {
    await token.approve(exchange.address, toWei(200));
    await exchange.addLiquidity(toWei(200), { value: toWei(100) });

    expect(await getBalance(exchange.address)).to.equal(toWei(100));
    expect(await exchange.getReserve()).to.equal(toWei(200));
  });
});
```

### Pricing function
现在我们想一下如何计算swap价格，一个朴素的想法也许是这样直接把价格和储备挂钩

$$
P_X​=\frac{x}{y}​,P_Y​=\frac{y}{x}​
$$

这有点道理因为exchange合约不和中心化交易所或者任何外部价格预言机交互，因此它不可能知道正确的价格，所以exchange合约本身就应当是一个价格预言机。它只能利用ether和token储备，这是我们计算价格的唯一信息。
由此写一个价格函数吧：
```solidity
function getPrice(uint256 inputReserve, uint256 outputReserve)
  public
  pure
  returns (uint256)
{
  require(inputReserve > 0 && outputReserve > 0, "invalid reserves");

  return inputReserve / outputReserve;
}
```

```javascript
describe("getPrice", async () => {
  it("returns correct prices", async () => {
    await token.approve(exchange.address, toWei(2000));
    await exchange.addLiquidity(toWei(2000), { value: toWei(1000) });

    const tokenReserve = await exchange.getReserve();
    const etherReserve = await getBalance(exchange.address);

    // ETH per token
    expect(
      (await exchange.getPrice(etherReserve, tokenReserve)).toString()
    ).to.eq("0.5");

    // token per ETH
    expect(await exchange.getPrice(tokenReserve, etherReserve)).to.eq(2);
  });
});
```
我们存入2000 token和1000 ether，期待token的价格是0.5 ether，ether的价格是2 token。然而这个测试失败了，它显示从exchange中获得了0 ether，Why？
原因是solidity中没有小数0.5被近似成了0，让我们增加精度来修复这个问题吧：
```solidity
function getPrice(uint256 inputReserve, uint256 outputReserve)
  public
  pure
  returns (uint256)
{
    ...

  return (inputReserve * 1000) / outputReserve;
}
```

```javascript
// ETH per token
expect(await exchange.getPrice(etherReserve, tokenReserve)).to.eq(500);

// token per ETH
expect(await exchange.getPrice(tokenReserve, etherReserve)).to.eq(2000);
```
现在1 token等于0.5 ether而1 ether 等于2 token了。
看起来好像什么问题，但是如果我们用2000 token来换取ether呢？我们会得到1000 ether，这将用尽exchange合约里的全部储备。
显然这个价格函数有点古怪，它竟然允许用尽合约储备这是我们不想看到的。
原因在于价格函数遵从恒等式它定义了$k$是个$x$与$y$的常数和，这种函数是一条直线

![[Pasted image 20250127161859.png]]

在x和y轴的交点处意味着允许它们为0！我们显然不想这样。

### Correct pricing function
回忆一下Uniswap是一个恒定乘积做市商（constant product market maker）这意味着它基于恒定乘积公式：

$$
x∗y=k
$$

这个公式能产生更好价格函数吗？
公式表明无论储备是多少，`k`都应保持不变。每笔交易都会增减ether或token的储备——让我们把这个逻辑放这个公式中：

$$
(x+Δx)(y−Δy)=xy
$$

Δx、Δy代表原始token和换得token的数量，从上式可得

$$
Δy=\frac{yΔx}{x+Δx}​
$$

现在函数关注于数量而不是价格
```solidity
function getAmount(
  uint256 inputAmount,
  uint256 inputReserve,
  uint256 outputReserve
) private pure returns (uint256) {
  require(inputReserve > 0 && outputReserve > 0, "invalid reserves");

  return (inputAmount * outputReserve) / (inputReserve + inputAmount);
}
```

```solidity
function getTokenAmount(uint256 _ethSold) public view returns (uint256) {
  require(_ethSold > 0, "ethSold is too small");

  uint256 tokenReserve = getReserve();

  return getAmount(_ethSold, address(this).balance, tokenReserve);
}

function getEthAmount(uint256 _tokenSold) public view returns (uint256) {
  require(_tokenSold > 0, "tokenSold is too small");

  uint256 tokenReserve = getReserve();

  return getAmount(_tokenSold, tokenReserve, address(this).balance);
}
```

```javascript
describe("getTokenAmount", async () => {
  it("returns correct token amount", async () => {
    ... addLiquidity ...

    let tokensOut = await exchange.getTokenAmount(toWei(1));
    expect(fromWei(tokensOut)).to.equal("1.998001998001998001");
  });
});

describe("getEthAmount", async () => {
  it("returns correct eth amount", async () => {
    ... addLiquidity ...

    let ethOut = await exchange.getEthAmount(toWei(2));
    expect(fromWei(ethOut)).to.equal("0.999000999000999");
  });
});
```
现在1 ehter = 1.998 token，0.999 ether = 2 token 这些金额与之前的定价功能生成的金额非常接近。不过，它们略微小了一些。Why？
这次的价格计算基于的恒定乘积公式实际上是一条双曲线：

![[Pasted image 20250127163807.png]]

双曲线和x与y轴不会相交自然储备也不会是0！
不难看出价格函数会滑点，单笔交易越大滑点越明显。从测试中可以看到我们的交易所得比预期的要少一些，这似乎是恒定乘积做市商的一个缺点，但这个机制保护了交易池不会被耗尽，这也符合供需关系，越紧俏的资产价格越高。
再写一个测试看看滑点对价格的影响吧
```javascript
describe("getTokenAmount", async () => {
  it("returns correct token amount", async () => {
    ... addLiquidity ...

    let tokensOut = await exchange.getTokenAmount(toWei(1));
    expect(fromWei(tokensOut)).to.equal("1.998001998001998001");

    tokensOut = await exchange.getTokenAmount(toWei(100));
    expect(fromWei(tokensOut)).to.equal("181.818181818181818181");

    tokensOut = await exchange.getTokenAmount(toWei(1000));
    expect(fromWei(tokensOut)).to.equal("1000.0");
  });
});

describe("getEthAmount", async () => {
  it("returns correct ether amount", async () => {
    ... addLiquidity ...

    let ethOut = await exchange.getEthAmount(toWei(2));
    expect(fromWei(ethOut)).to.equal("0.999000999000999");

    ethOut = await exchange.getEthAmount(toWei(100));
    expect(fromWei(ethOut)).to.equal("47.619047619047619047");

    ethOut = await exchange.getEthAmount(toWei(2000));
    expect(fromWei(ethOut)).to.equal("500.0");
  });
});
```
如上码，当我们想耗尽流动性时我们只能得到预期的一半。
最后再想说的是：我们最初基于准备金率的定价函数没有错。事实上，当我们交易的token数量与储备相比非常小时，这是正确的。但要建立AMM机制，我们需要更复杂的东西。

### Swapping functions
```solidity
function ethToTokenSwap(uint256 _minTokens) public payable {
  uint256 tokenReserve = getReserve();
  uint256 tokensBought = getAmount(
    msg.value,
    address(this).balance - msg.value,
    tokenReserve
  );

  require(tokensBought >= _minTokens, "insufficient output amount");

  IERC20(tokenAddress).transfer(msg.sender, tokensBought);
}
```
`_minTokens`是用户能换得的最小数量token，这个数量在UI中计算，始终包含滑点容差；用户同意至少获得该金额，但不能少于该金额。这是一个非常重要的机制，可以保护用户免受MEV拦截交易并修改资金池余额以牟利。
终于写到今天最后的代码啦
```solidity
function tokenToEthSwap(uint256 _tokensSold, uint256 _minEth) public {
  uint256 tokenReserve = getReserve();
  uint256 ethBought = getAmount(
    _tokensSold,
    tokenReserve,
    address(this).balance
  );

  require(ethBought >= _minEth, "insufficient output amount");

  IERC20(tokenAddress).transferFrom(msg.sender, address(this), _tokensSold);
  payable(msg.sender).transfer(ethBought);
}
```
这个函数用于从用户余额中获得数量为`_tokenSold`的token并发送`ethBought`数量的ether

## Conclusion
今天就到这里！虽然还没结束，但我们已经做了很多。我们的exchange合约可以接受用户的流动性，以一种防止流动性枯竭的方式计算价格。但一些重要的机制还没建立：
* 增加新的流动性会导致巨大的价格变化
* 流动性提供者没有回报；所有交换都是免费的
* 无法收回流动性
* 无法交换 ERC20
* factory合约仍未实现

我们将在未来的部分中实现这些功能！

# Part 2
## Adding more liquidity
在之前的章节我们讨论的`addLiquidity`还没写完，这个函数目前看起来是这样的：
```solidity
function addLiquidity(uint256 _tokenAmount) public payable {
  IERC20 token = IERC20(tokenAddress);
  token.transferFrom(msg.sender, address(this), _tokenAmount);
}
```
你能看出它有什么问题吗？
这个函数允许在任何时刻提供任意数量的流动性，正如我们所熟知的那样价格是以资产比例来计算的：

$$
P_X​=\frac{x}{y}​,P_Y​=\frac{y}{x}​
$$

当x和y是它们的资产储备量​，$P_X​$和$P_Y$代表的是ether和token的价格。
我们知道交换token时的资产储备改变不是线性的，这影响了价格，套利者通过平衡价格使其与大型中心化交易所的价格相匹配来获利。
我们实现的问题在于它没有对价格改变的幅度进行限制，换句话说它没有强制让新的流动性也执行当前的资产比例，这将导致价格有被操纵的可能，我们希望的是去中心化交易所的价格能尽可能接近中心化交易所的价格，让exchange合约充当价格预言机的角色。
所以我们必须确保增加流动性时不破坏原有交易池的资产比例，同时我们想允许在交易池还没初始化也即时储备为零时提供任意比例的流动性，这是一个重要的时刻，因为需要确定价格的初始值。
现在`addLiquidity`将有两个职责：
* 如果是为新池增加流动性，允许以任意比例提供
* 后续为交易池提供流动性时强制以相同比例提供
对于第一个职责在原有的代码上加个`if`就行啦
```solidity
if (getReserve() == 0) {
    IERC20 token = IERC20(tokenAddress);
    token.transferFrom(msg.sender, address(this), _tokenAmount);
```
第二个职责的实现如下码所示
```solidity
} else {
    uint256 ethReserve = address(this).balance - msg.value;
    uint256 tokenReserve = getReserve();
    uint256 tokenAmount = (msg.value * tokenReserve) / ethReserve;
    require(_tokenAmount >= tokenAmount, "insufficient token amount");

    IERC20 token = IERC20(tokenAddress);
    token.transferFrom(msg.sender, address(this), tokenAmount);
}
```
唯一的区别是，我们并没有存入用户提供的所有token，而是存入与发送的ether数量相匹配的token量，如果存入的token和ether数量不匹配则拒绝此次新增的流动性，以保证新加入的资产不会破环原有交易池的资产比例。

## LP-tokens
现在我们将讨论能让Uniswap正常运作的一个关键设计。
我们需要一个机制让LP能从他们的行为中获得收益，如果没有激励制度没人愿意把自己的资产放入一个第三方合约。
一个好方案是向每笔交易收手续费，交易者为LP提供的流动性支付一些费用把这些手续费作为LP的收益，这看起来很公平。
为了公平的提供激励，我们需要根据LP的贡献（即他们提供的流动性数量）按比例给予奖励。如果某人提供了池流动性的 50%，那么他就应该获得手续费的 50%。这很合理，对吗？
现在看来，这项任务相当复杂。不过，有一个优雅的解决方案： LP-tokens。
LP-tokens基本上是向流动性提供者发行的 ERC20 token，以换取他们的流动性。事实上，LP tokens就是股票：
1. 你可以获得LP-tokens以代表你提供的流动性
2. 获得的LP-tokens的比例和你在池中提供的流动性成正比
3. 手续费由持有的LP-tokens数量来按比例分配
4. LP-tokens可以换回流动性和累计费用
那么我们将如何根据所提供的流动性来计算发行的 LP-token数量呢？这不太容易，因为我们需要满足一些要求：
1. 每个已发行份额必须始终正确。当有人在我之后存入或取出流动性时，我的份额必须保持正确
2. 以太坊上的写操作（如在合约中存储新数据或更新现有数据）非常昂贵。因此，我们希望降低 LP-tokens的维护成本（即我们不想运行定期重新计算和更新份额）
想象一下我们铸造了很多token并把他们提供给LP。如果我们总是分发所有token，那就得不停的铸造还得不停的重新计算大家的份额，那假如只释放初始token的一部分，同样有可能把token都用光导致还得重新计算份额。
所以提供无限量token，在新流动性加入后就铸造新token才是王道。现在我们得想一个优雅的公式能让大家的份额在这种情况下也一直保持正确。
那么，该咋办嘞？
exchange合约存储了ether和token，所以我们应该基于它们的储备来计算。。。或者只根据它们中的一个来计算，我也不知道。Uniswap V1根据ether来计算的，但是V2版本允许了token之间的交易，所以该如何选择其实没有明确的参照。让我们继续讨论Uniswap V1的功能，稍后我们将看到当有两个ERC20时如何解决这个问题。
这个公式表示新LP-tokens数量是如何根据存入的ether来计算的：

$$
amountMinted=totalAmount∗\frac{ethDeposited}{ethReserve}​
$$

每一次新增流动性都按比例发行LP-tokens，但这也有点问题，如果有人提供了`ethReserve`那么多的流动性那份额还正确吗？
在修改`addLiquidity`前先让exchange合约继承ERC20
```solidity
contract Exchange is ERC20 {
    address public tokenAddress;

    constructor(address _token) ERC20("Zuniswap-V1", "ZUNI-V1") {
        require(_token != address(0), "invalid token address");

        tokenAddress = _token;
    }
```
现在，让我们更新addLiquidity：当增加初始流动性时，发行的LP-tokens数量等于存入的ether数量。
```solidity
function addLiquidity(uint256 _tokenAmount)
    public
    payable
    returns (uint256)
{
    if (getReserve() == 0) {
        ...

        uint256 liquidity = address(this).balance;
        _mint(msg.sender, liquidity);

        return liquidity;
```
后续将按照存入的ether按比例铸造的LP-token
```solidity
    } else {
        ...

        uint256 liquidity = (totalSupply() * msg.value) / ethReserve;
        _mint(msg.sender, liquidity);

        return liquidity;
    }
}
```

## Fees
在收取手续费前得先思考几个问题
* 我们的费用应该以什么方式支付，ether还是token
* 如何从交易中收取固定费率呢
* 如何根据比例给LP提供激励
同样，这看似是一项艰巨的任务，但我们已经具备了解决它的一切条件。
让我们思考一下最后两个问题。我们可以引入一种额外的付款方式，与交易一起发送。这些款项会被累积到一个基金中，任何流动性提供者都可以从中提取与其份额成正比的金额。这听起来是个合理的想法，而且令人惊讶的是，它几乎已经实现了：
1. 交易者已经发送了ether/tokens到exchange合约，为了收费我们可以直接从合约中提取它们提供的流动性
2. 我们的基金就是exchange合约中的储备，它可以用于累积费用，这也意味着储备将会随着时间的推移而增长，所以恒定乘积公式并不是那么恒定！然而，这并不能使它失效；和储备相比费用很小，而且没有办法操纵它来显著改变储备。
3. 现在我们需要回答第一个问题，LP应当按比例获得ether和token以及与LP-tokens成比例的累积费用
Uniswap的费用是0.3%，我们采用1%的比例以方便后续的测试。
```solidity
function getAmount(
  uint256 inputAmount,
  uint256 inputReserve,
  uint256 outputReserve
) private pure returns (uint256) {
  require(inputReserve > 0 && outputReserve > 0, "invalid reserves");

  uint256 inputAmountWithFee = inputAmount * 99;
  uint256 numerator = inputAmountWithFee * outputReserve;
  uint256 denominator = (inputReserve * 100) + inputAmountWithFee;

  return numerator / denominator;
}
```
因为没法用小数除法所以我们把分子分母都扩大100倍，费用则从分子中被减去。

$$
amountWithFee=amount∗\frac{100−fee}{100}​
$$

## Removing liquidity
要收回流动性，我们可以再次使用 LP-tokens：我们不需要记住每个LP存入的金额，可以根据 LP-tokens份额计算收回的流动性数量。
```solidity
function removeLiquidity(uint256 _amount) public returns (uint256, uint256) {
  require(_amount > 0, "invalid amount");

  uint256 ethAmount = (address(this).balance * _amount) / totalSupply();
  uint256 tokenAmount = (getReserve() * _amount) / totalSupply();

  _burn(msg.sender, _amount);
  payable(msg.sender).transfer(ethAmount);
  IERC20(tokenAddress).transfer(msg.sender, tokenAmount);

  return (ethAmount, tokenAmount);
}
```
当流动性收回时会以ether和token的形式按比例返回，但这会造成一定的折损。由于资产由美元计价。当流动性被收回时，余额可能与存入流动性时的余额不同。这意味着，LP将获得不同数量的ether和token，其总价格可能低于存入Uniswap时的价格。
为了计算取出的流动性我们可以用储备乘以LP的LP-token份额

$$
removedAmount=reserve∗\frac{amountLP​}{totalAmountLP}
$$

在收回流动性时会销毁LP-tokens以和它的底层资产相匹配。
## LP reward and impermanent loss demonstration
模拟一个完整的交易流程

1. LP存入100 ethers和200 tokens使得1 token = 0.5 ether, 1 ether = 2 token
```js
exchange.addLiquidity(toWei(200), { value: toWei(100) });
```
2. 一个用户交换了10 ether期望至少得到18 tokens事实上他得到18.0164个，它包含了滑点和1%的费用
```js
exchange.connect(user).ethToTokenSwap(toWei(18), { value: toWei(10) });
```
3. LP随即收回流动性
```js
exchange.removeLiquidity(toWei(100));
```
4. LP得到109.9 ethers（包含费用）和181.9836 tokens，这里能看出存入的总价和取出时不同，我们得到其他交易者的10 ethers但是付出了18.0164个token 这包含了其他用户支付的1%的费用。由于LP提供了全部的流动性所以他得到全部的费用。

## Conclusion
这是一篇很长的文章！希望LP-tokens对你来说不再是一个谜。
然而，我们还没有写完：exchange合约现在已经完成，但我们还需要实现factory合约，它作为交易所的注册器和连接多个交易所的桥梁，并让token-to-token成为可能。我们将在下一部分中实现它！

# Part 3
## What Factory is for
factory合约为exchange合约提供注册机制：每个新部署的exchange合约都由在factory中注册。这是个重要的机制它提供了对exchange进行搜索的基础。
factory提供的另一个实用工具是exchange模板，这样无需处理代码，node和部署脚本就可以上线一个exchange。所以今天我们将学习一个合约如何部署另一个合约。
Uniswap只有一个factory合约自然只有一个交易对注册表。然而没什么能阻止其他用户不通过官方factory来部署他们自己的factory甚至exchange。当这发生时这些exchange不会被Uniswap识别到当然也无法用它们在官方网站上交换token。
## Factory implementation
factory是一个注册表所以我们需要一个数据结构来记录exchage它将为地址做映射以允许通过token找到exchange
```solidity
pragma solidity ^0.8.0;

import "./Exchange.sol";

contract Factory {
    mapping(address => address) public tokenToExchange;

    ...
```
接下来`createExchange`函数将提供通过token地址来创建和部署exchange的功能
```solidity
function createExchange(address _tokenAddress) public returns (address) {
  require(_tokenAddress != address(0), "invalid token address");
  require(
    tokenToExchange[_tokenAddress] == address(0),
    "exchange already exists"
  );

  Exchange exchange = new Exchange(_tokenAddress);
  tokenToExchange[_tokenAddress] = address(exchange);

  return address(exchange);
}
```
它进行两个检查：
1. token 地址不是零地址（0x0000000000000000000000000000000000000000）
2. token 还未被注册过。
现在用token 地址实例化exchange，这需要我们导入"Exchange.sol"，这个实例化的过程类似于在OOP语言中实例化一个类，然而solidity中的`new`运算符不会实际部署合约它返回的值是合约的类型每一个合约都可以转换成一个地址。这正是我们在下一行做的事——得到新exchange的地址并存入注册表。
为了写完这个合约我们需要实现另一个函数——`getExchange`它允许我们通过其他合约的接口查询注册表。
```solidity
function getExchange(address _tokenAddress) public view returns (address) {
  return tokenToExchange[_tokenAddress];
}
```
factory就写完啦，真的太简单辣。
现在我们得改造exchange合约使得它能用factory来提供token-to-token的功能。
首先我们需要连接exchange和factory因为每一个exchange都需要知道factory的地址以外我们不想硬编码。所以得让合约变得更灵活些。为了连接exchagne和factory我们需要增加一个新的状态变量它存储着factory的地址。
```solidity
contract Exchange is ERC20 {
    address public tokenAddress;
    address public factoryAddress; // <--- new line

    constructor(address _token) ERC20("Zuniswap-V1", "ZUNI-V1") {
        require(_token != address(0), "invalid token address");

        tokenAddress = _token;
        factoryAddress = msg.sender;  // <--- new line

    }
    ...
}
```
## Token-to-token swaps
如何通过注册表实现token-to-token？
1. 创建一个标准的token-to-ether交易对
2. 不把ether发送给用户而是发给用户提供的一个exchange
3. 如果这个exchange存在则发送ether到exchange中以交换token
4. 返回交换后的token给用户
看起来合理，对吗？现在来实现它吧
```solidity
// Exchange.sol

function tokenToTokenSwap(
    uint256 _tokensSold,
    uint256 _minTokensBought,
    address _tokenAddress
) public {
    ...
```
这个函数接受三个参数，售卖的token数量，允许exchange返回的最低token数量，期望返回的token地址。
我们首先检查用户提供的token是否已经有exchange，如果未找到则抛出错误。
```solidity
address exchangeAddress = IFactory(factoryAddress).getExchange(
    _tokenAddress
);
require(
    exchangeAddress != address(this) && exchangeAddress != address(0),
    "invalid exchange address"
);
```
我们使用`IFactory`它是factory的接口，在与其他合约交互时使用接口是一个良好的编程实践。然而接口不允许访问状态变量这就是为什么我们在factory中实现`getExchange`函数，现在我们可以通过这个接口来访问facroty合约。
```solidity
interface IFactory {
  function getExchange(address _tokenAddress) external returns (address);
}
```
接下来我们用当前的exchange来用token交换ether并把用户的token发送到exchagne
```solidity
uint256 tokenReserve = getReserve();
uint256 ethBought = getAmount(
    _tokensSold,
    tokenReserve,
    address(this).balance
);

IERC20(tokenAddress).transferFrom(
    msg.sender,
    address(this),
    _tokensSold
);
```
最后一步是使用其他exchange来用ether换回token
```solidity
IExchange(exchangeAddress).ethToTokenSwap{value: ethBought}(
    _minTokensBought
);
```
这就行了！
发现问题了吗？看看`etherToTokenSwap`的最后一行
```solidity
IERC20(tokenAddress).transfer(msg.sender, tokensBought);
```
它发送购回的token给msg.sender。在solidity中msg.sender是动态的它指向发送这个调用的Account或者Contract。当用户调用合约函数它将指向用户的地址但是当合约调用其他合约时msg.sender指向合约。
由此`tokenToTokenSwap`会把token发到第一个exchange，我们可以再调用`ERC20(_tokenAddress).transfer(...)`来解决这个问题，但是有一个更优雅的方法：直接把token发给用户这样还能节约一些gas。为此我们需要把`etherToTokenSwap`分成两个函数
```solidity
function ethToToken(uint256 _minTokens, address recipient) private {
  uint256 tokenReserve = getReserve();
  uint256 tokensBought = getAmount(
    msg.value,
    address(this).balance - msg.value,
    tokenReserve
  );

  require(tokensBought >= _minTokens, "insufficient output amount");

  IERC20(tokenAddress).transfer(recipient, tokensBought);
}

function ethToTokenSwap(uint256 _minTokens) public payable {
  ethToToken(_minTokens, msg.sender);
}
```
`ethToToken`新增一个接受方地址作为参数，它使得我们能灵活选择要把token发给谁。
现在，我们需要另一个函数来向自定义接受方地址。
```solidity
function ethToTokenTransfer(uint256 _minTokens, address _recipient)
  public
  payable
{
  ethToToken(_minTokens, _recipient);
}
```
再给`ethToToken`包装一层作为token-to-token的方法
```solidity
    ...

    IExchange(exchangeAddress).ethToTokenTransfer{value: ethBought}(
        _minTokensBought,
        msg.sender
    );
}
```
搞定！