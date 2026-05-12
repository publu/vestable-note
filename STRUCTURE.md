# Structural Analysis: Liquidity, Transferability, and Pendle

This document examines the veStable as a market-design object. Where the liquidity actually sits, what the locked position looks like to secondary markets, how it interacts with Pendle and the rest of the DeFi stack, and what fails under stress.

## Three Liquidity States

A unit of value in this system can be in exactly one of three states at any time:

**1. Unlocked face.**
A normal stablecoin. Fully transferable ERC-20. Redeemable at par against the liquid pool. Earns no yield. This is the cash-equivalent leg of the supply.

**2. Locked position with liquid backing.**
The depositor locked, but the protocol routed their share into the liquid leg (T-bills, Aave, Morpho). Either no deal at their duration exists yet, or the protocol is at its illiquid deployment cap and parking the excess. Earns base yield only. Cannot redeem before lock expiry without selling the position on a secondary market or paying an exit fee.

**3. Locked position with illiquid backing.**
The depositor locked and the protocol routed their share into one or more illiquid deals (catalogs, private credit, term loans). Earns base yield plus a share of premium yield from the assets backing them. Cannot redeem before lock expiry without forcing the protocol to unwind illiquid positions, which is slow under normal conditions and impossible during stress.

The unlocked face is unambiguously liquid: AMM-listable, Curve-poolable, atomic swap. The interesting questions all sit in states 2 and 3.

## Transferability Of The Locked Position

Two design choices, each with downstream consequences.

**Non-transferable lock (Curve veCRV style).**
The lock is bound to the depositor's address. Cannot be sold. Only redeemed at expiry. Simple, clean term structure, no secondary market complications. No exit for a depositor who changes their mind. No composability inside the broader DeFi stack. No way for sophisticated capital to enter a position mid-lock.

**Transferable lock as NFT (Curve V2 / Solidly style).**
The locked position is an NFT carrying lock state and backing as on-chain metadata. Anyone can buy it on secondary markets. New owner inherits both the lock weight and the deployed backing. Gives depositors an exit. Enables secondary liquidity discovery for duration risk. Lets sophisticated buyers absorb risk from less sophisticated sellers. Introduces an external price for the locked position which can diverge from NAV. Creates a surface for wrapper arbitrage.

For a duration-as-yield-primitive product the NFT version is probably right. The whole premise is that duration is a tradeable property. Locking the locks entirely defeats the purpose of publishing a term structure.

## What The Locked Position Actually Is

A locked veStable NFT pays premium yield from its backing map along the way and matures into the face value of the stablecoin at unlock. That is structurally a fixed-income instrument: a coupon-plus-principal that converges to par at maturity.

It is functionally the same shape as a Pendle Principal Token bundled with its Yield Token. Pendle's whole architecture exists to price exactly this object after splitting it in two. The veStable keeps both legs bundled on one position.

This raises the question of whether the protocol should issue and trade these positions natively, or let Pendle (or a Pendle-like venue) wrap a fungible flavor and provide secondary liquidity externally.

## Pendle: How It Actually Works

Pendle's mechanics matter because the veStable's locked position is the same kind of object Pendle was built to trade.

The Pendle stack:

1. **SY (Standardized Yield) wrapper.** Any yield-bearing token can be wrapped into an SY that exposes a standard interface to the rest of the protocol.
2. **Split at a maturity.** SY at maturity `T` is decomposed into:
   - PT (Principal Token): redeems to one unit of underlying at `T`. Sold at a discount before `T`. Buying PT locks in a fixed yield-to-maturity.
   - YT (Yield Token): captures all yield accrued by the SY between now and `T`. Wasting asset that pays a stream and expires worthless.
3. **PT AMM per maturity.** A concentrated-liquidity AMM matches PT against the underlying. Pricing the discount on PT reveals the implied fixed yield.
4. **YT pricing.** YT trades by arbitrage against PT + SY (`PT + YT = SY` at any time before maturity).

Liquidity is concentrated at specific maturities. Each maturity needs its own PT AMM. Each new maturity is a fresh liquidity event.

## Pendle-wrapping A veStable: The Two Paths

To put a veStable on Pendle, three conditions need to hold:

- Yield stream is per-token, not per-holder. The veStable as designed pays per-position yield (each depositor has a unique backing map). A Pendle wrapper needs a fungible yield-bearing object, which requires standardizing the backing across a pool of positions.
- Maturity is well-defined. The veStable has continuous lock duration. Pendle needs discrete maturities, so a wrapper has to bucket positions into common tenors.
- The underlying is deep enough that PT and YT AMMs find counterparties at each maturity.

That gives two paths.

**Path A: Pendle becomes the external secondary market.**
A vault aggregator (call it stkVeStable) buys locked positions of a target tenor, manages the portfolio, and mints a fungible yield-bearing ERC-20. Pendle SY-wraps stkVeStable and splits it into PT/YT at standard maturities (3 month, 6 month, 12 month). Users wanting fixed yield buy PT. Users wanting variable-rate leverage buy YT. The veStable protocol does nothing extra. External composability emerges on its own.

This is the Convex-on-Curve outcome. The protocol gets distribution it did not build, but cedes the price discovery surplus to the aggregator. The aggregator captures the most desirable depositors (largest tickets, longest durations) and the underlying protocol risks becoming a pass-through.

**Path B: The veStable absorbs Pendle's job natively.**
The locked NFT already carries both principal commitment and variable yield rights. The protocol publishes a buyback bid at some discount-to-NAV that gives depositors a native exit. Bought-back positions sit in a protocol-owned pool. The protocol mints fungible fixed-rate paper against that pool, sized at common maturities. This is effectively running an internal Pendle market.

Higher complexity, keeps the fee surface inside the protocol, preserves price discovery as protocol-controlled. Path A is lower effort and cedes the surplus. The two paths are not mutually exclusive: a protocol could run a native buyback book while also tolerating an external Pendle wrapper.

## What Pendle Does Well That Should Be Copied

- **Discount-to-maturity is the UX.** A Pendle depositor sees "buy PT-USDC-MAR2027 at 5.2 percent APY," not "this position has utilization 0.8 and duration 0.5." Whatever the underlying mechanic, the user-facing screen must resolve to a yield number.

- **Fixed-rate exits.** A depositor who locked for 12 months at variable yield can lock in their expected yield by selling a PT-equivalent claim. The veStable can offer this natively or rely on Pendle to provide it.

- **Maturity bucketing concentrates liquidity.** Continuous duration in the underlying is fine. Secondary markets need rounded buckets (1M, 3M, 6M, 1Y, 2Y) to clear.

## What Pendle Does Badly That The veStable Avoids

- **Three tokens per maturity per asset (SY + PT + YT) fragments liquidity.** The veStable underlying is one stablecoin. The fragmentation only appears in the optional secondary layer, never at the base.

- **YT is a wasting asset.** Confusing for retail. The veStable locked NFT appreciates to NAV instead of decaying to zero, which is more intuitive.

- **No native duration matching on the asset side.** Pendle's yield is whatever its underlying yield-bearing token produces. The veStable's routing rule (lock duration drives deployment) means duration is matched at the protocol level, not just hedged at the user level via PT.

## Liquidity By State, Quantified

Let `L` = total veStable supply. Decompose:

- `U` = unlocked face
- `B` = locked, backing currently in liquid leg
- `P` = locked, backing currently in illiquid leg

`L = U + B + P`.

Redemption obligation in the next `d` days is `U + (B and P expiring within d)`. The protocol's liquid pool must cover this plus a stress buffer.

Practical constraint:

```text
liquid_pool >= U + sum(positions expiring within d) + stress_buffer
illiquid_pool = L - liquid_pool
illiquid_pool >= 0
liquid_pool >= 0.15 * L   (hard floor regardless of expiry schedule)
```

The 15 percent floor is a defensive number. The right value depends on observed redemption velocity and the time-to-liquidate of the worst-case illiquid asset. For a catalog book where positions take weeks to a quarter to exit cleanly, 15 percent is a reasonable starting point.

## Wave Redemption: The Stress Case

Failure scenario: a cohort that all locked four years ago all unlocks the same week. The protocol owes face value on all of `P` simultaneously, but `P`'s backing is in catalogs that cannot be sold instantly without taking a haircut.

Three mitigations, in order of strength:

**Stagger expiries at deposit.**
Add a small random component to the unlock timestamp (e.g., +/- 14 days). Smooths the curve so no week ever sees the full cohort.

**Redemption queue with NAV haircut.**
Depositors who want to exit at expiry join a queue ranked by lock-end. Payouts happen as backing matures or is sold. Anyone wanting to skip the queue takes a NAV haircut that funds an immediate exit.

**Secondary market for the locked NFT.**
Depositors sell their position rather than redeeming. The price discovers the discount-to-NAV the market is willing to pay for early liquidity. The protocol's solvency does not depend on liquidating backing; the secondary market handles it.

The third mitigation is the strongest because it converts a solvency problem into a price-discovery problem. As long as some buyer exists at some price, the protocol does not have to force-sell anything.

## Composability Surface

**AMMs.**
Unlocked face: standard ERC-20, Curve pool, Uniswap concentrated liquidity, Balancer composable stable. Locked NFT: non-fungible, so AMM listing requires either bucketed fungible vaults or NFT AMMs like Sudoswap (probably too thin for institutional size).

**Money markets.**
Unlocked face: Aave / Morpho / Spark accept immediately as collateral. Locked NFT: harder, because duration variance and NAV uncertainty complicate liquidation. Morpho Vaults (which can hold arbitrary assets and price them via oracle) is the realistic path. The protocol publishes a per-position NAV oracle that Morpho consumes.

**Pendle.**
Path A (external aggregator builds stkVeStable, gets Pendle-wrapped) or Path B (protocol issues fixed-rate paper against a buyback book natively).

**Convex-style aggregators.**
The veStable does not have a separate governance token with emissions to vote on. There is no bribe market. This kills the original Convex-on-Curve playbook. Aggregators can still buy locked positions to capture yield, but the dynamic is closer to a Yearn vault than to vlCVX. That absence of governance reflexivity is a feature, not a gap.

**Yearn-style vaults.**
Almost certainly the first composability layer that gets built. A vault buys locked positions across durations, rolls them at expiry, and mints a fungible yield-bearing token. Likely the default retail entry point if the lock UX is fiddly.

**Liquid wrappers.**
The lock-the-productive-asset / mint-a-tradable-claim pattern is identical to liquid staking. Expect a "liquid veStable" wrapper within months of any real launch. Convex emerged within six months of Curve's gauge weight system. Worth assuming the same here.

## Comparison Matrix

```text
                       | Pendle           | veStable        | Curve veCRV       | Maple syrupUSDC | Notional nToken
-----------------------+------------------+-----------------+-------------------+-----------------+-----------------
Tokens per position    | 3 (SY+PT+YT)     | 1 (NFT or ERC)  | 2 (CRV + veCRV)   | 1               | 1
Per-maturity AMMs      | yes              | optional        | no                | no              | no
User picks duration    | yes (maturity)   | yes (continuous)| yes (1-4y lock)   | no              | partial (fCash)
Duration-matched bkng  | no               | yes (routing)   | n/a               | no              | partial
Fixed-rate exit        | yes (PT)         | optional via wrap| no               | no              | yes (fCash)
Yield-token decay UX   | yes (YT)         | no              | n/a               | n/a             | n/a
Gov-token overhang     | low (PENDLE)     | none            | high (CRV/vlCVX)  | medium (MPL)    | low (NOTE)
Secondary liquidity    | deep at maturity | NFT or wrapper  | none (locked)     | cooldown queue  | continuous
Routing constraint     | none             | lock_to_asset   | none              | allocator       | governance
```

## Failure Modes Specific To The Structure

**The aggregator extracts the surplus (Path A risk).**
If stkVeStable becomes the dominant retail entry, the underlying protocol becomes a wholesale yield generator and the aggregator captures the price discovery layer. Mitigation: pre-empt with native fixed-rate issuance (Path B), or accept the trade.

**NAV oracle is the central point of failure.**
Any secondary liquidity for the locked NFT depends on a credible per-position NAV. If the oracle lags or is gameable, money markets pull collateral support and Pendle wrappers cannot price PT. Mitigation: an oracle that publishes both a conservative mark and an audited mark, with a delay tolerance built into the protocol's own redemption math.

**Routing concentration on the longest locks.**
If long deals always fill from the longest locks first, those locks bear concentrated underwriting risk. Mitigation: pro-rata across positions within a duration bucket so single-deal exposure is diversified across the cohort.

**Liquid wrapper centralization.**
If one wrapper (analogous to stETH) reaches 60 percent dominance, the protocol's term structure is effectively dictated by that wrapper's governance. Mitigation: design the lock mechanic so any wrapper has to honor the underlying duration metadata. The wrapper's tokens can be liquid, but the positions they hold cannot escape the lock weight.

**Pendle integration concentrating depositor selection.**
Pendle PT buyers want the most predictable yield. They will gravitate toward the safest backing. If the wrapper offers them only one slice of the veStable's backing curve, the rest of the protocol absorbs adverse selection. Mitigation: any Pendle-wrappable token aggregates a representative slice of the protocol's deployment, not the highest-quality slice.

## Summary

The veStable's base layer is a stablecoin with two metadata signals on each position: lock state (veDuration) and backing map (veUtilization). Unlocked face is fully liquid. Locked positions are fixed-income objects that look structurally identical to bundled Pendle PT + YT, just kept on one position instead of split.

The integration story with Pendle is not "compete" but "decide who runs the secondary market." Path A lets Pendle absorb it externally and cedes price discovery. Path B issues fixed-rate paper natively against a buyback book and keeps the surplus. Neither path requires changing the underlying mechanic.

In every comparable system except veStable, multiple tokens are needed to express duration, principal commitment, and yield share simultaneously. The veStable expresses all three as metadata on a single position and pushes the per-maturity AMM fragmentation cost into an optional secondary layer rather than baking it into the base.
