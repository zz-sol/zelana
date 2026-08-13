# Two-Layer Confidential Transfer Protocol — High-Level Design

## 1. Goal

Enable many institutions to issue confidential assets that share **one large
anonymity pool**, while each institution retains the ability to **audit only
its own issued asset**, wherever that asset moves inside the shared pool —
without needing to trust the pool operator, and without being able to see any
other institution's traffic.

Non-goals (v1): cross-asset atomic swaps inside the shielded pool, public
issuance-supply transparency inside the pool (that stays at Layer 1), and
recursive proof composition.

## 2. Architecture Diagram

![Two-Layer Confidential Transfer Architecture](architecture_diagram.svg)

*Blue = Layer 1 (Zether-shaped: accounts, `y = sk·G`, homomorphic ElGamal
balances, an institutional "auditor" ciphertext). Green = Layer 2
(Zcash-shaped: notes, commitments `cm`, nullifiers `nf`, one Merkle tree,
one proof per transaction rather than per action). Purple = the new
component — asset identity is proven via hidden registry membership rather
than shown in the clear, which is what turns the shared tree into one true
anonymity set instead of one per institution. See §4 below for why that
last point is load-bearing.*

## 3. Two-Layer Architecture

- **Layer 1 — Issuance Ledger (Zether-style, per institution).** Each
  institution operates its own confidential-account ledger (adapted Zether):
  accounts hold ElGamal-encrypted balances; transfers are homomorphic
  ciphertext updates with a proof of correctness. This is where an asset is
  actually minted/redeemed, and where the institution's regulator-facing
  audit trail lives natively (the institution runs this ledger).

- **Layer 2 — Shared Shielded Pool (Zcash-style, asset-hidden).** A single,
  shared note-commitment tree and nullifier set, used for all pool-internal
  transfers, regardless of issuing institution. Notes are Pedersen/Poseidon
  commitments to `(asset_id, value, owner, ...)`. **The asset identity is
  never revealed** in a transfer's public transcript — it is proven, in
  zero-knowledge, to be a validly registered asset, without saying which one.

- **The Bridge (Shield / Unshield).** A pair of circuits that move value
  between the two layers: *shield* debits a Layer-1 account and mints a
  Layer-2 note; *unshield* spends a Layer-2 note and credits a Layer-1
  account. Both are proven in the same Groth16/Baby Jubjub/Poseidon toolkit
  as Layer 2, so the whole system uses one proving stack.

## 4. Why Hiding the Asset ID Is the Crux of the Whole Design

A shared tree by itself is not sufficient for a shared anonymity set. If a
transaction publicly states "this note belongs to institution A's asset,"
an observer can trivially filter the tree back down to institution A's own
transfer history — recreating the small-pool problem the shared tree was
supposed to solve. The fix: every transfer proves membership of
`(asset_id, audit_pk, issuance_pk, value_base)` in a public registry Merkle
tree **without revealing which registry entry matched**. The only public
facts about a transfer are: it moved *some* validly registered asset,
conserved value, and correctly encrypted its values under *some* registered
audit key. This is the same mechanism behind Zcash Orchard's multi-asset
(ZSA) design, adapted here for a multi-institution setting.

## 5. Auditability Without Breaking Anonymity for Others

Every output note carries an **auditor ciphertext** — an exponential-ElGamal
encryption of the note's value under the *issuing institution's* registered
audit public key, using that asset's own value-base generator. The circuit
constrains this ciphertext to be correctly formed relative to the same value
committed in the note. Consequences:

- Institution A can decrypt every note tagged (invisibly, to everyone else)
  with A's own asset, anywhere in the shared tree, at any time — because it
  holds `audit_sk_A`.
- Institution A **cannot** decrypt institution B's notes: doing so requires
  `audit_sk_B`, which A does not have.
- No one — including the pool operator — can forge a valid transaction whose
  auditor ciphertext doesn't match reality, because it's a SNARK-enforced
  constraint, not a promise.

This is the "trapdoored Zcash per institution" property: audit visibility is
asset-scoped and cryptographically enforced, not access-controlled by
infrastructure.

## 6. Why No Separate Value-Commitment / Binding-Signature Layer

*(Note: §4.5 of the full protocol has since amended this — the value-balance
argument below still holds, but a separate transaction-binding signature
turned out to be required for an unrelated anti-malleability reason. See
the soundness review, Finding 6.)*

Classic Zcash (Sapling/Orchard) needs Pedersen value commitments and a
binding signature because each spend/output is proven by an independent
per-action circuit, later aggregated into one transaction outside the proof
system. Here, one Groth16 proof covers an entire transaction (all inputs and
outputs at once), so value conservation is simply an arithmetic constraint —
`Σ v_in = Σ v_out + public_delta` — enforced directly by the SNARK. This
removes an entire cryptographic subsystem without weakening soundness, and
noticeably shrinks the constraint count.

## 7. Comparison to Zcash's Own Solutions

Zcash has effectively already explored this design space twice — once for a
single asset, once for multiple assets — and it's worth being explicit
about what each of those designs did, and exactly where and why this
protocol departs from them.

### 7.1 Classic Zcash (Sprout / Sapling / Orchard) — single asset

The core mechanism is the one this protocol borrows for Layer 2: notes
committed into a Merkle tree, nullifiers revealed at spend time without
identifying which leaf they correspond to, and a proof that a valid input
note was consumed to produce valid output notes. Because Sapling and
Orchard prove **each spend/output independently** (an "action"), rather
than one proof per whole transaction, they need an extra mechanism to
enforce that everything nets to zero across a transaction assembled from
independently-produced actions: a homomorphic **Pedersen value commitment**
per action, summed by whoever assembles the transaction, checked against
zero via an aggregate **binding signature**. This also gives Orchard a
genuine capability this protocol does not have: independent, mutually
distrusting parties can each produce their own action proof without
revealing secrets to one another, and an untrusted third party can bundle
those actions into one transaction. Classic Zcash has exactly one asset
(ZEC), so none of this involves an "asset type" dimension at all.

### 7.2 Orchard ZSA (ZIP 226 / ZIP 227) — Zcash's own multi-asset extension

This is the closest existing precedent to what we're building, and the one
worth reading most carefully before treating this design as final. ZSA
extends Orchard to custom assets by giving each asset its own **value-base
generator** (so different assets' value commitments can't be confused or
forged against each other), while making a deliberate choice on the
transparency axis: **issuance and burn events for each asset are public**,
maintained as an on-chain issuance map, even though transfers inside the
pool remain shielded (amounts and counterparties hidden).

The stated rationale for that choice is that it gives *anyone* — not just
the issuer — an independently checkable supply invariant: sum the public
issuance/burn events for an asset, compare against expectations, and a
discrepancy is visible network-wide without needing any special key. This
rationale was borne out in practice: Orchard's own native pool (which
already hides transfer amounts, the same general category of information
this protocol hides more of) suffered a real inflation vulnerability
serious enough to require an emergency network-wide halt, and the very
fact that amounts were hidden made it hard for anyone to fully rule out
prior exploitation even after the fix shipped.

### 7.3 Where and why this protocol differs

| | Classic Zcash | Orchard ZSA | This protocol |
|---|---|---|---|
| Number of assets | One (ZEC) | Many, via asset base | Many, one per institution |
| Asset identity in a transfer | N/A | Public (asset base visible) | **Hidden** (proved via ZK registry membership) |
| Issuance/supply | N/A | **Public** issuance map | Private to the issuing institution (L1 ledger) |
| Externally verifiable supply invariant | N/A | Yes, by anyone | **No** — only the issuer, via its own audit key |
| Proof granularity | Per action, aggregated | Per action, aggregated | **One proof per whole transaction** |
| Cross-party transaction assembly | Yes (untrusted bundling) | Yes (untrusted bundling) | **No** — one prover needs all witnesses in a transaction |
| Curve / proof system | Jubjub / Groth16 (Sapling), Pallas-Vesta / Halo2, no trusted setup (Orchard) | Same as Orchard | Baby Jubjub / BN254, Groth16 (trusted setup required) |
| Issuance/redemption layer | N/A | Transparent issuance protocol, same chain | **Separate Zether-style per-institution ledger (Layer 1)**, external to the shielded pool |

Walking through the substantive rows:

- **We hide more than ZSA does.** ZSA hides amounts and counterparties but
  keeps asset identity public. We additionally hide asset identity itself
  (§4). This is not an incremental difference — it's the specific choice
  that makes the anonymity set the union of *all* institutions' activity
  rather than one pool per institution, which is this project's central
  goal and ZSA's design was never trying to achieve (Zcash's own assets
  don't need cross-issuer pooling).
- **We pay for that with ZSA's safety net.** ZSA's public issuance map is
  precisely what lets the network detect an inflation bug from outside, as
  the real 2026 Orchard incident underscored. Hiding asset identity removes
  the ability to compute a per-asset running total from public data at all
  — the reconciliation capability that remains (§5, and full protocol §9.5)
  is opt-in and issuer-only, not network-wide. This is the single largest
  point of divergence from Zcash's own risk posture, and it is flagged as
  an open question rather than a settled trade-off in `soundness_review.md`
  (Finding 1), not resolved here.
- **We add a layer Zcash doesn't have at all.** Neither classic Zcash nor
  ZSA has anything resembling Layer 1 — Zcash has no concept of a
  per-issuer institutional ledger with its own native audit capability,
  because Zcash's issuance (block-reward emission) isn't something an
  external institution needs to audit the way a bank or stablecoin issuer
  would need to audit its own confidential mint. This is the piece adapted
  from Zether rather than Zcash.
- **We simplified the proof structure, at the cost of a capability we
  judged unnecessary here.** Collapsing spend+output into one
  whole-transaction proof removes the need for Pedersen value commitments
  and a binding signature for balance purposes (§6) — but it also removes
  Orchard's untrusted multi-party bundling capability, since one prover now
  needs every witness in the transaction. We reviewed this trade-off
  explicitly (soundness review, Finding 9) and concluded it isn't a real
  loss for the stated use cases (no cross-institution atomic swaps or joint
  multi-sig spends are in scope), but it would need to be revisited if that
  scope ever changes.
- **We took on a strictly heavier trusted-setup dependency than current
  Zcash.** Sapling used Groth16 (trusted setup) but Orchard deliberately
  moved to Halo2 specifically to eliminate that dependency. We use Groth16
  again, for a different reason (BN254's native acceleration on our target
  chain), which means we are accepting a trust assumption Zcash's own
  current design has already moved past. This is already flagged as an
  operational dependency (§7, full protocol §9.6) but is worth naming
  explicitly as a step backward relative to Orchard specifically, not just
  an abstract cost.

In one sentence: **Zcash/ZSA hides amounts and keeps asset identity and
issuance public; we hide amounts and asset identity both, which is what
buys the cross-institution shared anonymity pool, but it comes at the
direct cost of the externally-verifiable supply invariant that Zcash's own
design — reinforced by a real incident — was built to preserve.** That
trade-off is the central open decision this project is carrying forward.

## 8. Cryptographic Toolkit

- **Embedded curve:** Baby Jubjub (Edwards curve over BN254's scalar field)
  — chosen because BN254 is Solana's natively accelerated pairing-friendly
  curve, so eventual on-chain Groth16 verification is cheap once syscalls
  are wired in (deferred per current scope).
- **Proof system:** Groth16 — small constant-size proof, 3-pairing
  verification, matches BN254 acceleration. Trade-off: requires a per-circuit
  trusted setup ceremony (flagged as an operational dependency, not solved
  here).
- **Hash:** Poseidon — arithmetic-circuit-friendly, used for note
  commitments, nullifiers, and all Merkle trees (pool tree and registry
  tree).
- **Range proofs:** plain bit-decomposition inside the same Groth16 circuit
  — cheaper here than Bulletproofs, since a second proof system would only
  add cost once a trusted setup is already required for everything else.

## 9. Security Properties (summary — see full protocol for reductions)

- **No inflation:** conservation constraints inside a sound SNARK prevent
  value creation at Layer 2; Layer 1 inflation resistance is inherited from
  Zether's existing security proof.
- **Anonymity set = union of all institutions' pool activity**, subject to
  DDH-type hardness on Baby Jubjub and Poseidon's random-oracle-like
  behavior.
- **Per-asset audit is complete and unforgeable**, subject to the audit
  secret key remaining private.
- **CRS integrity is a hard trust dependency** — a compromised Groth16
  trusted setup breaks soundness (inflation), not anonymity. This needs an
  explicit ceremony plan.

## 10. Deferred to Later Work

- `alt_bn128` syscall wiring for on-chain proof verification.
- Groth16 trusted setup ceremony design/governance.
- Registry governance (adding/removing institutions, key rotation, freeze).
- Fee/relayer model for shielded transactions.
- Diversified/stealth addresses for repeat-payment unlinkability.
- Cross-asset swaps inside the shielded pool.