---
title: "ERC223及ERC827实现代码欠缺安全考虑"
date: 2018-06-24T08:00:00+08:00
summary: "ERC223, ERC827的部分实现代码引入了任意函数调用缺陷，可能会对使用这部分代码的合约带来安全漏洞。如果需要实现上述规范接口，请仔细检查实现代码。这种合约本身允许用户自定义 `call()` 任意地址上任意函数的设计，十分危险。攻击者可以很容易地借用当前合约的身份来进行**任何操作**，比如盗取Token或者绕开权限检查等。"
slug: "Lacking Insights in ERC223 & ERC827 Implementation"
featuredImage: "1.png"
tags: ["ERC223", "ERC827", "Smart Contract", "Vulnerability"]
categories: ["Smart Contract"]
---

> ATN Token 中的 CUSTOM_CALL 漏洞深入分析

*本文部分内容基于安比（SECBIT）实验室团队与吴玉会（轻信科技）的讨论*

本文结论：ERC223, ERC827的部分实现代码引入了任意函数调用缺陷，可能会对使用这部分代码的合约带来安全漏洞。如果需要实现上述规范接口，请仔细检查实现代码。这种合约本身允许用户自定义 `call()` 任意地址上任意函数的设计，十分危险。攻击者可以很容易地借用当前合约的身份来进行**任何操作**，比如盗取Token或者绕开权限检查等。

影响范围：截止目前检测到以太坊上部署的受影响的ERC20合约数量：146

最新更新：

+ 火币网已经暂停了已经上线交易的相关问题Token[9][10]
+ ATN团队已经修复漏洞[1]

## CUSTOM_CALL 滥用事件回顾与分析

2018 年 6 月 20 日，AI Technology Network (ATN) 和慢雾团队披露了一起针对 ATN 智能合约的攻击事件，黑客于 2018 年 5 月 11 日利用 ATN Token 合约存在的漏洞，将自己地址设为 owner 并增发获利 1100 万 ATN。ATN 技术团队迅速发现问题、定位攻击方法并完成合约的升级修复 [1]。黑客利用了 ERC223 合约可传入自定义的接收调用函数与 ds-auth 权限校验等特征，在 ERC223 合约调用这个自定义函数时，合约调用自身函数从而造成内部权限控制失效。随后，百度安全的“隐形人真忙”也在先知安全大会上进行了“以太坊智能合约 call 注入攻击”的主题分享 [2]。这个漏洞源于一个较为常见的做法：在调用合约函数之后，可以再次调用一次另一个合约的任意函数，并且这个任意函数可以由合约调用发起者指定。但是 ATN 的合约漏洞恰恰暴露了这一常见做法非常危险的一面：合约调用者可能通过该功能绕开权限检查，或者以合约的身份发起对其它合约的攻击等等。

有安全隐患代码链接：

- https://github.com/Dexaran/ERC223-token-standard/blob/16d350ec85d5b14b9dc857468c8e0eb4a10572d3/ERC223_Token.sol#L70
- https://github.com/OpenZeppelin/openzeppelin-solidity/blob/master/contracts/token/ERC827/ERC827Token.sol

### ATN 事件漏洞分析

ERC223 是由 Dexaran 于 2017 年 3 月 5 日提出的一个 Token 标准草案 [3]，用于改进 ERC20，解决其无法处理发往合约自身 Token 的这一问题。ERC20 有两套代币转账机制，一套为直接调用 `transfer()` 函数，另一套为调用 `approve()` + `transferFrom()` 先授权再转账。当转账对象为智能合约时，这种情况必须使用第二套方法，否则转往合约地址的 Token 将永远无法再次转出。

下面代码为 ERC223 草案中的一段 **正确示例**，调用 `transfer()` 函数时，合约判断目标地址 `to` 是否是合约，如果是合约，则调用目标合约的 `tokenFallback()` 方法，从而实现合约对转入 Token 的处理。这段代码并没有 CUSTOM_CALL 滥用的问题。

```js
// 提案中的正确示例代码
contract ERC223 {
  function transfer(address to, uint value, bytes data) {
        uint codeLength;
        assembly {
            codeLength := extcodesize(_to)
        }
        balances[msg.sender] = balances[msg.sender].sub(_value);
        balances[_to] = balances[_to].add(_value);
        if(codeLength>0) {
            // Require proper transaction handling.
            ERC223Receiver receiver = ERC223Receiver(_to);
            receiver.tokenFallback(msg.sender, _value, _data);
        }
    }
}
```

ERC223 合约是 ERC20 合约的超集，目标为取代 ERC20 合约，成为新的 Token 合约标准。但提出以来至今一年多的时间仍未得到广泛接受，仅有少数项目采用了该提案。

下面是 ERC223 **错误实现代码**，ATN Token不幸地采用了这一段代码。用户被允许传入任意自定义的 `_custom_fallback`，从而任意调用目标 `_to` 地址上的任意方法！

```js
// 此代码有 CUSTOM_CALL 滥用问题
function transferFrom(
    address _from, 
    address _to, 
    uint256 _amount, 
    bytes _data, 
    string _custom_fallback
    ) 
    public returns (bool success)
{
    ...
    ERC223ReceivingContract receiver = ERC223ReceivingContract(_to);
    receiving.call.value(0)(byte4(keccak256(_custom_fallback)), _from, amout, data);
    ...
}
```

ATN 的漏洞分析报告中称其 Token 合约参考了 ERC223 标准的推荐实现 [4]。经过我们调查，发现其的确与 Dexaran 维护的 ERC223-token-standard 中 Recommended 分支的 `transfer()` 方法实现类似 [5]：

```js
// 此代码有 CUSTOM_CALL 滥用问题
function transfer(
    address _to, 
    uint _value, 
    bytes _data, 
    string _custom_fallback
    ) 
    public returns (bool success) 
{
    ...
    assert(_to.call.value(0)(bytes4(keccak256(_custom_fallback)), msg.sender, _value, _data));
    ...
}
```

这其实是很危险的行为！ConsenSys 维护的「以太坊智能合约 —— 最佳安全开发指南」中曾明确提示，要尽量避免合约的外部调用。但在此次攻击事件中，黑客传入的 `_custom_fallback` 为 `setOwner(address)`，传入的目标地址 `_to` 恰好是 ATN 合约本身，间接调用了 ATN 的 `setOwner(address)` 方法，使得 `msg.sender` 变为 ATN Token 合约本身，从而通过 ds-auth 库的 `isAuthorized()` 鉴权校验。

EVM 读取参数时并不会校验参数个数，在上述例子中，黑客调用了 `setOwner(adddress)` 函数，EVM 仅会读取最左边的 `_from` 参数。因此使用底层 `call()` 方法传参时，参数个数与函数所需不一致并不会引发报错，黑客很容易精心构造出所需的攻击参数。

### CUSTOM_CALL 滥用的危害

再回到 `_custom_fallback` 接口实现上。我们认为作为一个通用 Token 标准接口设计，设计者必须尽可能多地考虑整个系统生态的安全性，尽可能规避因使用不当引入的风险。倘若上面这种 `_custom_fallback` 接口设计得到广泛采纳，未来势必会出现更多类似的安全性问题。一个良好的接口设计，最好能做到精简、易用和无歧义。作者提案中 `tokenFallback()` 接口完全可以应对其原本想要解决的 ERC20 问题。而实现上引入自定义的 `_custom_fallback`，很容易对开发者产生误导并被滥用。

```js
  function approveAndCall(
    address _spender,
    uint256 _value,
    bytes _data
  )
    public
    payable
    returns (bool)
  {
    // require(_spender != address(this));
    approve(_spender, _value);
    require(_spender.call.value(msg.value)(_data));
    return true;
  }
```

通常当我们调用 ERC20 的 `approve()` 函数给一个智能合约地址后，对方并不能收到相关通知进行下一步操作，常见做法是利用 **接收通知调用**（`receiverCall`）来解决无法监听的问题。上面代码是一种实现方式，很不幸这段代码有严重的 CUSTOM_CALL 滥用漏洞。调用 `approveAndCall()` 函数后，会接着执行 `_spender` 上用户自定义的其他方法来进行接收者的后续操作。

下面敲黑板！

【**危害**】：这种合约本身允许用户自定义 `call()` 任意地址上任意函数的设计，十分危险。攻击者可以很容易地借用当前合约的身份来进行**任何操作**。

这通常会导致两种危险的后果：

- **后果一**：允许攻击者以缺陷合约身份来盗走其它 Token 合约中的 Token
- **后果二**：与 ds-auth 结合，绕过合约自身的权限检查
- **后果三**：允许攻击者以缺陷合约身份来盗走其它 Token 账户所授权（Approve）的 Token

后果一举例：假设缺陷 Token 合约 A 自身账户中拥有各种 Token B、 C、 D 等，攻击者只需将 `_spender` 设为想要盗取的目标 Token （如 B 的地址），再构造用于调用 `transfer(address,uint256)` 的 `_data`，即可轻松以合约 A 的身份将合约 A 中的各类 Token 转走。上面代码中对 `_spender != address(this)` 的校验，也仅能保护 A Token。

管理各种 Token 的智能合约，倘若也允许自定义 `call()`，其合约上的各种 Token 就十分危险了。

后果二举例：如 ATN 安全事件中，黑客也是借此漏洞利用 ATN 合约的身份，绕过了 ds-auth 的权限控制。

后果三举例：假设缺陷 Token 合约 A 被用户 X 授权（Approve）管理 10,000 个Token B，那么黑客也是借此漏洞调用`transferFrom()`函数来盗取Token B。

### ERC223 提案实现与接口定义不一致

进一步调查我们发现，ERC223 提案的文字接口描述中并没有提到 `_custom_fallback` 这一参数的引入和使用。

以下是该提案规定的接口：

```js
contract ERC223Interface {
    uint public totalSupply;
    function balanceOf(address who) constant returns (uint);
    function transfer(address to, uint value);
    function transfer(address to, uint value, bytes data);
    event Transfer(address indexed from, address indexed to, uint value, bytes data);
}
```

可以看到两个 `transfer()` 接口定义中均没有出现 `_custom_fallback` 参数。

我们在来看看提案的文字描述：

>  If the receiver is a contract ERC223 token contract will try to call tokenFallback function on receiver contract. If there is no tokenFallback function on receiver contract transaction will fail. tokenFallback function is analogue of fallback function for Ether transactions. It can be used to handle incoming transactions.

这段话的中心思想就是如果 Token 转账目标对象是 ERC223 合约，则尝试调用其 `tokenFallback()` 函数，如果目标对象不存在 `tokenFallback()` 函数，则让交易 fail 掉。`tokenFallback()` 在这里充当的作用就是类似以太转账里的默认 `fallback` 函数。

显然，ERC223 提案的初衷十分清晰，就是约定一个 `tokenFallback()` 接口作为 Token 合约标准，用于处理转入的 Token。ERC223 提案主分支的代码实现也没有 `_custom_fallback` 的问题。而作者推荐的 Recommended 分支里的代码却增加了一种引入 `_custom_fallback` 的 `transfer()` 实现，但是没有进行任何风险提示。

### ERC223 代码实现的其他问题

事实上，ERC223 Recommended 分支代码实现还存在其他问题。

`call()` 在处理 bytes 变量时会引发 evm 层面的 bug，导致数据不一致 [6]。

```js
// ERC223_Token.sol#L70
assert(_to.call.value(0)(bytes4(keccak256(_custom_fallback)), msg.sender, _value, _data));
```

event 处理 indexed 的 bytes 变量，在特殊情况下也会引发报错 [7]。

```js
// ERC223_Interface.sol#L18
event Transfer(address indexed from, address indexed to, uint value, bytes indexed data); 
```
 
由此可以推断 ERC223 的 Recommended 分支代码是不成熟的，我们不推荐使用。

## EVM 参数传递机制

下面我们解释一下 EVM 中函数调用与参数传递的机制，以便于对这个安全隐患的理解。以如下合约为例

```js
contract A{
    function transfer(address to, uint256 value){
  	      return;
    }
}
```

首先了解一下EVM参数传递机制：在调用函数时，如果目标函数有参数，正常情况下我们需要根据ABI指定的参数类型来构造输入。例如 `transfer(address to, uint256 value)` 在调用 `transfer()` 时，以太坊使用函数签名的哈希值前4字节作为 `function selector`，计算 `sha3(transfer(address,uint256))` 得到 `0xA9059CBB`，再拼接上`to`地址，256位补全为

```
0x0000000000000000000000003f5ce5fbfe3e9af3971dd833d26ba9b5c936f0be
```

，紧随其后拼接上`value`，同样256位补全为

```
0x000000000000000000000000000000000000000000000000000000e8d4a51000
```

最后得到完整的calldata：

```
0xa9059cbb0000000000000000000000003f5ce5fbfe3e9af3971dd833d26ba9b5c936f0be000000000000000000000000000000000000000000000000000000e8d4a51000
```

将这段 calldata 附加在交易中发送到目标智能合约地址即可实现函数调用。当以太坊节点收到交易时，将 calldata 与智能合约字节码一同加载到 EVM 中，字节码在编译时生成，也意味着对参数的处理在编译时也已经固定下来了。我们查阅以太坊黄皮书可以看到：

![](1.png)

现在我们来看一下实际编译出的字节码如何分离 `transfer`、`address`、`value`这三个参数，观察如下字节码片段：

```js
PUSH 80
PUSH 40
MSTORE
PUSH 4
CALLDATASIZE 
LT 
PUSH [tag] 1
JUMPI 
PUSH 0
CALLDATALOAD
PUSH 100000000000000000000000000000000000000000000000000000000
SWAP1
DIV 
PUSH FFFFFFFF
AND
DUP1
PUSH A9059CBB
EQ 			
PUSH 2
JUMPI
JUMPDEST
PUSH 0
DUP1
REVERT 
```

我们可以看到在字节码中，出于动态数组的考虑，只会判断 `calldata` 是否小于某个最小长度，但是不会检查参数是否过长。编译器会生成一系列 `CALLDATALOAD` 配合数学运算来分离出函数需要的参数。首先计算调用的目标函数：

`CALLDATALOAD` 指令将交易中的 calldata（`0xa9059cbb0000000000000000000000003f5ce5fbfe3e9af3971dd833d26ba9b5c936f0be000000000000000000000000000000000000000000000000000000e8d4a51000`）加载到栈中，然后使用除法运算，将数据前256位除以 `0x 100000000000000000000000000000000000000000000000000000000`, 得到`0xA9059CBB`，以此类推，每个参数都会用类似的方法分离出来。但是当参数过多的时候，字节码、EVM都不会处理，可以直接忽略，所以这个特性主要来源于编译器。黑客利用这一特性可以很容易地针对 CUSTOM_CALL 构造攻击参数。

## ERC827 的有安全隐患代码实现

类似的 ERC827 Token 提案也存在相同的问题 [8]。下面的代码来自于 **openzeppelin-solidity** 的 ERC827 问题代码实现：

```js
function transferAndCall(address _to, uint256 _value,bytes _data)
    public payable returns (bool)
{
    require(_to != address(this));
    super.transfer(_to, _value);
    require(_to.call.value(msg.value)(_data));
    return true;
}
```

该代码在 `transferAndCall()` 中转账功能完成后，会调用`_to`地址上的任意函数，并且参数由调用者任意指定。由于该函数检查了 `_to != address(this)`，因此代码不会产生 与 ds-auth 库结合后绕开权限检查的安全漏洞（`后果二`），但是可能会引入前文所提到的 `后果一`与`后果三`，即可以任意支配问题合约所拥有或管理的 Token（即以 this 合约为跳板攻击其它合约）。

此外，还有不少 ERC20 Token 加入了类似的 `call()` 自定义数据的实现。这种允许自定义 `call()` 任意地址上任意函数的设计，十分危险。在特殊情况下，甚至允许攻击者盗走合约中的各种 Token，以及绕过合约本身的权限控制。

## ERC20, ERC721 中关于“接收通知调用”正确的代码实现

正确的代码实现中，对于“接收通知调用”的处理应该将被通知函数的签名（signature）写死为固定值，避免由攻击者来**任意指定**的任何可能性。下面举两个例子说明正确的通知调用的写法：

1. 声明Receiver函数，并通过声明的函数进行接收通知调用

例如在以太坊官网(ethereum.org)维护的 ERC20 代码中关于通知调用的代码片段：

```js
function approveAndCall(address _spender, uint256 _value, bytes _extraData)
    public returns (bool success) 
{
    tokenRecipient spender = tokenRecipient(_spender);
    if (approve(_spender, _value)) {
        spender.receiveApproval(msg.sender, _value, this, _extraData);
        return true;
    }
}
```

调用通知采用了正常的函数调用方式。

2. 通过 Receiver 函数的**签名常量**进行接收通知调用

下面一段正确实现代码来自于 Consensys 维护的 Token-Factory 项目

```js
/* Approves and then calls the receiving contract */
function approveAndCall(address _spender, uint256 _value, bytes _extraData) returns (bool success) {
    allowed[msg.sender][_spender] = _value;
    Approval(msg.sender, _spender, _value);

    //call the receiveApproval function on the contract you want to be notified. This crafts the function signature manually so one doesn't have to include a contract in here just for this.
    //receiveApproval(address _from, uint256 _value, address _tokenContract, bytes _extraData)
    //it is assumed that when does this that the call *should* succeed, otherwise one would use vanilla approve instead.
    if(!_spender.call(bytes4(bytes32(sha3("receiveApproval(address,uint256,address,bytes)"))), msg.sender, _value, this, _extraData)) { throw; }
    return true;
    }
```

下面是正确实现“接收通知调用”的代码实现库

+ https://github.com/svenstucki/ERC677
+ https://github.com/OpenZeppelin/openzeppelin-solidity/blob/master/contracts/token/ERC721/ERC721BasicToken.sol#L349
+ https://github.com/ConsenSys/Token-Factory/blob/master/contracts/HumanStandardToken.sol
+ https://github.com/ethereum/ethereum-org/blob/b46095815f52cf328ecf7676b2b38284d48fba58/solidity/token-advanced.sol#L138

## 总结与反思

- ERC223 标准的实现与接口定义脱节，实现仓库存在两个分支接口不一致，使用**权威代码**时不能放松警惕
- ERC827 代码同样存在隐患
- 用 low-level call 一定要小心
- 需要深刻理解 EVM 中关于合约函数调用的实现机制

## Reference

- [1] [ATN抵御合约攻击的报告](https://atn.io/resource/aareport.pdf)
- [2] [以太坊智能合约call注入攻击](https://blog.csdn.net/u011721501/article/details/80757811)
- [3] [ERC-223 Token Standard Proposal Draft](https://github.com/ethereum/EIPs/issues/223)
- [4] [ATN.sol transferFrom()](https://github.com/ATNIO/atn-contracts/blob/7203781ad8d106ec6d1f9ca8305e76dd1274b181/src/ATN.sol#L114)
- [5] [ERC223_Token.sol transfer() function](https://github.com/Dexaran/ERC223-token-standard/blob/16d350ec85d5b14b9dc857468c8e0eb4a10572d3/ERC223_Token.sol#L70)
- [6] [ERC223-token-standard Issue 50](https://github.com/Dexaran/ERC223-token-standard/issues/50)
- [7] [ERC223-token-standard Issue 51](https://github.com/Dexaran/ERC223-token-standard/issues/51)
- [8] [ERC827Token.sol](https://github.com/OpenZeppelin/openzeppelin-solidity/blob/f18c3bc438b366f9cb3a8613f5be160c2cbced5e/contracts/token/ERC827/ERC827Token.sol#L73)
- [9] [New evilReflex Bug Identified in Multiple ERC20 Smart Contracts (CVE-2018-12702, CVE-2018-12703)](https://peckshield.com/2018/06/23/evilReflex/)
- [10] [HADAX Suspends 18T and GVE Deposits and Withdrawals](https://huobiglobal.zendesk.com/hc/en-us/articles/360000110521-HADAX-Suspends-18T-and-GVE-Deposits-and-Withdrawals)
