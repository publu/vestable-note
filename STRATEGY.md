# veStable Strategy

Why a duration-native stablecoin is worth shipping, what it collapses, and where the existing builds have stopped short.

## The Problem With The Current Stablecoin Stack

Yield-bearing stablecoins today split into two instruments:

1. A transferable stable (USDC, sUSDS face value, etc.).
2. A staked wrapper that accrues yield (sUSDe, sUSDS, sDAI, etc.).

The split is convenient for users but throws away the most important variable on the liability side: how long the holder is willing to leave capital in the system. The wrapper records "you are earning yield" but not "you are committed for N months." Issuers compensate by manually tiering products (30-day lockups, 1-year notes, governance-locked tokens) or by paying the same rate to all holders regardless of their duration. Both options waste information.

A veStable makes duration the native unit. The protocol observes a real liability term structure and prices yield against it.

## What It Collapses

A duration-native stablecoin replaces three separate mechanisms with one curve:

**Risk tranching (senior / junior LP tokens).**
Liquid holders effectively sit senior because they redeem first against the liquid pool. Locked holders sit junior because they eat the variability of the illiquid leg. No separate LP classes needed.

**Lockup tiers (30d / 90d / 1y buckets).**
Bucketed tiers force discrete pricing for a continuous preference. A single curve over duration covers every point a buyer might want without spinning up per-tier vaults.

**ve governance tokens.**
ve systems lock a secondary token whose price floats against the productive asset, creating reflexivity and FDV overhang. Locking the productive asset itself eliminates the second token entirely. The lock is metadata on the position, not a tradable claim.

One primitive, three problems solved.

## Where It Fits

The design assumes the asset side has two yield sources with different liquidity profiles:

- A liquid leg (T-bills, Aave, Morpho, repo) earning something near SOFR.
- An illiquid leg (private credit, music royalties, real-world cash flows, term loans) earning a premium.

The veStable lets the protocol pay each holder a blend weighted by their lock state. Aggregate lock weight sizes how much capital can sit in the illiquid leg without breaking redemption. The book is duration-matched by construction rather than by manual treasury management.

Any issuer running a mixed-backed stable is a candidate: real-world-asset issuers, on-chain credit funds, RWA aggregators, anything that today exposes a "fixed term note" alongside a stable face value.

## Precedent: Close Misses

Nobody has shipped this exact primitive. The near-misses each pick up one property but drop another.

**Notional nToken.**
Single ERC-20 earning a blended rate across maturities. Closest match on the single-token-blended-rate axis. Blend ratio is set by governance, not by depositor lock weight. nTokens stay perpetually liquid.

**Maple syrupUSDC.**
One token, yield derived from a blend across pools. Blend weights set by Maple's allocator. No depositor duration choice.

**Idle Adaptive Yield Split.**
Senior and junior APYs dynamically reweight by relative pool size. Closest match on the "aggregate ω sizes the liquid pool" property. Two separate tokens. Risk tranching, not duration tranching.

**BarnBridge SMART Yield.**
sBONDs let users pick a maturity and lock a fixed rate; juniors take variable. Two tokens, discrete maturities, risk tranche not duration blend. DAO sunset in 2023.

**Saffron, TermMax, Spectra, Pendle, Element, Sense, Tempus, Napier.**
All PT/YT splitters with discrete fixed maturities. Several dead (Element pivoted to Delv, Sense shut down 2024, Tempus killed by Lido). Common cause of death: per-maturity AMMs fragment liquidity, LPs lose to IL and yield-token decay, TVL never reaches escape velocity.

**Target-date / glide-path funds (Vanguard etc.).**
Single fund, continuous time-based reweighting between liquid and illiquid sleeves. Glide path is calendar-driven toward retirement, not chosen per-investor as a lock weight. Not duration-matched by depositor commitments.

**Interval funds (GS, BX, Apollo private credit).**
Blend public credit (liquid sleeve sized for quarterly redemptions) with private credit. Redemption gates are fund-level (typically 5% per quarter), not per-investor lock weights.

**Stable-value funds (401k).**
Insurance wrappers smooth a liquid-illiquid blend (book value vs market value). Every depositor gets the same blend. No per-user duration knob.

## Why Continuous ω Avoids The Graveyard

The dead fixed-term protocols all relied on per-maturity AMMs that fragmented liquidity. A single token with a continuous duration weight sidesteps the failure mode: there is no AMM per maturity because there is only one token. A user's lock state is metadata on their position, not a separately tradable asset. Liquidity stays unified and the issuer gets the term structure for free.

## What's Actually Unbuilt

Three primitives bolted together:

1. Idle's Adaptive Yield Split mechanism for the supply-sizing math.
2. Notional's nToken UX for the single-token blended rate.
3. Depositor-chosen ω (replacing governance-set blend ratios) as the lock-determined weight.

That combination is the open primitive.

## Tradeoffs And Open Questions

**Capital efficiency cap.**
If 30 percent of TVL stays liquid, only 70 percent deploys to the premium leg, capping aggregate yield. Average ω above some threshold (probably 0.5+) is needed for the economics to work.

**Smoothing.**
Premium yield is not a single number. It spikes and sags with the underlying asset's cash flow cycles. Two options: let locked users eat variance, or smooth via a rolling distribution backed by a buffer.

**Yield-only blend vs capital-allocation blend.**
(a) Capital stays in the liquid pool, blend is synthetic accounting, protocol owes locked users the premium yield from its books.
(b) Capital actually moves into the illiquid leg when a user locks, creating real economic exposure.
The honest version is (b) but it requires accurate per-depositor position tracking.

**Curve shape.**
Linear decay is the default. Exponential or sigmoid may better match the cash-flow timing of the underlying. Worth modeling against historical data from the target asset class.

**Wave redemption.**
If liquid holders all redeem at once the liquid pool drains. Either fire-sale the illiquid leg or queue redemptions. A hard floor on the liquid pool (around 15 percent of TVL regardless of average ω) plus a redemption queue covers the edge case.

**Liquid wrappers.**
Someone will build a liquid wrapper (yvVeStable or similar) and arbitrage the lockup. Either embrace it (Convex model) or accept it cannot be fully prevented.

**No first-loss protection.**
Risk-averse capital wants protection from variability, not just a smaller share of the same pool. This product fails them. A separate small senior tranche can be layered later for that buyer, distinct from the main veStable line.

## Open Design Choices

- Lock duration max: 1 year (capital efficiency) or 4 years (Curve-style)? Probably 1 year for a stable product.
- Boost curve shape: linear, exponential, or sigmoid.
- Liquid pool floor: hard 15 percent of TVL, or dynamic floor that scales with weighted average ω.
- Redemption queue mechanism for wave redemption.
- Tokenization: single stablecoin with per-holder duration metadata, or stablecoin plus a locked-face variant. Single-token is cleaner UX, dual-face is more legible to DeFi composability.

## Summary

One stablecoin. Multiple liquidity states. Decaying duration weight. Yield allocated by how useful the holder's liquidity is to the system.

The result is not stablecoin plus staked stablecoin. It is a redeemable stable asset with an endogenous yield surface determined by duration, utilization, and realized asset-side returns.
