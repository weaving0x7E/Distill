译自：
* https://jeiwan.net/posts/programming-defi-uniswap-1/
* https://jeiwan.net/posts/programming-defi-uniswapv2-2/
* https://jeiwan.net/posts/programming-defi-uniswapv2-3/
* https://jeiwan.net/posts/programming-defi-uniswapv2-4/

Author: Ivan Kuznetsov 
Content of this article is licensed under a [Creative Commons Attribution-NonCommercial-ShareAlike 4.0 International License](http://creativecommons.org/licenses/by-nc-sa/4.0/)

# Part 1
## Introduction
截至2021年6月共推出过三个版本的Uniswap。第一个版本于2018年推出允许换ether和其他token，还可以进行chained swap，以实现token与token之间的交换。V2于2020年三月推出相比于V1它允许ERC20之间直接交换，当然也允许用chained swap交换任意token。V3于2021年五月推出本次升级极大的提升了资本利用率它允许LP从资金池中取出更大比例的流动性，同时仍能获得同样的回报。在本系列中我们将深入每个版本并从零开始构建一个简单版本的Uniswap。

## What is Uniswap
简单的说Uniswap是一个去中心化交易所（DEX）它的目的时提供一个中心化交易所的替代方案。它在Ethereum上运行并且是完全依赖代码本身来运转的，没有任何人能在这套机制中有任何特权。
在。
它底层依赖一套算法能允许建立pool或token pairs为其提供流动性使得用户可以利用这种流动性来交换token，这种算法被称为自动做市商(automated market maker)或者LP。
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
x是ether储备y是token储备k是常量。Uniswap需要k保持恒定无论x与y的储备如何。当你用ether交易其他币对时你存入ether到合约中并得到一定数量的token。Uniswap确保在交换后k不变（但这也不绝对）。这个公式也负责价格计算，下文将详述这一机制。

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
Uniswap V1只有两个合约：Factory和Exchange
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
现在我们想一下如何计算交换价格，一个朴素的想法也许是这样直接把价格和储备挂钩
$$
P_X​=\frac{x}{y}​,P_Y​=\frac{y}{x}​

$$
这有点道理因为exchange合约不和中心化交易所或者任何外部价格预言机交互，所以它不可能知道正确的价格，所以exchange合约本身就应当是一个价格预言机。它只能利用ether和token储备，这是我们计算价格的唯一信息。
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
原因是solidity只支持近似的整数除法0.5被近似成了0，让我们增加精度来修复这个问题吧：
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
现在1 token等于0.5 ether而1 ether 等于2 token了
看起来好像什么问题，但是如果我们用2000 token来换取ether呢？我们会得到1000 ether，这将用尽exchange合约里的全部储备
显然这个价格函数有点古怪，它允许用尽合约储备这是我们不想看到的。
原因在于价格函数遵从恒等式它定义了`k`是个`x`与`y`的常数和，这种函数是一条直线
![[Pasted image 20250127161859.png]]
在x和y轴的交点处意味着允许它们为0！我们显然不想这样。

### Correct pricing function
回忆一下Uniswap是一个恒定乘积做市商（constant product market maker）这意味着它基于恒定乘积公式：
$$
x∗y=k
$$
这个公式能产生更好价格函数吗？
公式表明无论储备是多少，`k`都保持不变。每笔交易都会增减ether或token的储备——让我们把这个逻辑放这个公式中：
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
双曲线和x与y轴不会相加自然储备也不会是0，这意味着储备是无限的！
不难看出价格函数会动态滑点，单笔交易越大滑点越明显。从测试中可以看到我们的交易所得比预期的要少一些，这似乎是恒定乘积做市商的一个缺点，然而这个机制保护了交易池不会被耗尽，这也符合供需关系，越紧俏的资产价格越高。
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
最后再想说的是：我们最初基于准备金率的定价函数没有错。事实上，当我们交易的代币数量与储备相比非常小时，这是正确的。但要建立AMM机制，我们需要更复杂的东西。

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
今天就到这里！虽然还没结束，但我们做了很多。我们的exchange合约可以接受用户的流动性，以一种防止流动性枯竭的方式计算价格，并允许用户交换代币和代币。但一些重要的机制还没建立：
* 增加新的流动性会导致巨大的价格变化
* 流动性提供者没有回报；所有交换都是免费的
* 无法移除流动性
* 无法交换 ERC20
* factory合约仍未实现

我们将在未来的部分中实现这些功能！