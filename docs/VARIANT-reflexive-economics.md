# Variant memo, reflexive economics without time locks

Written 2026-07-19 from a proposal by [@kxcd](https://github.com/kxcd). STATUS: a sanctioned exploration, not the
architecture of record. DESIGN.md v4 with the frozen (and 2026-07-19 amended) term economics remains
the design this project builds. This memo scopes the variant so it can be evaluated on equal terms,
and it graduates to its own design branch only if the exploration produces a full alternative
architecture worth specifying.

The live conversation on this memo happens in the repository discussions. The economics of this
variant, and in particular the backer-exit question below, are discussed in
[discussion #1](https://github.com/hilawe/dashdollar/discussions/1). @kxcd's related
oracle proposal (masternode-sourced pricing, a separate evaluation item in TODO.md Phase 2) is
discussed in [discussion #2](https://github.com/hilawe/dashdollar/discussions/2). Decisions reached
in those threads get folded back into this memo and the build plan, so the memo stays the record
and the threads stay the conversation.

## The proposal

Positions carry no time locks. A backer may open or close a position at any time. The collateral
requirement is not a function of a chosen term but is reflexive to market conditions, moving with
the DASH price and the outstanding stablecoin supply. When the token trades above parity, the
required collateral per token falls, which invites new minting, and the new supply presses the price
back toward a dollar. The redemption side is unchanged in spirit, holders redeem for underlying
collateral when the token trades below parity. @kxcd's motivating argument against time
locks is custody exposure, since collateral locked for years on a non-quantum-safe L1 is a standing
target, and a holder cannot respond to a cryptographic break until the lock expires.

## What survives from v4 unchanged

Nearly all of the machinery, because the architecture was kept economics-agnostic. The bridge and
escrow custody path, the signed-intent mint binding, the commit-first redemption queue with
deterministic admission, the normalized-share apportionment ledger, the oracle, the capability
matrix, the terminal unwind, and the auditor conventions all operate the same way whether the
collateral schedule is term-based or reflexive. A variant build would change parameter logic inside
the actuator's published rules, not the custody or settlement structure.

## What changes

- The term curve disappears, and with it the position-expiry transitions (close, roll, or dormant)
  defined in D5. Positions become open-ended with close-at-will.
- The collateral requirement becomes a published function of price and outstanding supply. That
  function is new consensus-relevant surface for the actuator rules and needs the same
  deterministic, reproducible treatment as the mint inequality.
- The 200-to-500-percent by-term schedule is replaced by whatever range the reflexive function
  spans, and the below-floor dynamic adjustment has to be reconciled with it, since both now react
  to the same market signal and could fight each other.

## The first question the variant must answer

Time locks are not only heritage, they are the pool's capital stability. A locked backer cannot
run. Remove the locks and a DASH crash produces two simultaneous exits, holders redeeming (designed
for, immediate pro-rata with back-pressure and caps) and backers closing positions to pull
collateral out (structurally prevented mid-term in v4). The backing becomes flighty at the exact
moment it is needed. Any serious version of this variant needs a mechanism that slows or prices
backer exit under stress, for example exit fees that scale with the aggregate ratio, notice periods,
or a close queue subject to the same admission discipline as redemption. Reflexivity itself needs
care on the downside, because a rule that raises collateral requirements as DASH falls chokes the
minting that parity support needs, while a rule that loosens them under-collateralizes new positions
in a falling market.

## How it gets decided

Not by argument. Because both models share the same machine underneath, the Phase 3 economic replay
harness can run both through identical historical DASH drawdowns with endogenous behavior (mint
cessation, DASH selling, redemption queue dynamics, and for this variant, backer-exit dynamics) and
compare parity retention, pool drawdown, and terminal-trigger proximity. The comparison plan is a
standing item in TODO.md Phase 3. Until those numbers exist, the frozen-template economics remain
the design of record and this memo remains the record of the alternative.
