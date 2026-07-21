---
title: "DashDollar, a plain explainer"
date: "21 July 2026"
geometry: margin=1in
fontsize: 11pt
---

Working name for a Dash-native, decentralized US-dollar stablecoin. This document explains, in
plain terms, what the project is, what has been built and tested so far, what the first
simulation results say, and what happens next. It is written for a reader who has not seen the
formal design record, and it ends with the working plan in outline form.

**Status on 21 July 2026.** The architecture is designed, reviewed, and locked. A simulation
harness now exists and has replayed the design's economics against nine years of DASH price
history, with results bound to reproducible receipts. No normative specification, no on-chain
code, and no testnet deployment exists yet, and several core mechanisms still depend on facts
about the Dash technology stack that have not been confirmed with the Dash team. Where a
property rests on an assumption rather than a measurement, this document says which.

A note on two layers used throughout. Dash Core is the base blockchain, the Layer 1 (L1) that
holds DASH coins and does the mining and settlement. Dash Platform is the application layer
built on top, the Layer 2 (L2) that runs user identities, documents, and a native token system.
A Dash Improvement Proposal (DIP) is Dash's formal protocol-change document, the same idea as a
Bitcoin BIP.

## 1. What it is, and what it is not yet

DashDollar is a proposed token intended to hold a value of one US dollar (USD), backed entirely
by DASH that users lock up as collateral, with no company holding the reserves and no bank
account behind it. It carries over the economic template of DigiByte's DigiDollar, adapted to
run on Dash.

Three things make it distinctive:

- It is **over-collateralized**. Every DashDollar in circulation is backed by more than a dollar
  of DASH locked behind it, between two and five dollars of DASH depending on how long the
  backer commits, so the dollar target has a cushion against the DASH price falling.
- It is **non-custodial in intent**. No single party holds the collateral. Collateral is locked
  on L1 and can only be released by a large quorum of Dash masternodes signing off. The limits
  of that claim are spelled out in Section 6.
- It lives **almost entirely on Dash Platform**. The only steps that touch the base chain are
  moving DASH into the system (peg-in) and paying it back out (peg-out). Everything else, the
  tokens, positions, prices, and accounting, lives on L2. This is the "use our own stack"
  mandate the Dash colleagues set for the project.

It is not yet a specification and not yet deployable code. It is a locked architecture, a set of
open feasibility questions for the Dash team, and a working simulator whose first results are
summarized in Section 8.

## 2. Why it exists

The Dash colleagues asked for a stablecoin that is Dash-native, decentralized, and built on the
Dash Platform stack rather than bolted on from outside. A dollar-stable token that people can
hold and pay with, backed by DASH and governed by Dash's own consensus rather than by a
custodian, gives the network a stable unit of account without importing an external stablecoin
and its trust assumptions. The economic shape was borrowed from DigiByte's DigiDollar because
that template was already worked out, and the job here was to make it fit Dash's two-layer
architecture.

## 3. The one hard rule, and what it forces

The strictest constraint on the whole design is the boundary between the two layers.

- On L1 (Dash Core), two operations and no others: locking DASH into the system through a DIP-2
  asset lock (peg-in), and releasing DASH out of the system through a DIP-7 quorum-signed asset
  unlock (peg-out).
- On L2 (Dash Platform), everything else: the DashDollar token itself, the collateral positions,
  the price feed, the accounting, the redemption logic, and the rules that decide who may mint
  or redeem.

The colleagues set this boundary as a hard requirement, and it shaped every later decision. A
key consequence is that Dash Platform has data contracts and a native token system but no
general on-chain computation. On a smart-contract chain, a contract could take a live price,
work out how many DashDollars a deposit should create, and have every validator run that same
calculation and agree on the answer. Dash Platform does not work that way. It can check the
paperwork, but it cannot do the math.

To make that concrete, suppose someone locks 500 dollars of DASH on a term whose rule requires
2.50 dollars of collateral for every DashDollar. The system should let them mint up to 200
DashDollars. Nothing in a data contract can read the live DASH price, do that division, and
enforce the 200, because the price is an outside number the platform cannot pull into a
consensus calculation. So an authorized off-chain actor computes the 200, signs a statement
authorizing exactly that mint against that deposit at that price under that parameter set, and
submits it. The platform then enforces what it actually can, namely that the signer holds the
mint authority, that the document is well formed, and that the same authorization is never used
twice. It does not re-derive the 200.

The same split runs through the whole system. The mint size, the redemption payout, the pooled
collateral ratio and its adjustment, the price itself, the periodic share-ledger updates, and
the terminal-unwind accounting are all computed off-chain by a named signer and submitted as
authorized actions, while the platform enforces only structure, ownership, uniqueness, and who
may act. The rules are deterministic and the inputs are published, so anyone can recompute any
authorized number and challenge a wrong one. Much of the design's machinery exists because the
numbers are attested and audited after the fact rather than proven up front by consensus.

## 4. How the price is held at a dollar

A point of precision before the mechanics. DashDollar is not pegged in any hard sense, and
neither is the system it borrows from. In a free market anyone may trade it at any price they
wish. What the system provides is a set of incentives that draws the market price toward parity
with the US dollar, so that trading it much above or below a dollar is a losing proposition for
whoever does it. A stablecoin is only as good as the forces that pull it back to a dollar, and
DashDollar uses two.

The floor under the price is the **redemption right**. Any holder can hand in one DashDollar and
receive a dollar's worth of the underlying DASH collateral, in proportion to the pool. If
DashDollar trades below a dollar, redeeming it for a full dollar of DASH is profitable, that
buying pressure removes supply, and the price is pulled back up. The design makes this right
immediate for any holder, meaning it is never made to wait behind the backers' commitment terms,
though it is subject to throughput limits described later. Section 8 reports what the historical
replay says about how much work this force actually does.

The ceiling over the price is **minting**. If DashDollar trades above a dollar, a backer can
lock DASH collateral, mint new DashDollars, and sell them for more than the collateral cost, and
that new supply pushes the price back down. The over-collateralization requirement sets how
cheaply new supply can be created.

The cushion that keeps the whole thing solvent when DASH itself falls is the
**over-collateralization**, between 200 and 500 percent depending on term. There is **no forced
liquidation** of individual backers. Most collateralized stablecoins liquidate a backer the
moment their collateral ratio drops, which cascades in a crash. DashDollar instead uses a
system-wide dynamic adjustment that tightens as the aggregate ratio approaches a roughly 120
percent floor, and only as a last resort a terminal unwind (Section 5). The dynamic adjustment
is a throttle that slows the bleeding, not a repair that restores lost value, and the design
records that distinction.

## 5. The moving parts

**The token.** DashDollar is a native Dash Platform token whose freeze, blacklist, and
administrative transfer powers are permanently disabled. The only special authority that exists
is minting and burning, held by a protocol role, not a person. Once you hold DashDollars, no one
can freeze or claw them back.

**Positions and collateral.** A backer opens a position by locking DASH and minting DashDollars
against it, choosing a term from 30 days up to a capped maximum in the two-to-three-year range.
The exact cap gets fixed alongside the other economic parameters, and it was deliberately
brought down from an earlier 10-year ceiling because a decade of custody on today's
cryptography is a long exposure window. Longer terms require more over-collateralization. At the
end of a term nothing is auto-liquidated. The owner can close the position, roll it over, or do
nothing and leave the collateral as backing indefinitely.

**Peg-in and minting.** A backer locks DASH on L1 with a DIP-2 asset lock. The mint waits for
ChainLock finality, Dash's near-instant confirmation from masternode quorums, so the lock cannot
be reversed out from under a freshly minted balance. The mint is bound to a signed intent that
names all the relevant details, and that intent can be consumed exactly once, so a deposit
either mints or is refunded but never both.

**Peg-out and redemption.** The flow commits first and prices later, so nobody gets a free
option on the price. The holder submits one redemption request that moves the tokens into escrow
and joins a public queue, with a stated tolerance for price slippage. Requests leave the queue
in a deterministic order within per-period caps. The admitted request is priced against the
current oracle price, with a published cap on the total cost (fee plus spread) so there is no
hidden variable exit charge. A single action then reserves the collateral, burns the escrowed
tokens, and creates the payout claim, or cancels and returns everything, with the same one-time
authorization covering both outcomes. Every escrowed balance has a published timeout, and every
entry and return is logged publicly. The payout goes out through a DIP-7 quorum-signed unlock on
L1.

**The actuator.** Because L2 cannot do the pricing math itself, an off-chain actor computes the
authorized quantities and helps sign the operations. This actuator runs deterministic, publicly
reproducible rules, so anyone can recompute every authorization it issues and catch a false one.
It is a small, fixed launch group with hardware-backed keys and a published rotation procedure,
kept separate from the oracle with distinct keys.

**The oracle.** The DASH-to-USD price comes from independent operators, each reading several
markets, combining them with medians, and signing the result under a threshold signature. Stale
prices halt price-dependent operations, and unusual price dispersion forces a conservative
fallback. The raw signed observations are published so a bad operator is attributable after the
fact. A colleague has proposed an alternative where the masternodes themselves source and sign
prices into blocks, and that option is on the evaluation list.

**The accounting.** Who is owed what, as backers enter and leave and prices move, is tracked
with a normalized-share ledger that keeps separate running indices for collateral and debt. Each
position records the index values in force when it entered, so only movement after entry applies
to it. Updates are batched per period rather than written one event at a time.

**An alternative under evaluation.** A Dash colleague, kxcd, has proposed a different economic
shape: no time locks at all, with the collateral requirement responding continuously to the DASH
price and the outstanding supply instead of being set by a chosen term. His motivating argument
is custody exposure, since collateral locked for years on today's cryptography is a standing
target, and it is the same argument that brought the maximum term down to the 2-to-3-year range.
Nearly all of the machinery described above survives under his model unchanged. What changes is
the pool's capital stability, because a locked backer cannot run and a close-at-will backer can,
so a crash produces two simultaneous exits instead of one, and any serious version of the
variant needs a mechanism that slows or prices backer exit under stress. The proposal is scoped
in the repository as `docs/VARIANT-reflexive-economics.md`, the conversation lives in the
repository's discussion threads, and the plan (Section 10, item 6) is to settle it by running
both economic models through the same 27 historical crash episodes rather than by argument.

**The terminal unwind.** If the system is provably insolvent past a public, objective trigger,
it stops taking new business and pays out an estate in a fixed order. Reversible in-flight
operations are unwound rather than paid at face value, all token holders are paid pro rata as
one class, and the classification of in-flight operations is a deterministic function others can
recompute and challenge, so the operator has no discretion to favor anyone at the end.

## 6. The trust model

This is the part the design takes the most care to state plainly.

The collateral sits on L1 behind the bridge and can only be released by a DIP-7 asset unlock
signed by the Dash Platform validator quorum, a large set of masternodes. On the current
understanding, L1 validates only that the quorum signature is present and valid, not that the
underlying L2 logic was correct. That makes the signing path a **collective custodian**. It is
decentralized, since it is a large quorum rather than one company, but it is not a cryptographic
guarantee that funds can only ever move correctly. No trustless claim is available under this
architecture, and the design makes none. Whether L1 can be made to prove more than the bare
signature is one of the open questions in Section 9.

Alongside the quorum, the actuator has specific powers, and the design lists them in a
capability matrix that separates what the platform can prevent from what it can only detect
after the fact. In plain terms, the actuator on its own could refuse service, delay or reorder
redemptions within the caps, or strand expired escrow. If certain platform-level protections
turn out to be unavailable, it could also initiate an unlock without a matching burn, and
validators following the rules would sign it. Each such power is written down rather than
hidden, and every discretionary action is logged publicly. The point of the exercise was not to
claim the system is trustless. It was to state where the trust sits and how large the damage
from each role could be.

The design accepts this collective-custodian trust as its working basis for now, and it records
a better path as future work, though the path is hard. A cryptographic version would gate the
base-chain release so collateral can only move against a proof that a real DashDollar burn
happened, using a consensus rule of the kind Dash Core is already exploring for other purposes.
Even then the ultimate trust would still rest on the majority of the masternode set following
the rules, because that burn proof is secured by the same masternodes, so this is an improvement
to the trust model rather than a route to full trustlessness.

## 7. How the design and code are checked

The architecture did not come from one author's first draft. Requirements were frozen first with
the owner. Five independent designs were then produced from an architecture-free brief by
frontier models across different families, each working without sight of the others, alongside
the author's own design. Convergences across the independent designs were treated as forced by
the requirements, and divergences became the deliberate decisions to reason through and record.
The synthesized design then went through three full adversarial review rounds, each by multiple
independent reviewers instructed to hunt for what earlier passes missed. Round one overturned
architecture-level decisions. Round two reworked mechanisms without overturning architecture.
Round three overturned nothing at the architecture level, and the design was locked at that
point.

The same discipline now applies to code. The simulation harness described in the next section
went through 19 adversarial passes across two review rounds before its results were accepted.
Among the defects those reviews caught were one experiment whose conclusion was reversed by its
own setup and one headline number contaminated by measuring past the point where the simulated
system had failed. Every result below survived that process. Each is bound to a machine-readable
receipt recording the exact data, code, and parameters that produced it.

## 8. What the historical replay says so far

A simulation harness now replays the design's economics against the full daily DASH-USD price
history available from late 2017 through the present, 3,175 daily closes. The history contains
27 distinct crash episodes of 30 percent depth or more, including the 2018 bear market in four
legs, the March 2020 crash (36.8 percent in one day), May 2021 (54.3 percent in one week), and
the 2022 bear market. The worst declines within any one day, week, and month across the whole
window are 37.2, 54.3, and 65.9 percent.

Four results, each with its number:

1. **A frozen pool fails, a redeeming pool survives.** A pool seeded at 300 percent
   collateralization with all flows switched off breaches the design's failure trigger in 7 of
   the 27 episodes, bottoming at 66.9 percent backing in the 2019 slide. The same pool with the
   modeled redemption behavior switched on fails in 0 of 27. Redemption is not just an exit
   valve. Paying a redeemer one dollar of DASH per token removes proportionally less collateral
   than debt whenever backing exceeds 100 percent, so every redemption deleverages the pool.
2. **Redemption friction is the dominant risk lever.** Sweeping the behavioral assumptions
   across 243 parameter combinations, the worst outcomes (5 of 27 episodes failing) all occur
   where redeeming is expensive (a 3 percent round-trip cost) and the market underprices the
   solvency risk. Every combination where redeeming is cheap survives all 27 episodes. For the
   design, this says the published cap on total redemption cost is not merely a fairness
   feature. It is the parameter the system's survival is most sensitive to.
3. **The dollar target bends before it breaks.** Across the sweep, the modeled market discount
   reaches 15 to 28 percent during the worst weeks of the deepest crashes before parity
   recovers. That is the price of a design with no forced liquidations under a 65 percent
   monthly collateral crash, under these behavioral assumptions.
4. **The conclusions are not hostage to the softest assumption.** The model cannot observe an
   above-par price, so it assumes minting stays available whenever no discount exists. Turning
   minting off entirely, the worst case of that assumption, moves the headline from 0 of 27
   episodes failing to 1 of 27. The bound is published in the same receipt as the result.

Three cautions belong next to those numbers. The behavior functions are stated assumptions with
swept coefficients, not measurements of real users. The alternative economic model a colleague
proposed (no time locks, collateral requirements that respond to the market) is scoped but not
yet built, so the comparison that discussion is waiting on does not exist yet. And the simulated
failure trigger is a placeholder for the real terminal mechanism, so failure counts mark where
the design's alarm would sound, not a full account of what an unwind would pay.

## 9. The open questions that gate everything

The immediate next step is not more building. Eight feasibility facts need answers from the Dash
Platform team or from direct testnet probing, because they decide whether large parts of the
design are enforced by consensus or are policy that auditors can only check after the fact. The
three that matter most:

1. Can one Platform action atomically consume escrowed tokens, create a claim, and reserve
   collateral together? The safety class of redemption depends on the answer.
2. Can a signed intent be consumed exactly once across a token action and a document, making
   mint-versus-refund and burn-versus-cancel mutually exclusive by construction?
3. What does a DIP-7 unlock actually prove on L1? If the base chain verifies that a matching L2
   request exists, at-most-once payment is a protocol guarantee. If it only checks the quorum
   signature, the collective-custodian trust floor of Section 6 is confirmed as the floor.

Favorable answers make the strong version of the design buildable. Mixed answers, the likely
case, mean building with the trust model stated plainly and several properties labeled as policy
with public detection. Unfavorable answers force a choice between materially weaker guarantees
and asking the Dash core team for a base-layer protocol change, which is a much larger and
slower path.

## 10. The working plan, in outline

The plan below is ordered by what blocks what. Items marked done are complete and review-closed.

1. **Design** (done). Requirements frozen, five-design clean-room round, three adversarial
   review rounds, architecture locked, maximum term amended to the 2-to-3-year range.
2. **Project infrastructure** (done). Private canonical repository with review evidence, a
   curated public repository with a leak-gated exporter, and public discussion threads where the
   colleague conversation now lives.
3. **Replay harness, first two layers** (done). Frozen price data with provenance, a
   27-episode crash catalog, an integer-arithmetic pool simulator with conservation checks, a
   behavior layer with stated assumptions, and receipt-bound results. Nineteen adversarial
   passes across two rounds, closed with zero findings outstanding.
4. **Feasibility answers** (next, blocking). Put the eight Phase 0 questions to the Dash
   Platform team, with the testnet-probeable subset measured directly on a local network. The
   question message for the Core lead is drafted and waiting to send.
5. **Owner decisions** (open, not blocking the questions). The shape of the collateral schedule
   across the amended term range, and the functional form of redemption demand. The harness
   currently brackets both with stated stand-ins and says so in its artifacts.
6. **Model B and the comparison** (after a position-close transition is added to the
   simulator). Build the colleague's no-time-lock reflexive model, run both models through the
   same 27 episodes, and take the numbers back to the public discussion thread.
7. **Specification** (after the feasibility answers). Write SPEC.md, the normative state
   machine with exact arithmetic and test vectors. Its first full multi-model review round is
   the review discipline's required follow-up to the design lock.
8. **Custody proof on testnet** (after specification, kills the design if it fails). Prove the
   bridge end to end: a peg-in credits an escrow identity the depositor cannot raid, the
   actuator threshold can move it, and a burn-gated peg-out survives a validator rotation.
9. **Parameters under stress** (after Model B and the spec). Fix the term cap, fees, and caps
   from replay results across both economic models, with the redemption-cost cap treated as the
   survival parameter the sweep showed it to be.
10. **Hardening and a bounded launch** (last). Adversarial testing, a capped initial issuance,
    and staged removal of launch controls after full cycles across key rotations.

## 11. Where it stands

The thinking is done and stress-tested on paper, and the first layer of that thinking now runs
as tested code against nine years of price history. The trust assumptions are stated rather than
hidden, the simulation's own assumptions are published with their sensitivity bounds, and the
project's critical path runs through eight questions to the Dash Platform team, not through more
design. After those answers, the work is specification, a testnet custody proof, and the
economics comparison the public discussion is waiting on.
