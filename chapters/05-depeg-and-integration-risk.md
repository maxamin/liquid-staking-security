# Depeg History and Integration Risks

An LST has two prices. The **redemption rate** is what the protocol would pay per token through its own withdrawal path. The **market price** is what someone will pay on a DEX or exchange right now. They usually sit close together, because arbitrageurs buy cheap LSTs and redeem them. When redemption is slow, blocked or doubted, the two separate. Integrations that confuse the two get hurt.

## The stETH discount of 2022

Before the Shapella upgrade, stETH could not be redeemed at all. Its market price was held near 1 ETH only by the expectation of future redemption and by liquidity in the Curve stETH/ETH pool. In May and June 2022, during the Terra collapse and the stress around lenders and funds such as Celsius and Three Arrows Capital, large holders sold stETH to raise ETH. Public data and press reports show stETH trading at a discount of several percent to ETH for weeks, with the Curve pool becoming heavily imbalanced toward stETH.

Nothing was wrong with Lido's accounting. The discount came from liquidity and time: an exit that would take months is worth less than ETH today. Leveraged positions that looped stETH as collateral to borrow ETH (for example on Aave) faced liquidation risk if lending markets priced stETH from the market, and bad-debt risk if they did not.

After withdrawals were enabled in 2023, the arbitrage path became real, and discounts have generally been smaller and shorter. They have not disappeared: exit queue length, oracle delays and a protocol pause can all reopen the gap.

## Pricing an LST in an integration

| Method | Pros | Cons |
|---|---|---|
| Hard-coded 1:1 with ETH | Simple, no oracle | Wrong for reward-bearing tokens; ignores depeg; bad debt in a run |
| Protocol exchange rate (`stEthPerToken`, `getExchangeRate`) | Not manipulable by DEX trades; tracks rewards | Ignores market discount; trusts the LST's own oracle; can be pushed if the LST's accounting is attackable |
| Market price feed (LST/ETH or LST/USD) | Reflects real exit value | Thin markets manipulable; feed may lag |
| Exchange rate with cap on growth (e.g. Aave's capped price adapters) | Limits damage from inflated rate | Still ignores market discount |
| Min(exchange rate, market price) | Conservative | Can trigger liquidations on brief market wicks |

There is no universally right answer. Choose by the integration's role:

- **Lending collateral** (you lose if overvalued): use a conservative price, cap rate growth, and set loan-to-value with the historical discount in mind.
- **Borrowed asset** (you lose if undervalued): the opposite bias applies.
- **AMM or vault share pricing:** never use the pool's own spot price; see **oracle-security-engineering**.

```solidity
// Capped exchange-rate adapter (illustrative)
function price() external view returns (uint256) {
    uint256 rate = IWstETH(wsteth).stEthPerToken();
    uint256 maxRate = snapshotRate + snapshotRate * maxYearlyBps * (block.timestamp - snapshotTime) / (365 days * 10_000);
    return rate > maxRate ? maxRate : rate;
}
```

## Other integration hazards

- **Rebasing in vaults.** A vault holding stETH sees its balance grow. If it tracks deposits in its own storage, the surplus may be claimable by the first caller of some sweep function, or stuck forever. Wrap to wstETH or account in shares.
- **Transfer rounding.** stETH transfers can deliver 1–2 wei less than requested. Code that requires `received == amount` reverts.
- **Rate manipulation through donation.** For exchange-rate LSTs whose rate is `totalAssets / totalSupply` with `totalAssets` read from a balance, donating ETH can move the rate. This is the ERC-4626 inflation attack pattern; check whether the LST's rate is reporting-based or balance-based.
- **Pause and upgrade.** LST contracts can be paused or upgraded by governance. Lending markets should know what happens if transfers stop.
- **Restaked LSTs.** A liquid restaking token wraps an LST and adds new slashing conditions and withdrawal delays. Its depeg risk stacks on top of the LST's; see **restaking-shared-security-risk**.
- **Cross-chain LSTs.** Bridged wstETH on L2s depends on the bridge and on a rate relayed from L1, which can be stale. See **cross-chain-messaging-protocols**.

## A quick risk model

For a lending market, estimate the price drop at which looped positions become insolvent, and compare it with historical discounts:

```python
# leverage loop: deposit LST, borrow ETH at LTV, buy more LST, repeat
ltv, liq_threshold = 0.90, 0.93
leverage = 1 / (1 - ltv)                       # infinite-loop leverage
debt_ratio = ltv                               # debt / collateral value at entry
for discount in [0.0, 0.01, 0.03, 0.05, 0.07]:
    health = (1 - discount) * liq_threshold / debt_ratio
    print(f"discount {discount:.0%}: leverage {leverage:.1f}x, health {health:.3f}",
          "LIQUIDATABLE" if health < 1 else "")
```

With these numbers, a discount above roughly three percent makes maximally looped positions liquidatable. If the oracle uses market price, a few-percent depeg causes a liquidation cascade; if it uses the exchange rate, the protocol is exposed to bad debt only if the LST truly loses value.

## Worked example

_Not executed in this book's CI; verify locally._

Measure the 2022 discount yourself on a fork.

1. Pick blocks around mid-June 2022 using a block explorer's date-to-block tool.
2. For each block, start `anvil --fork-url $ETH_RPC_URL --fork-block-number <n>` (or use `cast call --block <n>` against an archive RPC) and query the Curve stETH/ETH pool:

```bash
POOL=0xDC24316b9AE028F1497c275EB9192a3Ea0f67022
cast call $POOL "get_dy(int128,int128,uint256)(uint256)" 1 0 100000000000000000000 --block $N --rpc-url $ETH_RPC_URL
cast call $POOL "balances(uint256)(uint256)" 0 --block $N --rpc-url $ETH_RPC_URL
cast call $POOL "balances(uint256)(uint256)" 1 --block $N --rpc-url $ETH_RPC_URL
```

3. Record how much ETH 100 stETH bought, and the pool's ETH/stETH balance split. Plot over time with matplotlib.
4. Feed the observed discounts into the risk model above with a lending market's real LTV and liquidation threshold from its documentation.

## Common pitfalls

- Treating the exchange rate as the price a liquidator can actually get.
- Treating a market price from one thin pool as reliable.
- Assuming post-Shapella depegs cannot happen.
- Forgetting that wrapped, bridged and restaked versions each add risk.
- Strict equality checks on rebasing token transfers.

## Exercises

1. Write a Foundry test showing a vault that holds stETH and tracks deposits in storage leaking the rebase surplus to an attacker; fix it with wstETH.
2. Implement `min(exchangeRate, marketPrice)` with a staleness check on the market feed and fuzz it.
3. Repeat the worked example for rETH using a Balancer or Curve pool of your choice.

## Further reading

- Lido stETH integration guide in the Lido docs: <https://docs.lido.fi>
- Aave governance forum discussions of LST price adapters: <https://governance.aave.com>
- **oracle-security-engineering** and **restaking-shared-security-risk** in this wiki
- Curve stETH pool on Etherscan for historical data
