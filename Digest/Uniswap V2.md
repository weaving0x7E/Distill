译自：
* https://jeiwan.net/posts/programming-defi-uniswapv2-1/
* https://jeiwan.net/posts/programming-defi-uniswapv2-2/
* https://jeiwan.net/posts/programming-defi-uniswapv2-3/
* https://jeiwan.net/posts/programming-defi-uniswapv2-4/

Author: Ivan Kuznetsov 

Content of this article is licensed under a [Creative Commons Attribution-NonCommercial-ShareAlike 4.0 International License](http://creativecommons.org/licenses/by-nc-sa/4.0/)
# Part 1
## Introduction
Uniswap是一个运行于以太坊的去中心化交易所。它是完全自动化、拒绝人为操作、去中心化的。它经历了多次开发迭代：第一个版本于2018年十一月上线，第二个版本于2020年五月上线，第三个版本也是最后一个版本于2021年三月上线。
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
我们将从core合约开始，首先关注最重要的机制。这些合约是非常通用的，它们的调用需要前置准备——这种低级结构减少了易被攻击的范围，并使整个结构更精细。（我也看不太懂作者到底想说啥。。。😓）
