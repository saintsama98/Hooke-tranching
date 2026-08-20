# Strata

> Study of a live system. One tranching engine, several unrelated yield sources, and a
> published parameter set per market. This document works through the economics from what
> enters the structure to what each party is left holding.

Strata is a generalised risk tranching protocol. A single engine splits a yield source
into a senior and a junior tranche, and the same engine has been pointed at several
different strategies without being rewritten for any of them. It launched on Ethena USDe
in October 2025 and has since been extended to other sources including Neutrl NUSD,
Hastra PRIME and Saturn USDat. Reported value locked is roughly 78 million dollars, with
a cumulative figure above 160 million cited by the protocol.

It is the most useful protocol in this repository for a reason unrelated to its size.
Strata publishes the parameters that drive its yield split, market by market, and keeps
them separate from the arithmetic that consumes them. The invariant, the loss branch and
the accrual are exact and auditable. The values that decide how the split behaves are
declared numbers with names, published per market, and changeable only behind a timelock.
That is the separation argued for in
[`../01-fundamentals/`](../01-fundamentals/README.md), running in production across
several heterogeneous strategies at once, and it is rare enough to be worth studying by
someone with no interest in the protocol itself.

The parameters below are the whole of the calibration surface. The baseline premium the
senior tranche always pays runs from 5 percent on Hastra PRIME to 100 percent on Saturn
USDat. The maximum additional premium accruing as senior value grows runs from 0 percent
on Saturn to 15 percent on most others. The exponent controlling how quickly the premium
rises is 0.3 in every market without exception, which turns out to matter a great deal and
is the subject of Section 4.3.

The subject of what follows is the money: what enters the structure, what event causes it
to be measured, the rule by which it is divided, the order in which loss is borne, and the
levers by which each of those may be altered. Every figure and formula is taken from the
published documentation and cited inline. Quantities derived rather than quoted are marked
as such and the derivation is shown. This account is documentation led rather than source
led, because the contracts are deployed and audited but were not published at the time of
writing.

---

## 1. Structural summary

Strata is not a yield source. It is a wrapper that interposes a two tranche capital structure between a depositor and an external yield strategy, and it is deliberately generic over that strategy: the same engine is applied to a delta neutral synthetic dollar (Ethena), a market neutral fund (Neutrl, M1 Capital), a tokenised preferred equity dividend (Strategy's STRC via Saturn), and a warehouse lending facility financing home equity lines of credit (Figure via Hastra). Source: [Protocol Overview](https://docs.strata.markets/technical-documentation/protocol-overview), [market pages](https://docs.strata.markets/).

Each market is assembled from the same set of functional parts, and it is worth naming
them by what they do rather than by what they are called, because the division of labour
is the design. There is a senior claim and a junior claim, both of which are transferable
across chains. There is an orchestrator that sequences the accounting and the calls into
the strategy. There is a calculation layer that is pure, holding no assets and computing
the value of each claim and the exchange rates between them. There is a strategy that
deploys, stakes and retrieves assets and reports what the whole position is worth. There
is a feed that pushes an external benchmark rate on chain. There is a set of deferred
settlement locks, one for assets and one for shares and one that passes through the
unbonding of the underlying. And there is a configuration layer through which every
parameter change must pass, in two steps and behind a delay.

The separation that matters is the third of those. The layer that computes what each
tranche is worth holds nothing and can move nothing, so an error in the valuation is a
mispricing rather than a theft. That is the same discipline Centrifuge applied in its
earlier work and the same discipline Royco applies today, and it is close to a
precondition for a tranched structure being auditable at all.

---

## 2. What comes in

Three distinct inflows drive the model. They are of different kinds and must not be conflated.

### 2.1 Principal

A depositor supplies a base asset and receives shares in one tranche. The accepted asset set is per market and deliberately wide at the entry point: the Ethena market accepts `USDe` or `sUSDe`; the Neutrl market accepts `NUSD`, `USDC`, `USDT`, `USDe`, or `sNUSD`; the Midas and Hastra markets accept `USDC`; the Saturn market accepts `USDat`. A `TrancheDepositor` utility normalises the input and routes it. Minting fees are documented as waived across all markets at the time of writing. Sources: [srUSDe](https://docs.strata.markets/markets/ethena-usde/srusde), [jrNUSD](https://docs.strata.markets/markets/neutrl-nusd/jrnusd).

Principal is then forwarded to the `Strategy`, which stakes it into the underlying instrument — `sUSDe`, `sNUSD`, `mHYPER`, `mM1-USD`, STRC, or Democratized Prime. The tranche vaults therefore never hold idle base asset in the steady state; they hold accounting claims against a single commingled position.

### 2.2 Yield

Yield is not received as a transfer. It is *observed* as growth in the strategy's reported total value:

```
gain = totalTVL(t1) − totalTVL(t0)
```

This is the single most consequential design decision in the model. Because the yield is a measured delta on a commingled pool rather than an identified cashflow, the entire structure is a **net asset value tranching** rather than a **cashflow tranching**: there is no payment to sequence, only a balance to attribute. A negative `gain` is handled by the same arithmetic path as a positive one, which is why loss absorption requires no separate machinery. Source: [Protocol Overview](https://docs.strata.markets/technical-documentation/protocol-overview).

### 2.3 The exogenous rate signal

The senior tranche's floor is not chosen by the protocol; it is imported. An `AprPairFeed` fed by an `AaveAprPairProvider` publishes the benchmark on chain, and an operational role calls `onAprChanged` / `updateRoundData` to refresh it. Sources: [Contracts Details](https://docs.strata.markets/technical-documentation/contracts-details), [Roles and Permissions](https://docs.strata.markets/technical-documentation/roles-and-permissions).

The benchmark definition differs materially by market, and each definition encodes a different commercial claim:

| Market | Base asset | Underlying strategy | Benchmark (senior floor) |
| --- | --- | --- | --- |
| Ethena USDe | USDe | sUSDe | Supply weighted average of USDC and USDT lending rates, Aave v3 Core |
| Neutrl NUSD | NUSD | OTC / funding rate arbitrage | Ethena sUSDe APY |
| Midas mHYPER | USDC | Multi chain stablecoin yields (Hyperithm) | Aave v3 Core USDC/USDT weighted average **+ 3% APY** |
| Midas mM1-USD | USDC | M1 Capital Master USD Fund | **0%** |
| Saturn USDat | USDat | STRC preferred equity dividend | **65% of the STRC dividend rate** |
| Hastra PRIME | USDC | Democratized Prime (Figure HELOC warehouse) | None specified |

Sources: [Dynamic Yield Split](https://docs.strata.markets/protocol-mechanism/dynamic-yield-split), [Ethena USDe](https://docs.strata.markets/markets/ethena-usde), [Neutrl NUSD](https://docs.strata.markets/markets/neutrl-nusd), [Midas mHYPER](https://docs.strata.markets/markets/midas-mhyper), [Midas mM1-USD](https://docs.strata.markets/markets/midas-mm1-usd), [Saturn USDat](https://docs.strata.markets/markets/saturn-usdat), [Hastra PRIME](https://docs.strata.markets/markets/hastra-prime).

A benchmark of 0% (mM1-USD) reduces the senior claim to a pure proportional split with no guarantee. A benchmark of "Aave + 3%" (mHYPER) is a floor set *above* the risk free proxy, which transfers a standing subsidy from junior to senior irrespective of performance. These are not incidental settings; they are the commercial terms of each deal expressed as a parameter.

---

## 3. What triggers the model

There is no scheduled accrual, no epoch, and no keeper driven settlement. The model advances **lazily, on interaction**. Two classes of event move it:

1. **Any protocol action** — deposit, mint, withdraw, or redeem on either tranche. Each begins by calling Update Accounting before any share arithmetic is performed.
2. **A feed update** — `onAprChanged` or `updateRoundData`, called by `UPDATER_FEED_ROLE`, which is not timelocked.

Between events, no state changes. The interval `t0 → t1` over which yield is attributed is therefore the interval between interactions, not a fixed period. This is the design Quantstamp audited under the scope name "discrete accounting mechanism". Sources: [Protocol Overview](https://docs.strata.markets/technical-documentation/protocol-overview), [Roles and Permissions](https://docs.strata.markets/technical-documentation/roles-and-permissions), [Audits](https://docs.strata.markets/technical-documentation/audits).

The ordered sequence executed on each trigger:

1. `Strategy` reports `totalTVL` at `t1`.
2. `Accounting` compares against the stored `totalTVL` at `t0`; `gain = totalTVL(t1) − totalTVL(t0)`.
3. If a reserve percentage is set, a portion of `gain` is redirected to `reserveTVL`.
4. The remainder is provisionally credited to `jrtTVL`.
5. The senior target gain is computed (Section 4) and **debited from `jrtTVL`, credited to `srtTVL`**.
6. The invariant `jrtTVL + srtTVL + reserveTVL == totalTVL` is restored.

Step 5 is the waterfall. It is expressed as a transfer out of the junior balance rather than as a priority over an incoming payment, and it is unconditional in sign — the junior balance is debited whether or not step 4 credited anything. That single property is what makes the loss waterfall and the income waterfall the same code path.

---

## 4. The division rule — Dynamic Yield Split

### 4.1 The stated formulae

Let `TVL_sr` and `TVL_jr` be the two tranche balances, and define the senior share

```
r  =  TVL_ratio_sr  =  TVL_sr / (TVL_sr + TVL_jr)
```

The base rate is derived from the underlying instrument's own exchange rate growth over the interval:

```
Growth_Factor  =  ExchangeRate_T1 / ExchangeRate_T0 − 1
APR_base       =  Growth_Factor / (T1 − T0) × 1 year
```

The risk premium — the fraction of the base yield that the senior tranche surrenders to the junior tranche in exchange for the loss protection it receives — is a one parameter power law in the senior share:

```
Risk_Premium_sr  =  x  +  y · r^k
```

The senior rate is the greater of its floor and its post premium share:

```
APR_sr  =  MAX( APR_target ,  APR_base · (1 − Risk_Premium_sr) )
```

The senior entitlement is then accrued through a monotonic index, so that the senior claim compounds and never decreases in index terms:

```
Interest_Factor   =  APR_sr · (T1 − T0) / 1 year
Target_Index_T1   =  Target_Index_T0 · (1 + Interest_Factor)
Senior_Gain       =  TVL_sr · (Target_Index_T1 / Target_Index_T0 − 1)
```

The junior rate is published as:

```
APY_jr  =  (APY_base − APY_sr) · r / (1 − r)  +  APY_base
```

Senior coverage and junior overperformance are defined as:

```
Senior coverage        =  Total TVL / TVL_sr   =  1 / r
Junior overperformance =  APY_jr / APY_base
```

Sources: [Dynamic Yield Split](https://docs.strata.markets/protocol-mechanism/dynamic-yield-split), [Protocol Overview](https://docs.strata.markets/technical-documentation/protocol-overview).

### 4.2 The junior formula is not an independent rule

The published junior expression is frequently read as a second pricing rule alongside the senior one. It is not. Expanding it:

```
(b − s)·r/(1−r) + b  =  [ (b − s)r + b(1 − r) ] / (1 − r)
                     =  [ br − sr + b − br ] / (1 − r)
                     =  ( b − s·r ) / (1 − r)
```

which is exactly the rearrangement of the conservation identity `s·r + j·(1−r) = b`. The junior tranche is therefore a pure residual: it receives whatever the pool earned less whatever the senior index demanded, divided by its own share. There is one pricing decision in the model (the senior rate) and one accounting consequence (the junior balance). *Derived; verified numerically in Section 6.*

The immediate corollary is the junior's exposure multiple:

```
∂j/∂s  =  −r / (1 − r)   ≡   −λ
```

Every additional percentage point granted to the senior tranche costs the junior tranche `λ` points, where `λ` is the senior to junior balance ratio. This is the leverage figure that governs the entire risk profile, and it is set by capital composition rather than by any parameter.

### 4.3 The premium curve is far flatter than "dynamic" implies

Published parameters, all markets using `k = 0.3`:

| Market | `x` | `y` | `k` |
| --- | --- | --- | --- |
| Ethena USDe | 10% | 12.5% | 0.3 |
| Neutrl NUSD | 15% | 15% | 0.3 |
| Midas mHYPER | 12.5% | 15% | 0.3 |
| Midas mM1-USD | 12.5% | 15% | 0.3 |
| Saturn USDat | 100% | 0% | 0.3 |
| Hastra PRIME | 5.0% | 7.5% | 0.3 |

Source: [Dynamic Yield Split](https://docs.strata.markets/protocol-mechanism/dynamic-yield-split).

An exponent of `0.3` makes `r^k` strongly concave, so nearly all of the premium's variation occurs at senior shares below 50% — a region the protocol never occupies, because a senior share of 50% corresponds to 200% coverage. Evaluating the Ethena curve across the domain (*derived*):

| `r` | Coverage `1/r` | `Risk_Premium_sr` | `λ = r/(1−r)` |
| --- | --- | --- | --- |
| 0.10 | 1000.0% | 16.27% | 0.11 |
| 0.25 | 400.0% | 18.25% | 0.33 |
| 0.50 | 200.0% | 20.15% | 1.00 |
| 0.75 | 133.3% | 21.47% | 3.00 |
| 0.8713 | 114.8% | 21.99% | 6.77 |
| 0.90 | 111.1% | 22.11% | 9.00 |
| 0.9524 | 105.0% | 22.32% | 20.01 |
| 1.00 | 100.0% | 22.50% | ∞ |

Over the entire operating range from 200% coverage down to the 105% circuit breaker, the Ethena premium moves 20.15% → 22.32% — a span of **2.17 percentage points of the base APY**. The equivalent spans for the other markets are 2.82pp (Neutrl, both Midas markets) and 1.41pp (Hastra).

The economic reading is that the yield split is, in practice, a near static proportional division with a marginal tilt, and that the compensation the junior tranche actually receives for taking incremental risk comes almost entirely from `λ` — the shrinking of its own denominator — rather than from any repricing by the premium curve. The premium is close to constant; the leverage is not. A protocol seeking a genuinely rate responsive premium in the 100–130% coverage band would need `k` well above 1, not below.

The Saturn market is the limiting case and is instructive: `x = 100%, y = 0%` sets `Risk_Premium_sr = 100%` identically, so `APR_base · (1 − 1) = 0` and the `MAX` always selects the floor. The senior claim there is a **fixed rate instrument** paying 65% of the STRC dividend, with the junior tranche taking the entire residual and the entire variance. The same engine expresses both a participating senior and a fixed rate senior purely through parameter choice.

---

## 5. Fees and the third bucket

Value leaves the structure at three points.

**Performance fee** — a fixed percentage of the yield on pooled collateral, allocated to the protocol treasury. Levied on the pool, and therefore ahead of the tranche split, so it reduces `APR_base` for both tranches proportionally rather than falling on either one. Values: 5% (Ethena, Hastra PRIME), 7.5% (Neutrl, mHYPER, mM1-USD), 0% (Saturn USDat).

**`reserveTVL`** — a configurable share of each interval's `gain` diverted before the junior credit, governed by `setReserveBps` under a 48-hour timelock, with `RESERVE_MANAGER_ROLE` controlling redistribution and treasury withdrawal. This is the only structurally retained buffer in the design and is the model's closest analogue to traditional **excess spread capture**: yield trapped in a third account rather than released to the equity claim. The documentation does not publish the current basis point setting, and the invariant `jrtTVL + srtTVL + reserveTVL == totalTVL` is the only place the bucket is visible. Sources: [Protocol Overview](https://docs.strata.markets/technical-documentation/protocol-overview), [Roles and Permissions](https://docs.strata.markets/technical-documentation/roles-and-permissions).

**Redemption fee** — charged on exit, and per the [Mechanism Overview](https://docs.strata.markets/protocol-mechanism/mechanism-overview) "distributed to respective tranche holders with portion to treasury", with the treasury share governed by `setFeeRetentionBps` under a 48-hour timelock. Its principal function is therefore not revenue but **anti dilution**: a levy on the exiting holder that accretes to those who remain.

The fee schedule is tiered on coverage, and the direction is counterintuitive on first reading — fees are highest when coverage is *strongest* and fall to zero when coverage is weakest:

| Market | Senior exit fee (by coverage) | Junior exit fee (by coverage) | Junior cooldown (by coverage) | Min. coverage |
| --- | --- | --- | --- | --- |
| Ethena USDe | 0.025% (flat) | 0.10% (flat) | 7d (Ethena unbonding) | 105% |
| Neutrl NUSD | 0.05% >120% · 0.025% 110–120% · 0% <110% | 0.20% >120% · 0.10% 110–120% · 0% <110% | none >120% · 7d 110–120% · **35d** <110% | not published |
| Midas mHYPER | 0.05% >120% · 0.025% 110–120% · 0% <110% | 0.20% >120% · 0.10% 110–120% · 0% <110% | none >120% · 7d 110–120% · **21d** <110% | not published |
| Midas mM1-USD | 0.05% >120% · 0.025% 110–120% · 0% <110% | 0.20% >120% · 0.10% 110–120% · 0% <110% | none >120% · 14d 110–120% · **28d** <110% | not published |
| Saturn USDat | 0.05% >130% · 0.025% 115–130% · 0% <115% | 0.20% >130% · 0.10% 115–130% · 0% <115% | 7d >130% · 14d 115–130% · **28d** <115% | 107.5% |
| Hastra PRIME | 0.05% >115% · 0.025% 110–115% · 0% <110% | 0.15% >115% · 0.075% 110–115% · 0% <110% | none >115% · 7d 110–115% · **14d** <110% | 105% |

Sources: the six market pages cited in Section 2.3; the srNUSD/jrNUSD fee ranges are separately confirmed as 0–5 bps and 0–20 bps at [FAQs](https://docs.strata.markets/resources/faqs).

The inversion is coherent once the two instruments are read as substitutes rather than complements. Under stress the protocol does not need a *price* on exit, because it has already imposed a *lock* — cooldowns lengthen to 14, 21, 28, or 35 days, and below the minimum coverage threshold junior redemption is suspended outright. Charging a fee on top would serve no incremental deterrent purpose while raising the round trip cost for exactly the new junior capital the protocol is trying to attract at that moment. Waiving the fee at low coverage makes recapitalisation cheap precisely when recapitalisation is what the structure requires. *This reading is an interpretation; the documentation states the schedule but not its rationale.*

The generalised primitive is stated plainly in the [Protocol Overview](https://docs.strata.markets/technical-documentation/protocol-overview): senior coverage may be split into **up to three ranges**, and each range may apply any combination of **SharesLock**, **AssetsLock**, and **Fee**. The market tables above are instantiations of that one mechanism.

---

## 6. The loss waterfall

Step 5 of the accounting cycle branches on whether the interval's gain covers the senior index accrual.

**Case A — `gain ≥ Senior_Gain`.** The senior tranche receives its full target. The junior tranche retains the remainder. The invariant holds.

**Case B — `gain < Senior_Gain`.** The shortfall is met by debiting `jrtTVL`. The senior index still advances at `APR_sr`; the junior exchange rate falls. The invariant holds.

Case B has no lower guard other than the junior balance itself. As the documentation states, the junior tranche "can lose the principal as well during prolonged negative performance or an underlying default event as it acts as the first loss capital", and on exhaustion "the senior tranche may also incur a loss of principal". Sources: [Risks and Mitigations](https://docs.strata.markets/protocol-mechanism/risks-and-mitigations), [FAQs](https://docs.strata.markets/resources/faqs), [Junior Tranche](https://docs.strata.markets/introduction/junior-tranche).

The consequence worth stating precisely: **Case B fires on a yield shortfall, not only on a credit loss.** If the underlying earns 3.0% while the floor demands 3.75%, the junior balance is debited even though nothing has defaulted. The junior tranche is short a floor rate as well as long the credit — it has written an interest rate option to the senior tranche in addition to a first loss guarantee, and the two exposures are settled through the same debit.

### 6.1 The debit is capped in code

The [Cyfrin audit](https://github.com/Cyfrin/cyfrin-audit-reports/blob/main/reports/2025-10-08-cyfrin-strata-tranches-v2.0.pdf) publishes the splitting block, which shows two properties the prose documentation does not:

The transfer to the senior tranche is capped, and the cap is the economically important
part. Two things follow from how the debit is bounded. The invariant of Section 3 is
enforced rather than merely asserted, so a division that failed to conserve value would
not settle at all. And the senior claim on the junior tranche is bounded by the junior
balance less one whole unit, which means the senior floor is not an unconditional
entitlement even before the junior tranche is legally exhausted. It is an entitlement
capped at what the junior tranche actually holds, after which the senior claim reverts to
whatever the strategy earned. The reversion to base yield on junior exhaustion is
therefore a binding term of the instrument and not only a statement in prose.

First, the invariant of Section 3 is **enforced by revert**, not merely asserted in documentation. Second, and more consequentially, the senior's claim on the junior is bounded by `saturatingSub(jrtNavT1, 1e18)` — the transfer cannot drive the junior net asset value below one whole unit. The senior floor is therefore not an unconditional entitlement even before the junior tranche is legally exhausted: it is an entitlement *capped at the junior balance less one unit*, after which the senior necessarily reverts to whatever the strategy earned. The reversion to base yield on junior exhaustion is thus a coded term, not only a prose statement.

The audit also records the inverse branch. Where `srtGainTarget < 0`, the loss is transferred to the junior tranche as profit — commented in the source as "should never happen, jic" — and the finding at 7.3.5 was that this branch credited `jrtNavT0` rather than `jrtNavT1`, so the invariant check would revert. It was fixed in commit `5332b3` and verified.

### 6.2 Reconciliation against live figures

Taking the published Ethena market state — `srUSDe` TVL $89.24M at 3.75% 7-day APY, `jrUSDe` TVL $13.18M at 7.59% 7-day APY ([DefiLlama](https://defillama.com/protocol/strata-markets)) — the model reproduces the reported junior yield exactly:

- `r = 89.24 / 102.42 = 0.87131`, so senior coverage is `1/r = ` **114.77%**, and `λ = 6.77`.
- Implied base rate `b = s·r + j·(1−r) = 3.75(0.87131) + 7.59(0.12869) = ` **4.2442%**.
- Substituting into the published junior formula returns **7.5900%**, against 7.59% reported. The identity of Section 4.2 is confirmed to four significant figures.

The informative part is the senior side. The split only senior rate at this state would be `b · (1 − 0.21994) = ` **3.3107%**, whereas the reported rate is 3.75%. The `MAX` is therefore selecting the **floor**, not the split: the Ethena market is currently operating in the regime where the senior tranche is being paid its Aave benchmark guarantee rather than its proportional share.

The junior tranche is funding the difference. The subsidy is `(3.75 − 3.3107) · λ = ` **2.97 percentage points** of junior APY — that is, of the 7.59% the junior tranche earns, roughly 2.97pp is being surrendered to honour the senior floor, leaving the junior with about 4.62pp attributable to the premium and residual. Under a base rate above roughly 4.8% the floor would stop binding and the split would resume governing. *All figures in this subsection are derived from the two published APYs and two published TVLs; they assume the reported APYs are net of the 5% performance fee and that the 7-day trailing window is representative.*

### 6.3 Sensitivity

Applying Ethena parameters with a 3.75% floor across base rates and senior shares (*derived*):

| Base APY | `r = 0.50` (cov 200%) | `r = 0.8713` (cov 114.8%) | `r = 0.9524` (cov 105%) |
| --- | --- | --- | --- |
| 10.00% | sr 7.98% · jr 12.02% (1.20×) | sr 7.80% · jr 24.89% (2.49×) | sr 7.77% · jr 54.66% (5.47×) |
| 8.00% | sr 6.39% · jr 9.61% (1.20×) | sr 6.24% · jr 19.91% (2.49×) | sr 6.21% · jr 43.72% (5.47×) |
| 6.00% | sr 4.79% · jr 7.21% (1.20×) | sr 4.68% · jr 14.93% (2.49×) | sr 4.66% · jr 32.79% (5.47×) |
| 4.24% | sr 3.75% · jr 4.74% (1.12×) | sr 3.75% · jr 7.59% (1.79×) | sr 3.75% · jr 14.13% (3.33×) |
| 4.00% | sr 3.75% · jr 4.25% (1.06×) | sr 3.75% · jr 5.69% (1.42×) | sr 3.75% · jr 9.00% (2.25×) |
| 3.50% | sr 3.75% · jr 3.25% (0.93×) | sr 3.75% · **jr 1.81%** (0.52×) | sr 3.75% · **jr −1.50%** |
| 3.00% | sr 3.75% · jr 2.25% (0.75×) | sr 3.75% · **jr −2.08%** | sr 3.75% · **jr −12.01%** |

Rows in which the floor binds are all those at or below 4.24%. The junior yield crosses zero at `b = floor · r`: 3.267% at the current senior share, 3.571% at the 105% breaker, 1.875% at 200% coverage.

Two properties follow. First, the junior tranche's breakeven base rate *rises* as coverage falls — the thinner the junior layer, the higher the underlying yield it needs merely to avoid losing principal. Second, at the 105% circuit breaker the junior's exposure multiple is `λ ≈ 20`, so a 1pp shortfall against the floor costs the junior tranche 20pp of annualised return. The junior position is most leveraged at exactly the coverage level at which its redemption is suspended.

---

## 7. Circuit breakers and the coverage state machine

Coverage `1/r` is the single state variable governing protective behaviour. Three responses are documented, in escalating order.

1. **Fee and cooldown tiering** (Section 5), applied continuously across up to three coverage bands.
2. **Minimum coverage threshold** — at 105% (Ethena, Hastra PRIME) or 107.5% (Saturn USDat), the protocol suspends **senior minting** and **junior redemption** simultaneously. Sources: [Risks and Mitigations](https://docs.strata.markets/protocol-mechanism/risks-and-mitigations), [srUSDe](https://docs.strata.markets/markets/ethena-usde/srusde), [jrUSDe](https://docs.strata.markets/markets/ethena-usde/jrusde).
3. **Junior shortfall pause** — `setJrtShortfallPausePrice`, held by `PAUSER_ROLE` with no timelock, sets a junior exchange rate level at which activity halts. Together with `setMinimumJrtSrtRatio` and `setMinimumJrtSrtRatioBuffer` under `OWNER_ROLE` (48-hour timelock), these are the parameters that define where the breakers sit. Source: [Roles and Permissions](https://docs.strata.markets/technical-documentation/roles-and-permissions).

> `diagrams/strata-coverage-states.d2`

The pairing at threshold 2 is the structurally significant one. Halting senior minting prevents `r` from rising further, since new senior capital would directly dilute coverage. Halting junior redemption prevents `r` from rising by the other route, since junior exits shrink the denominator. Senior redemption is never halted — which is correct as to seniority, and also means the only permitted flow at the breaker is the one that *improves* coverage. The structure corrects itself, but it does so by immobilising the junior holder.

The protocol describes the equilibrium as self balancing: "thinner coverage increases junior yields, attracting more liquidity" ([FAQs](https://docs.strata.markets/resources/faqs)). Section 4.3 qualifies the mechanism — the attraction comes overwhelmingly from `λ` rather than from the premium curve, and it operates only while the base rate exceeds the floor. Below the floor the same leverage runs in reverse, and the incentive to supply junior capital inverts at precisely the point the structure most needs it. The fee waiver at low coverage (Section 5) is best read as a deliberate offset to that inversion.

---

## 8. The governance surface is the economic surface

Every quantity in Sections 4 through 7 is a mutable parameter. The published role map:

| Role | Powers | Timelock |
| --- | --- | --- |
| `PAUSER_ROLE` | `setActionStates` (pause deposits/redemptions), `setJrtShortfallPausePrice` | **none** |
| `UPDATER_FEED_ROLE` | `onAprChanged`, `updateRoundData` | **none** |
| `UPDATER_STRAT_CONFIG_ROLE` | `setRiskParameters`, `setCooldowns` | 24h |
| `RESERVE_MANAGER_ROLE` | reserve redistribution, treasury withdrawal | 24h |
| `PROPOSER_CONFIG_ROLE` | `scheduleExitFeeChange` | none (proposal only) |
| `OWNER_ROLE` | APR feed and provider, `setReserveBps`, `setFeeRetentionBps`, `setMinimumJrtSrtRatio`, `setMinimumJrtSrtRatioBuffer`, `setRoundStaleAfter` | 48h |

Custody sits with an administrative multisig of three of four and an operational
multisig of two of three, behind timelocks of forty eight and twenty four hours
respectively. The addresses are published and are omitted here because they are an
implementation detail rather than an economic one.

The material observation: `setRiskParameters` — which carries `x`, `y`, `k`, and the floor — sits behind a **24-hour** delay, while `setReserveBps` and `setFeeRetentionBps` sit behind 48 hours. The senior tranche's "guaranteed minimum yield" is thus a governance commitment revocable within a day, not a property of the contract. The rate feed itself is updatable with no delay at all. Any standard that intends the senior guarantee to be an invariant rather than a policy must locate the floor somewhere other than a mutable configuration slot, or must bound the direction and magnitude of permitted changes at the contract level.

Six audits are published across Quantstamp, Cyfrin, and Guardian, with two further reports on the pre deposit vaults. Notably, "coverage aware redemption mechanism" and "discrete accounting mechanism" were scoped as separate engagements, which corroborates their treatment as the two load bearing subsystems. Source: [Audits](https://docs.strata.markets/technical-documentation/audits).

---

## 9. Assessment against the common shape of the sector

Assessed against the five parts described in [`common-shape.md`](../03-protocol-landscape/README.md).

**Valuation and loss accounting: provided, and unusually cleanly.** Discrete per interaction accounting on a measured TVL delta, with a monotonic `Target_Index` for the senior claim and a floating exchange rate for the junior claim. The senior index never decreases; the junior rate may fall below 1. The three bucket invariant is explicit and total.

**The waterfall: provided, two tranches, perpetual.** Senior first on income and junior first on loss, through a single unconditional debit. No mezzanine, no sequential paydown, no turbo or PIK features. Pro rata within each tranche.

**Coverage tests and cure: partial, and further than Centrifuge, but still short of a cure.** Coverage gates issuance and redemption exactly as Centrifuge's minimum subordination ratio gates DROP issuance, and Strata additionally tiers exit fees and locks across coverage bands, which is a finer instrument than Centrifuge's binary gate. Strata also does something Centrifuge does not: Case B **actively diverts junior capital to satisfy the senior interest entitlement**, which is a genuine interest coverage cure rather than a passive absorption.

What remains absent is the principal side cure. No mechanism redeems senior notional to restore the coverage ratio; the structure never delevers. When coverage falls, Strata freezes the numerator and the denominator and waits for the underlying to recover or for new junior capital to arrive. The traditional overcollateralisation test — divert cashflow to *retire* senior principal until the ratio is restored — is unrepresented, which is the same gap this repository's [`../01-fundamentals/README.md`](../01-fundamentals/README.md) identifies across the sector.

**Settlement: provided, but not through ERC-7540.** Tranches are ERC-4626 with LayerZero OFT transferability. Asynchronous settlement, which the structure plainly requires, is built alongside ERC-4626 as three separate lock contracts — `ERC20Cooldown` (asset lock), `UnstakeCooldown` (underlying unbonding passthrough), and `SharesCooldown` (share lock) — rather than through the request/claim lifecycle ERC-7540 defines for exactly this case. The choice is worth recording: a protocol that needed variable duration, state dependent deferred redemption declined the standard written for asynchronous vaults and built a parallel mechanism.

### Three findings carry beyond Strata

1. **The coverage band triple is a general idea.** Up to three ranges over senior coverage, each applying any combination of a share lock, an asset lock and a fee, is a compact and complete way to describe exit friction that varies with the state of the pool. It is materially more expressive than a single subordination gate, and it is the most transferable part of the design. See [`../04-liquidity/README.md`](../04-liquidity/README.md).

2. **The fixed arithmetic and the declared parameters are cleanly separated.** The invariant, the Case A and Case B branch, and the `Target_Index` accrual are exact and auditable. The values `x`, `y`, `k`, the floor, and the coverage bands are parameters published per market. That is the separation described in [`../01-fundamentals/README.md`](../01-fundamentals/README.md), running in production across seven different strategies on one engine.

3. **A floor rate is an obligation, and here it is not typed as one.** The junior tranche is short an interest rate floor as well as long first loss credit exposure, and the model settles both through the same debit, so the two are indistinguishable in the accounting. Whether the floor survives the exhaustion of the junior layer is answered in the code rather than declared: the transfer to the senior tranche is capped at the junior balance less one unit, per Section 6.1, so the senior reverts to the base rate as the junior is exhausted. That answer is legible to someone reading the source and to nobody else.

---

## Sources

- Strata documentation index: [docs.strata.markets](https://docs.strata.markets/) · [llms.txt](https://docs.strata.markets/llms.txt)
- [Why Strata](https://docs.strata.markets/introduction/why-strata) · [Senior Tranche](https://docs.strata.markets/introduction/senior-tranche) · [Junior Tranche](https://docs.strata.markets/introduction/junior-tranche)
- [Mechanism Overview](https://docs.strata.markets/protocol-mechanism/mechanism-overview) · [Dynamic Yield Split](https://docs.strata.markets/protocol-mechanism/dynamic-yield-split) · [Risks and Mitigations](https://docs.strata.markets/protocol-mechanism/risks-and-mitigations)
- [Protocol Overview](https://docs.strata.markets/technical-documentation/protocol-overview) · [Contracts Details](https://docs.strata.markets/technical-documentation/contracts-details) · [Roles and Permissions](https://docs.strata.markets/technical-documentation/roles-and-permissions) · [Security](https://docs.strata.markets/technical-documentation/security) · [Audits](https://docs.strata.markets/technical-documentation/audits)
- Market pages: [Ethena USDe](https://docs.strata.markets/markets/ethena-usde) ([srUSDe](https://docs.strata.markets/markets/ethena-usde/srusde), [jrUSDe](https://docs.strata.markets/markets/ethena-usde/jrusde)) · [Neutrl NUSD](https://docs.strata.markets/markets/neutrl-nusd) ([srNUSD](https://docs.strata.markets/markets/neutrl-nusd/srnusd), [jrNUSD](https://docs.strata.markets/markets/neutrl-nusd/jrnusd)) · [Midas mHYPER](https://docs.strata.markets/markets/midas-mhyper) · [Midas mM1-USD](https://docs.strata.markets/markets/midas-mm1-usd) · [Saturn USDat](https://docs.strata.markets/markets/saturn-usdat) · [Hastra PRIME](https://docs.strata.markets/markets/hastra-prime)
- [FAQs](https://docs.strata.markets/resources/faqs)
- Audit reports: [Cyfrin, protocol v1 (2025-10-08)](https://github.com/Cyfrin/cyfrin-audit-reports/blob/main/reports/2025-10-08-cyfrin-strata-tranches-v2.0.pdf) · [Cyfrin, shares cooldown (2026-01-23)](https://github.com/Cyfrin/cyfrin-audit-reports/blob/main/reports/2026-01-23-cyfrin-strata-shares-cooldown-v2.0.pdf) · [Guardian, tranches](https://github.com/GuardianAudits/Audits/blob/main/Strata/Strata_Tranches_report.pdf) · [Quantstamp, discrete accounting](https://certificate.quantstamp.com/full/strata-discrete-accounting/02318e87-e35f-4e96-81ad-192253203d55/index.html) · [Quantstamp, protocol v1](https://certificate.quantstamp.com/full/strata-tranches/3c3a4037-2a92-468c-a4f3-5ea498e7b539/index.html) · [Quantstamp, redemption fee](https://certificate.quantstamp.com/full/strata-update-to-tranches/d7a903b7-80cf-42db-8433-79186fdd8be2/index.html)
- Third party: [DefiLlama — Strata Markets](https://defillama.com/protocol/strata-markets) · [Aleare Research — Strata: Bringing Risk-Tranching to DeFi Yields](https://alearesearch.substack.com/p/strata-risk-tranching)
