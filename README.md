# veStable: Stablecoin Duration as a Primitive

Most yield-bearing stablecoins split liquidity into two instruments: the transferable stablecoin and a staked wrapper. The stablecoin is money; the wrapper is yield. That architecture is simple, but it throws away the most important variable on the liability side: duration.

A **veStable** makes duration native to the stablecoin itself.

The base asset remains a normal redeemable stablecoin. Hold it unlocked and it behaves like cash. Lock it, and the protocol records both amount and remaining lock time. That lock mints no separate claim on principal. It creates decaying `ve` weight:

```text
ve_i = locked_amount_i * remaining_lock_time_i / max_lock_time
```

With a 4-year max lock, a fresh 4-year lock receives full weight, a 2-year lock receives half weight, and all weight decays toward zero as unlock approaches. Users must extend duration to maintain weight.

The important distinction is that this `ve` balance is not primarily governance power over emissions. It is a duration-adjusted claim on variable yield.

A minimal veStable has three liquidity states:

```text
Unlocked stablecoin
- fully liquid
- no yield

Minimum lock, e.g. 1 week
- earns base yield
- backed by T-bills, repo, money markets, or other liquid collateral

Longer lock
- earns base yield
- receives pro-rata priority over premium yield
- premium yield comes from less liquid opportunities that require dated capital
```

Yield distribution becomes two-layered:

```text
base_yield_pool -> distributed across locked stablecoin balances
premium_yield_pool -> distributed across live ve weight
```

So for user `i`:

```text
user_base_yield =
  base_yield_pool * locked_balance_i / total_locked_balance

user_premium_yield =
  premium_yield_pool * ve_i / total_ve
```

Longer locks do not set a fixed APY. They increase priority over whatever premium yield the system actually produces.

That matters because the realized rate is endogenous. It depends on the amount of premium yield generated, the amount of capital competing for it, and the shape of the lock curve. If many users lock long, premium yield is diluted across more `ve`. If few users lock long while premium yield is abundant, realized yield expands. The protocol is not promising a rate; it is clearing yield against duration.

This turns the stablecoin liability side into an observable term structure:

```text
How much supply is liquid?
How much is locked for one week?
How much is locked for six months?
How much is locked for two years?
How much is locked for four years?
```

That curve can govern what the asset side is allowed to do. Instant-liquidity holders should not be paid for illiquid deployment. Long-duration holders should not receive a fixed subsidy disconnected from actual returns. Yield should follow the usefulness of liquidity.

In the standard model, a yield-bearing stablecoin hides this inside a wrapper. In a veStable, the stablecoin itself becomes the market for dated liquidity.

The primitive is simple:

> one stablecoin, multiple liquidity states, decaying duration weight, variable yield allocated by how useful the user's liquidity is to the system.

The result is not `stablecoin + staked stablecoin`. It is a redeemable stable asset with an endogenous yield surface determined by duration, utilization, and realized asset-side returns.
