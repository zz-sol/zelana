# Goals for the Protocol Redesign

Criteria the new spec has to satisfy, based on the requirement discussions
with the Solana Foundation institutional team. If this document and
[high_level_design.md](high_level_design.md) ever disagree, this one wins.

## Background

We went through four candidate models before settling on a direction.

A single shared pool (the Helius-style approach) puts every institution's
funds in one anonymity set. Institutions won't accept this: they don't want
their funds commingled with anyone else's, even other compliant parties.
A variant with one shared tree but per-institution audit keys is
cryptographically fine but was rejected just as firmly, since resting funds
still sit in one shared structure.

Ring-based designs (anonymous Zether, PGC) fit the token-2022 model nicely
but have small, sender-chosen anonymity sets, push decoy selection onto
wallets, and produce 10–27 KB transactions that don't fit on Solana even at
4x the current limit.

Zcash ZSA (ZIP 226/227) is the closest existing precedent but fails on
three counts: it's still one shared Orchard pool, issuance is transparent
by design so supply and TVL are public, and there's no programmability for
things like repos or DvP. The institutional team confirmed all three are
dealbreakers.

The agreed direction is a Zcash-style pool/nullifier protocol with one pool
per institution.

## Decisions

Locked 2026-08-20. These resolve forks the discussions left open.

**D1. Inter-institution peg: bilateral credit with periodic net
settlement.** A cross-pool transfer creates a bilateral claim between the
two institutions, and claims are periodically net-settled in the public
base asset. Settlement leaks only the net flow per period; how long the
period is (less leakage vs. more counterparty exposure) is up to each
institution pair, not a protocol constant. We rejected a shared reserve
(re-introduces commingling) and per-transfer settlement (defeats TVL
hiding).

**D2. Pure Zcash-style core on BN254, no token-2022 CT dependency.** The
core is a single proving stack: Groth16, Baby Jubjub, Poseidon. Everything
the CT layer would have provided is covered natively — TVL hiding by
cross-pool shielded transfers (burn a note in pool A, mint in pool B, prove
conservation in ZK, amount never public), dummy traffic by zero-value notes
which are indistinguishable by construction, and auditability by in-circuit
auditor ciphertexts (exponential ElGamal over Baby Jubjub). Boundary
leakage is the same as the CT-based variant either way: the outermost
public-asset hop is visible in both. The payoff is that the curve25519/BN254
gap disappears from the core. A CT compatibility adapter (cross-curve
boundary proof) becomes an optional extension; S1/S2 below are downgraded
accordingly.

**D3. Programmability via Solana atomicity plus a coordinator; the pool
core stays transfer-only.** DvP/repo legs live in different pools and each
is proven independently by its own institution, so witnesses are never
shared. One Solana transaction bundles both legs and native atomicity makes
them all-or-nothing. Each leg's proof binds an intent hash (a public input
covering both legs' conditions) so a single leg can't be extracted from the
mempool and replayed on its own. No shielded VM, no binding signatures, no
circuit changes beyond that one public input.

## Hard requirements

**R1 — Isolation of resting funds.** Every institution gets its own pool:
own commitment tree, own nullifier set, own wrapped mint. One institution's
resting liquidity never sits in a shared on-chain structure with another's,
and the institution controls its own pool.

**R2 — Hidden TVL, including for wrapped public assets.** The total value
resting in a pool must not be derivable from public data. This was
explicitly confirmed to apply to public assets like USDC, not just natively
confidential tokens. Mechanism (per D1/D2): shield/unshield against the
public base asset is visible at the boundary, but cross-pool transfers move
value between institutions without touching the public asset, so once those
start happening the per-institution resting TVL becomes unobservable.
Bilateral claims settle netted per D1.

**R3 — Confidential amounts and balances everywhere.** No plaintext amount
appears in any pool transaction, deposit, withdrawal, or inter-institution
settlement.

**R4 — Unlinkability within a pool, Zcash-style.** Deposits and
withdrawals must be unlinkable via a membership proof over the pool's full
note set. Sender-chosen decoy rings are out: weaker privacy, decoy logic in
the wallet, oversized transactions.

**R5 — Anonymity-set inflation via dummy traffic.** Each pool's real user
base may be small, so the protocol must support injecting artificial volume
(zero-value transactions) that is indistinguishable from real traffic. This
is what makes R1 compatible with a meaningful anonymity set, and R3 is the
prerequisite: only with hidden amounts can fake and real transactions look
the same.

**R6 — Private inter-institution transfers.** Institutions transfer value
to each other (repos, settlement) on a need-to-know basis: the two
counterparties learn what they need, third parties learn nothing — not the
amount, and ideally not the frequency of bilateral flows beyond what's
unavoidable. The transfer must not leak either side's TVL, which is exactly
why these go through cross-pool shielded transfers rather than the public
base asset; net settlement per D1 bounds the residual leakage.

**R7 — Issuer-scoped auditability.** The issuing institution can audit its
own asset (recover amounts for its regulator-facing trail) wherever the
asset moves, without a trusted third party and without visibility into
other institutions' assets. Audit is read-only: no spend authority, no
forced forfeiture.

**R8 — Programmability for institutional workflows.** Repos and DvP
settlement are the named use cases, and the lack of them in ZSA was a
stated dealbreaker. The design must either support conditional spends
natively or compose cleanly with Solana programs that orchestrate them.

**R9 — Transactions fit Solana.** Every protocol transaction fits within
Solana's transaction limits as they exist (or are realistically scheduled
to exist) at deployment time. This drives the choice of compact SNARKs over
BN254 verified via native syscalls. The 10 KB+ transactions of anonymous
PGC are the counterexample.

**R10 — No consensus or validator changes.** The protocol deploys as
programs on unmodified Solana. A new validator type or a Canton-style
permissioned network is over-engineering.

## Preferences

**S1 — Token-2022 CT compatibility as an optional adapter** (revised per
D2). The core does not depend on confidential transfers. An adapter — a
cross-curve proof linking a curve25519 CT ciphertext to a BN254-native
commitment at shield/unshield time — may be specified later so CT-account
holders can enter a pool without a public-amount hop.

**S2 — Adoption path for existing mints.** Existing token-2022 mints
should be able to use a pool with minimal migration. Without the S1
adapter, CT balances enter via a decrypt-to-public hop (one boundary amount
leaked); wrapped-mint migration via the token-wrap program is the accepted
fallback.

**S3 — Reuse existing tooling** where possible: token-wrap for
conversions, existing ZK ElGamal and range-proof infrastructure, BN254
syscalls for verification.

**S4 — Supply integrity stays provable to the issuer.** Hiding TVL removes
the public supply invariant, so the issuer must retain the ability to
verify no-inflation of its own asset, and optionally publish
proof-of-reserve attestations without revealing values.

## Non-goals

- A shared anonymity pool across institutions. This was the central goal of
  the previous design and is now reversed: institutions rejected
  commingling in any form.
- Hiding which institution or asset a pool belongs to. Pools are publicly
  attributable; privacy comes from hidden amounts, unlinkability, and dummy
  traffic.
- Forced forfeiture or issuer seizure of user funds. Audit is read-only.
- Validator or consensus modifications (R10).
- Cross-asset atomic swaps inside a single pool. Cross-asset flows happen
  at the settlement layer; DvP coordination per R8 stays in scope at the
  program layer.

## Engineering constraints

- On-chain verification uses the BN254 pairing syscalls (Groth16, or
  KZG-based PLONK as fallback); proof plus public inputs must respect R9.
- One proving stack throughout the core (D2). The curve25519/BN254 gap
  exists only in the optional S1 adapter, never inside pool circuits.
- Groth16 needs a trusted setup; ceremony design and governance is real
  work, and a universal-setup alternative should be costed as a comparison
  point.
- Dummy-traffic operations (R5) need a specified fee/cadence model: who
  pays, how it's rate-limited, and how indistinguishability holds at the
  RPC/mempool level (relayers).

## Open questions

Resolved by the decisions above: the peg and settlement model (D1),
the programmability mechanism (D3), proof granularity (D3 — one proof per
leg, one prover per leg, atomicity from the Solana transaction, so no
binding-signature or multi-party assembly machinery), and the audit key
architecture (D2 — in-circuit auditor ciphertexts; with per-institution
pools, the old registry-membership machinery for hiding the audited asset
is unnecessary and dropped).

Still open, to resolve in the spec:

1. Anonymity across pools. A cross-pool transfer reveals which two
   institutions transacted, since pool addresses are public. Is that
   acceptable leakage, or do we need dummy cross-pool traffic mirroring R5?
2. Settlement leakage over time. Formalize what the sequence of periodic
   net settlements reveals, and whether settlement transactions need
   padding or batching.
3. Cross-pool authorization. What does the receiving pool check before
   minting against a bilateral claim — a signature, claim-ledger state,
   exposure limits on-chain or off?
4. Dummy-traffic economics: fees, cadence, who pays, relayer design.

## Acceptance checklist

A candidate spec is done when:

- [ ] each institution's resting funds live in a structure no other
      institution's funds enter (R1)
- [ ] no public observer can compute any pool's TVL, including for wrapped
      USDC-class assets (R2)
- [ ] no plaintext amount appears anywhere in the transaction flow (R3)
- [ ] deposit/withdrawal unlinkability is proven against the full pool
      note set (R4)
- [ ] dummy transactions are indistinguishable from real ones under a
      stated adversary model (R5)
- [ ] the inter-institution transfer protocol is specified end to end,
      including peg backing and a settlement leakage analysis (R6)
- [ ] the issuer can decrypt exactly its own asset's flows, read-only (R7)
- [ ] at least one repo or DvP flow is walked through end to end (R8)
- [ ] worst-case transaction size is computed and fits Solana (R9)
- [ ] zero validator changes required (R10)
