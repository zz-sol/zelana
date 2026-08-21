# Goals for the Protocol Redesign

Criteria the new spec has to satisfy, based on the requirement discussions
with the team. 

## Background

We went through four candidate models before settling on a direction.

- A single shared pool (the Helius-style approach) puts every institution's
funds in one anonymity set. Institutions won't accept this: they don't want
their funds commingled with anyone else's, even other compliant parties.
A variant with one shared tree but per-institution audit keys is
cryptographically fine but was rejected just as firmly, since resting funds
still sit in one shared structure.

- Ring-based designs (anonymous Zether, PGC) fit the token-2022 model nicely
but have small, sender-chosen anonymity sets, push decoy selection onto
wallets, and produce 10–27 KB transactions that don't fit on Solana even at
4x the current limit.

- Zcash ZSA (ZIP 226/227) is the closest existing precedent but fails on
three counts: it's still one shared Orchard pool, issuance is transparent
by design so supply and TVL are public, and there's no programmability for
things like repos or DvP. The institutional team confirmed all three are
dealbreakers.

The agreed direction is a Zcash-style pool/nullifier protocol with one pool
per institution.

## Non-features

The following are not covered by our protocol. Although the protocol may
support it with some extension.

- Inter-institution peg. 

The detailed mechanism can vary. Here we highlight one possible solution:
bilateral credit with periodic net settlement. 

A cross-pool transfer creates a bilateral claim between the
two institutions, and claims are periodically net-settled in the public
base asset. Settlement leaks only the net flow per period; how long the
period is (less leakage vs. more counterparty exposure) is up to each
institution pair, not a protocol constant. We rejected a shared reserve
(re-introduces commingling) and per-transfer settlement (defeats TVL
hiding).

- Programmability.

Programmability is achieved via Solana atomicity plus a coordinator; the pool
core stays transfer-only and does not provide programmability. 

DvP/repo legs live in different pools and each
is proven independently by its own institution, so witnesses are never
shared. One Solana transaction bundles both legs and native atomicity makes
them all-or-nothing. Each leg's proof binds an intent hash (a public input
covering both legs' conditions) so a single leg can't be extracted from the
mempool and replayed on its own. No shielded VM, no binding signatures, no
circuit changes beyond that one public input.

## Features

- Every institution gets its own pool:

own commitment tree, own nullifier set, own wrapped mint. One institution's
resting liquidity never sits in a shared on-chain structure with another's,
and the institution controls its own pool.

— Hidden TVL (optional?). 

The total value
resting in a pool must not be derivable from public data. This was
explicitly confirmed to apply to public assets like USDC, not just natively
confidential tokens. Shield/unshield against the
public base asset is visible at the boundary, but cross-pool transfers move
value between institutions without touching the public asset, so once those
start happening the per-institution resting TVL becomes unobservable.

— Confidential and Unlinkability. Classical Zcash setup.

— Anonymity-set inflation. 

Each pool's real user
base may be small, so the protocol must support injecting artificial volume
(zero-value transactions) that is indistinguishable from real traffic. 

— Private inter-institution transfers. 

Institutions transfer value
to each other (repos, settlement) leaks nothing to third parties. This
includes the amount, and ideally not the frequency of bilateral flows beyond what's
unavoidable. Additionally, in the case the transfer choose to not leak either side's TVL, this can be achieved via cross-pool shielded transfers.

— Auditability.

The issuing institution can audit its
own asset (recover amounts for its regulator-facing trail) wherever the
asset moves, without a trusted third party and without visibility into
other institutions' assets. Audit is read-only: no spend authority, no
forced forfeiture (or do we want forced forfeiture and frozen?).

## Open questions

1. Anonymity across pools. A cross-pool transfer reveals which two
   institutions transacted, since pool addresses are public. Is that
   acceptable leakage, or do we need dummy cross-pool traffic?

2. Settlement leakage over time. Formalize what the sequence of periodic
   net settlements reveals, and whether settlement transactions need
   padding or batching.

3. Cross-pool authorization. What does the receiving pool check before
   minting against a bilateral claim — a signature, claim-ledger state,
   exposure limits on-chain or off?
   
4. Dummy-traffic economics: fees, cadence, who pays, relayer design.
