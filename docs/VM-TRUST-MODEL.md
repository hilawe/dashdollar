# Trust model under a native VM (forward-looking note)

Written 2026-07-18. This note is HYPOTHETICAL and non-normative. It assumes Dash Platform ships
consensus-side execution (a VM), which contradicts the pinned scoping fact that no smart contracts
exist or are on this design's horizon. Its purpose is to record, while the analysis is fresh, which
trust legs a VM would remove and which would survive it, so a future roadmap change can be evaluated
in minutes instead of re-derived. DESIGN.md v4 remains the architecture of record and is designed
for the no-VM world.

## What the VM absorbs

With the mint inequality, redemption pricing, apportionment indices, admission ordering, escrow
timeouts, and terminal classification all executed in consensus, the actuator's discretionary rows in
the capability matrix collapse. Unbacked mints become invalid by construction rather than detectable
after the fact. The mint-then-burn bypass dies with them, because no reachable state contains tokens
the consensus math did not authorize. The signer set that today does the math reduces to a
permissionless keeper that submits transactions anyone could submit, carrying liveness weight but no
safety weight. Every property DESIGN.md labels CONDITIONAL on VERIFY 1 and 2 resolves to
consensus-backed, and most of the auditor machinery shifts from safety-critical detection to
ordinary monitoring.

## The trust legs that remain

Three legs survive a VM, and they become the whole trust story.

**1. The oracle committee.** A VM executes math on the price it is given and cannot know the true
DASH price. Sustained false prices still permit undercollateralized issuance or excess release, at
consensus speed and with consensus confidence. The oracle leg is untouched by a VM and becomes the
largest remaining discretionary surface. The existing mitigations carry over unchanged, namely
independent operators, multi-venue medians, dispersion and staleness breakers, threshold signing,
and attributable signed observations. Anything stronger (validity bounds enforced in consensus, for
example rejecting a certificate that moves more than X percent per epoch) becomes newly possible
with a VM and should be specified if this note ever activates.

**2. The L1 unlock path.** The bridge does not improve by itself. Absent a Core-side change, an
Asset Unlock is still released against a quorum threshold signature, and the signing quorum remains
a collective custodian able to sign an unlock the L2 state never authorized. What the VM changes is
that the covenant complement finally becomes meaningful. With mint math in consensus, a burn proof
actually attests that backed tokens were destroyed, so a proof-verifying unlock rule on L1 would
close the loop that the burn-only covenant cannot close today. The VM and the covenant are
complements, and each without the other leaves a hole. VM without covenant leaves the quorum able to
bypass L2 entirely. Covenant without VM verifies proofs an adversary can manufacture.

**3. The masternode honest-majority floor.** Permanent and unchanged. Platform consensus, the
signing quorums, and Core validation all draw from one correlated masternode population. A VM moves
work from a named signer set into that population's consensus, which is a real reduction in trusted
parties, but it cannot go below the substrate. No configuration of this system reaches
cryptographic trustlessness against a colluding masternode majority.

**4. Parameter and upgrade authority (smaller, but named).** Someone still sets the term curve, the
collateral ratios, the haircut caps, and approves contract or VM-code upgrades. With a VM this
becomes the main remaining human lever, and its governance (timelocks, public notice, immutability
milestones) matters more, not less, once everything else is consensus-executed.

## The resulting trust statement

Under a VM plus a proof-verifying unlock covenant, the trusted set narrows from
quorum-plus-actuator-plus-oracle to quorum-plus-oracle, where the quorum term is honest-majority
Platform consensus rather than a discretionary signer set, and the oracle term is bounded by
whatever validity rules consensus enforces on certificates. That is the best trust model this
architecture can reach on this substrate. It is a strong improvement and still not trustlessness,
and documentation written for that world should say so as plainly as v4 says it for this one.

## Activation trigger

This note activates only if the Dash colleagues indicate consensus-side computation is on a real
roadmap. The standing soft question lives in TODO.md's future-work section. If it activates, the
sequence is to specify oracle validity bounds, revisit the covenant feasibility analysis (its non-forgeable-authorization bar becomes satisfiable), and re-run the capability
matrix with the actuator rows removed and the parameter-authority rows expanded.
