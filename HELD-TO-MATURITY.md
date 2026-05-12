# Held To Maturity: What veStable Refuses To Be

The veStable is a tokenized term-deposit primitive. The unlocked face is a redeemable stablecoin backed by liquid assets. The locked position is a non-transferable claim that matures into face value at a chosen duration, backed by duration-matched illiquid assets. There is no third-party wrapper, no secondary market, no aggregator-issued liquid token, and no protocol-side mechanism for recreating the liquidity-on-illiquidity mismatch the protocol exists to avoid.

## The Class Of Product Being Replaced

A pattern keeps recurring in DeFi: depositors hold a claim they believe is daily-liquid, the protocol holds an asset book that takes weeks to quarters to liquidate, calm hides the mismatch, stress exposes it. The break mode is always the same. Liability-side redemption velocity exceeds asset-side liquidation capacity, the protocol fire-sells at a discount, halts redemptions, or socializes losses.

Specific products in this class:

- **Anchor on UST and equivalents.** 20 percent paid on theoretically liquid stablecoin deposits, funded from collateral yields that could not actually be liquidated at the promised rate. Collapse was the predictable outcome.
- **Money market lending books (Aave, Compound, Spark).** Theoretical instant withdrawal for lenders, term loans for borrowers. Utilization curves cushion the mismatch in calm and stress events (March 2020, Iron Bank, every depeg cascade) expose it.
- **Liquid staking derivatives (stETH, rETH, and the family).** Token trades 1:1 in calm. Queue length blows out under stress (June 2022 stETH depeg). The 1:1 pricing pretense is the source of fragility, not a feature.
- **Tokenized treasury wrappers (Ondo OUSG, Mountain USDM, Maple Treasury).** Daily-liquid claim against assets with weeks-to-months redemption windows. Liquidity sleeves buy time. Asset and liability tenors still do not match.
- **On-chain private credit wrappers (Maple syrupUSDC, Goldfinch FIDU).** Term loans on the asset side dressed up as ERC-20s with cooldown queues on the liability side. Better than the others. Still mismatched.

Each survives in calm and breaks in stress. The fragility is structural, not behavioral.

## What veStable Does Instead

Two products, neither carrying the mismatch.

**Unlocked face.**
A normal stablecoin backed by liquid assets only: T-bills, Aave / Morpho deposits, money-market exposures. Daily-redeemable at par. Earns base yield. No promise of premium yield. No illiquid backing.

**Locked face.**
A non-transferable position bound to the depositor's address. Depositor picks a duration at deposit. Protocol routes the position into duration-matched illiquid assets. Position is closed until expiry. Earns base yield plus the realized premium yield from the assets backing it. At expiry it unwinds to face value and joins the unlocked supply.

Structurally the locked face is a CD or a closed-end interval-fund position. It does not pretend to be liquid. Anyone who needs liquidity buys the unlocked face.

## What The Protocol Refuses To Build Or Support

**No transferable lock.**
The position is bound to the depositor's address. Cannot be sold, gifted, or used as collateral inside the protocol's integrations. The duration commitment is the product. Allowing transfer creates a secondary market whose pricing becomes the de facto term structure and undoes the matching property.

**No protocol-side NAV oracle for third parties.**
A wrapper needs per-position pricing. The protocol does not publish that oracle. Without it, third-party wrappers either invent their own pricing (and inherit the credit risk) or wait until expiry (which removes the liquidity advantage that motivated the wrapper).

**No documented Pendle SY adapter.**
Pendle works by SY-wrapping any yield-bearing token. The protocol does not publish or maintain integration code, does not list integrations in documentation, does not coordinate maturity schedules with Pendle, does not subsidize PT or YT pool liquidity. A third party who wants to ship this externally bears the integration risk and the credit risk.

**No Yearn-style aggregator endorsement.**
A vault that holds locked positions and mints a fungible yield token re-imports the liability mismatch at the wrapper layer. The protocol does not co-sign, integrate, or feature any such product. Aggregators that pursue the route do so as credit-risk intermediaries on their own balance sheet, not as composable extensions of the protocol.

**No early redemption window.**
No quarterly cooldown. No 5 percent haircut buyback. No emergency exit. The lock is the product. Users who need liquidity buy the unlocked face. Users who bought the locked face have priced their own duration tolerance and the protocol does not provide a mulligan.

## Why "No Aggregators" Is The Stance

A wrapper exists to convert an illiquid position into a liquid claim. Every wrapper that succeeds at this re-imports the fragility the underlying protocol shed. The wrapper has a run, the wrapper's pricing breaks, the wrapper's holders panic. The underlying protocol either gets dragged in or watches the wrapper collapse with the protocol's reputation attached.

The veStable's structural advantage is precisely that no liquid claim on illiquid assets exists in the entire stack. Allowing aggregators to manufacture one externally hands the fragility back to the ecosystem with the protocol's product as the input.

Two practical responses to "someone will build it anyway":

1. Without protocol-side support (NAV oracle, transferable position primitive, integration documentation), the wrapper is materially harder to build. Most never ship.
2. If a wrapper does ship externally, its credit risk is its own. The underlying lock is non-transferable, so the wrapper holds depositor IOUs against itself, not protocol claims. A run on the wrapper does not propagate to the protocol's term structure or solvency.

## The Zero-Utilization Floor Still Holds

The contingent lock from MECHANICS.md still applies. The lock binds only on the deployed portion of the position. If the protocol has no illiquid deployments yet, the lock is functionally void: every depositor earns base yield and can withdraw freely. The depositor's commitment is only enforced where the protocol used it.

This is the symmetric guarantee. The protocol does not extract liquidity it does not deploy, and the depositor does not promise liquidity the protocol cannot use. Both sides commit or neither does.

## What The Protocol Gives Up By Doing This

The pitch of "we are the future of liquid yield" disappears. Composability with the major DeFi yield primitives (Pendle, Yearn, Convex) disappears. The TVL flywheel that comes from being a building block for other people's products disappears.

What remains is a smaller, more durable narrative: tokenized term deposits with honest duration matching, positioned against bank CDs and closed-end credit funds rather than against other DeFi protocols. The narrative TAM is smaller. The actual TAM in capital looking for this exposure (private credit allocators, institutional treasuries, family offices, conservative DAO treasuries) is plausibly larger, and the durability of the product is meaningfully higher because there is no class of stress event that breaks the design.

## One Line

The veStable is not a DeFi yield wrapper. It is the tokenized version of "you bought a CD, you hold it to maturity, and nobody invented a synthetic claim against it on top."
