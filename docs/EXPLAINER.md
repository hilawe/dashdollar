---
title: "DashDollar, a plain explainer"
date: "18 July 2026"
geometry: margin=1in
fontsize: 11pt
---

Working name for a Dash-native, decentralized US-dollar stablecoin. This document explains,
in plain terms, what the project is, what has been produced so far, and what happens next depending
on the answers to a set of still-open feasibility questions. It is written for a reader who has not
seen the formal design record.

**Status on 18 July 2026.** This is a design, not a running system. The architecture is locked after
review, but no normative specification, no code, and no on-chain deployment exists yet, and several
core mechanisms depend on facts about the Dash technology stack that have not yet been confirmed with
the Dash team. Nothing here should be read as a delivered or proven guarantee. Where a property
depends on an unconfirmed fact, this document says so.

A note on two layers used throughout. Dash Core is the base blockchain, the Layer 1 (L1) that holds
DASH coins and does the mining and settlement. Dash Platform is the application layer built on top,
the Layer 2 (L2) that runs user identities, documents, and a native token system. A Dash Improvement
Proposal (DIP) is Dash's formal protocol-change document, the same idea as a Bitcoin BIP.

## 1. What it is, and what it is not yet

DashDollar is a proposed token that is meant to stay worth one US dollar (USD), backed entirely by
DASH that users lock up as collateral, with no company holding the reserves and no bank account
behind it. It carries over the economic template of DigiByte's DigiDollar, adapted to run on Dash.

Three things make it distinctive:

- It is **over-collateralized**. Every DashDollar in circulation is backed by more than a dollar of
  DASH locked behind it, between two and five dollars of DASH depending on how long the backer
  commits, so the dollar target has a cushion against the DASH price falling.
- It is **non-custodial in intent**. No single party holds the collateral. Collateral is locked on
  L1 and can only be released by a large quorum of Dash masternodes signing off. The limits of
  that claim are spelled out in Section 6.
- It lives **almost entirely on Dash Platform**. The only steps that touch the base chain are moving
  DASH into the system (peg-in) and paying it back out (peg-out). Everything else, the tokens,
  positions, prices, and accounting, lives on L2. This is the "use our own stack" mandate the Dash
  colleagues set for the project.

It is not yet a specification, not yet code, and not yet tested against the real Dash Platform. It is
an agreed architecture with its trust assumptions stated and a list of facts that must be checked
before anyone builds it.

## 2. Why it exists

The Dash colleagues asked for a stablecoin that is Dash-native, decentralized, and built on the Dash
Platform stack rather than bolted on from outside. A dollar-stable token that people can hold and pay
with, backed by DASH and governed by Dash's own consensus rather than by a custodian, gives the
network a stable unit of account without importing an external stablecoin and its trust
assumptions. The economic shape was borrowed from DigiByte's DigiDollar because that template
was already worked out, and the job here was to make it fit Dash's two-layer architecture.

## 3. The one hard rule: base chain for the bridge, application layer for everything else

The strictest constraint on the whole design is the boundary between the two layers.

- On L1 (Dash Core), two operations and no others: locking DASH into the system through a DIP-2 asset
  lock (peg-in), and releasing DASH out of the system through a DIP-7 quorum-signed asset unlock
  (peg-out).
- On L2 (Dash Platform), everything else: the DashDollar token itself, the collateral positions,
  the price feed, the accounting, the redemption logic, and the rules that decide who may mint or
  redeem.

The colleagues set this boundary as a hard requirement, and it shaped every later decision. A key
consequence is that Dash Platform has data contracts and a native token system but no general
on-chain computation. On a smart-contract chain, a contract could take a live price, work out how
many DashDollars a deposit should create, and have every validator run that same calculation and
agree on the answer. Dash Platform does not work that way. It can check the paperwork, but it cannot
do the math.

To make that concrete, suppose someone locks 500 dollars of DASH on a term whose rule requires 2.50
dollars of collateral for every DashDollar. The system should let them mint up to 200 DashDollars.
Nothing in a data contract can read the live DASH price, do that division, and enforce the 200,
because the price is an outside number the platform cannot pull into a consensus calculation. So an
authorized off-chain actor computes the 200, signs a statement authorizing exactly that mint against
that deposit at that price under that parameter set, and submits it. The platform then enforces what
it actually can, namely that the signer holds the mint authority, that the document is well formed,
and that the same authorization is never used twice. It does not re-derive the 200.

The same split runs through the whole system. The mint size, the redemption payout, the pooled
collateral ratio and its adjustment, the price itself, the periodic share-ledger updates, and the
terminal-unwind accounting are all computed off-chain by a named signer and submitted as authorized
actions, while the platform enforces only structure, ownership, uniqueness, and who may act. The
rules are deterministic and the inputs are published, so anyone can recompute any authorized number
and challenge a wrong one. Much of the
design's machinery exists because the numbers are attested and audited after the fact rather than
proven up front by consensus.

## 4. How the price is held at a dollar

A point of precision before the mechanics. DashDollar is not pegged in any hard sense, and neither
is the system it borrows from. In a free market anyone may trade it at any price they wish. What the
system provides is a set of incentives that draws the market price toward parity with the US dollar,
so that trading it much above or below a dollar is a losing proposition for whoever does it. A
stablecoin is only as good as the forces that pull it back to a dollar, and DashDollar uses two.

The floor under the price is the **redemption right**. Any holder can hand in one DashDollar and
receive a dollar's worth of the underlying DASH collateral, in proportion to the pool. If DashDollar
ever trades below a dollar, redeeming it for a full dollar of DASH is profitable, that buying pressure
removes supply, and the price is pulled back up. The design makes this right immediate for any holder,
meaning it is never made to wait behind the backers' commitment terms, though it is subject to
throughput limits described later.

The ceiling over the price is **minting**. If DashDollar trades above a dollar, a backer can lock DASH
collateral, mint new DashDollars, and sell them for more than the collateral cost, and that new supply
pushes the price back down. The over-collateralization requirement sets how cheaply new supply can be
created.

The cushion that keeps the whole thing solvent when DASH itself falls is the **over-collateralization**,
between 200 and 500 percent depending on term. There is **no forced liquidation** of
individual backers. Most collateralized stablecoins liquidate a backer the moment their collateral
ratio drops, which cascades in a crash. DashDollar instead uses a system-wide dynamic adjustment that
tightens as the aggregate ratio approaches a roughly 120 percent floor, backed by a funded loss
reserve, and only as a last resort a terminal unwind (Section 5). The dynamic adjustment is a throttle
that slows the bleeding, not a repair that restores lost value, and the design is explicit about that
distinction.

## 5. The moving parts

**The token.** DashDollar is a native Dash Platform token whose freeze, blacklist, and administrative
transfer powers are permanently disabled. The only special authority that exists is minting and
burning, and that authority is held by a protocol role, not a person. Once you hold DashDollars, no
one can freeze or claw them back.

**Positions and collateral.** A backer opens a position by locking DASH and minting DashDollars
against it, choosing a term from 30 days up to a capped maximum in the two-to-three-year range (the
exact cap gets fixed alongside the other economic parameters, and it was deliberately brought down
from an earlier 10-year ceiling because a decade of custody on today's cryptography is a long
exposure window). Longer terms require more over-collateralization.
Positions are documents on Dash Platform, and the platform checks their structure and ownership but
not the price math. At the end of a term nothing is auto-liquidated. The owner can close the position
by acquiring and burning its share of debt and withdrawing the collateral, or roll it over at current
terms, or simply do nothing and leave the collateral as backing indefinitely, still redeemable
against and never force-liquidated.

**Peg-in and minting.** A backer locks DASH on L1 with a DIP-2 asset lock. The mint waits for
ChainLock finality, Dash's near-instant confirmation from masternode quorums, so the lock cannot be
reversed out from under a freshly minted balance. The mint is bound to a signed intent that names all
the relevant details, and that intent can be consumed exactly once, so a deposit either mints or is
refunded but never both.

**Peg-out and redemption.** This is the most reworked part of the design, rebuilt across the review
rounds until it stopped leaking value. The flow, in order:

1. **Commit first.** The holder submits one redemption request that moves the tokens into escrow and
   joins a public queue, with a stated tolerance for price slippage. No price is fixed yet, so there
   is nothing for a holder to game by waiting.
2. **Admit in a fixed order.** Requests leave the queue in a deterministic order, within per-period
   caps that limit how much can be redeemed at once.
3. **Price at admission.** The admitted request is priced against the current oracle price under the
   holder's slippage tolerance, with a published cap on the total cost (fee plus spread) so there is
   no hidden variable exit charge.
4. **Reserve, burn, and claim in one step.** A single action reserves the collateral, burns the
   escrowed tokens, and creates the payout claim, or cancels and returns everything. The same
   one-time authorization covers both outcomes, so no path both burns and refunds.
5. **Escrow has a deadline.** Every escrowed balance has a published timeout after which it becomes
   returnable, and every entry and return is logged publicly so stuck funds are visible.

The actual dollar payout then goes out through a DIP-7 quorum-signed unlock on L1.

**The actuator.** Because L2 cannot do the pricing math itself, an off-chain actor computes the
authorized quantities and helps sign the operations. This actuator runs deterministic, publicly
reproducible rules, so anyone can recompute every authorization it issues and catch a false one. It is
a small, fixed launch group with hardware-backed keys and a published rotation procedure. It is kept
separate from the oracle, with distinct keys and rotation cadences.

**The oracle.** The DASH-to-USD price comes from independent operators, each reading several markets,
combining them with medians, and signing the result under a threshold signature. Stale prices halt
price-dependent operations, and unusual price dispersion forces a conservative fallback. The raw
signed observations are published so a bad operator is attributable after the fact.

**The accounting.** Tracking who is owed what, as backers enter and leave and prices move, is done
with a normalized-share ledger. It keeps separate running indices for collateral and for debt, and
each position records the index values in force when it entered, so only movement after entry applies
to it. Updates are batched per time period rather than written one-redemption-at-a-time, because
per-event updates would force every redemption through a single slow chain of writes.

**The terminal unwind.** If the system is provably insolvent past a public, objective trigger, it
stops taking new business and pays out an estate in a fixed order. The design hardened this path so it
cannot be gamed in the final window: reversible in-flight operations are unwound rather than paid at
face value, all token holders are paid pro rata as one class, and the loss reserve joins the estate on
the holders' side. The trigger is objective and the classification of in-flight operations is a
deterministic function others can recompute and challenge, so the operator has no discretion to favor
anyone at the end.

## 6. The trust model

This is the part the design takes the most care to state plainly.

The collateral sits on L1 behind the bridge and can only be released by a DIP-7 asset unlock signed by
the Dash Platform validator quorum, which is a large set of masternodes. On the current understanding,
L1 validates only that the quorum signature is present and valid, not that the underlying L2 logic was
correct. That makes the signing path a **collective custodian**. It is decentralized, since it is a
large quorum rather than one company, but it is not a cryptographic guarantee that funds can only ever
move correctly. No trustless, non-custodial claim is available under this architecture, and the design
makes none. Whether L1 can be made to prove more than the bare signature is one of the open questions
in Section 9, and the answer sets whether this trust floor stays where it is or improves.

Alongside the quorum, the actuator has specific powers, and the design lists them in a capability
matrix that separates what the platform can prevent from what it can only detect after the fact. In
plain terms, the actuator on its own could refuse service, delay or reorder redemptions within the
caps, or strand expired escrow, and if certain platform-level protections turn out to be
unavailable it could also initiate an unlock without a matching burn, which honest validators would
then sign. Each such power is written down rather than hidden, and every discretionary action is
logged publicly so misbehavior is at least visible in near real time. The point of the exercise was
not to claim the system is trustless. It was to state where the trust sits and how large the damage
from each role could be.

The design accepts this collective-custodian trust as its working basis for now, and it records a
better path as future work, though the path is hard. A cryptographic version would
make the base chain release collateral only against proof that a genuine, properly backed redemption
occurred, using a consensus rule of the kind Dash Core is starting to explore. The catch is that
proving a mere token burn is not enough, because the party that computes mints could itself create
and burn tokens to manufacture such a proof, so the rule would have to verify a full holder-authorized
redemption with the exact payout fixed, which pulls much of the system's own logic down onto the base
chain. Even if that were built, the ultimate trust would still rest on the honest majority of the
masternode set, so it is an improvement to the trust model rather than a route to full trustlessness,
and for now the plain collective-custodian model is what the design uses.

## 7. How the design was produced, and why that supports confidence

The architecture did not come from one author's first draft. It came from a clean-room, multi-model
process designed to remove single-author blind spots.

- Requirements were frozen first with the owner: the economics, the strict L1-and-L2 boundary, and
  the rule that peg-in and peg-out are first-class required flows.
- Those requirements were packaged into an architecture-free brief that stated the problem and the
  constraints but withheld any preferred solution.
- Five independent designs were produced from that brief by frontier models across different
  families, each working without sight of the others, alongside the author's own design.
- The designs were then compared. Points where independent designers converged were treated as
  forced by the requirements. Points where they diverged became the deliberate decisions to reason
  through and record.
- The synthesized design then went through three full adversarial review rounds, each by multiple
  independent reviewers instructed to hunt for what earlier passes missed. Findings were folded in
  as versions two, three, and four.

The trajectory of those rounds is the real evidence for calling the architecture settled. Round one
overturned architecture-level decisions. Round two reworked mechanisms without overturning
architecture. Round three overturned no architecture decision at all, and every remaining finding was
a mechanism correction, a completeness gap, or a claim-strength label. Decreasing depth across rounds, with
the structure going still by the third, is the signal the process is built to detect. All of that
review evidence is preserved verbatim as the basis for the "converged" claim.

## 8. What has been done so far

- **Requirements frozen** with the owner, including the economics and the hard layer boundary.
- **Five independent clean-room designs** produced and compared, plus the author's own.
- **A synthesized architecture record** written and revised through three adversarial review rounds
  into its current version four.
- **Architecture locked at version four** on 18 July 2026, with the owner confirming the disposition
  and declining a fourth review round on the architecture record, on the basis that further defect
  hunting belongs against the normative specification, not the architecture prose.
- **A phased build plan** written, ordered so the riskiest assumptions get tested first.
- **A private version-controlled home** for the project, with the full review evidence preserved, a
  clean secret scan, a local pre-push leak guard, and a standing rule that the private repository is
  never made public as-is and any public version is a separately curated export.

The normative specification, any code, any testnet result, and any confirmed answer to the
feasibility questions below do not exist yet.

## 9. What happens next, and how the outcome depends on the results

The immediate next step is not to build. It is to get eight feasibility facts answered by the Dash
Platform team or by direct testnet probing. These answers decide whether large parts of the design
are backed by consensus (the platform enforces them) or are merely policy that auditors can detect
violations of after the fact (the platform cannot enforce them). That difference is the difference
between a strong guarantee and a promise with a smoke alarm attached.

### 9.1 The eight open questions, in plain terms

1. Can a single platform action atomically consume escrowed tokens, create a claim, and reserve
   collateral together, or be made equivalent by the platform's uniqueness rules? The safety of
   redemption depends on this.
2. Can a signed intent be consumed exactly once across both a token action and a document, so that
   minting and refunding, or burning and cancelling, are mutually exclusive by construction?
3. What exactly does a DIP-7 unlock prove on L1? Does the base chain verify that a valid L2 request
   exists and matches, or does it merely accept the quorum signature? Does it prevent the same
   withdrawal from being signed twice?
4. Can an asset lock credit the dedicated reserve identity, and can the lock payload carry the
   one-time claim-key commitment the custody design needs?
5. Is there any stablecoin-aware check that can run before a peg-out is signed, without a platform
   protocol change?
6. Can escrow-held balances be prevented from being spent on ordinary platform fees, which would
   otherwise leak value out of the accounting?
7. What is the real throughput for withdrawals and for repeated updates to a single shared document?
   This sets the redemption caps and the accounting batch cadence.
8. Can a token transfer into an escrow identity carry a consensus-enforced, time-based automatic
   return, so stranded escrow releases itself?

### 9.2 The three answers that matter most, and what each one means

Three of these dominate the outcome.

- **Atomicity and single-consumption (questions 1 and 2).** If the platform can enforce these, then
  redemption and minting are safe by construction: you cannot burn without reserving, and you cannot
  both mint and refund the same deposit. If it cannot, those safety properties drop to actuator
  policy. Auditors would still detect a violation, but the platform would not prevent it, so a
  faulty or dishonest actuator becomes a live risk rather than a blocked one.
- **DIP-7 unlock semantics (question 3).** If the base chain proves the L2 request and blocks a
  repeat signing, then paying each redemption at most once is a protocol guarantee, and the trust
  floor of Section 6 improves. If the base chain only accepts the quorum signature, then at-most-once
  payment is custodian policy, the collective-custodian framing is confirmed as the floor, and a
  colluding or buggy signer set is the thing standing between collateral and a double payment.

### 9.3 The three overall scenarios

Rolling those answers up, the project lands in one of three places.

- **Favorable.** The platform enforces atomicity and single-consumption and the base chain proves
  the L2 request. Most of the design proceeds as a consensus-backed system, the redemption and mint
  safety properties are real guarantees, and the trust floor is better than the current
  assumption. This is the strongest story and it proceeds straight to specification and build.
- **Mixed, and most likely.** Some protections are enforceable and some are not, and in particular
  the base chain probably only checks the quorum signature. The design still proceeds, but with the
  collective-custodian trust floor stated plainly and several properties labeled as policy with
  auditor detection rather than hard guarantees. This is viable and was designed for from the start,
  which is why the capability matrix and the public logging exist. The claims made to users are more
  measured.
- **Unfavorable.** The platform can enforce neither atomicity nor single-consumption, and the base
  chain cannot prevent a duplicate unlock. Then there are two choices, and neither is quick. Accept
  materially weaker guarantees, with meaningful trust placed in the actuator and the quorum and a lot
  resting on after-the-fact detection, or ask the Dash core team for a protocol change, a new or
  amended DIP, so the base layer can enforce what the design needs. A protocol change is a much
  larger, slower ask and would reshape the timeline, but the design is structured so that this is the
  fallback rather than a surprise.

### 9.4 The build road after the questions are answered

Assuming the answers are favorable enough to continue, the plan is ordered by risk.

- **Custody slice on testnet.** Prove the bridge end to end before anything else, so that a peg-in
  credits an escrow identity, the depositor cannot pull it back, the actuator threshold can, and a
  burn-gated peg-out signs and confirms across a rotation of the validator quorum. If this fails, the
  design fails, which is why it comes first.
- **Normative specification.** Write the full state machine, the exact integer arithmetic, and the
  test vectors as a specification document. This becomes the artifact that future reviews and any two
  independent implementers are held against, and its first full multi-model review round is the
  follow-up pass the review discipline calls for after convergence.
- **Actuator and oracle.** Build the deterministic, publicly reproducible authorizer and the
  multi-operator signed price oracle, kept separate from each other.
- **Economics under stress.** Replay the parity incentives against historical DASH crashes,
  including how people would actually behave in one, minting stopping and DASH being sold and the
  redemption queue filling. If parity fails even with immediate redemption, the maximum term gets
  shortened until it holds. This is its own gate, independent of the platform-feasibility gate, and even a technically
  sound design has to pass it.
- **Hardening and a bounded launch.** Adversarial testing, a capped initial issuance with a funded
  reserve for chain reorganizations, and only later a migration that removes the launch-time
  controls once the system has run through several full key rotations.

## 10. Where it stands

The architecture is settled and documented, including the places where it has to trust a
quorum rather than pure cryptography. The evidence behind calling it settled is preserved. The next
move is not code, it is eight questions to the Dash Platform team, because their answers decide
whether the strong version of this design is buildable, whether the measured version is what ships, or
whether a base-layer protocol change is on the table. Only after those answers does specification and
building begin, and even then the economics still have to survive a replay of a real DASH crash before
any parameters are fixed.

Put simply, the thinking is done and stress-tested on paper, the trust assumptions are stated rather
than hidden, and the project is at the point where it needs facts from the Dash stack, not more
design.
