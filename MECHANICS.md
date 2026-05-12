# veDuration and veUtilization

Two yield-bearing signals that live on the same veToken. No PT/YT split. No separate governance token. No per-maturity AMM.

## The Two Questions

A duration-aware stablecoin has two facts it has to publish for the system to be honest:

**veDuration (the commitment question).**
How long has the depositor promised to keep their capital in the system?

**veUtilization (the deployment question).**
How much of that capital is the protocol actually using in long-duration positions, versus parked in the liquid leg?

Most existing systems answer one and ignore the other.

- Curve `ve` answers commitment (lock weight) but says nothing about whether the locked capital is being put to work.
- Maple, Idle, Notional answer deployment (asset mix) but the depositor has no way to signal duration preference.
- Pendle splits commitment (PT) and yield share (YT) into separate tokens, which works but fragments liquidity across maturities.

A veStable that wants to underwrite long-duration assets needs both answers from the same data structure.

## Why Both Signals Are Needed

A counterparty about to put up a multi-year deal (a music catalog acquisition, a private credit tranche, a long-dated repo line) needs two things confirmed before they commit:

1. The liability backing this deal is committed for at least as long as the deal lasts.
2. The committed capital is actually routed to the deal, not redirected into the liquid leg.

veDuration alone gives you (1). veUtilization alone gives you (2). Either one in isolation lets the protocol fake the other:

- A protocol with great veDuration numbers but no veUtilization is sitting on locked capital it cannot deploy. Depositors are paid for a commitment that produces no premium yield.
- A protocol with great veUtilization but no veDuration is paying out premium yield against capital that can leave next week. The first wave of redemptions blows up the deal book.

The two together close the loop. The liability side and the asset side see the same term structure.

## Both Signals On One Token

Pendle separates the principal claim (PT) and the yield claim (YT) into two transferable tokens. Each maturity needs its own PT/YT pair, which spawns its own AMM, which fragments liquidity. The fixed-term protocol graveyard (Element, Sense, Tempus, Napier) is full of teams that died from this.

A veStable carries the same two signals as metadata on a single position:

```text
veToken_i.amount        = face value of the deposit
veToken_i.lock_state    = remaining lock time, decaying toward zero
veToken_i.backing       = {asset_id: share} showing what the protocol deployed against this position
```

`lock_state` is the veDuration signal. `backing` is the veUtilization signal. The token is one redeemable claim. The two signals are readable on-chain. No PT, no YT, no AMM per maturity.

## How Weight Is Computed

Define for each position:

```text
duration_i    = remaining_lock_time_i / max_lock_time
utilization_i = share_of_position_in_illiquid_leg
```

Both are in `[0, 1]`. The honest weighting uses a floor, not a sum:

```text
effective_weight_i = locked_amount_i * min(duration_i, utilization_i)
```

`min(...)` is intentional. A depositor cannot claim premium yield for capital the protocol parked in T-bills, even if they locked it for four years. The protocol cannot claim it is deploying long-duration capital that the depositors never committed. Both halves must be true.

Premium yield is then distributed pro-rata across `effective_weight`:

```text
user_premium_yield_i = premium_yield_pool * effective_weight_i / sum(effective_weight)
```

Base yield (T-bills, Aave, Morpho) is distributed pro-rata across `locked_amount`, since base yield is earned on every dollar regardless of duration or utilization:

```text
user_base_yield_i = base_yield_pool * locked_amount_i / total_locked
```

## Routing As The Unifier

The cleanest implementation is to let lock duration drive deployment routing on the asset side. When a new illiquid deal needs to be filled, the protocol pulls capital from positions whose remaining lock covers the deal's maturity. Long locks fill long deals first.

Under this rule, the two signals stop being two separate dials. Lock duration directly determines what backs your position, which means:

- Your duration choice is your utilization choice.
- The protocol's deal flow is what realizes your yield.
- Each position's backing map is the audit trail.

If the protocol cannot find a deal matching your duration, your capital sits in liquid backing and earns base yield only. That is the honest answer to "why am I locked but not earning premium": there is no deal at your duration yet.

## The Contingent Lock

The signal `min(duration, utilization)` implies something stronger than a fixed-term lock. The lock is binding only on the portion of the position the protocol actually deployed into the illiquid leg. The undeployed portion stays liquid even when the user has signed a long lock commitment.

```text
deployed_i  = sum(backing_i.illiquid)        <- locked, illiquid
liquid_i    = amount_i - deployed_i           <- withdrawable on demand
```

A user who locks 100 USD for one year, when the protocol routes 60 USD into a 9-month catalog deal and parks 40 USD in T-bills, is locked on exactly 60 USD. The other 40 USD is a stablecoin sitting in a money-market vault that happens to live inside the protocol. They can pull it any time.

This closes the worst asymmetry of a Curve-style hard lock: "you committed, the protocol did not deploy, you eat the illiquidity anyway." Under the contingent lock, illiquidity follows deployment. No deployment, no illiquidity.

## The Zero-Utilization Steady State

When the protocol has no illiquid deployments at all:

- Every user's `utilization_i = 0`
- `effective_weight_i = 0` for everyone
- `premium_yield_pool` has no inflows because no illiquid asset is producing yield
- Everyone earns base yield only, distributed pro-rata across `locked_amount`
- Locked positions and unlocked face are functionally identical: same yield, no enforceable lock

In that state the veStable looks structurally identical to plain sUSDS or USDC-in-Morpho. Flat curve. No effective lock. The whole duration apparatus is dormant.

This is intended. The duration mechanism only switches on when there is something for it to govern. The mechanism does not impose costs in the absence of asset-side deployment, and it cannot quietly tax depositors during a period of low protocol activity. The user-facing rule is symmetric: no work from the protocol means no commitment from the depositor.

## What This Replaces

The single-token shape collapses four mechanisms that protocols usually bolt on separately:

1. **Senior / junior tranches.** Liquid holders sit senior because they redeem first against the liquid pool. Locked holders sit junior because they hold deployed backing.
2. **Discrete lockup tiers.** Continuous `duration` metadata replaces 30d / 1y / 4y buckets.
3. **ve governance tokens.** Lock weight is applied directly to the productive asset, killing the secondary token's reflexivity and FDV overhang.
4. **PT / YT splits.** Principal commitment and yield claim live as metadata on one token, so there is no AMM per maturity to fragment liquidity.

Four primitives. One token.

## Failure Modes

**Lock and no deploy.**
User locks four years, protocol has no four-year deals, capital sits in liquid earning base. Mitigation: publish the deal pipeline so users see what they are queueing for. Optionally let users specify a maximum duration they consent to being deployed against, so a four-year lock can opt to only back deals up to two years.

**Adverse routing concentration.**
If long deals are filled from the longest locks first, the longest committers are the most exposed to a single underwriting mistake. Mitigation: pro-rata across positions within a duration bucket rather than serial fill, so a bad deal spreads across the cohort.

**Wave unlock at the long end.**
A cohort that all locked four years ago all unlocks the same week. Mitigation: small randomization of expiry at deposit time, plus a redemption queue at the unlock edge.

**Game via repeated short locks.**
A user cycles seven-day locks to look committed. The duration decay handles this naturally: a seven-day lock contributes seven-day-equivalent weight, not four-year-equivalent weight. Premium yield share collapses to near zero.

**Liquid wrapper arbitrage.**
Someone builds a yvVeStable that mints a liquid claim against locked positions, Convex-on-Curve style. Mitigation: do not fight it. The underlying lock metadata is still on-chain and the protocol's term structure is unchanged. The wrapper is just a secondary liquidity venue, not an escape hatch.

**Forced redemption against deployed backing.**
A user with high `utilization` wants to exit. The protocol cannot sell their backing instantly without taking a haircut on the illiquid leg. Mitigation: redemption queue ranked by lock-end proximity. Positions whose backing is closest to maturing leave first. Positions still deep in illiquid backing wait.

## Summary

veDuration is how long the depositor committed.
veUtilization is how much of that commitment is actually working.
Both signals live as metadata on one veToken.
Premium yield is routed through the `min` of the two.
The liability side and the asset side see the same term structure, and a counterparty considering a long-duration deal has a published answer to "will this liquidity still be here when I need it?"
