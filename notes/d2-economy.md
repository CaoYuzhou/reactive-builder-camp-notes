# D2｜读懂费用：用 cast 查询 Lasna 系统合约参数

- 网络：Reactive Lasna 测试网（chainId 5318007）
- RPC：https://lasna-rpc.rnk.dev/
- 系统合约：`0x8888888888888888888888888888888888888888`
- 查询时间：2026-11-17（Lasna 区块 8522774 前后）

## 三条 cast 命令和输出

```bash
export REACTIVE_RPC=https://lasna-rpc.rnk.dev/
export SYSTEM_CONTRACT_ADDR=0x8888888888888888888888888888888888888888

cast call $SYSTEM_CONTRACT_ADDR "_gasPriceCoeffPer1000()" --rpc-url $REACTIVE_RPC | cast to-dec
# 1000

cast call $SYSTEM_CONTRACT_ADDR "_extraGas()" --rpc-url $REACTIVE_RPC | cast to-dec
# 100000

cast call $SYSTEM_CONTRACT_ADDR "_maxChargeGas()" --rpc-url $REACTIVE_RPC | cast to-dec
# 50000
```

不加 `| cast to-dec` 时，`cast call` 返回的原始值：

| 参数 | 原始返回（uint256） | 十进制 |
|---|---|---|
| `_gasPriceCoeffPer1000()` | `0x…03e8` | 1000 |
| `_extraGas()` | `0x…0186a0` | 100000 |
| `_maxChargeGas()` | `0x…c350` | 50000 |

说明：本机 Foundry 还在安装中，这次的输出是对同一 RPC 发 `eth_call`（`cast call` 底层发的就是这个请求）拿到的，数值与 `cast call` 的输出一致；装好 Foundry 后会用上面三条命令重跑一遍确认。

三个值和文档写的默认值一致。另外查了一下当时的 gas 价格：`cast gas-price --rpc-url $REACTIVE_RPC` → 100000000000（100 gwei）。

## 1. 一次 react() 的费用怎么算？

```
fee = p_base × C × (g_used + K)
```

- `p_base`：基础 gas 价格，取 `tx.gasprice` 和 `block.basefee` 中较大的那个（现在 Lasna 是 100 gwei）
- `C`：系数 = `_gasPriceCoeffPer1000 / 1000` = 1000 / 1000 = **1**（不打折也不加价）
- `g_used`：这次 `react()` 实际消耗的 gas
- `K`：固定附加 gas = `_extraGas` = **100,000**

举例：一次 `react()` 实际用了 50,000 gas
→ fee = 100 gwei × 1 × (50,000 + 100,000) = 15,000,000 gwei = **0.015 lREACT**

几个推论：
- 固定附加的 100,000 gas 比一次简单 `react()` 本身还贵，**每次触发都有一笔"起步价"**。订阅很频繁的事件（比如热门池子的 Sync、全部代币的 Transfer）很快会把钱烧完，所以要设次数上限
- `_maxChargeGas`（50,000）**不进入费用公式**，它是系统合约调用 `pay()` 结算欠款时给的 gas 上限。自定义的 `pay()` 逻辑超过这个预算会失败，留下欠款
- 用 1 ETH 换 100 lREACT 的话，按上面例子大约够 6,600 次 `react()`

## 2. balance、reserves、debt 分别是什么？为什么有一点 debt 就停止执行？

| | 放在哪里 | 含义 |
|---|---|---|
| **balance** | 合约自己 | 合约自己能花的钱（直接往合约转账就会增加） |
| **reserves** | 系统合约（或目标链的 Callback Proxy） | 预付的手续费额度，用 `depositTo(合约地址)` 存进去，只能用来付费 |
| **debt** | 系统合约（或 Callback Proxy） | 已经产生但还没付的费用 |

收费顺序：先扣 reserves → 不够就请合约用自己的 balance 通过 `pay()` 付 → 还付不出就记成 debt。存钱时先还 debt，剩下的才进 reserves，所以一个合约不会同时有 reserves 和 debt。

**为什么有一点 debt 就停：** 系统合约处理每一笔 reactive 交易时，**第一步就是检查 debt**，在花任何 gas 之前。只要 debt > 0，系统合约就不调用 `react()`，Callback Proxy 也不投递回调。这样网络不会替一个付不起钱的合约继续干活。合约变成 inactive 时会 emit `BlacklistContract`，没有报错，只是"安静地停了"，所以需要自己监控。补救方法：用 `depositTo()` 存钱（先还清欠款），或者合约有 `coverDebt()` 的话用自己的 balance 还。直接往欠款的合约转账不能还债，钱可能卡在里面。

## 3. react() 回滚了还收费吗？对合约设计意味着什么？

**照样收费，不退。** 系统合约是在执行之后按实际消耗的 gas 收费的；`react()` 回滚会被捕获、记一条 `ReactiveContractReverted` 事件，然后照常扣费。回调失败也一样（记 `CallbackFailure`，照收）。

对设计的影响：
- **不要用 `require` / `revert` 来"过滤"不需要的事件。** 条件不满足时直接 `return`，花的 gas 更少，而且不会制造一堆失败记录
- **订阅尽量精确**：能用 topic1（indexed 参数）过滤的就在订阅里过滤，少收无关事件，每个事件都要付至少 100,000 gas 的附加费
- **加次数上限和金额上限**，达到上限后自动取消订阅，避免一个 bug 把余额烧光
- **目标链的回调函数写简单**，不要轻易 revert；失败也要付费，而且同一事件可能被重复送达，要做幂等（比如记录已处理的事件，避免重复执行）
- **两边都要留钱并监控余额**：Reactive 合约用 lREACT，回调合约用目标链的 ETH

参考：[Economy & System Interactions](https://dev.reactive.network/economy)、[Origins & Destinations](https://dev.reactive.network/origins-and-destinations)
