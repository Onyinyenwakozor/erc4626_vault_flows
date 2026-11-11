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
5. The largest flags, and the largest outflow neither method flagged, were checked against the vault's onchain Deposit and Withdraw events.

## Results

1. **Steakhouse Prime Instant:** 4 withdrawal days flagged by z score, 1 by IsolationForest. Largest: 30 Sep 2026, 21.79m USDC, driven by a single strategy contract.
2. **Gauntlet USDC Prime V2:** 4 days flagged by z score, none by IsolationForest. Largest flagged: 29 Oct 2026, 19.41m USDC, spread across many wallets. The largest outflow in the whole sample, 45.71m USDC on 22 Apr 2026, was missed by both methods.
3. **kpk USDC Prime V2:** 5 days flagged by z score, 1 by IsolationForest only. Largest: 2 Feb 2026, 8.87m USDC, driven by one Safe multisig that also drove the 30 Jun flag.

Each vault's largest flagged outflow came from a different source: one strategy, many wallets, or one large holder. Every estimate checked against onchain events matched within 0.1%.

## Flag checks against onchain data

DeFiLlama takes its daily snapshot at about 23:00 UTC, so each row covers 23:00 UTC the day before to 23:00 UTC that day. The Steakhouse and Gauntlet checks use that exact window. The kpk checks use the UTC calendar day.

### Steakhouse Prime Instant, 30 Sep 2026

Model estimate: net outflow of 21.79m USDC, z score of minus 3.73

Onchain, 29 Sep 23:02 to 30 Sep 23:01 UTC: 27 withdrawals totalling 29,454,768 USDC and 36 deposits totalling 7,674,931 USDC, net outflow 21,779,837 USDC. Matches the estimate within 0.1%.

Main driver: one withdrawal of 26,237,393.31 USDC at 15:43:11 UTC on 30 Sep 2026

Wallet: 0xbaed9839573d349e42dfbf23a8916e5ab9caf2e3 (contract MorphoCompounder, TokenizedStrategy implementation 0x254A93feff3BEeF9cA004E913bB5443754e8aB19)

Transaction: [0x76adc316e3f2bd0526e6dbe5e42b7139655642388c1476e0439a88f6cc73edf3](https://etherscan.io/tx/0x76adc316e3f2bd0526e6dbe5e42b7139655642388c1476e0439a88f6cc73edf3)

### Gauntlet USDC Prime V2, 29 Oct 2026

Model estimate: net outflow 19.41m USDC, z score minus 4.13

Onchain, 28 Oct 23:01 to 29 Oct 23:01 UTC: 51 withdrawals totalling 22,911,932 USDC and 10 deposits totalling 3,507,356 USDC, net outflow 19,404,577 USDC. Matches the estimate within 0.1%.

Drivers: many wallets exiting in the same window, routed through VaultBundlesV1 at 0x02912516d49de997db75b9d7858faae59209650b. Largest two:

1. 6,602,678.71 USDC at 22:42:11 UTC, owner 0x651fac183d2ac9753bec39f7530adf1b873f0314, tx [0x4f9990439130560251a4c9a8d838182d7dc9b22803da76693525b053f0e1e153](https://etherscan.io/tx/0x4f9990439130560251a4c9a8d838182d7dc9b22803da76693525b053f0e1e153)
2. 5,196,578.05 USDC at 21:51:35 UTC, owner 0x84d1b180d67ba40a4cb7aeb78af6a8bf80fc5c63, tx [0x2c3d5b2c5e27507e284937432fe315338928010b910c782836181ba7de9fb9a2](https://etherscan.io/tx/0x2c3d5b2c5e27507e284937432fe315338928010b910c782836181ba7de9fb9a2)

### Gauntlet USDC Prime V2, 22 Apr 2026 (missed by both methods)

Model estimate: net outflow 45.71m USDC, z score minus 2.45, the largest single day outflow in the sample. Not flagged by either method.

Onchain, 21 Apr 23:01 to 22 Apr 23:02 UTC: 34 withdrawals totalling 55,365,283 USDC and 65 deposits totalling 9,658,291 USDC, net outflow 45,706,992 USDC. Matches the estimate within 0.1%.

Drivers:

1. 14,998,614.20 USDC, owner 0x8092c20351cf4048b464df2144dc8a4dd49ce71d, tx [0xfbfed3b742409f4034693961ed18760e9e436c123a2fa8532202f03c1664309c](https://etherscan.io/tx/0xfbfed3b742409f4034693961ed18760e9e436c123a2fa8532202f03c1664309c)
2. 14,252,716.87 USDC, owner 0xd33847382f821094d9079256c25ca1794ab408c6, tx [0x1be059c3fe2810b024d7c009f7ed997ae543a58870c70356f6cf54bf78bec7e6](https://etherscan.io/tx/0x1be059c3fe2810b024d7c009f7ed997ae543a58870c70356f6cf54bf78bec7e6)
3. 10,939,812.90 USDC, owner 0x68e7e72938db36a5cbbca7b52c71dbbaadfb8264, tx [0x042ceca97d317250771bc7a17d259682152fafc6bb03f02179be8fa5a3e0d8a2](https://etherscan.io/tx/0x042ceca97d317250771bc7a17d259682152fafc6bb03f02179be8fa5a3e0d8a2)

The next day the vault took in a net 33.47m USDC. 0x8092c20351cf4048b464df2144dc8a4dd49ce71d deposited 19.47m USDC back in two transactions ([0xc3dd64063aa54a485fd655159750bfff88f0c88b7468a9e5a461c9b6d1b5967f](https://etherscan.io/tx/0xc3dd64063aa54a485fd655159750bfff88f0c88b7468a9e5a461c9b6d1b5967f) and [0xfdbd00a09dcee342bd587606fe9e165e5e55107d5e6c527398cb08f8e5eaedff](https://etherscan.io/tx/0xfdbd00a09dcee342bd587606fe9e165e5e55107d5e6c527398cb08f8e5eaedff)), so a large part of this outflow was money rotating out and back.

Why it was missed: the two days before had net inflows of 19.77m and 45.10m USDC (z scores of 5.14 and 5.77). Those widened the 30 day baseline, so this outflow only reached a z score of minus 2.45, short of the minus 3 threshold. IsolationForest likely spent its 1% anomaly quota on the large inflow days, which the outflow filter then discards.

### kpk USDC Prime V2, 2 Feb 2026 (IsolationForest only)

Model estimate: net outflow 8.87m USDC, z score minus 2.32, below the z threshold

Onchain, 2 Feb 2026 UTC: 3 withdrawals totalling 8,868,789 USDC, no deposits. Matches the estimate within 0.1%.

Driver: 8,021,669.19 USDC at 12:57:11 UTC by Safe multisig 0x4f2083f5fbede34c2714affb3105539775f7fe64, tx [0x3446f277e2dbd6942d7ee6c7a628e4ed7ca4bbf7b2b8d4411976468c05742c88](https://etherscan.io/tx/0x3446f277e2dbd6942d7ee6c7a628e4ed7ca4bbf7b2b8d4411976468c05742c88)

### kpk USDC Prime V2, 30 Jun 2026 (z score)

Model estimate: net outflow 4.39m USDC, z score minus 7.01

Onchain, 30 Jun 2026 UTC: 2 withdrawals totalling 7,889,012 USDC, 4 deposits totalling 3,504,882 USDC, net outflow 4,384,130 USDC. Matches the estimate within 0.1%.

Driver: 4,920,000 USDC at 14:08:59 UTC by the same Safe 0x4f2083f5fbede34c2714affb3105539775f7fe64, tx [0x819ffcbeeff33982291962902ccf10d4eb35844df2656357e61b79692a0d8b89](https://etherscan.io/tx/0x819ffcbeeff33982291962902ccf10d4eb35844df2656357e61b79692a0d8b89)

Method note: Deposit and Withdraw events were pulled with eth_getLogs from a public Ethereum RPC, filtered by vault address and the ERC4626 event signatures, and summed over the window stated for each check.

## Limits

1. TVL comes from DeFiLlama, not onchain data, and is measured in USD. Only USDC vaults are used so that price moves don't show up as flows.
2. apyBase is a smoothed annual figure, not the return earned each day, so return volatility looks lower than it really is.
3. The data is daily. A deposit and a withdrawal inside the same snapshot window cancel out. DeFiLlama snapshots at about 23:00 UTC, so onchain checks must use that window, not the calendar day.
4. Missing days are left blank, not filled in.
5. The z score can flag tiny moves when recent flows are close to zero. Gauntlet on 18 Jan flagged a $1,025 outflow from a $102k vault, and a minimum outflow filter of $250k removes it. kpk on 17 Mar shows a z of minus 79: the $500k outflow was real (about 7% of the vault), but the z value is inflated because the days before it were almost flat.
6. The opposite also happens: large inflows widen the baseline and hide the next outflow. Gauntlet's 45.71m outflow on 22 Apr, the largest in the sample, was missed by both methods for this reason.
7. There are no labelled anomalies, so precision can't be scored formally. Instead, five days were checked against Deposit and Withdraw events, and all five estimates matched within 0.1%. The 3 sigma threshold and 1% contamination are still assumptions.
8. Gauntlet's 92.89% max TVL drawdown comes from its first weeks, when TVL fell from 15.4m on 27 Jan to 1.1m on 4 Feb 2026. It has not been checked onchain.

## Next step

1. Build the baseline from outflow days only, or use a median based z score, so large inflows can't hide the next outflow. Fit IsolationForest on outflow days only for the same reason.
2. Automate the event check with web3.py and use `convertToAssets()` for the real share price, so every day is checked against onchain data, not just the flagged ones.

## How to run

Open the notebook in Colab and click **Runtime > Run all**. No API key is needed.
