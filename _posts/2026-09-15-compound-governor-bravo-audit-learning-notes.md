---
layout: post
title: "读懂我的第一份智能合约审计报告：Compound Governor Bravo 学习总结"
date: 2026-09-15
description: "从治理状态机、Timelock、代理存储和输入校验出发，记录我第一次完整学习智能合约审计报告时纠正的几个误解。"
tags: [Solidity, Smart Contract Security, Compound, Governor Bravo, Audit Learning]
permalink: /governor-bravo-audit-learning-notes/
image: /assets/images/governor-bravo-hero.png
---

# 读懂我的第一份智能合约审计报告：Compound Governor Bravo 学习总结

> 这是一篇学习记录，不是我独立完成的安全审计。文中的发现来自 OpenZeppelin 已公开的历史审计报告；我的工作是对照当时的代码和修复记录，理解问题为什么成立，并记录自己从误解到理解的过程。

## 为什么从 Governor Bravo 开始

我有 C++ 开发经验，也接触过 Ethereum、Solana、DEX 和预测市场，但第一次系统阅读 Solidity 审计报告时，代理、`delegatecall`、Timelock 和治理状态机仍然是明显的知识缺口。

我选择 [OpenZeppelin 的 Compound Governor Bravo 审计报告](https://www.openzeppelin.com/news/compound-governor-bravo-audit)，是因为它的审计范围只有三个治理相关文件，报告结构清楚，也没有 AMM、借贷或预言机的大量数学背景。它很适合用来建立第一个“报告 → 源码 → 修复 → 可迁移检查项”的学习闭环。

OpenZeppelin 审计的是 Compound Protocol 的 commit [`f86c247f6f81e14f8e0fd78402653a0b8371266a`](https://github.com/compound-finance/compound-protocol/tree/f86c247f6f81e14f8e0fd78402653a0b8371266a/contracts/Governance)，范围包括：

- `GovernorBravoDelegate.sol`：治理逻辑；
- `GovernorBravoDelegator.sol`：可升级代理；
- `GovernorBravoInterfaces.sol`：共享存储布局、事件、结构体和接口。

报告统计为 0 个 Critical、0 个 High、2 个 Medium、5 个 Low 和 5 个 Note。对我而言，最有价值的不是严重漏洞，而是这些问题共同说明了：治理系统的安全性依赖状态边界、参数边界、迁移过程和升级兼容性。

## 我先建立的系统模型

Governor Bravo 在 Governor Alpha 的治理机制上增加了可调的 `proposalThreshold`、`votingDelay` 和 `votingPeriod`，还增加了弃权票、投票理由和治理逻辑升级能力。

这里的调用和状态关系可以简化成：

<figure class="article-figure">
  <img src="{{ '/assets/images/governor-bravo-architecture.svg' | relative_url }}" alt="Governor Bravo 的代理、治理逻辑、存储与 Timelock 关系图">
  <figcaption>代理提供稳定入口并保存状态，Delegate 提供逻辑，Timelock 控制通过提案的延迟执行。</figcaption>
</figure>

我最初把报告中“inherits many of the mechanisms from Governor Alpha”理解成 Solidity 的合约继承。对照源码后才发现，`GovernorBravoDelegate` 实际继承的是存储和事件定义，而不是 `GovernorAlpha`。报告表达的是机制沿用，不是 `contract GovernorBravoDelegate is GovernorAlpha` 这种语言层面的继承。

## 提案通过，不等于参数已经生效

我一开始没有把 `Succeeded`、`Queued` 和 `Executed` 分清，甚至把 Timelock 和投票窗口、重复交易检查混在一起。源码把这几个边界写得很明确：

<figure class="article-figure">
  <img src="{{ '/assets/images/governor-bravo-lifecycle.svg' | relative_url }}" alt="Governor Bravo 提案从 Pending、Active 到 Succeeded、Queued、Executed 的状态图">
  <figcaption>投票通过、进入 Timelock 队列和实际执行是三个不同的安全边界。</figcaption>
</figure>

- `votingDelay` 和 `votingPeriod` 决定何时开始投票、投票持续多久；
- 投票通过后，提案只是 `Succeeded`，还没有改变目标合约的状态；
- 调用 `queue` 会设置 `eta`，此后状态才是 `Queued`；
- Timelock 延迟结束不会自动执行，仍然需要显式调用 `execute`；
- 只有 `execute` 成功，提案中的参数修改或其他操作才真正生效。

我现在对 Timelock 的理解是：它把“治理同意”与“操作生效”拆开，给参与者留下观察、评估和应对异常提案的时间。重复动作检查只是排队路径上的一个约束，投票窗口也由另外两个参数控制；它们都不是 Timelock 延迟本身的目的。

## 报告中最值得保留的几个问题

### M01：有权限，不代表参数一定安全

审计版本的构造参数缺少系统性的输入检查，`_setVotingDelay` 和 `_setVotingPeriod` 也可以接受任意 `uint`。例如，如果把 `votingPeriod` 设置为 `0`，后续提案的 `startBlock` 和 `endBlock` 会相同；按 `state` 函数的区块判断，它不会获得正常的 `Active` 投票窗口，治理可能因此无法正常推进。

我最初的错误是把“这个值在业务上不合理”当成“合约不会接受这个值”。EVM 不理解治理语义：只要授权调用能够到达，代码又没有显式拒绝，`0` 就是一个可以写入 storage 的普通数值。

这也纠正了我对访问控制的另一个误解：

- access control 回答“谁可以改”；
- input validation 回答“可以改成什么”。

即使调用者是 Timelock、治理合约或多签，也仍然可能发生提案参数写错、编码错误或权限被滥用。`onlyAdmin` 不能代替上下界检查，反过来也一样。

OpenZeppelin 将 M01 标为 Medium。报告记录的 [修复 PR #5](https://github.com/Arr00-Blurr/compound-protocol/pull/5) 为治理参数增加了合理范围，并排除了部分零地址；但 `_setImplementation` 仍然没有输入检查，因此报告将其状态写为 **Partially fixed**。这里不能把“加入了一些校验”写成“整个问题已经彻底解决”。

### M02：重复交易限制是一条接口设计约束

提案排队时，Governor Bravo 会根据 `target`、`value`、函数签名、calldata 和 `eta` 计算动作标识，并拒绝已经排队的相同动作。因此，同一执行时间下的完全相同调用不能重复出现。

这看起来不像典型的“攻击者窃取资产”漏洞，更像治理接口与 Timelock 组合后产生的约束：未来为治理设计函数时，不能默认同一提案可以用相同参数连续调用同一个动作。OpenZeppelin 的建议也是把这一行为明确记录下来，让后续开发者在接口设计时考虑它。

这个 finding 让我意识到，审计报告不仅检查能不能被攻击，也检查系统行为是否被准确表达。未记录的限制会在未来集成或升级时变成可靠性风险。

### L01：治理迁移会冻结所有未执行的 Alpha 提案

我起初把这个问题理解成“迁移时漏掉了某一个提案”。实际情况更系统化：Bravo 会继承 Alpha 当前的 `proposalCount`，并把它保存为 `initialProposalId`；而 Bravo 的 `state` 只接受大于 `initialProposalId` 的新提案 ID。因此，迁移时所有尚未执行的 Alpha 提案都不能再通过 Bravo 的状态机执行。

这不是单个数据遗漏，而是旧状态机与新状态机之间的迁移兼容性问题。报告建议在升级前明确告知用户，必要时在 Bravo 重新提案，或者为已在 Timelock 排队的旧提案设计专门的执行方式。

### L02：`initialize` 不能只靠名字保证只执行一次

审计版本的 `initialize` 是 `public`，只检查 `msg.sender == admin`，没有“一次且仅一次”的状态约束。它可以再次修改 `timelock`、`comp` 和三个治理参数。

我以前容易把 `initialize` 当成天然等价于构造函数。代理模式下并非如此：实现合约的初始化通常是一次普通的外部调用，是否只能调用一次必须由状态检查或 initializer 机制保证。这个问题后来在 [修复 PR #4](https://github.com/Arr00-Blurr/compound-protocol/pull/4) 中加入一次性约束，报告标记为 **Fixed**。

### N05：升级真正保留的是 slot，不是变量含义

`GovernorBravoDelegator` 通过 `delegatecall` 执行 Delegate 的逻辑。执行过程中：

- 代码来自 Delegate；
- storage 和合约地址属于 Delegator；
- `msg.sender` 仍是调用代理的原始调用者。

因此，仅替换实现地址不会自动清空治理状态。但这不等于升级一定安全：如果新实现改变了变量顺序或类型，它会用新的含义解释代理里原有的 slot，旧数据还在，却可能被读成完全不同的变量。

OpenZeppelin 没有在审计版本中发现已发生的 storage collision，但指出自定义代理缺少对 function clashing 的控制，而且存储变量类型混排会增加未来升级出错的概率。[修复 PR #9](https://github.com/Arr00-Blurr/compound-protocol/pull/9) 对存储声明进行了更可预测的排序和说明，报告将该项标为 **Fixed**；报告结论仍然建议优先采用成熟的代理实现，而不是自行设计代理。

## 这次学习改变了我的读报告方式

第一次读完时，我关注的是“有几个 Medium、有没有严重漏洞”。回头看，真正形成方法的是下面四个步骤：

1. 先画出模块职责、权限和状态存放位置，不急着读 finding 标题。
2. 把每个 finding 拆成前置条件、代码行为、被破坏的性质、影响和修复。
3. 分清报告中的事实、我从代码得到的推论，以及报告之外的扩展知识。
4. 对照审计 commit 和修复 PR，确认修复状态，不能拿现在的代码替代当时的上下文。

以后再看治理或可升级合约，我会固定检查：

- 可调参数是否同时有调用者限制和合理的上下界；
- `Succeeded`、`Queued`、`Executed` 是否被清楚区分；
- Timelock 延迟、过期窗口和重复动作规则是否被文档化；
- initializer 是否只能执行一次，初始化是否可能被抢先或重复调用；
- 迁移时旧提案、旧权限和待执行操作如何处理；
- 新旧实现的 storage layout 是否兼容；
- implementation 地址的升级入口是否验证目标并有清晰的治理流程。

## 仍然没有假装掌握的部分

这份报告让我建立了代理与治理的基本模型，但我还没有通过动手实验完整验证 storage layout 升级，也没有彻底掌握 fallback 中 assembly 对返回数据和 revert 的转发细节。这些内容目前只能列为后续学习项，不能因为读过一份报告就写成自己的熟练能力。

第一篇文章的价值也正在这里：它不是证明我已经是审计员，而是留下可复查的证据——我在哪里理解错了，源码如何推翻了原来的直觉，以及下一次审计时我会多检查什么。

## 参考资料

- [OpenZeppelin：Compound Governor Bravo Audit](https://www.openzeppelin.com/news/compound-governor-bravo-audit)
- [Compound Protocol：审计 commit 与范围代码](https://github.com/compound-finance/compound-protocol/tree/f86c247f6f81e14f8e0fd78402653a0b8371266a/contracts/Governance)
- [GovernorBravoDelegate.sol（审计版本）](https://github.com/compound-finance/compound-protocol/blob/f86c247f6f81e14f8e0fd78402653a0b8371266a/contracts/Governance/GovernorBravoDelegate.sol)
- [GovernorBravoDelegator.sol（审计版本）](https://github.com/compound-finance/compound-protocol/blob/f86c247f6f81e14f8e0fd78402653a0b8371266a/contracts/Governance/GovernorBravoDelegator.sol)
- [GovernorBravoInterfaces.sol（审计版本）](https://github.com/compound-finance/compound-protocol/blob/f86c247f6f81e14f8e0fd78402653a0b8371266a/contracts/Governance/GovernorBravoInterfaces.sol)
- [M01 修复 PR #5](https://github.com/Arr00-Blurr/compound-protocol/pull/5)
- [L02 修复 PR #4](https://github.com/Arr00-Blurr/compound-protocol/pull/4)
- [N05 修复 PR #9](https://github.com/Arr00-Blurr/compound-protocol/pull/9)
