# ERC4626 vault flows: separating returns from deposits and withdrawals

## Question

When a vault's TVL moves, how much is yield and how much is money moving in or out? And which withdrawal days are unusual enough to look into?

## Vaults

All three are Morpho Vault V2 USDC vaults on Ethereum, run by different curators. Each was checked on Etherscan for `asset()` and `convertToAssets()` to confirm it is ERC4626.

| Vault | Curator | Vault address | DeFiLlama pool ID |
|---|---|---|---|
| Steakhouse Prime Instant | Steakhouse | 0xbeef088055857739C12CD3765F20b7679Def0f51 | 931ea9be-5f4d-428e-beaf-205fc5b4e2b5 |
| Gauntlet USDC Prime V2 | Gauntlet | 0x8c106eedad96553e64287a5a6839c3cc78afa3d0 | 71b34441-5a46-431b-a9b3-8c081cd0d74c |
| kpk USDC Prime V2 | kpk | 0x4ef53d2caa51c447fdfeeedee8f07fd1962c9ee6 | 052e3f85-e489-4bee-9e34-c7898ef1d449 |

Underlying asset for all three: USDC, 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48

## Method

1. Daily TVL and apyBase from the DeFiLlama yields API.
2. Daily return from apyBase: `r = (1 + apyBase / 100) ** (1 / 365) - 1`
3. Estimated net flow: `net_flow = TVL_today - TVL_yesterday * (1 + r)`. Whatever TVL change the return does not explain is money moving in or out. Flows are also scaled by the previous day's TVL so vaults of different sizes compare.
4. Two anomaly methods for withdrawal spikes: a 30 day rolling z score (flag at minus 3 or lower, with outflow above $250k), and IsolationForest from scikit learn as a second check.
5. The largest flags were checked against the vault's onchain Deposit and Withdraw events.

## Results

1. **Steakhouse Prime Instant:** 4 withdrawal days flagged by z score, 1 by IsolationForest. Largest: 30 Sep 2026, 21.79m USDC, driven by a single strategy contract.
2. **Gauntlet USDC Prime V2:** 4 days flagged by z score, none by IsolationForest. Largest: 29 Oct 2026, 19.41m USDC, spread across many wallets.
3. **kpk USDC Prime V2:** 5 days flagged by z score, 1 by IsolationForest only. Largest: 2 Feb 2026, 8.87m USDC, driven by one Safe multisig that also drove the 30 Jun flag.

Each vault's largest outflow came from a different source: one strategy, many wallets, or one large holder.

## Flag checks against onchain data

### Steakhouse Prime Instant, 30 Sep 2026

Model estimate: net outflow of 21.79m USDC, z score of minus 3.73

Onchain, 29 and 30 Sep 2026 UTC: 59 Withdraw events totalling 33.29m USDC and 71 Deposit events totalling 11.73m USDC, net outflow 21.56m USDC. The estimate is within about 1% of the onchain figure.

Main driver: one withdrawal of 26,237,393.31 USDC at 15:43:11 UTC on 30 Sep 2026

Wallet: 0xbaed9839573d349e42dfbf23a8916e5ab9caf2e3 (contract MorphoCompounder, TokenizedStrategy implementation 0x254A93feff3BEeF9cA004E913bB5443754e8aB19)

Transaction: [0x76adc316e3f2bd0526e6dbe5e42b7139655642388c1476e0439a88f6cc73edf3](https://etherscan.io/tx/0x76adc316e3f2bd0526e6dbe5e42b7139655642388c1476e0439a88f6cc73edf3)

### Gauntlet USDC Prime V2, 29 Oct 2026

Model estimate: net outflow 19.41m USDC, z score minus 4.13

Onchain, 29 Oct 2026 UTC: withdrawals 20.13m USDC, deposits 3.51m USDC, net outflow 16.62m USDC. Outflows continued on 30 Oct (net 5.92m USDC), so the gap most likely reflects DeFiLlama's snapshot time.

Drivers: many wallets exiting on the same day, all routed through VaultBundlesV1 at 0x02912516d49de997db75b9d7858faae59209650b. Largest three:

1. 6,602,678.71 USDC at 22:42:11 UTC, owner 0x651fac183d2ac9753bec39f7530adf1b873f0314, tx [0x4f9990439130560251a4c9a8d838182d7dc9b22803da76693525b053f0e1e153](https://etherscan.io/tx/0x4f9990439130560251a4c9a8d838182d7dc9b22803da76693525b053f0e1e153)
2. 5,196,578.05 USDC at 21:51:35 UTC, owner 0x84d1b180d67ba40a4cb7aeb78af6a8bf80fc5c63, tx [0x2c3d5b2c5e27507e284937432fe315338928010b910c782836181ba7de9fb9a2](https://etherscan.io/tx/0x2c3d5b2c5e27507e284937432fe315338928010b910c782836181ba7de9fb9a2)
3. 2,973,724.88 USDC at 13:32:35 UTC, owner 0xd620aadabaa20d2af700853c4504028cba7c3333, tx [0xb7ff3307b9b5fb027e588b422cbe7eac158542fb97884aef3bc638b0f2629321](https://etherscan.io/tx/0xb7ff3307b9b5fb027e588b422cbe7eac158542fb97884aef3bc638b0f2629321)

### kpk USDC Prime V2, 2 Feb 2026 (IsolationForest only)

Model estimate: net outflow 8.87m USDC, z score minus 2.32, below the z threshold

Onchain, 2 Feb 2026 UTC: 3 withdrawals totalling 8,868,789 USDC, no deposits. Matches the estimate within 0.1%.

Driver: 8,021,669.19 USDC at 12:57:11 UTC by Safe multisig 0x4f2083f5fbede34c2714affb3105539775f7fe64, tx [0x3446f277e2dbd6942d7ee6c7a628e4ed7ca4bbf7b2b8d4411976468c05742c88](https://etherscan.io/tx/0x3446f277e2dbd6942d7ee6c7a628e4ed7ca4bbf7b2b8d4411976468c05742c88)

### kpk USDC Prime V2, 30 Jun 2026 (z score)

Model estimate: net outflow 4.39m USDC, z score minus 7.01

Onchain, 30 Jun 2026 UTC: 2 withdrawals totalling 7,889,012 USDC, 4 deposits totalling 3,504,882 USDC, net outflow 4,384,130 USDC. Matches the estimate within 0.1%.

Driver: 4,920,000 USDC at 14:08:59 UTC by the same Safe 0x4f2083f5fbede34c2714affb3105539775f7fe64, tx [0x819ffcbeeff33982291962902ccf10d4eb35844df2656357e61b79692a0d8b89](https://etherscan.io/tx/0x819ffcbeeff33982291962902ccf10d4eb35844df2656357e61b79692a0d8b89)

Method note: Deposit and Withdraw events were pulled with eth_getLogs from a public Ethereum RPC, filtered by vault address and the ERC4626 event signatures, and summed per UTC day.

## Limits

1. TVL comes from DeFiLlama, not onchain data, and is measured in USD. Only USDC vaults are used so that price moves don't show up as flows.
2. apyBase is a smoothed annual figure, not the return earned each day, so return volatility looks lower than it really is.
3. The data is daily. A deposit and a withdrawal on the same day cancel out, and the snapshot time varies. That's why Gauntlet's 29 Oct estimate (19.41m) is higher than the onchain figure for that calendar day (16.62m).
4. Missing days are left blank, not filled in.
5. The z score can flag tiny moves when recent flows are close to zero. Gauntlet on 18 Jan flagged a $1,025 outflow from a $102k vault, and a minimum outflow filter of $250k removes it. kpk on 17 Mar shows a z of minus 79: the $500k outflow was real (about 7% of the vault), but the z value is inflated because the days before it were almost flat.
6. There are no labelled anomalies, so precision can't be scored formally. Instead, the four largest flags were checked against Deposit and Withdraw events: three matched within 1%, and the fourth gap came from snapshot timing. The 3 sigma threshold and 1% contamination are still assumptions.

## Next step

Automate the event check with web3.py and use `convertToAssets()` for the real share price, so every day is checked against onchain data, not just the flagged ones.

## How to run

Open the notebook in Colab and click **Runtime > Run all**. No API key is needed.
