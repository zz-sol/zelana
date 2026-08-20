# Protocol Redesign — Goals & Acceptance Criteria

This document distills the institutional requirements into concrete criteria that the redesigned
protocol specification must satisfy. Each requirement is traced back to the
discussion that produced it.

**Status of this document:** normative input to the new spec. Where the
current [high_level_design.md](high_level_design.md) conflicts with a
requirement here, this document wins.

---

## 1. Context

Three candidate models were evaluated in the discussions and rejected:

| Model | Why rejected |
|---|---|
| **Helius-style single shared pool** | One anonymity set commingles all institutions' funds; institutions refuse to have their funds "in the same place" as others, even compliant others. |
| **Single shared tree + per-institution audit keys** | Cryptographically sound, but institutions rejected it outright ("super against it") — resting funds are still commingled in one structure. |
| **Anonymous Zether / PGC (Monero-style rings)** | Small, locally-chosen anonymity sets; decoy selection pushed onto wallets; transaction sizes of ~10–27 KB do not fit Solana transactions even at 4× the current limit. |
| **Zcash ZSA (ZIP 226/227)** | Single shared Orchard pool (same commingling problem); issuance transparent by design (supply/TVL public); no programmability for repos/DvP. Confirmed as insufficient: "unfortunately we definitely need these requirements." |

The agreed direction ("Sounds like the perfect way forward then") is a
**Zcash-style pool–nullifier protocol with one pool per institution**.

---

## 1a. Design Decisions (locked 2026-08-20)

These resolve forks left open by the discussions; the spec must follow them.

- **D1 — Inter-institution peg: bilateral credit + periodic net settlement.**
  A cross-pool transfer creates a bilateral claim between the two
  institutions; claims are periodically net-settled in the public base asset
  (e.g. USDC). Settlement leaks only the *net* flow per period; the
  period/exposure-limit trade-off (longer period = less leakage, more
  counterparty exposure) is a per-institution configuration, not a protocol
  constant. Rejected alternatives: shared reserve (re-introduces
  commingling), per-transfer settlement (defeats TVL hiding).

- **D2 — Pure Zcash-style core on BN254; no Token-2022 CT dependency.**
  The core protocol is a single proving stack (Groth16 / Baby Jubjub /
  Poseidon over BN254). The roles the CT layer would have played are covered
  natively: TVL hiding via **cross-pool shielded transfers** (burn note in
  pool A, mint note in pool B, ZK conservation proof, amount never public);
  dummy traffic via native zero-value notes (indistinguishable by
  construction); auditability via in-circuit auditor ciphertexts
  (exponential ElGamal over Baby Jubjub). Boundary leakage is identical to
  the CT-based variant (the outermost public-asset hop is visible in both).
  Consequence: the curve25519 ↔ BN254 gap disappears from the core; a CT
  compatibility adapter (cross-curve boundary proof) is an **optional
  extension**, not core (downgrades S1/S2 accordingly).

- **D3 — Programmability via Solana atomicity + coordinator program; pool
  core stays transfer-only.** DvP/repo legs live in different pools, each
  proven independently by its own institution (witnesses never shared); one
  Solana transaction bundles both legs, native atomicity makes them
  all-or-nothing. Each leg's proof binds an **intent hash** (public input
  covering both legs' conditions) to prevent a single leg being extracted
  and replayed standalone. No shielded VM, no binding signatures, no
  circuit changes beyond the one public input.

---

## 2. Hard Requirements (MUST)

### R1 — Per-institution isolation of resting funds
Every institution gets its own pool (own commitment tree, own nullifier set,
own wrapped mint). Resting liquidity of one institution is never commingled
with another's in any shared on-chain structure. The institution controls its
own pool ("institutions want their own place where they can do their own
things").

### R2 — Hidden TVL, including for wrapped public assets
The total value resting in an institution's pool must not be derivable from
public data. This applies to natively confidential tokens **and** to wrapped
public assets such as USDC (explicitly confirmed: "yes theyre expecting to be
able to hide tvl of public assets too"). Mechanism (per D1/D2): shield/unshield
against the public base asset is visible at the boundary, but cross-pool
shielded transfers move value between institutions without touching the
public asset; once such transfers occur, per-institution resting TVL becomes
unobservable (bilateral claims settle netted per D1).

### R3 — Confidential amounts and balances everywhere
All balances and transfer amounts inside the system are encrypted
(confidential-transfer style). No plaintext amounts appear in any pool
transaction, deposit, withdrawal, or inter-institution settlement.

### R4 — Unlinkability within a pool (Zcash-style, not ring-style)
Deposits into a pool and withdrawals out of it must be cryptographically
unlinkable, via proof of membership in the pool's full note set
(global-within-pool anonymity set), **not** via sender-chosen decoy rings.
Ring-based designs are explicitly rejected (weaker privacy, wallet-side decoy
complexity, oversized transactions).

### R5 — Anonymity-set inflation via indistinguishable dummy traffic
Because each pool's real user base may be small, the protocol must support
injecting artificial volume (e.g. zero-value in/out transactions) that is
**cryptographically indistinguishable** from real transactions. This is what
makes R1 (isolated pools) compatible with a meaningful anonymity set. R3 is a
prerequisite: only with confidential balances can fake and real transactions
be indistinguishable.

### R6 — Private inter-institution transfers ("need-to-know" privacy)
Institutions must be able to transfer value to each other (repos, settlement)
such that:
- the two counterparties learn what they need to know,
- third parties learn nothing (not the amount, and ideally not the fact or
  frequency of bilateral flows beyond what is unavoidable),
- the transfer does not leak either institution's TVL (see R2 — this is
  precisely why inter-institution transfers are cross-pool shielded
  transfers rather than movements of the public base asset; net settlement
  per D1 bounds the residual leakage).

### R7 — Issuer-scoped auditability
The issuing institution must be able to audit its own asset (view amounts /
recover its regulator-facing trail) wherever that asset moves, without a
trusted third party, and without gaining any visibility into other
institutions' assets. Audit capability is read-only: no spend authority, no
forced forfeiture.

### R8 — Programmability for institutional workflows
The protocol must leave room for additional logic on top of private
transfers — repos and DvP settlement are the named use cases. ZSA's lack of
programmability was a stated dealbreaker. Concretely: the design must either
support conditional/escrowed spends natively or compose cleanly with Solana
programs that orchestrate them.

### R9 — Transactions fit Solana
Every protocol transaction must fit within Solana's transaction limits as
they exist (or are realistically scheduled to exist) at deployment time.
Proof systems and encodings must be chosen accordingly (compact SNARKs over
BN254, verified via native syscalls). The ~10 KB+ transactions of anonymous
PGC are the explicit counterexample.

### R10 — No consensus/validator changes
The protocol deploys as on-chain programs on unmodified Solana. Building a
new validator type or a Canton-style permissioned network is explicitly
rejected as over-engineering.

---

## 3. Strong Preferences (SHOULD)

### S1 — Token-2022 CT compatibility as an optional adapter (revised per D2)
The core protocol does not depend on Token-2022 confidential transfers. A
compatibility adapter (cross-curve boundary proof linking a curve25519 CT
ciphertext to a BN254-native commitment at shield/unshield time) MAY be
specified as an optional extension so CT-account holders can enter a pool
without a public-amount hop.

### S2 — Adoption path for existing confidential mints
Existing Token-2022 mints (with or without confidential transfers enabled)
should be able to use a pool with minimal migration. Without the S1 adapter,
CT balances enter via a decrypt-to-public hop (one boundary amount leaked);
wrapped-mint migration via the token-wrap program is the accepted fallback.

### S3 — Reuse existing tooling
Prefer existing, audited components: token-wrap program for conversions,
existing ZK ElGamal / range-proof infrastructure where applicable, BN254
syscalls for proof verification.

### S4 — Supply integrity remains provable to the issuer
Hiding TVL (R2) removes the public supply invariant. The issuer must retain
the ability to verify no-inflation of its own asset, and optionally to
publish proof-of-reserve / proof-of-supply attestations without revealing the
values themselves.

---

## 4. Non-Goals

- **A single shared anonymity pool across institutions.** This was the
  central goal of the previous design (high_level_design.md §1) and is now
  explicitly reversed: institutions rejected commingling in any form.
- **Hiding the asset/institution identity of a pool.** Pools are
  per-institution and publicly attributable; privacy comes from confidential
  balances + unlinkability + dummy traffic, not from hiding which
  institution a pool belongs to.
- **Forced forfeiture / issuer seizure of user funds.** Audit is read-only.
- **Validator or consensus modifications** (R10).
- **Cross-asset atomic swaps inside a single pool** — cross-asset flows are
  handled at the inter-institution settlement layer, not inside a pool.
  (DvP coordination per R8 remains in scope at the program layer.)

---

## 5. Engineering Constraints

- **Proof verification on-chain:** BN254 pairing syscalls (Groth16 or
  KZG-based PLONK); proof + public inputs must respect R9.
- **Single proving stack (per D2):** Groth16 / Baby Jubjub / Poseidon over
  BN254 throughout the core. The curve25519 ↔ BN254 gap exists only in the
  optional S1 adapter (cross-curve equality proof at the pool boundary),
  never inside pool circuits.
- **Trusted setup:** if Groth16 is used, ceremony design/governance is
  required; a universal-setup or transparent alternative should be costed as
  a comparison point.
- **Dummy-traffic economics (R5):** fee cost and submission cadence of
  artificial volume must be specified (who pays, how rate-limited, how
  indistinguishability is preserved at the RPC/mempool level, e.g. relayers).

---

## 6. Open Questions

Resolved by the decisions in §1a:
- Inter-institution peg & settlement → **D1**.
- Programmability mechanism → **D3** (coordinator + atomicity + intent hash).
- Proof granularity → **D3**: one proof per leg, one prover per leg;
  cross-leg atomicity comes from the Solana transaction, so no binding
  signature / multi-party assembly machinery is needed.
- Audit key architecture → **D2**: in-circuit auditor ciphertexts
  (Baby Jubjub exponential ElGamal). With per-institution pools the previous
  design's registry-membership machinery for *hiding* the audited asset is
  unnecessary and is dropped.

Still open, to resolve in the spec:

1. **Anonymity across pools:** does a cross-pool transfer reveal *which
   two* institutions transacted (pool addresses are public)? Is hiding the
   counterparty pair a requirement or acceptable leakage? Dummy cross-pool
   traffic may be needed, mirroring R5.
2. **Settlement leakage model (D1):** formalize what a periodic net
   settlement reveals over time (sequence of net flows) and whether
   settlement transactions themselves need padding/batching.
3. **Cross-pool transfer authorization:** what the receiving pool checks
   before minting against a bilateral claim (issuer signature? claim-ledger
   state? exposure limit enforcement on-chain vs. off-chain).
4. **Dummy-traffic economics (R5):** fee cost, cadence, who pays, and
   mempool/RPC-level indistinguishability (relayers).

---

## 7. Acceptance Checklist

A candidate spec satisfies this document iff:

- [ ] Each institution's resting funds live in a structure no other
      institution's funds enter (R1)
- [ ] No public observer can compute any pool's TVL, including for wrapped
      USDC-class assets (R2)
- [ ] No plaintext amount appears anywhere in the transaction flow (R3)
- [ ] Deposit→withdrawal unlinkability proven against the full pool note
      set (R4)
- [ ] Dummy transactions are indistinguishable from real ones under a
      stated adversary model (R5)
- [ ] Inter-institution transfer protocol specified end-to-end, including
      peg backing and settlement leakage analysis (R6)
- [ ] Issuer can decrypt/audit exactly its own asset's flows, read-only (R7)
- [ ] At least one concrete repo or DvP flow walked through end-to-end (R8)
- [ ] Worst-case transaction size computed and shown to fit Solana (R9)
- [ ] Zero validator changes required (R10)
