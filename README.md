# DashDollar

Design work for a decentralized, non-custodial US-dollar stablecoin native to the Dash network.
The token and all of its logic live on Dash Platform (L2). Dash Core (L1) is touched only twice,
for collateral peg-in through a DIP-2 asset lock and for peg-out through a DIP-7 quorum-signed
asset unlock. The economics carry over DigiByte's DigiDollar template, with over-collateralized
backer positions, no per-position forced liquidation, and a system of incentives that draws the
market price to parity with the dollar rather than any hard peg.

The project is in its design phase. The architecture record is complete and locked (`DESIGN.md`,
version 4). The normative specification is the next deliverable and is gated on a set of platform
feasibility questions listed in `TODO.md` Phase 0. This repository contains design documents, not
running code.

Where to start reading:

- `docs/EXPLAINER.md` is the plain-language account of what the system is, how the parity
  incentives work, the honest trust model, and what happens next depending on the feasibility
  answers.
- `DESIGN.md` is the architecture record, including the capability matrix and the feasibility
  items to verify.
- `TODO.md` is the phased build plan, ordered riskiest assumption first.
- `docs/VM-TRUST-MODEL.md` is a forward-looking note on how the trust model would change if Dash
  Platform ever shipped consensus-side execution.
- `docs/VARIANT-reflexive-economics.md` scopes an alternative economic model under evaluation.

The design was produced by a clean-room exercise in which five independent designs were drafted
from an architecture-free requirements packet and then synthesized, followed by three full
adversarial review rounds by independent frontier models from different families. Findings were
folded as versions two through four, and the third round overturned no architecture decision.
The full review evidence is preserved privately.

DashDollar is a working name. The final token name is a product decision for the Dash community.
