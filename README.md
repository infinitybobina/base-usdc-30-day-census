# USDC on Base: a 30-day census and a payment-attribution scan

This repository publishes three labeled measurements of USDC activity on Base:

- **Pass 1 — full census.** Blocks 50,784,977–52,080,977 (the 30 days ending 2026-10-02): every USDC `Transfer` log, aggregated per UTC hour.
- **Window scan — suffix share across all USDC transactions.** Blocks 50,767,968–52,020,807 in 40-block windows, anchored to chain tip 52,063,968. No clock dates are stored in this artifact.
- **Pass 2 — sampled calldata scan.** Every 50th block of the census range (25,921 blocks, ~2%), read with full transactions; two selectors; suffixes parsed from calldata tails; values taken from calldata arguments.

Every number below carries the measurement it came from. Denominators are never mixed.

## Key numbers

| Metric | Value | Measurement |
|---|---|---|
| Blocks scanned | 1,296,001 | Pass 1 |
| USDC Transfer logs, 30 days | 110,334,933 | Pass 1 |
| Logged transfer value, 30 days | $2,803,375,123,478 | Pass 1 |
| Mean log size | $25,408 | Pass 1 |
| Count peak / trough | 14:00 / 23:00 UTC (1.70×) | Pass 1 |
| Dollar column CV | 6.8% | Pass 1 |
| Window blocks | 1,200 (40-block windows) | Window scan |
| USDC transactions in windows | 49,924 | Window scan |
| Transfer logs in windows | 98,916 | Window scan |
| Transactions carrying app-code suffix | 5,120 (10.3%) | Window scan |
| Window blocks with ≥1 suffix | 992 | Window scan |
| Value on app codes inside windows | $5,753,418.14 (sum of logs, not scaled) | Window scan · `observer_facilitator_total.json` |
| Registry-address txs in windows | 92, none suffixed | Window scan · `observer_facilitator_total.json` |
| Sampled blocks | 25,921 (every 50th, ~2%) | Pass 2 |
| Candidate payment calls | 64,757 | Pass 2 |
| Candidate value (calldata) | $37,046.39 | Pass 2 |
| Candidates with suffix | 28,044 (43.3%, 95% CI ±0.4 p.p.) | Pass 2 |
| Value of suffixed candidates | $79.25 (0.21% of candidate value) | Pass 2 |
| Registry facilitator candidates | 10 calls, $14.08 | Pass 2 |
| Suffix ∩ facilitator | 0 | Pass 2 |
| Calls to hub 0x0770…c137 | 27,972 (all suffixed), $51.64 | Pass 2 |
| Missed blocks | 0 | Pass 2 |

Candidate composition (Pass 2): 36,785 `transferWithAuthorization` calls on the USDC contract ($36,994.75, of which 72 are suffixed, worth $27.61) plus 27,972 settle-pattern calls on the hub ($51.64, all suffixed).

## What we measured

### Pass 1 — full census (baseline)

Every block of the census range, every `Transfer` log of USDC (`0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913`), aggregated per UTC hour. This is a network baseline, not an agent-payment profile: a multi-hop payment is counted at each hop. The count profile swings 1.70× across the day (peak 14:00 UTC); the dollar profile is nearly flat (CV 6.8%).

### Window scan — suffix share across all USDC transactions

Anchored to chain tip 52,063,968. Windows cover blocks 50,767,968–52,020,807. No clock dates are stored in this file. For every USDC transaction in a window, the enclosing transaction's calldata tail is checked for an ERC-8021 suffix. The windows contain 49,924 USDC transactions and 98,916 Transfer logs; 992 window blocks contain at least one suffixed transaction. Result: 5,120 transactions (10.3%) carry an app code. The value attributed to app codes is $5,753,418.14 — a sum of Transfer logs inside the tagged transactions, so multi-hop transfers are counted at each hop, exactly as in the baseline; it is not unique payment volume. Separately, 92 transactions touched facilitator addresses from the public registry; none carried a suffix. This scan is intentionally not scaled to the month.

### Pass 2 — sampled calldata scan

Every 50th block from 50,784,977 to 52,080,977, read with full transactions. Candidate = a call matching one of two selectors: `transferWithAuthorization` (0xe3ee160e, EIP-3009) on the USDC contract, or settle-pattern calls (0x92669141) on hub 0x0770d2124c0a581c28cfc47a659817145e6cc137. Suffixes are parsed from the calldata tail (marker ×8, schemaId, codesLength read from the end). Values come from the calldata argument — the authorized amount — not from receipts: the public RPC rejected batch receipt reads, and per-block receipt polling did not fit the run.

Checks, in order. Before the run: a control transaction from an earlier scan was re-decoded — builder code `bc_o3dj3qk8` and 16 minor units matched; its block is off the step-50 grid, so it is not in the dataset. After COMPLETE: four sampled transactions were receipt-verified — the calldata value matched the single USDC Transfer log exactly (2, 1573, 17113, 23099 minor units).

## Findings

1. **The network breathes with human hours; suffixed calldata does not.** Pass 1 counts swing 1.70× across the day. The suffixed-call series in Pass 2 is flat (918–1,464 per hour, peak 17:00 UTC) with no night trough. We report the shape only: 99.7% of that series is one unverified hub (Finding 4), so this text does not call it agent behavior.
2. **Attribution metadata is present at both denominators we can measure.** 10.3% of USDC transactions in the window scan carry an app code; 43.3% of direct payment calls in the block sample carry a suffix. Different sets, measured differently — neither substitutes for the other.
3. **The money attached to suffixes is small inside the sampled denominator:** $79.25 of $37,046.39 candidate value (0.21%). The window scan attributes $5,753,418.14 to app codes inside its windows — hop-counted like the baseline, and deliberately not scaled. No monthly dollar figure is derived in this document.
4. **Suffixed calls are concentrated in one contract.** 27,972 of 28,044 (99.7%) target 0x0770…c137, which is not verified as x402 infrastructure and is unattributed.
5. **Registry addresses tagged nothing in either measurement.** The window scan saw 92 transactions touch listed facilitator addresses; none carried a suffix (those 92 calls were not dissected, so their settlement role is unmeasured). The block sample found 10 candidate calls involving registry addresses ($14.08); intersection with suffixes = 0. This supports the working hypothesis that listed facilitators do not tag their own transactions. Whether the registry reflects actual settlement addresses is an open question our data does not answer.

## Limitations (read before citing)

- **Two selectors, no routers.** Router/aggregator-wrapped payments are invisible to Pass 2. Its totals are a floor for direct calls of two known shapes, not an estimate of the whole payment flow.
- **Values come from calldata.** `value_max = value_sum` in all 64,757 rows by construction. Receipt verification covered four transactions post-run; the control transaction was checked pre-run and is off-grid.
- **We do not scale dollars.** The ±0.4 p.p. interval applies to the 43.3% call share only. Linear ×50 scaling of sampled dollars is a separate assumption we decline to make: sampled value is concentrated (the 10:00 UTC hour alone holds $5,302 of $37,046).
- **Sums count hops.** The $2.8T baseline and the window scan's $5,753,418.14 both sum Transfer logs; a multi-hop payment is counted at each hop. Neither is settlement volume.
- **The hub is unattributed.** We do not know who operates 0x0770…c137 or whether it is x402-compatible.
- **The window scan describes its windows.** 10.3% and $5,753,418.14 characterize 1,200 blocks, not the month.

## Reproduce

- `artifacts/observer_hours.csv` — Pass 1 hourly baseline (all USDC transfers)
- `artifacts/observer_run.json` — window-scan totals and top codes
- `artifacts/observer_facilitator_total.json` — window-scan totals: attributed amount, registry-address transactions, window blocks with suffix
- `artifacts/observer2_hours.csv` — Pass 2 hourly: all candidates / suffixed / facilitator
- `artifacts/observer2_summary.json` — Pass 2 totals and acceptance checks
- `docs/DECISION.md` — methodological decisions and acceptance gates, in order

Method in one paragraph: an exhaustive log census, fixed windows, and a deterministic block sample (every 50th block, stated range) with calldata parsing. No heuristics, no ML classification, no private data sources. Every number above can be recomputed from the artifacts.

## What would break these findings

- Facilitators publish settlement addresses that change the intersection — re-runnable within a day
- 0x0770…c137 is attributed to a known protocol — the concentration finding becomes a named finding
- A router-inclusive denominator — shares and totals would move, as stated above

We publish what we measured. Corrections with reproducible evidence are welcome — issues are open.
