# Open Questions for Cat / Institutional Team

Context: we have redesigned the protocol around per-institution shielded
pools ([high_level_design.md](high_level_design.md)), following the
direction agreed in the discussions (Zcash-style pools, one per
institution, hidden TVL, repo/DvP support). Before we commit the detailed
spec, the following points need confirmation from the institutional side.
They are ordered by how much they would change the design.

---

## Q1 — TVL hiding for public assets: confirming what is achievable

For a transparent base asset like USDC, the strongest guarantee any scheme
can offer (ours and the wrapped-mint scheme discussed earlier alike) is:

- the public **vault balance** of each institution's pool is visible at
  all times (USDC itself is transparent — this is unavoidable);
- **hidden** are: every individual transfer, all amounts, all
  counterparties, all gross flows, and the *divergence* between the vault
  balance and the institution's true position (i.e. outstanding bilateral
  claims accrued through inter-institution transfers);
- each periodic net settlement reveals **one net flow per counterparty
  pair per period**. Longer settlement periods leak less but carry more
  bilateral credit exposure; this cadence would be configured per
  institution pair.

For natively confidential assets (institution-issued bonds, deposit
tokens, etc.) TVL and supply are fully hidden from everyone except the
issuer.

**Question:** does this match institutions' expectation of "hiding TVL of
public assets"? Specifically, is periodic-net-flow visibility at
settlement acceptable, and do institutions have a view on settlement
cadence (daily? weekly? threshold-triggered)?

## Q2 — Visibility of counterparty pairs on inter-institution transfers

A cross-pool transfer necessarily touches both institutions' pool
programs on-chain. Third parties therefore see **that** institution A and
institution B transacted (not the amount, not the instrument, not which
clients) and how often.

**Question:** is "A and B interacted at time t" acceptable leakage, or do
institutions require hiding the counterparty pair itself? If the latter,
we would add scheduled zero-value dummy cross-pool transfers and/or a
routing layer — feasible, but it adds cost and operational coordination,
so we only want to build it if it is actually required.

## Q3 — Receiving institution as a required online party

Our design gates every incoming cross-pool transfer on the receiving
institution's signature. This is deliberate: it is the receiving
institution's real-time acceptance of the sender institution's IOU and
its bilateral exposure-limit control (it can see the incoming amount and
refuse past its credit limit with that counterparty) — analogous to
accepting a correspondent-banking credit.

**Question:** is it acceptable operationally that an institution must be
online (running a signing service) to *receive* inter-institution
transfers? And do institutions want exposure limits enforced this way
(per-transfer acceptance) or as pre-authorized limits (sign once per
counterparty per period up to a cap)?

## Q4 — Bilateral credit + periodic net settlement as the peg model

Inter-institution transfers do not move the underlying public asset per
transfer; instead they accrue bilateral claims that are net-settled
periodically in the base asset (this is what makes TVL hiding possible,
see Q1). Between settlements, each institution carries unsecured
bilateral exposure to its counterparties, bounded by the limits in Q3 —
economically similar to traditional bilateral netting arrangements, and
non-performance at settlement is a credit/legal event handled by the
bilateral agreement, not by the protocol.

**Question:** is bilateral netting with off-chain legal backing the right
model, or do institutions expect collateralized/pre-funded settlement
(which would leak more, since collateral movements in a public asset are
visible)?

## Q5 — Repo term enforcement stays off-protocol

In our design a repo's opening and closing legs are each atomic on-chain
DvPs (both legs settle together or not at all — no principal risk within
a leg). However, whether the *closing leg happens at all* at maturity is
not enforced by the protocol: if the cash taker defaults at term, the
cash giver's remedy is retaining the collateral notes, per the bilateral
repo agreement — exactly as in traditional bilateral repo.

**Question:** is contractual enforcement of the term sufficient, or do
institutions expect on-chain enforcement of the closing leg (e.g. escrow
of the collateral for the term)? On-chain term escrow is buildable but
locks the collateral (no rehypothecation during the term) and adds
protocol complexity.

## Q6 — Client custody model within a pool

Within an institution's pool, notes are bearer-style: whoever holds a
note's spending key controls it, and the institution's audit key sees
amounts but **not** client identities, and cannot move client funds
(no forced forfeiture — confirmed earlier as read-only audit).

**Question:** for institutions' own clients, who holds the spending keys —
the client directly (self-custody within the bank's pool), the
institution as custodian, or both (2-of-2)? This does not change the
protocol, but it changes the wallet/key-management deliverables and what
"the institution controls its own pool" means operationally.

## Q7 — Dummy-traffic operations

Anonymity within a small pool is inflated by injecting zero-value
transactions indistinguishable from real ones. Someone must generate and
pay fees for this traffic (likely the institution itself or a relayer it
runs), at a cadence that determines the privacy level.

**Question:** are institutions comfortable operating (and paying for)
their own dummy-traffic generation as part of running a pool, or is this
expected to be a service provided by the protocol operator / a third
party?
