---
title: "'Move' Programming Language: The Highlight of Libra"
date: 2019-07-10T08:00:00+08:00
summary: "Among publications of Libra, the most attractive thing would be the brand new 'Move' programming language. After the first glance at 'Move' whitepaper, we came to the idea that this is the future of smart contract language."
featuredImage: "cover1.png"
slug: "'Move' Programming Language: The Highlight of Libra"
author: Guo Yu
tags: ["Facebook", "Libra", "Smart Contract", "Programming Language"]
categories: ["Libra", "Programming Language", "Smart Contract"]
contentCopyright: "If you need to reprint, please indicate the author and source of the article."
---

Abstract: Would Facebook bring dawn to smart contracts?


Everyone must have been bombarded by Facebook news recently.

Libra, the cryptocurrency project started by Facebook, got published officially on June 18th along with its website and whitepaper offering global currency & financial service to billions of people. Also, it has several technical whitepapers to explain the new programming language 'Move' and consensus protocol -- LibraBFT. Its source code has been published on GitHub, and the test network has gone online. Libra is designed as a permissioned blockchain as the developing team states that no mature solution in the permissionless (public) blockchain is available for billions of people and transition to the public blockchain would begin in five years.

Among publications of Libra, the most attractive thing would be the brand new 'Move' programming language. After the first glance at 'Move' whitepaper, we came to the idea that this is the future of smart contract language.

Lefteris Karapetsas, a developer from Berlin, said on Twitter:

> Their design goals seem to overlap, or even aim to replace Ethereum? 
> 

CryptoPuzzleDream, the founder of PuzzleToLife.com, holds this opinion:

> I think 'move' programming language released by $FB could be more exciting than Libra
> 

A standard geek James Clark states:

> I'm usually pretty skeptical of anything related to cryptocurrency, but here's one piece of Libra that looks potentially interesting: a bytecode programming language called Move with semantics inspired by linear logic. 
> 

This is the very thought that just came across our mind:

> 'Move' is the smart contract platform language designed for digital assets.


## Three Applications of 'Move'

1. Issuing cryptocurrencies, tokens, and digital assets
2. Handling blockchain transactions
3. Managing validators


## The Bottom-up Static Typing

The static typing of Move, which is essentially a logical constraint, is much stricter than Ethereum smart contract language. The modern programming languages like Rust, Golang, Typescript, Haskell, Scala, OCaml all select static typing due to the advantage that many simple program bugs could be detected during compilation instead of execution.

> Well-typed programs never get stuck. 

This is a technical jargon in programming languages saying that the code would be highly reliable if it passes type checking.

However, 'Move' is not 100% statically typed, due to practicability requirements. It tries to perform type checking during compilation rather than deployment. Some type checking tasks have to be postponed during execution, while type safety is guaranteed.

One excellent design concept of 'Move' is that its virtual machine is statically typed. The layer above is IR (Intermediate Representation), which is also typed. The top layers of 'Move' would offer more high-level languages to financial applications adhering to static typing to avoid simple bugs in smart contracts.

## First-class Resources

The word `First-class Resources` seems pretty much academic. So what is the meaning? The so-called first class in programming language refers to prior programmed objects during coding.

So what does `Resources` refer to? It is another academic term about `value`. `Value` could be copied arbitrarily while `Resources` could just get consumed, not copied. `Resources` are similar to cokes that decrease if someone drinks daily. `Value`, on the contrary, are words on a notebook. You and other people could read and copy them every day in your mind while words would not disappear.

Traditional programming languages, including Ethereum smart contract language, record digital assets in `values` which might be incorrect numbers. In fact, there are plenty of wrong records, e.g., Jack transferred to John, and John gets 10 dollars while Jack's balance remains. Many issues in this field significantly reduced people's trust in the smart contract future.

Instead, the 'Move' contract applies the 'resource type' integrating the traditional linear logic. Digital assets are defined by resource types so that they could fit attributes in linear logic:

1. Digital assets could not be copied
2. Digital assets could not vanish

The real principle of `First-class Resources` is that `digital assets are first-class citizens`. Now we could infer that 'Move' is the smart contract language designed exactly for managing digital assets. Technically, digital assets could be variables in contracts to be stored, assigned, and become parameters/return values of functions & processes. The static typing of 'Move' enables the compiler to examine most errors of resources during compilation and before deployment, enhancing smart contracts' security.

Here is one sentence selected from the whitepaper abstract:

> First-class resources are a very general concept that programmers can use not only to implement safe digital assets but also to write correct business logic for wrapping assets and enforcing access control policies.


## Contract Security Design

'Move' contract design is properly integrated with security. First of all, 'Move' does not support dynamic dispatch at all. Here is an explanation of dynamic dispatch - a flexible language mechanism: we could write lots of functions, processes, or subprograms in the code for the main program calling to perform different tasks. Suppose that we could know which function the program would call or the calling sequence, then these calls are static. On the contrary, if we have no idea about function calling details until program execution, then it would be dynamic. Obviously, dynamic dispatch is much more flexible than static dispatch.

Nevertheless, flexibility means more bugs. Many modern programming languages support dynamic dispatch to an extent on the level of language itself, e.g., dynamic binding caused by inheritance in an object-oriented language. These dynamic properties make program reasoning and formal verification harder and the program itself riskier. There are many dynamic properties in Ethereum smart contracts, such as delegatecall, parameters in the form of contracts and function pointers, while 'Move' supports none of the dynamic dispatch or property. All contract execution path could be determined during compilation to be thoroughly analyzed and verified.

Before execution, 'Move' contracts would get verified by a bytecode verifier for all types of bugs. In the meantime, the bytecode undergoes interpretation with types for execution and verification.

'Move' has stringent restrictions on mutable contract variables and borrows some design concepts from Rust. At any time, only one pointer could modify the mutable variables to avoid messing up data. In Ethereum Solidity, you could define many pointers for only one variable, leading to bugs once the code logic is not designed correctly.

'Move' modular system does support cyclic recursion dependency compared to EVM, so there would be no re-entrancy bug.

## Advanced Module System

'Move' module system is designed in the style of a functional programming language (OCaml, Coq, SML), according to the whitepaper:

> Move modules are similar to smart contracts in other blockchain languages. ..., However, modules enforce strong data abstraction — a type is transparent inside its declaring module and opaque outside of it.

The module system could encapsulate the digital asset concept perfectly. We could manage digital assets via public module interfaces along with their authorities flexibly. The ERC20 token of Ethereum is in the form of a contract, while a token of 'Move' is a box passed freely like a resource without showing details inside. Meanwhile, the module system abstraction is based on its static typing, and type security could be guaranteed by smart contract virtual machine verification.

The 'Move' module system offers a solid base for formal verification, as we could define invariants inside the module. The invariant is a strict restraint of digital asset state providing valuable information for formal verification automation. Additionally, the black-box abstraction of module systems enables the formal verification to be modularized, costing less. It would be much easier to write program analyzers and symbolic executors based on the 'Move' module system, as the abstraction makes the contract logic simple for inference.


## The Future 'Move' Smart Contract

Although 'Move' seems rough and immature, this direction is exciting. We could see from the 'Move' language that Facebook desires to build a grand digital asset platform, which originally belongs to future Ethereum.

Why do we love 'Move'? Here are three reasons:

1. Integrating programming language researching results and EVM smart contract language experience
2. Prioritizing the security and correctness of smart contracts
3. Innovating smart contract language design for financial apps rather than following routines (not applying WASM, LLVM or modifying EVM)

> No traditional scheme stands out in blockchain; the future is all about innovation.
