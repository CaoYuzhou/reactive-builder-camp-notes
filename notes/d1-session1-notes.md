# D1｜Online Session 1 回放笔记（开营 + Q&A）

- 时间：2026-11-16（周一）20:00 北京时间，回放：https://x.com/i/broadcasts/2JQeMJmQnXdrYK （约 51 分钟）
- 分享人：Reactive 黑客松评委 Constantine；ETHPanda 介绍本期安排

## 三条笔记要点

1. **Reactive 解决的是"谁来盯链、谁来发交易"的问题。** 一句话概括就是：链 A 上发生了某个事件 → Reactive 合约在 `react()` 里跑你的 Solidity 判断逻辑 → 向任一支持的目标链发回调。原本要自己跑机器人 / Keeper 的事，交给网络来做。最适合的项目是由链上事件触发、能替代一个本来要自己跑的机器人、通常监听一条链去另一条链执行、关键路径留在链上；只依赖链下数据（合约根本不 emit 事件）或 Reactive 只是装饰的项目不适合。

2. **Omni 版本（10 月底主网迁移）改了很多写法，旧教程和 AI 生成的代码要小心。** CometBFT 共识，Reactive 上即时最终性、约 1 秒出块；不再有 ReactVM 双状态，Reactive 合约就是普通智能合约；`react()` 从 `vmOnly` 改成 `onlySystem`，事件由系统合约 `0x8888…8888` 送达；cast 等标准 EVM 工具可以直接用。

3. **动手前要知道的坑：** Reactive 合约为 `react()` 付费、目标链回调合约为回调付费，有欠款就停止执行；回调可能失败（目标合约 revert 时代理会 emit `CallbackFailure`），目标函数要写得简单、不会意外 revert；源链事件一旦被观察到就视为最终，同一事件可能来两次，要防止重复执行；Base Sepolia 现在 200ms 出块，计时要用时间戳而不是区块数。

## 其他记录

- **Reactive Omni Buildathon**：11/5–12/5，奖金 5,500 美元（以 REACT 支付），在 DoraHacks 报名和提交。三个赛道：
  - 赛道 1 Every Second Counts（$2,000 + $250）：Lasna + Unichain Sepolia（1 秒出块）/ Base Sepolia（200ms），套利、预测市场、再平衡金库等；希望看到全链路延迟实测。
  - 赛道 2 Set It and Forget It（$2,000 + $250）：任一支持链上的个人自动化，比如组合追踪、LP 价格提醒、自定义再平衡规则；不需要合约经验，可以用 AI 编程工具；希望看到"你敢让它自己跑"的工具。
  - 赛道 3 Reactive Meets AI（$1,000）：AI 必须是运行系统的一部分，而不只是写代码的工具。
- **评审标准**：Execution（链上能看到 reactive 交易和回调，没有就不继续评分）、Integration（Reactive 合约真的在干活）、Usefulness（解决真实问题）、Craft（代码可读、Demo 能跑、文档清楚）。
- **本期安排**：4 场 Online Session（11/16 开营、11/19 技术 Q&A、11/25 项目讨论、11/29 Demo Day）；提交凭证是 GitHub 学习仓库 / 合约地址 / 交易哈希；**结业 = 核心任务 ≥ 600 分 + 两个必做任务**；WCB 上做任务不等于报名黑客松，DoraHacks 需要自己报名提交。
- 资料：开发文档 dev.reactive.network；支持链 dev.reactive.network/origins-and-destinations；模板仓库 github.com/ethpanda-org/reactive-builder-camp
