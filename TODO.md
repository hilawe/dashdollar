# DigiDollar on Dash Platform, initial TODO

Seeded 2026-07-18 from the clean-room round's divergences and novel ideas. Ordered by riskiest
assumption first, matching the unanimous verdict of all four designs that the bridge and escrow
mechanics get validated before anything else.

## Phase 0, feasibility facts (blocking, answered with the Dash Platform team or by testnet probe)

Reordered 2026-07-18 after review round 2 (see DESIGN.md v3).

- [ ] Atomicity FIRST: can one Platform transition consume escrowed tokens, create a claim
      document, and reserve collateral (or can uniqueness indices serialize the equivalent)?
      The redemption safety class (consensus-backed vs policy-backed) depends on the answer.
- [ ] Intent single-consumption: canonical signed intent consumed exactly once, referenced by both
      a token action and a document, making mint and abort mutually exclusive by construction.
- [ ] DIP-7 asset-unlock mechanics in full: unique withdrawal index or mandatory-input conflict,
      expiry, re-signing across quorum rotation, fee source and replacement rules, and whether L1
      validation proves the L2 request's existence and content or merely accepts the quorum
      signature. At-most-once payment is custodian policy until this proves otherwise.
- [ ] Confirm an asset lock can credit the dedicated reserve identity and that the lock payload can
      carry the one-time claim-key commitment (D4 v2). Per-position escrow identities only if
      identity cost proves negligible and the key-custody lifecycle can be specified.
- [ ] Confirm group signer mechanics: fixed identity roster with explicit membership updates, or
      any path to a rotating threshold signer.
- [ ] Confirm whether any stablecoin-aware verification can precede peg-out signing without a
      Platform protocol change. If none, the burn gate is actuator policy alone, document it.
- [ ] Confirm whether escrow-held credits can be prevented from paying Platform fees (fee spending
      is otherwise an unmodeled conservation transition).
- [ ] Measure withdrawal limits, asset-unlock expiry, and re-sign behavior across validator quorum
      rotation on testnet.
- [ ] (Round 3) Measure singleton-document update throughput for serial causally-dependent writes
      by one identity (grounds the epoch-batched index cadence and admission caps).
- [ ] (Round 3) Confirm whether a native-token transfer to an escrow identity can carry any
      consensus-enforced time-bound reversion (escrow self-release; if not, escrow return stays an
      actuator action in the capability matrix).

## Phase 1, custody slice (kills the design if it fails)

- [ ] Testnet proof: peg-in credits an escrow identity, the depositor cannot withdraw it, the
      actuator threshold can, and a burn-gated peg-out signs and confirms across a quorum rotation.
- [ ] Idempotent redemption claims: re-sign after failure without double-payment, fee replacement
      constrained to a reserved conflicting input.
- [ ] Property-test harness for the three conservation equations across arbitrary interleavings of
      mint, burn, refund, replacement, reorg, and delayed observation.

## Phase 2, authorizer and oracle

- [ ] Deterministic actuator engine, publicly reproducible, no policy discretion. Publish the
      reference implementation so anyone can recompute every authorization.
- [ ] Oracle committee: per-operator multi-venue aggregation, median of medians, dispersion and
      staleness circuit breakers, signed raw observations on Platform for attributability.
- [ ] Evaluate the masternode-sourced oracle topology proposed by a Dash colleague (2026-07-19): MNs
      read prices from several exchanges, the network combines them, discards outliers,
      threshold-signs, and commits the result on-chain (coinbase on L1 or an L2 equivalent). Compare
      against the independent-operator committee on bootstrap cost, liveness, attributability, and
      the correlated-substrate concentration the capability matrix flags.
- [ ] Separation of actuator and oracle memberships, distinct keys and rotation cadences.
- [ ] Cold-root membership rotation with public handover period.

## Phase 2a additions (after review round 2)

- [ ] Full signed mint intent: canonical domain-separated encoding (all bound fields per DESIGN.md
      D4 v3), published test vectors.
- [ ] Redemption quote-escrow-reserve-burn state machine with the price-lock moment, reservation
      nonce, and nontransferable claims.
- [ ] Claim-level relational invariants and legality-validating auditors (per-intent aging,
      absolute deadlines, cumulative caps, published bounds).
- [ ] Global cumulative factor apportionment with lazy settlement, global rounding bucket, and
      split-and-merge invariance tests (same history, one position vs a thousand splits).
- [ ] Loss reserve boundary: asset, funding, seniority, target, replenishment, three-way public
      reporting (gross, redeemable, restricted).
- [ ] Admission control and back-pressure for redemption (reserve slot and collateral before burn,
      deterministic service order, per-epoch exposure caps, disclosed as a qualification to
      "immediate").
- [ ] Terminal-unwind barrier: snapshot classification of in-flight intents, fixed estate priority,
      objective public trigger evidence with notice and challenge procedure.
- [ ] Signed-action manifest format (input snapshot, oracle certificate, parameter version,
      fixed-point trace, intent id, expected counter-leg, signer epoch) so auditors reproduce every
      authorized action.

## Phase 2b, normative specification (added after review round 1)

- [ ] Complete transition-vector state machine: every balance-changing operation (peg-in accept,
      claim, mint, fees, reserve funding and use, redemption commit, unlock confirm gross of miner
      fee with the bearer named, refunds, mint abort, terminal unwind) as an explicit vector over
      all state variables, with the invariant holding only at paired checkpoints and bounded
      tolerance for in-flight states so auditors alarm on persistent mismatches, not transient ones.
- [ ] Deterministic integer fixed-point arithmetic specification with canonical rounding and
      published test vectors for every formula.
- [ ] Executable reference model with fault injection (crashes, delayed messages, rotations,
      duplicate submissions, fee changes, reordered confirmations), checking conservation at every
      reachable anchored checkpoint.
- [ ] Mint-abort recovery path (timeout refund of funded-but-unissued positions to the original L1
      sender on proof no mint occurred).

## Phase 3, positions, token, and economics

- [ ] Data contract: positions, deposits, requests, system parameters, price certificates.
- [ ] Native token registration with freeze, blacklist, and admin-transfer permanently disabled.
- [ ] Term curve, adjustment multiplier, haircuts, below-floor recapitalization rule, stressed
      redemption payout with pro-rata cap. Replay against historical DASH drawdowns before fixing
      parameters.
- [ ] Fix the exact maximum-term cap inside the owner-amended 2-to-3-year range (decision
      2026-07-19, quantum-exposure argument accepted, DESIGN.md constraints line). The economic
      replay may shorten it further and never lengthens it. Background kept for the record: this
      design's pooled quorum-managed collateral can migrate to new cryptography by protocol upgrade
      mid-term (unlike per-UTXO script timelocks), so the exposure is quorum keys and pool custody
      rather than frozen outputs, which softens but does not remove the argument.
- [ ] Reflexive-economics variant: keep docs/VARIANT-reflexive-economics.md alive as the comparison
      target, and when the replay harness exists, run both economic models through the same
      historical-crash replays on the same machine (frozen-template terms vs no-time-lock reflexive
      collateral), so the colleague conversation gets numbers instead of instincts.
- [ ] Immediate any-holder pro-rata redemption (D3 v2), plus position-owner lifecycle (add
      collateral, extend, transfer, close at maturity). Economic replay must test the peg under
      historical DASH drawdowns WITH endogenous behavior (mint cessation, DASH selling, queue
      dynamics), and if the peg fails even with immediate redemption, shorten the maximum term.
- [ ] Funded loss-absorption design: origination-fee reserve sizing, mint-stop threshold, recovery
      state, terminal haircut rule (the dynamic adjustment is a throttle, not a repair).
- [ ] Epoch snapshot batching and the one-epoch minting delay for new positions, with per-epoch
      issuance caps, and explicit labeling of which orderings are enforced versus actuator policy.
- [ ] Volatility-scaled mint and redemption fees. Conservative-price fallback in borderline epochs.

## Phase 4, hardening and rollout

- [ ] Terminal in-kind unwind only (the short-timeout oracle-free exit was removed in v2 after the
      oracle-withholding attack finding): immutable trigger, pro-rata senior claims, explicit
      abandonment of the USD promise.
- [ ] Actuator-signed public checkpoint attestations for light clients. Transferable authenticated
      redemption-claim receipts.
- [ ] Permissionless auditor tooling and the fail-closed alarm convention.
- [ ] Adversarial testing: unbacked mint attempts, counterfeit position documents, reused burns,
      conflicting replacements, compromised-actuator requests.
- [ ] Bounded launch: issuance ceiling, funded reorganization reserve, raise limits only after full
      cycles across several key rotations.
- [ ] Versioned-immutability migration removing launch controls.

## Process

- [ ] Per-change adversarial reviews (different model family) continue on every non-trivial change,
      per the standing review discipline.
- [ ] Trust documentation states plainly that the peg-out quorum is a collective custodian under
      this architecture, with blast radii per role.

## Future work, deferred (non-blocking, revisit later)

Recorded 2026-07-18. The design proceeds on the collective-custodian trust model, where the Platform
validator quorum's honest majority is trusted to sign only legitimate asset unlocks (see DESIGN.md
forced-core item 7 and the capability matrix). That is accepted as the working basis and is NOT a
blocker. This section holds improvements to look at later, not gating work.

Context (2026-07-18): Dash Core V24 is nearly finished and reportedly already carries covenant
primitives, so a covenant-based trust-minimization could in principle be added before the next
release, IF the exact rule were specified quickly. An independent review by a different model family
pressure-tested that idea. The conclusion is to NOT chase the V24 window with a quick rule, for the
concrete reasons below. This supersedes the more optimistic single-bullet note first written here.

- [ ] DECISION (conditional, not permanent): keep the collective-custodian model for v4 and do not
      pursue a burn-only covenant WHILE the mint math lives in an authorized signer set. The verdict
      is contingent on that condition, and it flips if the mint math ever moves into consensus (see
      the VM contingency below), so external communication should state the dependency, not a flat
      "not worth it". The reason it fails today is a concrete bypass. Because the actuator holds mint authority, a rule
      that gates the unlock on a genuine Platform burn proof does not remove the actuator-alone theft
      path. A compromised actuator can mint unbacked DashDollar to an identity it controls, burn it,
      present a real burn proof, and release real collateral (the mint-then-burn bypass). A burn-only
      covenant catches an actuator BUG that omits a burn, not a malicious actuator, so it buys little
      while implying more.
- [ ] The bar a useful covenant must clear (why it is not small). To actually remove the
      actuator-alone predicate, Core would have to verify a NON-ACTUATOR-FORGEABLE redemption
      authorization, created atomically by Platform consensus from holder-controlled pre-existing
      tokens, binding the exact DASH amount, recipient script, fee, token and contract id, and a
      once-only redemption id, none of which the actuator can forge with its mint, burn, reserve, or
      document keys. Pinning the exact DASH amount pulls the oracle, collateral ledger, rounding, and
      position rules into what Core must check, which approaches reimplementing DashDollar on L1 and
      contradicts the everything-on-L2-except-peg mandate.
- [ ] Correction to the earlier framing. DIP-27 is a single GLOBAL credit pool and Asset Unlock
      transactions have no inputs, so there is no discrete stablecoin collateral output to attach the
      PR #187 pattern to. Pool isolation must be solved first (a tagged DashDollar sub-pool with its
      own accounting, or a rule that every credit-pool unlock proves its Platform source), else a
      malicious signer just uses the legacy Asset Unlock path against the same fungible pool. PR #187
      is precedent for hardcoded Core-enforced spend conditions, NOT evidence that Core already has a
      generic Platform-proof-verification engine.
- [ ] Coupling to mint authority. The same root appears on the mint side. Platform cannot enforce the
      price-dependent mint inequality in consensus (data-contract validation cannot, the authorizer
      set must), so mint is inherently actuator-trusted. Removing actuator-alone theft therefore
      requires constraining mint in consensus too, which Platform's no-on-chain-computation model
      cannot do. The unlock covenant and the mint trust are one problem, not two.
- [ ] The only justified V24-window activity is a time-boxed feasibility DESIGN of the full rule (the
      conditions above), NOT an implementation or a quick merge. Revisit only if that full rule can be
      specified and can pass normal consensus-change review before V24 freezes. The full consultation is preserved in the private review evidence.
- [ ] VM contingency (recorded 2026-07-18). The mint-side trust problem exists because the mint math
      runs in the authorized signer set instead of a VM. If Platform ever ships consensus-side
      execution, the picture changes structurally. The mint inequality would run in consensus, so
      unbacked mints become invalid by construction rather than detectable, the mint-then-burn bypass
      dies, the actuator reduces to a permissionless keeper role, and most CONDITIONAL properties in
      DESIGN.md resolve to consensus-backed. A burn proof then actually means backed tokens were
      destroyed, so the L1 proof-verifying covenant becomes the meaningful complement it currently is
      not. The trusted set narrows from quorum-plus-actuator-plus-oracle to quorum-plus-oracle (the
      oracle leg and the masternode honest-majority floor survive a VM, since a VM cannot know the
      true price). Pinned fact from scoping stands: Platform has NO smart contracts and none on this
      design's horizon, so v4 is correctly designed for the world where the VM never arrives. Soft
      question for the Dash colleagues alongside Phase 0: is consensus-side computation on any real
      roadmap? The answer sets how much of the actuator scaffolding is permanent versus transitional.
      The full analysis of the trust legs that remain under a VM (oracle, L1 unlock path, masternode
      floor, parameter authority) lives in `docs/VM-TRUST-MODEL.md`, with its activation trigger.
