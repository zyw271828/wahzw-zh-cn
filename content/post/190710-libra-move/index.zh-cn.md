---
title: "Move语言：我眼中的 Libra 最大亮点"
date: 2019-07-10T08:00:00+08:00
summary: "Libra 的一系列发布中，新的编程语言 Move 尤为吸人眼球。第一时间看了 Move 的白皮书，嗯，这也许才是未来智能合约语言该有的样子。"
featuredImage: "cover1.png"
slug: "'Move' Programming Language: The Highlight of Libra"
author: 郭宇
tags: ["Facebook", "Libra", "Smart Contract", "Programming Language"]
categories: ["Libra", "Programming Language", "Smart Contract"]
---

相信各位和我一样，今天被 Facebook 刷爆朋友圈。

Facebook 发起的加密数字货币项目 Libra 于今天（6月18日）正式公开亮相。Libra 同步发布了多语言官网和白皮书，定位为面向数十亿人的全球货币和金融服务基础设施。Libra 还发布了多个技术白皮书，详细介绍了其新开发的编程语言 Move 和共识协议 LibraBFT。Libra 源码已在 GitHub 开源，测试网络也已上线。目前设计为许可链（联盟链），其声称当前非许可链（公链）不存在成熟的解决方案能够支撑数十亿人的使用需求，并表明将在发布五年内开始转向非许可链的过渡工作。

Libra 的一系列发布中，新的编程语言 Move 尤为吸人眼球。第一时间看了 Move 的白皮书，嗯，这也许才是未来智能合约语言该有的样子。

一位来自柏林的开发者 Lefteris Karapetsas 在社交平台上提出了自己的观点：

> Their design goals seem to overlap, or even aim to replace Ethereum? 
> 
> 他们的设计目标似乎有些重叠，或者甚至旨在代替以太坊？

PuzzleToLife.com 的创始人 CryptoPuzzleDream 认为：

> I think "move" programming language released by $FB could be more interesting than libra
> 
> 我认为FB发布的 “move” 语言比 libra 更有趣。

James Clark 是一名标准极客，他说：

> I'm usually pretty skeptical of anything related to cryptocurrency, but here's one piece of Libra that looks potentially interesting: a bytecode programming language called Move with semantics inspired by linear logic. 
> 
> 我通常对与加密货币相关的任何东西都表示怀疑，但是Libra 中有一部分看起来相当有趣：一个被称为 Move 的字节码编程语言，其语义受线性逻辑的启发。

而我脑子里闪过是这样一句话：

> Move 是为「数字资产」而生的智能合约平台型语言


## Move 语言的三大用处

1. 发行数字货币，Token，和数字资产
2. 灵活处理区块链交易
3. 验证器（Validator）管理


## 自底向上的静态类型系统

Move 采用的是静态类型系统，类型系统本质上是一种逻辑约束。相比以太坊的智能合约语言来说要严格地多。现代的编程语言比如 Rust, Golang, Typescript，Haskell, Scala, OCaml 都不约而同采用了静态类型系统，他们的优点是，很多编程低级错误都可以在编译的时候发现，而不是拖到运行期才爆出 bug。

> Well-typed programs never get stuck. 

这是程序语言（PL）领域的一句黑话：一个类型无误的代码永远不会跑飞。意思是，如果一段合约代码经过了类型检查，那么可靠性会相当高。

Move 也没有设计成一个 100% 静态类型检查的语言，那样会降低实用性。 Move 提到了尽量让类型检查在编译的时刻进行，而不是等部署到链上之后。当然有些类型检查不得不放到运行期来做，但是仍然可以保证类型安全。

Move 有个非常好的设计思路是，从虚拟机开始就是静态类型化的，然后往上是一个有一定易读性的中间语言层，IR层（Intermediate Representation），也是类型化的。将来，Move 上层将会提供更多的面向各种金融应用的高级语言，那些语言自然也是静态类型，保证智能合约不再会发生非常低级的错误。

## First-class Resources 理念

`First-class Resources` 这个词相当得学术，中文翻译过来叫「资源是一等公民」，
这究竟什么意思呢？所谓编程语言的一等公民就是编程语言在编程的时候首要考虑的被编程对象。

那么`资源`，`Resources`又是什么呢？ 这也是一个很学术的名字。`Resources`是和`Value` 相对应
的概念。`Value`是可以随意拷贝的，而`Resources`只能被消耗，不能被拷贝。`Resources`就像可乐，
你喝了一瓶就少了一瓶，而`Value`，就好比写在本子上的英文单词，每天早上都可以念一遍，念完他不会消失，如果你记住了，那就在脑子里拷贝了一份。不仅你可以念，我也可以念，你可以背，我也可以背。

传统的编程语言，包括以太坊智能合约语言中，对于数字资产的记账是采用 `Value` 的方式，这会导致一个问题：记账是有可能记错的。事实上记错账的智能合约相当得多，比如：张三向李四转账，李四的账户多了10块钱，但是张三的账户余额却没改。在过去两年里，各种智能合约记账漏洞甚至一度搞得大家对智能合约的未来丧失了信心。

Move 合约吸收了传统理论「线性逻辑」的思想，提出了一种类型，叫做「资源类型」（Resources Types）。数字资产可以用「资源类型」来定义，这样一来，数字资产就像资源一样，满足线性逻辑的一些特性：

1. 数字资产不能被复制
2. 数字资产不能凭空消失

`First-class Resources` 的真正含义是「数字资产是一等公民」。这句话可以引申出，Move 是为操作数字资产而生的智能合约语言。从技术角度讲，数字资产可以作为合约的变量，数字资产可以存储，可以赋值，可以作为函数/过程的参数，也可以作为函数/过程的返回值。而 Move 的静态类型系统使得智能合约代码能够在编译期，也就是部署前就可以通过编译器检查出绝大多数的资源使用错误。保证智能合约不再像以前那样的脆弱不堪。

摘用白皮书摘要中的一句话：

> First-class resources are a very general concept that programmers can use not only to implement safe digital assets but also to write correct business logic for wrapping assets and enforcing access control policies.

> 作为一等公民的资源是一种非常普遍的概念，程序员不仅可以用它实现安全的数字资产，同时也可以编写正确的业务逻辑，实现正确的访问控制策略。


## 合约安全性设计

Move 合约在设计时，充分考虑了安全性。首先 Move 完全不支持动态指派（Dynamic Dispatch）。好，我这里解释下什么是 Dynamic Dispatch，通俗地说，这是一种非常灵活的语言机制。在程序里面是可以写很多的函数，或者过程，或者子程序。然后一个主程序可以来调用这些函数/过程/子程序，来分别完成不同的功能。如果程序在运行之前，我们就能知道它到底都调用了哪个函数，或者以某种顺序调用很多函数，那么这些函数调用是「静态指派」的，如果在运行之前，我们不清楚某一步的函数调用究竟是调用了哪一个函数，直到程序运行的时候，通过观察，我们才能知道的话，那么这个函数调用被称为是「动态指派」的。「动态指派」显然要比「静态指派」灵活的多。

但是灵活同时也意味着更容易出问题。很多现代编程语言都或多或少支持动态指派，也就是从语言层面直接支持，比如面向对象语言中的「继承」导致的「动态绑定」。动态特性是不利于程序的推理，更不利于形式化验证（Formal verification），也更容易出安全问题。在以太坊智能合约设计中就存在许多「动态特性」，比如支持函数指针做参数，合约做参数，delegatecall等等。除此之外，还有「高阶函数指针」，「回调函数」，「异常处理函数」等等都属于「动态指派」。而 Move 语言则剔除了任何形式的「动态指派」或者「动态特性」，即所有的合约执行路径都能在编译的时候确定，然后可以进行非常充分的分析、验证。

Move 合约在运行前都会经过一个字节码验证器（Bytecode verifier）进行校验，这个验证器可以检查出各种类型错误，同时也进行充分的静态分析。同时字节码在解释执行的时候，仍然是带着类型，一边运行，一边检查。

Move 语言对合约中的「可修改变量」进行了非常严格的限制，并且从 Rust 语言那边偷师了一些设计理念。保证任何时刻只能由一个指针对可修改变量进行修改，这样避免造成混乱。在以太坊 Solidity 里面，可以定义很多的指针指向同一个变量，如果程序员没考虑清楚代码逻辑，就会很容易出各种奇葩问题。

与以太坊 EVM 平台相比，Move 模块系统不再支持循环递归依赖，完美解决合约重入一类的漏洞（Re-entrancy）。

## 强悍的模块系统

Move 模块系统采用的是一种函数式编程语言（OCaml, Coq, SML）风格的设计，按照白皮书的说法：

> Move modules are similar to smart contracts in other blockchain languages. ..., However, modules enforce strong data abstraction — a type is transparent inside its declaring module and opaque outside of it.

模块系统可以很好地将数字资产的概念打包封装，对于数字资产的操作比如通过模块的公开接口，并且在这个接口上可以做灵活的权限控制。在写以太坊智能合约的时候，以太坊上的 ERC20 Token 是作为一个合约而存在，而在 Move 语言中，一个 Token 可以被想象成一个箱子，被随意像资源一样传递，但是同时不会暴露箱子内部细节。同时模块系统的抽象也完全基于它的静态类型系统，并且类型安全性完全可以由智能合约虚拟机来检查保证。

Move 的模块系统为智能合约的形式化验证提供了非常好的基础，在模块内部可以定义「不变式」。所谓的不变式是指对数字资产内部状态的一个严格约束，这个约束可以为形式化验证的自动化提供非常有价值的信息。而且，模块系统的「不透明抽象」可以使形式化验证工作变得模块化，成本更低。在Move 模块系统上编写程序分析器，符号执行器也会简单很多，因为经过抽象，可以把合约逻辑变得非常简单，易推理。当然对于普通智能合约开发者而言，利用好模块系统，可以让合约代码变得异常清晰。


## 面向未来的 Move 智能合约

Move 虽然看起来还略显粗糙和稚嫩，但是这个方向仍然让人激动人心，从 Move 语言层面可以看到 Facebook 的野心，是想做一个庞大的数字资产平台。这个角色本来应该属于以太坊。

我为什么有点喜欢上了 Move，想了想，大概下面三个原因：

1. 汲取了 PL (程序语言)领域研究成果，同时也吸收了 EVM 智能合约语言的经验教训。
2. 从设计上无比重视「智能合约安全性、正确性」
3. 没有墨守成规（既没有用WASM，LLVM，也没有在 EVM 上直接修改），而是积极创新，是设计构思真正适合金融应用的智能合约语言



> 在区块链领域里面，凡是套用传统方法的方案，无一胜出，唯有创新才有未来。
