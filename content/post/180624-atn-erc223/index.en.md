---
title: "Lacking Insights in ERC223 & ERC827 Implementation"
date: 2018-06-24T08:00:00+08:00
summary: "This bug originates from a common practice: calling an arbitrary function appointed by the contract caller from another contract after invoking a function in the current one, while the bug in ATN contract reveals the danger of this approach: the contract caller could bypass authority checks or attack others with the identity of contract itself via this feature."
slug: "Lacking Insights in ERC223 & ERC827 Implementation"
featuredImage: "1.png"
tags: ["ERC223", "ERC827", "Smart Contract", "Vulnerability"]
categories: ["Smart Contract"]
contentCopyright: "If you need to reprint, please indicate the author and source of the article."
---

> An Analysis of ATN Token's CUSTOM_CALL Bug

On June 20th, 2018, AI Technology Network (ATN) reported an attack on ATN smart contract: an hacker set his address as `owner` via a bug in ATN Token contracts and issued 11 million ATN tokens for himself on May 11th, 2018. ATN team located the bug, discovered the hacking method and upgraded the contract in no time [1]. The hacker made use of passing custom fallback functions in ERC223 contracts along with `ds-auth` approving check, then called the function of this contract when ERC223 contract invoked this CUSTOM_CALL. Afterwards, '隐形人真忙' of Baidu Security also shared experience of 'call injection attack in Ethereum smart contracts' in Xianzhi Security Summit. This bug originates from a common practice: calling an arbitrary function appointed by the contract caller from another contract after invoking a function in the current one, while the bug in ATN contract reveals the danger of this approach: the contract caller could bypass authority checks or attack others with the identity of contract itself via this feature. 

Number of ERC20 contracts with similar vulnerabilities deployed on Ethereum at present: 146


Links of Risky Code:

- https://github.com/Dexaran/ERC223-token-standard/blob/16d350ec85d5b14b9dc857468c8e0eb4a10572d3/ERC223_Token.sol#L70

- https://github.com/OpenZeppelin/openzeppelin-solidity/blob/dc1e352cc49565ba0a33c346006655cf63681ff5/contracts/token/ERC827/ERC827Token.sol

Update:

- ERC827 code was removed from open-zeppelin [2]

### Analysis on ATN Incident

ERC223 is a draft of Token standard raised by Dexaran on March 5th, 2017 [3] to solve the problem of handling tokens sent to an ERC20 contract. ERC20 has 2 ways to transfer tokens: one is calling `transfer()` directly and another is invoking `approve()` + `transferFrom()` - approve first, then transfer. Taking the later approach is required when the smart contract takes the receiver role, otherwise tokens sent to the contract address would be locked forever.

Here is a **correct sample** of ERC223 draft: when calling `transfer()`, the contract checks if `to` address is a contract. If so, it calls `tokenFallback()` in the target contract to handle tokens sent to the contract. There is no CUSTOM_CALL abusing.

```js
// Correct draft code sample
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

ERC223 is a superset of ERC20 designed to replace ERC20 as a new Token contract standard, while not being widely accepted for over a year. Only a small portion of projects employed it.

This is an **incorrect implementation** of ERC223 taken by ATN Token unfortunately. The user is permitted to pass arbitrary `_custom_fallback` to call any functions from `_to` address.

```js
// CUSTOM_CALL abusing
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

Report on ATN bugs stated that its contract referred to a recommended implementation of ERC223 [4]. After investigating this issue, we found that it was indeed similar to `transfer()` in Recommended branch of ERC223-token-standard repo maintained by Dexaran [5]:

```js
// CUSTOM_CALL abusing
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

This is actually risky. '*A Guide to Smart Contract Security Best Practices*' by ConsenSys urges developers avoiding contract external calling. The hacker passed `setOwner(address)` as `_custom_fallback` in this attack and the target address `_to` was exactly ATN contract itself, thus invoking `setOwner(address)` of ATN contract indirectly. `msg.sender` became ATN Token contract itself so as to pass the test `isAuthorized()` in `ds-auth`.

EVM does not check the number of parameters when reading them. After the hacker called `setOwner(adddress)`, EVM only read `_from` on left. Therefore no error would emerge on differences between parameter numbers and the required sum by the function when applying low-level `call()` for parameter passing. The hacker had no difficulty constructing attacking parameters as a result.

### Danger of Abusing CUSTOM_CALL

Back to `_custom_fallback` implementation. As a common Token interface, the designer should, in our opinion, take as many cases as possible into account to get rid of introducing potential risks and vulnerabilities. Suppose `_custom_fallback` interface above gets accepted widely, more similar security problems are yet to come. A robust interface design should be simple, easy-handling and straightforward. The `tokenFallback()` interface in the draft could have dealt with the ERC20 issue intended for, whereas introducing `_custom_fallback` would disrupt developers and become abused.

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

In most cases, when we pass a smart contract address to ERC20 `approve()`, the other side cannot get relevant notifications for next steps and  a common solution is `receiverCall`. The snippet above is one implementation and unluckily contains severe CUSTOM_CALL abusing. After executing `approveAndCall()`, it would run other operations defined by `_spender`.

Pay attention to this one.

Consequences: These types of contracts are designed to permit users defining `call()` with functions on any addresses, which is highly risky. By capturing the contract's identity, hackers could easily conduct **any operations**.

This often leads to 3 dangerous results:

- **First**：Allow an attacker to steal tokens in other contracts with the identity of a buggy contract
- **Second**：Bypass authority checks in the contract with the help of `ds-auth`
- **Third**：Allow an attacker to steal approved tokens in other accounts with the identity of a buggy contract

An example of first one: Suppose buggy contract A has B/C/D tokens, a hacker could set `_spender` address to a target token contract (e.g. B's address) and select `_data` for calling `transfer(address,uint256)`, there would not be any barrier for the hacker to transfer out A's tokens with the identity of contract A. `_spender != address(this)` in code above could only protect A Token.

If smart contracts managing all kinds of tokens allow custom `call()`, tokens inside them are all in danger.

An example of second one: The hacker in ATN incident made use of the contract's identity to bypass `ds-auth` authority restriction.

An example of third one: Imagine that a user X approve buggy contract A of managing 10,000 B tokens, a hacker could steal B calling `transferFrom` via this bug.

### Inconsistency Between ERC223's Draft & Interface

With further investigation, we found no description of using `_custom_fallback` in ERC223 interface draft.

This is the interface defined by the draft:

```js
contract ERC223Interface {
    uint public totalSupply;
    function balanceOf(address who) constant returns (uint);
    function transfer(address to, uint value);
    function transfer(address to, uint value, bytes data);
    event Transfer(address indexed from, address indexed to, uint value, bytes data);
}
```

No parameters called `_custom_fallback` appear in two `transfer()` interfaces.

Let us inspect the draft description:

>  If the receiver is a contract ERC223 token contract will try to call tokenFallback function on receiver contract. If there is no tokenFallback function on receiver contract transaction will fail. tokenFallback function is analogue of fallback function for Ether transactions. It can be used to handle incoming transactions.

The gist is that if the token receiver is an ERC223 contract then call its `tokenFallback()`. Fail the trade if the target has no `tokenFallback()`. Here `tokenFallback()` works as a default `fallback` in Ethereum transaction.

ERC223 draft has an explicit goal: defining `tokenFallback()` for Token contracts for handling received tokens. While there is no `_custom_fallback` in the main branch of ERC223 code, the Recommended branch introduces an implementation of `transfer()` with `_custom_fallback` without warnings.

### Other Bugs in ERC223 Implementation

In fact, ERC223 Recommended branch has other bugs.

When dealing with bytes, `call()` would trigger a bug in EVM level, causing inconsistency in data [6].

```js
// ERC223_Token.sol#L70
assert(_to.call.value(0)(bytes4(keccak256(_custom_fallback)), msg.sender, _value, _data));
```

Under certain circumstances, errors would arise in event handling indexed bytes variables [7].

```js
// ERC223_Interface.sol#L18
event Transfer(address indexed from, address indexed to, uint value, bytes indexed data); 
```

It is safe to say ERC223 Recommended branch is not reliable, please try not to apply the code.

## Mechanism of Passing Parameters in EVM

To better understand this vulnerability, we would explore EVM's mechanism of calling functions and passing parameters. Take a look at this sample

```js
contract A{
    function transfer(address to, uint256 value){
  	      return;
    }
}
```

First comes an introduction to parameter passing in EVM: When calling a function, if it has parameters, we need to construct the input according to types appointed by ABI normally. For example, if Ethereum invokes `transfer()` in the form of `transfer(address to, uint256 value)`, it takes first 4 bytes of function signature's hash value as `function selector` and computes `sha3(transfer(address,uint256))`, then the result is `0xA9059CBB`, plus the address of `to`, the 256 bits becomes

```
0x0000000000000000000000003f5ce5fbfe3e9af3971dd833d26ba9b5c936f0be
```

`value` is also plugged into 256 bits computation

```
0x000000000000000000000000000000000000000000000000000000e8d4a51000
```

Finally we have a full calldata:

```
0xa9059cbb0000000000000000000000003f5ce5fbfe3e9af3971dd833d26ba9b5c936f0be000000000000000000000000000000000000000000000000000000e8d4a51000
```

Send the transaction along with the calldata, Ethereum could complete function calling. When an Ethereum node receives a request, it loads calldata and smart contract byte code into EVM. Byte code gets generated in compilation such that processing parameters are done in the mean time. The byte code only checks if `calldata` is shorter than a minimum requirement rather than if it is too long. The compiler would generate a series of `CALLDATALOAD` with mathematical operations to extract parameters required by the function. First it computes the target function invoked:

`CALLDATALOAD` instruction would load the calldata (`0xa9059cbb0000000000000000000000003f5ce5fbfe3e9af3971dd833d26ba9b5c936f0be000000000000000000000000000000000000000000000000000000e8d4a51000`) of the trade into the stack and divide the first 256 bits by `0x100000000000000000000000000000000000000000000000000000000`, then gets `0xA9059CBB`. Every other parameter would be extracted in a similar way. However, byte code and EVM would not process parameters when there are too many of them and leave out this step instead. All in all, this feature originates from the compiler. The hacker could easily construct attacking parameters on CUSTOM_CALL.

## Risky ERC827 Implementation

The similar ERC827 Token draft possesses this issue as well [8]. Code below comes from an ERC827 buggy implementation by **openzeppelin-solidity**:

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

After the program finished transaction in `transferAndCall()`, it would call a function on `_to` address with parameters set by callers. Since the check `_to != address(this)`, the code cannot bypass the authority check combined with `ds-auth` library(`second result`), while might introduce `first result` and `second result` above, managing buggy contracts' tokens(attack other contracts via 'this' contract).

Aside from this, numerous ERC20 Token contracts implement similar `call()` which is highly risky. It permits attackers stealing contracts' tokens and bypassing authority checks in some cases.

## Correct ERC20 & ERC721 Implementation on 'receiverCall'

A correct 'receiverCall' program should hard-code the signature of called function to prevent from getting **arbitrarily appointed** by an attacker. Here are 2 correct 'receiverCall' examples:

1. Conduct 'receiverCall' via declaring Receiver function

Take ERC20 code maintained by ethereum.org as an example:

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

'receiverCall' acts like a normal function call.

2. run 'receiverCall' by **signature constant** of Receiver function

This correct snippet comes from Token-Factory by ConsenSys

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

There are some repositories implementing 'receiverCall' correctly

+ https://github.com/svenstucki/ERC677
+ https://github.com/OpenZeppelin/openzeppelin-solidity/blob/master/contracts/token/ERC721/ERC721BasicToken.sol#L349
+ https://github.com/ConsenSys/Token-Factory/blob/master/contracts/HumanStandardToken.sol
+ https://github.com/ethereum/ethereum-org/blob/b46095815f52cf328ecf7676b2b38284d48fba58/solidity/token-advanced.sol#L138

*Special thanks to Yuhui Wu from Qingxin Tech who discussed the topic with us and provided feedback and comment. For more information, check out the [Awesome Buggy ERC-20 Tokens](https://github.com/sec-bit/awesome-buggy-erc20-tokens) open-source repo maintained by SECBIT Labs*.

## Conclusion

- ERC223 standard acts differently from interface definition and two branches in official repo are implemented inconsistently. Be cautious with **official code**
- ERC827 is vulnerable as well
- Be careful using low-level call
- EVM's mechanism on calling contract functions needs to be well understood

## Reference

- \[1\][ATN Report](https://atn.io/resource/aareport.pdf)
- \[2\][OpenZeppelin removed ERC827](https://github.com/OpenZeppelin/openzeppelin-solidity/pull/1045)
- \[3\][ERC-223 Token Standard Proposal Draft](https://github.com/ethereum/EIPs/issues/223)
- \[4\][ATN.sol transferFrom()](https://github.com/ATNIO/atn-contracts/blob/7203781ad8d106ec6d1f9ca8305e76dd1274b181/src/ATN.sol#L114)
- \[5\][ERC223_Token.sol transfer() function](https://github.com/Dexaran/ERC223-token-standard/blob/16d350ec85d5b14b9dc857468c8e0eb4a10572d3/ERC223_Token.sol#L70)
- \[6\][ERC223-token-standard Issue 50](https://github.com/Dexaran/ERC223-token-standard/issues/50)
- \[7\][ERC223-token-standard Issue 51](https://github.com/Dexaran/ERC223-token-standard/issues/51)
- \[8\][ERC827Token.sol](https://github.com/OpenZeppelin/openzeppelin-solidity/blob/f18c3bc438b366f9cb3a8613f5be160c2cbced5e/contracts/token/ERC827/ERC827Token.sol#L73)
- \[9\][New evilReflex Bug Identified in Multiple ERC20 Smart Contracts (CVE-2018-12702, CVE-2018-12703)](https://peckshield.com/2018/06/23/evilReflex/)
- \[10\][HADAX Suspends 18T and GVE Deposits and Withdrawals](https://huobiglobal.zendesk.com/hc/en-us/articles/360000110521-HADAX-Suspends-18T-and-GVE-Deposits-and-Withdrawals)

