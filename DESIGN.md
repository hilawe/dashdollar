# DigiDollar on Dash Platform, design synthesis (v4, post-review-round-3, architecture LOCKED 2026-07-18)

v4 synthesized 2026-07-18. Lineage: v1 from a clean-room round of five independent designs. v2
folded round 1 (BLOCK, BLOCK, APPROVE-WITH-FIXES). v3 folded round 2 (BLOCK, BLOCK, BLOCK,
APPROVE-WITH-FIXES). Round 3 reviewed v3 with four fresh passes: BLOCK, BLOCK, BLOCK,
APPROVE-WITH-FIXES. No round-3 finding overturned an architecture decision. All were mechanism
corrections inside v3 folds, document-completeness defects, or honesty corrections, and they are
folded here. FORCED marks clean-room consensus. CHOSEN marks deliberate decisions. VERIFY marks
feasibility facts to confirm before build.

## Scope and specification boundary (added in v4)

This document is the ARCHITECTURE RECORD: decisions, mechanisms, trust model, and the properties
the normative specification must deliver. It is not the normative specification. The complete
state machine (all state variables, conservation equations, transition vectors including the
reservation, cancellation, and terminal-classification states introduced since v2, reference and
uniqueness rules, deadlines, integer domains, rounding-remainder ownership, and test vectors)
is the SPEC.md deliverable of the build plan's specification phase, and that spec, not this
document, is the artifact future soundness reviews and the two-independent-implementers test run
against. Until SPEC.md exists, no safety property described here should be treated as delivered.
Properties that depend on unresolved VERIFY items are CONDITIONAL: if Platform can enforce them
they are consensus-backed, otherwise they are actuator policy with auditor detection, and every
such property is labeled below. This boundary statement resolves a recurring review finding that
prose references to equations "standing" elsewhere are not implementable.

Constraints fixed by the owner: DigiDollar economics frozen (over-collateralization 200 to 500
percent by term, terms 30 days to 10 years, no per-position forced liquidation, aggregate dynamic
collateral adjustment near a 120 percent floor, threshold-signed multi-oracle price). AMENDED by the
owner 2026-07-19: the maximum term is capped in the 2-to-3-year range, with the exact cap fixed in
Phase 3 parameter fitting. The trigger was @kxcd's quantum-exposure argument against
decade-long custody on a non-quantum-safe L1, accepted by the owner. This is a requirements
amendment by the requirements owner, not an architecture change, and the v4 lock stands. Boundary
strict: peg-in (DIP-2 asset lock) and peg-out (DIP-7 quorum-signed asset unlock) are the only L1
touchpoints. Platform has data contracts and the native token layer only, no general computation.

## The forced core (unchanged through three review rounds)

1. Native Platform token. Freeze, blacklist, administrative transfer permanently disabled. Mint
   and burn authority only, held by a protocol role.
2. Positions are data-contract documents. Validation enforces structure and ownership, not price
   math.
3. Collateral pooled on L1 behind the bridge, released only by a quorum-signed asset unlock.
4. An off-chain computing, threshold-signing actuator is unavoidable, running deterministic,
   publicly reproducible rules.
5. Mints gate on ChainLock finality of the L1 lock.
6. Oracle: independent operators, several venues each, medians, threshold signature, freshness
   bounds, circuit breakers, attributable signed observations.
7. The honest trust verdict: L1 validates only the quorum signature on an unlock. The signing path
   is a collective custodian. No threshold-proof non-custodial claim is available or made.
8. Riskiest assumptions first: bridge mechanics, redemption atomicity, and (added in round 3) the
   apportionment ledger.

## Who signs what, and the capability matrix (v4, expanded)

The L1 unlock signature is always produced by the Platform validator quorum (DIP-7). The actuator
cannot produce L1 signatures. Its custody power is initiation through escrow-identity keys, plus
every discretionary step the protocol routes through it. These capabilities are INDEPENDENTLY
SUFFICIENT where marked: no second threshold must fail for the harm to occur. Every unlock
initiation carries an actuator-threshold-signed burn receipt (burn transaction identifier plus
state proofs) so fake-burn initiations are attributable. Every quote request and issuance, escrow
entry and return, and admission decision is logged append-only on Platform so selective treatment
is publicly observable in near real time.

| Threshold alone | Can steal or extract | Can freeze or distort | Cannot |
|---|---|---|---|
| Actuator | Initiate unlocks without burns (honest validators sign); mint unbacked supply; trigger terminal unwind prematurely or refuse to trigger it; admit favored claims before a terminal cutoff | Refuse mints, burns, initiations; withhold or delay quotes selectively; manipulate admission ordering within the caps; withhold return of expired escrows (holder funds stranded in escrow identities); delay factor updates | Forge L1 signatures; seize balances held outside escrow (transfers permissionless) |
| Validator quorum | Pending the DIP-7 validation check: possibly sign an unlock without a valid L2 request; fabricate L2 state a lagging auditor accepts | Censor state transitions; refuse to sign unlocks | Initiate withdrawals from escrow identities without actuator keys |
| Oracle committee | Sustained false prices permit undercollateralized issuance or excess release | Staleness halts price-dependent operations; borderline dispersion forces conservative pricing | Move collateral or tokens |
| Membership-rotation root (if used) | Delayed: replace actuator membership, then anything the actuator can do | Delayed freeze via replacement | Act within the public handover delay undetected |
| Requires genuine collusion | Rewriting history against external holders of prior checkpoint attestations | | |

Actions Platform PREVENTS versus violations that are only DETECTABLE is a required distinction in
SPEC.md for every row. All rows share one correlated substrate, the masternode population.

**Future work pointer (non-normative, does not change v4).** The design accepts the
collective-custodian trust model above as its working basis, meaning the Platform validator quorum's
honest majority is trusted to sign only legitimate asset unlocks. A cryptographic trust-minimization
path is desirable later but is NOT small, and the naive version does not work. Gating the L1 Asset
Unlock on a mere Platform burn proof would NOT remove the actuator's "initiate unlocks without burns"
power, because the actuator holds mint authority and could mint unbacked supply, burn it, and present
a genuine burn proof (a mint-then-burn bypass). Removing that independently sufficient predicate would
require Core to verify a non-actuator-forgeable redemption authorization that binds the exact released
amount, which pulls the oracle and collateral math onto L1 and approaches reimplementing the
stablecoin there. This is recorded as deferred future work, scoped in TODO.md "Future work,
deferred", and the trust floor would remain the masternode honest majority regardless. dashpay/dips
PR #187 is precedent for hardcoded Core-enforced spend conditions, not a generic Platform-proof
engine.

## Deliberate choices (v4 state)

**D1. Authorizer.** Unchanged from v3: one dedicated actuator group distinct from the oracle
committee, smaller fixed launch roster, hardware-backed keys, published rotation and handover
protocol, no slashable-bond claims.

**D2. Redemption (v4, rebuilt again after round 3 broke the v3 flow twice).** Round 3 found the v3
quote-first flow granted a free option (a binding price quote before any commitment is a lookback
put the holder exercises only when the market moves in their favor) and left both halves of the
intermediate state unsafe (no cancellation transition for a reserved-but-unburned claim, and
escrow return that silently depended on actuator goodwill). v4 flow:

1. COMMIT FIRST. The holder submits one redemption intent that simultaneously transfers the tokens
   to escrow and enters the on-chain append-only request queue, with a declared slippage
   tolerance. No binding price exists yet, so there is nothing to option.
2. ADMIT DETERMINISTICALLY. Admission out of the request queue follows a deterministic ordering
   key (queue position by anchored arrival, hash-tiebroken) within the per-epoch exposure caps.
   The request queue reserves NO collateral, so the caps genuinely bind. CONDITIONAL: if Platform
   cannot enforce the ordering, admission is actuator policy, and the append-only log plus the
   capability matrix say so.
3. PRICE AT ADMISSION. The admitted intent is priced against the current certificate under the
   holder's slippage tolerance (tolerance exceeded: the intent returns to the holder unburnt).
   Pricing uses the median with a DISCLOSED total haircut cap: the conservative spread plus the
   fixed redemption fee together may never exceed a published bound, resolving the round-3 finding
   that an uncapped high-side spread is a hidden variable exit fee. The oracle unit (USD per DASH)
   and the exact payout equation are SPEC.md deliverables.
4. RESERVE, BURN, CLAIM as one intent-consuming step. The admitted, priced intent is consumed
   exactly once by either (a) the reservation-burn-claim transition, which reserves gross
   collateral including maximum L1 fee, burns the escrowed tokens, activates the claim, and steps
   the apportionment indices, or (b) the cancellation transition, which releases everything back
   to the holder. Burn and cancellation consume the SAME one-time authorization, so no schedule
   produces both. CONDITIONAL on VERIFY 1 and 2: consensus-backed if Platform can enforce the
   combined transition or serialize it by uniqueness index, otherwise policy-backed and labeled.
5. ESCROW LIFECYCLE. Every escrowed balance has a published hard timeout: lacking a consumed
   intent after the timeout, it becomes returnable, every entry and return (success or timeout) is
   logged publicly, and auditors alarm on stuck escrows. Escrow return remains an actuator ACTION
   (Platform cannot self-release without a time-reversion primitive, VERIFY 8), so the stranding
   power appears in the capability matrix rather than being hidden.
6. At-most-once L1 payment remains custodian policy pending the DIP-7 conflict check. Claims are
   nontransferable.

**D3. Redemption rights.** Immediate any-holder pro-rata redemption stands, with the round-2
qualifications (no maturity subordination, disclosed caps, back-pressure) and the round-3 repairs
above (two-queue split, deterministic admission, commit-before-price). "Immediate" means the
protocol never subordinates exit to position maturities, not that throughput is unlimited.

**D4. Binding a mint to its lock.** Unchanged from v3 (single reserve identity, full
domain-separated signed intent consumed exactly once, mint and abort mutually exclusive on the
same intent identifier, authorization expiry). A round-3 reviewer honesty correction applies: the
mutual exclusion is CONDITIONAL on VERIFY 2, and if Platform cannot enforce single-consumption it
is actuator policy with auditor detection, stated as such.

**D5. Solvency and apportionment (v4, ledger corrected).** Round 3 established that one cumulative
factor cannot carry both debt reduction and collateral release (the ratios differ whenever price
moved between events) and that late-entering positions need cohort treatment. v4 adopts a
NORMALIZED-SHARE LEDGER: separate collateral and debt indices, each position storing entry
snapshots of both indices, so only index movement after entry applies to it. Index updates are
batched per epoch (round 3 showed per-redemption singleton-document updates serialize all
redemptions through one causally-dependent write chain), with redemptions recording gross
consumption immediately and the indices settling the epoch. Split-and-merge neutrality, the global
rounding-remainder bucket, and the full equation set (mint, redeem, settle, split, merge, close,
rollover) are SPEC.md deliverables with property tests. POSITION EXPIRY now has defined
transitions, closing a gap latent since v2: at term expiry nothing auto-releases; the owner may
close (acquire and burn the position's effective debt, then withdraw effective collateral, both
factor-settled) or roll (re-underwrite at current price, term curve, and adjustment); an owner who
does nothing leaves the position as pool backing indefinitely, still redeemable-against and never
force-liquidated, with its collateral withdrawal rights simply dormant until closure. The loss
reserve keeps its v3 boundary and gains its terminal-mode placement (below).

## The terminal unwind (v4, barrier hardened)

Round 3 found two ways the v3 barrier moved value to the fast: reserved claims paid at par ahead
of pro-rata holders (a public objective trigger makes that a guaranteed bank run during the notice
window) and unissued depositors ranked behind holders (a premature trigger confiscates collateral
from someone who never received tokens). v4 rules:

- Admissions stop at the OBJECTIVE TRIGGER EVIDENCE itself, not at the end of a notice period.
- At the barrier, classification of in-flight intents is a DETERMINISTIC, publicly recomputable
  function of the barrier checkpoint state, reservation nonces, and intent timestamps, submitted
  as a signed recomputable snapshot others can verify and challenge. No actuator discretion in
  classification.
- Reversible states unwind, they do not join the estate: funded-but-unminted deposits are refunded
  BEFORE estate distribution, escrowed-but-unburnt tokens return to holders, and reservations not
  yet signed and broadcast to L1 are VOIDED back into the pro-rata pool. Only unlocks already
  signed and broadcast retain their claim, because L1 may confirm them regardless.
- The estate then pays one class, all token holders pro rata (voided reservations included), then
  unissued-refund stragglers procedurally, then position-owner residuals. The restricted loss
  reserve joins the estate on the holders' side (its purpose, loss absorption, has arrived).
- Premature triggering, refusal to trigger, and pre-cutoff favoritism are actuator capabilities
  listed in the matrix.

## The core invariant (v4 requirements)

The relational requirements stand from v3 (claims reference consumed burn intents, mints reference
consumed deposit intents, auditors validate transition legality, per-intent aging with absolute
deadlines and cumulative caps, reclassification never resets age). The complete arithmetic model,
including the reservation and terminal states, lives in SPEC.md per the specification boundary
above, and the two-implementers state-root test is its acceptance criterion.

## Oracle and emergency modes

Unchanged from v3 except as amended by the terminal-barrier section and the disclosed total
haircut cap. Volatility scaling remains mint-side only.

## Feasibility items to verify with the Dash Platform team (v4 order)

1. Atomicity of the reservation-burn-claim transition (or uniqueness-index serialization).
   Redemption safety class depends on it.
2. Intent single-consumption across a token action and a document (mint-abort and burn-cancel
   mutual exclusion).
3. DIP-7 unlock mechanics in full, including whether L1 validation proves the L2 request or merely
   accepts the quorum signature.
4. Reserve-identity crediting and claim-key commitment room in the asset-lock payload.
5. Any stablecoin-aware check before peg-out signing without a protocol change.
6. Escrow credits segregated from fee payment.
7. Withdrawal throughput and singleton-document update throughput (grounds the admission caps and
   the epoch-batched index cadence).
8. Whether a native-token transfer to an escrow identity can carry any consensus-enforced
   time-bound reversion (escrow self-release; if not, escrow return stays an actuator action).

## Review record and process disposition

Clean-room: five designs. Round 1: BLOCK, BLOCK, APPROVE-WITH-FIXES, folded to v2 (D3 reversal,
invariant rewrite, custody honesty). Round 2: BLOCK, BLOCK, BLOCK, APPROVE-WITH-FIXES, folded to
v3 (atomic redemption intent, full signed mint intent, relational invariant, apportionment
mechanism, capability matrix). Round 3: BLOCK, BLOCK, BLOCK, APPROVE-WITH-FIXES, folded to this v4
(commit-before-price redemption, two-queue admission, ledger correction to normalized shares with
entry snapshots, terminal barrier hardening, expiry transitions, matrix expansion, specification
boundary). Round 3 overturned no architecture decision. The synthesis-level iteration is therefore
recorded as CONVERGED AT THE ARCHITECTURE LEVEL, and the remaining review findings are
specification obligations carried into SPEC.md, which becomes the artifact under adversarial
review in the specification phase. The owner confirmed this disposition on 2026-07-18 and
declined a fourth full round on the architecture record, on the basis that a fourth pass would
re-review mechanism detail that SPEC.md will formalize and that SPEC.md's own fresh full
multi-model round will examine with more precision. That SPEC.md round is the follow-up full
pass the review discipline requires after a converging round, and it runs once the Phase 0
feasibility answers have settled which conditional mechanisms survive. The architecture record is
LOCKED at v4 as of 2026-07-18. Changes from here are specification work in SPEC.md under
per-change adversarial review, not further synthesis rounds on this document.

## Honest latency and cost expectations

As v3, plus: admission is deterministic within caps, pricing happens at admission (not at quote
request), and epoch-batched index settlement means a redemption's position-level accounting may
settle one epoch behind its burn. Worst case unbounded. Fees: L1 miner fees both legs, Platform
state fees, origination fee (volatility-scaled) funding the loss reserve, and a redemption cost
whose TOTAL (fixed fee plus conservative-pricing spread) never exceeds one published cap.
