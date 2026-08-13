# Full Protocol Specification: Two-Layer Confidential Multi-Institution Transfer

Status: pre-spec, for cryptographic cross-examination. Not yet a wire
protocol / implementation spec.

---

## 0. Scope and Design Philosophy

This document specifies a system with two layers:

- **Layer 1 ("L1")**: a per-institution confidential-account ledger, adapted
  from Zether (Bünz, Agrawal, Zamani, Boneh, 2020). L1 is where assets are
  actually issued and redeemed. This document adapts Zether's parameters to
  this system's curve/proof choices and its added institutional-audit
  ciphertext, but does not re-derive Zether's core security proof — that is
  cited, not reproduced.

- **Layer 2 ("L2")**: a single shared shielded pool, structurally similar to
  Zcash Orchard, but with the asset type hidden via zero-knowledge registry
  membership rather than revealed in the clear (unlike Orchard's original
  Sapling-style single-asset design, and going further than Orchard ZSA's
  on-chain-visible asset base in some configurations — here the asset is
  fully hidden from the public transcript). This layer is specified in full
  constraint-level detail, since it is the novel component.

- **Bridge**: shield / unshield circuits linking L1 accounts to L2 notes.

All zero-knowledge statements are proven with **Groth16** over **BN254**,
using **Baby Jubjub** as the embedded curve and **Poseidon** as the
arithmetic hash function, for both commitment/nullifier derivation and
Merkle trees.

On-chain proof verification, in production, is intended to use Solana's
native `alt_bn128_pairing` / `alt_bn128_addition` / `alt_bn128_multiplication`
syscalls. **This wiring is explicitly deferred** — the protocol below assumes
a generic "verify Groth16 proof against public inputs" oracle, which may
initially be implemented off-chain or as an unoptimized on-chain routine.

---

## 1. Notation and Parameters

- `F_r`: the scalar field of BN254 (= base field of Baby Jubjub).
- `E`: Baby Jubjub, a twisted Edwards curve over `F_r`, prime-order subgroup
  of order `ℓ`, base point `G`.
- `Poseidon(...)`: a Poseidon hash instance over `F_r`, domain-separated per
  use (commitments, nullifiers, Merkle nodes, key derivation) via fixed
  constant tags, written `Poseidon_TAG(...)`.
- `L`: bit-length bound for note/account values (recommended `L = 40`–`48`;
  see §9.4 for the trade-off this bounds).
- `D_pool`: depth of the Layer-2 note-commitment Merkle tree (e.g. 32).
- `D_reg`: depth of the asset registry Merkle tree (e.g. 16–20; sized to
  number of institutions expected, not number of notes).
- Groth16 CRS: one per circuit (`C_transfer`, `C_shield`, `C_unshield`, and
  L1's transfer circuit `C_L1`), each requiring a trusted setup ceremony
  (see §11.1).

---

## 2. Participants and Keys

### 2.1 Institutions

Each institution `i` registers, at setup time:

- `(audit_sk_i, audit_pk_i = audit_sk_i · G)`: an ElGamal keypair. Holding
  `audit_sk_i` is what lets institution `i` decrypt L2 notes of its own
  asset. This key must be kept private by the institution; its compromise
  breaks confidentiality of that institution's flows (not the pool's
  overall soundness).
- `issuance_pk_i`: authorizes minting/redemption at Layer 1 (mechanism
  inherited from Zether-style issuance; out of scope for the L2-specific
  detail here).
- `asset_id_i = Poseidon_ASSET(issuance_pk_i)`: **derived, not freely
  chosen.** Binding `asset_id_i` deterministically to `issuance_pk_i` closes
  a registration-front-running/squatting attack that exists if `asset_id`
  is an arbitrary institution-chosen field element: without this binding, a
  malicious registrant could observe an institution's intended identifier
  and register it first (with the attacker's own `audit_pk`), so that when
  the real institution shields its issued asset, its notes are silently
  auditable by the attacker instead. Deriving `asset_id_i` from
  `issuance_pk_i` means squatting requires controlling the corresponding
  `issuance_sk_i`, which registration should also require a signature proof
  of (see §2.2).
- `value_base_i = HashToCurveBabyJubJub(asset_id_i)`: a nothing-up-my-sleeve
  curve point used as the "value generator" for this asset's exponential
  ElGamal encryptions (both L1 balances and L2 audit ciphertexts use this).
  **This must be computed by the deterministic `HashToCurveBabyJubJub`
  function, not supplied as an arbitrary registrant-chosen point.** An
  adversarially or carelessly chosen `value_base_i` — in particular the
  identity element, or any point with a known small-order/degenerate
  relationship — silently breaks constraint (i)'s ability to bind `ct_j` to
  the true value `v_j^out` for that asset: e.g. if `value_base_i = O`
  (identity), `ct_j`'s second component no longer depends on `v_j^out` at
  all, so the auditor ciphertext stops encoding the value it is supposed to
  certify. This is a self-inflicted loss of that institution's own
  auditability (§9.5 discusses why that auditability matters more than it
  might first appear), not a cross-institution attack, but is easy to
  prevent at registration time and should not be left as a
  registrant-supplied field.

### 2.2 Asset Registry

A public, append-only Merkle tree `Registry` of depth `D_reg`, whose leaves
are:

```
leaf_i = Poseidon_REG(asset_id_i, audit_pk_i.x, audit_pk_i.y, issuance_pk_i, value_base_i.x, value_base_i.y)
```

`registry_root` is a public input to every L2 circuit. Registry updates
(adding an institution, rotating a key) are an operational/governance
process, deferred to §11.3 for *who* is allowed to admit entries — but
regardless of that governance model, admission of a new leaf must require
a signature under `issuance_sk_i` over `(audit_pk_i, value_base_i)`,
verified before insertion, so that registration itself requires control of
`issuance_sk_i` (consistent with `asset_id_i` being derived from
`issuance_pk_i`, §2.1) and `value_base_i` can be checked against the
canonical `HashToCurveBabyJubJub(asset_id_i)` computation at admission
time rather than trusted as registrant input. Cryptographically, the
protocol only needs "some agreed current `registry_root`" beyond that.

### 2.3 Users

A user holds:

- `ask ∈ F_r`: spending key (uniformly random).
- `nk = Poseidon_NK(ask)`: nullifier key, derived deterministically.
- `pk = ask · G`: owner public key, stored in notes they receive.

(v1 simplification: no diversified addresses / IVK-OVK split. Noted as
future work in §12.)

---

## 3. Layer 2: Note Structure

A note is a tuple:

```
note = (asset_id, v, pk, rho, rcm)
```

- `asset_id ∈ F_r`: which institution's asset this note represents. **Never
  revealed publicly** — only proven, in zero knowledge, to be a value for
  which a registry entry exists.
- `v ∈ [0, 2^L)`: value.
- `pk`: owner's public key (a Baby Jubjub point).
- `rho ∈ F_r`: freshness/uniqueness tag, sampled uniformly at note creation.
- `rcm ∈ F_r`: commitment randomness, sampled uniformly at note creation.

### 3.1 Note Commitment

```
cm = Poseidon_CM(asset_id, v, pk.x, pk.y, rho, rcm)
```

`cm` is inserted as a leaf into the shared Layer-2 note-commitment Merkle
tree of depth `D_pool`. The current tree root is called `anchor`.

### 3.2 Nullifier

At spend time, the owner computes:

```
nf = Poseidon_NF(nk, rho, cm)
```

`nf` is published and checked against a global nullifier set (append-only,
reject if already present — this is the double-spend check).

**Why `cm` is included, not just `rho`:** a simpler `nf = Poseidon_NF(nk,
rho)` is sufficient for unlinkability, but Faerie-Gold-class attacks
historically arose from a note commitment admitting more than one valid
opening — an attacker crafts a single `cm` for which two different `rho`
values (or other fields) both "open" it, letting the same note be spent
twice under two different nullifiers, or letting a recipient be shown a
note that looks funded but is not actually spendable as claimed (the
original Zcash Sprout "Faerie Gold" vulnerability, and the related
"InternalH" second-preimage vulnerability from an under-strength
commitment hash). Binding `cm` directly into `nf`'s derivation removes the
need to separately assume `cm` has no alternate opening under the *specific
hash construction and truncation width used in practice*: even if some
future implementation error weakens `Poseidon_CM`'s effective collision
resistance (e.g. by truncating its output — the exact class of mistake
behind the historical "InternalH" bug), a colliding `cm` cannot be
re-opened to a different `rho` without also changing `nf`, since `nf` is
now a function of `cm` itself. This is a Groth16-cost-free hardening (`cm`
is already computed by constraint (a)/(d); it is only an extra hash input
here) and is deliberately conservative, matching Orchard's own reasoning
for including `cm` in its nullifier derivation. Implementers must still
use the full, untruncated Poseidon output for `cm` — reintroducing a
truncated commitment hash for efficiency would reintroduce exactly the bug
class this is meant to defend against.

Two notes' nullifiers remain unlinkable to an outside observer without
knowledge of `nk`, assuming Poseidon behaves as a random oracle for this
purpose.

### 3.3 Dummy Notes

To support a fixed circuit shape (`N_in` inputs, `N_out` outputs per
transaction — v1 recommends `N_in = N_out = 2`) with fewer real notes,
unused input/output slots are filled with dummy notes: `v = 0`, and dummy
inputs are exempted from the Merkle-membership check (see §4, constraint
(a)). A dummy note's nullifier is still computed and published (to keep
public-input shape uniform) but is harmless since it cannot collide with a
real nullifier except with negligible probability.

Dummy notes still carry the transaction's shared `asset_id` witness like
any real note (see §4.3(g)) — there is no separate "asset exemption" for
dummies. This is simpler than an exemption and equally safe: since `v = 0`
is already forced for dummy notes, they cannot smuggle value regardless of
which `asset_id` they're tagged with, so there is nothing to gain from
letting them diverge, and removing the exemption removes a special case a
future implementer could otherwise get wrong.

### 3.4 Recipient Note Delivery (off-circuit, informational only)

The sender encrypts `(asset_id, v, rho, rcm, memo)` to the recipient via
standard ECDH-derived-key AEAD encryption using an ephemeral Baby Jubjub
keypair and the recipient's `pk`. **This is not a circuit-proven step.** If
a sender constructs this ciphertext incorrectly, the only consequence is
that the recipient fails to discover/spend the note — an availability
problem for the two parties involved, not a soundness or auditability
problem for the system, since the actual value transferred is fixed by the
committed values proven correct in-circuit regardless of what the
delivered plaintext claims.

---

## 4. Layer 2 Circuit: `C_transfer`

One Groth16 proof per Layer-2 transaction. Covers `N_in = 2` input notes and
`N_out = 2` output notes (extendable; see §12).

### 4.1 Public Inputs

1. `anchor` — pool tree root the input notes are claimed to belong to.
2. `registry_root`.
3. `nf_1, nf_2` — nullifiers of the two input notes.
4. `cm_1, cm_2` — commitments of the two output notes.
5. `ct_1, ct_2` — auditor ciphertexts for the two output notes (each a pair
   of Baby Jubjub points; see §4.4).
6. `public_delta ∈ F_r` (signed) — nonzero only for shield/unshield-linked
   transactions; `0` for a pure pool-internal transfer.

### 4.2 Private Witnesses

For each input note `j ∈ {1,2}`:
- `asset_id, v_j^in, pk_j, rho_j^in, rcm_j^in, ask_j`
- Merkle authentication path `path_j` (length `D_pool`) and leaf index
- `dummy_j ∈ {0,1}`

For each output note `j ∈ {1,2}`:
- `v_j^out, pk_j^out, rho_j^out, rcm_j^out`
- `dummy_j' ∈ {0,1}`

Shared across the transaction:
- `asset_id` (single witness value; all real input/output notes must match
  this)
- `audit_pk, issuance_pk, value_base` — the registered tuple for `asset_id`
- `reg_path` — Merkle path of the registry leaf, length `D_reg`
- `r_1, r_2` — ElGamal randomness used for `ct_1, ct_2`

### 4.3 Constraints

**(a) Input note validity.** For each input `j`:
```
cm_j^in = Poseidon_CM(asset_id, v_j^in, pk_j.x, pk_j.y, rho_j^in, rcm_j^in)
MerkleVerify(anchor, cm_j^in, path_j) = 1   OR   dummy_j = 1
```
(Dummy inputs additionally force `v_j^in = 0`.)

**(b) Ownership.**
```
pk_j = ask_j · G
```

**(c) Nullifier correctness.**
```
nk_j = Poseidon_NK(ask_j)
nf_j = Poseidon_NF(nk_j, rho_j^in, cm_j^in)     (must equal public nf_j)
```

**(c') Nullifier distinctness (within this transaction).**
```
nf_1 ≠ nf_2
```
This is not optional. Without it, a prover could present the *same*
underlying note as both input slots (reusing `ask, rho, cm` identically in
both), producing `nf_1 = nf_2`, and have constraint (f) below count that
one note's value twice while the on-chain nullifier-set check only ever
sees one nullifier value inserted — a same-transaction double-count that
inflates the transaction's effective input value using a single real note.
This is the multi-input analogue of the classic "don't let a transaction
list the same UTXO twice" bug, and it must be enforced *both* here, in the
circuit (so a proof cannot even be constructed claiming to spend the same
note twice within one transaction), *and* by the on-chain verifier, which
must reject a transaction whose declared nullifier list contains a
duplicate before consulting the historical nullifier set (the historical
set alone cannot catch this, since neither copy exists in it yet at the
start of the transaction).

**(d) Output commitment correctness.** For each output `j`:
```
cm_j^out = Poseidon_CM(asset_id, v_j^out, pk_j^out.x, pk_j^out.y, rho_j^out, rcm_j^out)
```
(must equal public `cm_j`)

**(e) Range checks.** For each real `v` (input and output), bit-decompose
into `L` boolean-constrained bits and check the sum matches `v`:
```
v = Σ_{k=0}^{L-1} b_k · 2^k,   b_k ∈ {0,1}
```

**(f) Value conservation.** `public_delta` is not used directly as a signed
field element; it is decomposed via two *witnessed and range-checked*
non-negative parts, `delta_pos` and `delta_neg`, with:
```
public_delta = delta_pos - delta_neg        (as a field equation)
delta_pos · delta_neg = 0                    (at most one is nonzero)
0 ≤ delta_pos < 2^L,   0 ≤ delta_neg < 2^L   (same bit-decomposition range check as (e))
v_1^in + v_2^in + delta_pos = v_1^out + v_2^out + delta_neg
```
**This range check on `delta_pos`/`delta_neg` is required, not cosmetic.**
`public_delta` is a *public input*, supplied by whoever submits the
transaction, and F_r arithmetic is modular. If `public_delta` (or its
sign-decomposed parts) were left unconstrained in bit-length, a prover
could choose a `public_delta` that is enormous — large enough to wrap
around the field modulus — making the field equation in (f) hold for
values of `v_j^in`/`v_j^out` that satisfy their own small range checks
individually, while the *effective* real-world delta implied by wraparound
is something else entirely. This is a standard instance of the
"under-constrained circuit via missing range check" bug class documented
across production ZK systems (Circom `LessThan`-without-bit-length-pinning
incidents, missing leaf range checks in Merkle-tree implementations, and
others catalogued in community ZK bug trackers); the general lesson is
that *every* field element that participates in an arithmetic equality
meant to represent bounded real-world quantities needs its own explicit
range constraint, public input or not — it is not sufficient to range-check
only the witnessed values and assume public inputs are "already fine"
because the verifier can inspect them; the verifier only checks the field
equation, not the real-world magnitude the prover intended.

**(g) Asset consistency.** All input and output notes (real and dummy —
see §3.3) share the same `asset_id` witness.

**(h) Registry membership.**
```
leaf = Poseidon_REG(asset_id, audit_pk.x, audit_pk.y, issuance_pk, value_base.x, value_base.y)
MerkleVerify(registry_root, leaf, reg_path) = 1
```
(`value_base` is taken directly from this registry leaf — it is *not*
independently recomputed via an in-circuit hash-to-curve, which would add
cost for no additional guarantee, since the registry entry is already the
canonical source of truth for `value_base`.)

**(i) Auditor ciphertext correctness.** For each output `j`:
```
ct_j = ( r_j · G ,  v_j^out · value_base + r_j · audit_pk )
```
(must equal the public `ct_j`)

### 4.4 Notes on the Auditor Ciphertext

`ct_j` is an **exponential ElGamal** ciphertext of `v_j^out` under
`audit_pk` (the audit key of whichever institution's `asset_id` this
transaction uses — itself hidden by (h)). The institution decrypts with
`audit_sk`:

```
ct_j.2 - audit_sk · ct_j.1 = v_j^out · value_base
```

then recovers `v_j^out` by discrete-log search bounded to `[0, 2^L)`
(baby-step-giant-step, or an institution-maintained incremental lookup
table — practical since the institution observes its own decryptable
stream continuously and does not need to search cold each time). This
bound is why `L` is capped rather than a full field element (§9.4).

Only the institution whose `asset_id` was actually used can perform this
decryption, since it requires `audit_sk` for the *matching* institution.
Constraint (h) ensures the `(asset_id, audit_pk)` pair used is a real
registered pair — a malicious sender cannot substitute an arbitrary
`audit_pk` they control, because that pair would not be present in
`Registry` and (h) would fail.

### 4.5 Transaction Binding (correction to the earlier "no binding
machinery needed" simplification)

The high-level design document argued that folding value conservation
into one whole-transaction Groth16 proof removes the need for Sapling/
Orchard-style Pedersen value commitments and an aggregate binding
signature, since (f) is already enforced by the SNARK. That argument holds
for *value-balance* purposes. It does **not** cover a separate malleability
class Zcash's own protocol had to defend against independently: a
malleator lifting a valid shielded description (proof + its public inputs)
out of the transaction context it was created for and relocating it into a
*different* transaction — one with a different fee, different transparent
components, or an otherwise different overall transaction hash — while
leaving the shielded statement itself (and its Groth16 proof) intact and
still verifying. Zcash's documented fix for exactly this problem is a
digital signature, generated with an ephemeral or dedicated key at proof-
construction time, that binds the shielded description to the specific
transaction it was built for.

This system needs the equivalent. Concretely: each `C_transfer`/`C_shield`/
`C_unshield` proof must be accompanied by a signature — under a key the
prover controls at proving time (e.g. re-using `ask` for a transfer, or a
transaction-specific ephemeral key otherwise) — over a hash of the *entire*
transaction as it will be broadcast (all public inputs, any transparent fee
fields, any other bundled components), verified by the on-chain program
alongside the Groth16 proof itself. Without it, the public-input immutability
of a single Groth16 statement is not sufficient protection: Groth16 proofs
are known to be malleable (re-randomizable to a different but equally valid
proof of the *same* public inputs), and more importantly, nothing in the
circuit as specified ties the proof to one specific *transaction envelope*
rather than any envelope carrying the same public inputs — an omission this
document's earlier draft did not address. This is now a required primitive,
not an optional hardening measure, and should be specified in full at the
wire-format stage (deferred alongside §11.4's fee model, since fee fields
are exactly the kind of transaction-envelope data that needs to be inside
the bound hash).

---

## 5. Layer 1: Adapted Zether Accounts

Layer 1 is specified at the level of "what is adapted from Zether," not
re-derived from first principles; see Bünz et al. (2020) for the base
protocol and its security proof.

### 5.1 Account State

Each L1 account `u` (institution-scoped) has:
- `y_u = sk_u · G`: long-term public key.
- `CT_u = (r_u · G, \; b_u · G + r_u · y_u)`: exponential ElGamal encryption
  of balance `b_u` under `y_u`, using base point `G` as the value
  generator (institution-internal; does not need to match any L2
  `value_base`, since L1 and L2 have separate encryption contexts).
- `CT_u^{aud} = (r_u' · G, \; b_u · value_base_i + r_u' · audit_pk_i)`: a
  second ciphertext of the same balance under the institution's own
  `audit_pk_i`, giving the institution native visibility into every
  account on its own ledger (this is the direct L1 analogue of Zether's
  published "auditor extension").

### 5.2 Transfer / Debit-Credit

Adapted from Zether's confidential transfer: prover (account owner) knows
`sk_u`, the plaintext balance `b_u` (tracked client-side, standard Zether
assumption — not re-derived from the ciphertext on each use), and an amount
`v`, and proves:
- `0 ≤ v ≤ b_u`, `0 ≤ b_u - v` (range proofs, same bit-decomposition
  technique as L2, folded into the same Groth16/Baby Jubjub/Poseidon
  toolkit rather than Zether's original Sigma+Bulletproof combination —
  this is the one substantive adaptation from the original paper, made for
  proving-stack uniformity, not because the original scheme was unsound).
- The new ciphertext `CT_u'` (and `CT_u'^{aud}`) are correctly re-randomized
  updates reflecting `b_u - v`, using the additive homomorphism of
  exponential ElGamal (publicly checkable arithmetic on ciphertexts; only
  the *knowledge and range* of the update needs proving, not the
  homomorphic arithmetic itself).
- The receiving account's ciphertext is correspondingly credited by `v`.

### 5.3 Concurrency: Why L1 Accounts Need Epochs or an Equivalent

An account-ciphertext model like §5.1–5.2 has an inherent liveness problem,
well documented in Zether's own literature (not a bug specific to this
adaptation, but one this system inherits and must not ignore): a transfer
proof is constructed against a *specific* observed `CT_old`. If any other
transaction touching the same account is confirmed first — even an
unrelated, honest one, such as an incoming credit from a different
counterparty — `CT_old` changes on-chain, and the pending proof, built
against the now-stale ciphertext, becomes invalid and must be discarded and
rebuilt. An adversary who can reliably front-run a target account's pending
transaction (e.g. a relayer with mempool visibility) can use this to
repeatedly grief a specific account's transactions without needing to
break any cryptographic assumption — this is a liveness/availability
attack, not a soundness break, but it is real and specifically documented
in the original Zether paper and its follow-ups as a problem requiring an
explicit mitigation (Zether's own fix batches transactions into fixed
**epochs**, with all transactions in an epoch proven against the same
epoch-opening balance and applied atomically at epoch boundaries, rather
than each transaction racing against the latest live ciphertext).

This system's Layer 1 must adopt an equivalent mechanism — either Zether's
epoch batching directly, or a per-account sequence-number/lock scheme
(analogous to an Ethereum-style account nonce) that at minimum ensures a
proof's validity window is well-defined and re-submission after a
legitimate concurrent update is a normal, expected, and cheap client-side
retry rather than an exploitable race. This is deferred to the
implementation-level L1 specification (this document only adapts Zether's
cryptographic core, per §0/§5's scoping) but is flagged here explicitly so
it is not silently dropped in translation — nothing about moving to Baby
Jubjub/Groth16 changes this concurrency property, since it is a property of
the *account model*, not the proof system underneath it.

### 5.4 Issuance / Redemption

Institution mints by directly setting an account's ciphertext to encrypt an
issued amount, authorized by `issuance_sk_i`, logged for the institution's
own regulator-facing record. Redemption is the reverse. This is
intentionally not shielded/private at L1 — L1 is the transparent-to-its-
operator ledger of record; privacy is L2's job.

---

## 6. Bridge: Shield and Unshield

### 6.1 `C_shield`

Moves value from an L1 account into a fresh L2 note.

**Public inputs:** `CT_old, CT_new` (L1 ciphertext before/after, both
readable on-chain), `registry_root`, output commitment(s) `cm_j`, auditor
ciphertext(s) `ct_j`.

Note: the shielded **amount is not a public input** — only the before/after
L1 ciphertexts and the resulting L2 commitment/auditor-ciphertext pair are
public. The amount itself stays hidden inside the proof, which is a strictly
stronger privacy property than revealing a shield amount in the clear.

**Private witnesses:** `sk_u`, `b_old` (locally tracked L1 balance opening
`CT_old`), `v` (amount shielded), L1 re-randomization values for `CT_new`,
and everything `C_transfer` needs to construct and audit-encrypt the new L2
note(s) (§4.2, restricted to the output side only — there is no L2 input
note to spend here).

**Constraints:** union of the relevant L1 debit constraints from §5.2
(knowledge of `sk_u`, `b_old`, range proof on `v` and `b_old - v`, correct
`CT_new`/`CT_new^{aud}` update) and the output-note constraints (d), (e),
(g), (h), (i) from §4.3, with `v` playing the role of `v_j^out`.

### 6.2 `C_unshield`

The mirror image: spends an L2 note (input-note constraints (a)–(c), (e),
(g), (h) from §4.3) and credits an L1 account with the corresponding
homomorphic update to `CT_new` (analogous to §5.2's credit side). Public
inputs are `anchor`, `registry_root`, `nf` (of the spent L2 note),
`CT_old, CT_new` (L1). As with shielding, the unshielded amount is not a
public input.

---

## 7. On-Chain State and Verification Flow (protocol-level, syscall wiring deferred)

Global state:
- `anchor` history (recent Merkle roots of the L2 pool tree, to allow proofs
  built against a slightly-stale root),
- L2 nullifier set,
- `registry_root` (current),
- per-account L1 ciphertexts.

Per transaction, the verifying program:
1. Checks the claimed `anchor` is a recognized recent root.
2. Checks the claimed `registry_root` matches the current registry root.
3. Checks none of the transaction's nullifiers are already present in the
   nullifier set; if so, rejects (double-spend).
4. Verifies the Groth16 proof against the declared public inputs, using the
   circuit-appropriate CRS (`C_transfer`, `C_shield`, or `C_unshield`).
5. On success: inserts new nullifiers into the nullifier set, inserts new
   commitments into the pool tree (updating `anchor`), and updates any L1
   ciphertexts touched.

Step 4 is, for now, a generic "verify Groth16 proof" call; wiring this to
`alt_bn128_pairing` and friends is explicitly out of scope for this document
and tracked as a deferred integration item (§12).

---

## 8. Adversary Model

The adversary:
- Observes all on-chain data: full history of `anchor` roots, all
  commitments, all nullifiers, all ciphertexts, all proofs, the registry
  and its history.
- Does not hold any user's `ask`, any institution's `audit_sk_i`, or the
  Groth16 CRS toxic waste (assumed destroyed post-ceremony; see §11.1).
- May submit arbitrary transactions (adaptive, active adversary), subject to
  proof verification.

Anonymity is analyzed as indistinguishability of two transaction sequences
differing only in which registered asset(s) and which honest users are
involved, given everything on-chain.

---

## 9. Security Analysis

### 9.1 No Inflation (L2)

Given Groth16 soundness (knowledge-soundness under the CRS/toxic-waste-
destroyed assumption, standard bilinear-group assumptions for BN254) and
constraint (f), any accepted `C_transfer` proof implies
`Σ v_in = Σ v_out + public_delta` over the true committed values. Combined
with nullifier-set uniqueness (no note spent twice) and registry membership
(g)/(h) preventing cross-asset value smuggling, total committed value per
asset inside the L2 pool cannot increase except via a valid `C_shield`
proof, which itself is tied to a verified L1 debit. Global no-inflation thus
reduces to: Groth16 soundness (L2 + bridge) ∧ Zether's own no-inflation
proof (L1, cited, not re-derived here).

### 9.2 Anonymity

Two transactions using different registered assets and different users are
computationally indistinguishable to an observer, assuming:
- **DDH-type hardness on Baby Jubjub** (semantic security of the ElGamal-
  style auditor ciphertexts; owner-key unlinkability),
- **Poseidon behaves as a random oracle** for this purpose (nullifier and
  commitment unlinkability without the relevant secret keys),
- **Groth16 zero-knowledge** (the proof itself leaks nothing beyond the
  declared public inputs — critically, `asset_id` is *not* a public input
  anywhere in `C_transfer`, `C_shield`, or `C_unshield`).

The resulting anonymity set for any given transfer is the full set of
Layer-2 activity across *all* registered institutions during the relevant
window — this is the mechanism that delivers the "large shared pool" goal,
and it depends on asset-hiding (§4.3(h)) and the single-tree structure
(§3.1) jointly; either one alone is insufficient (see high-level design,
§3/§6).

### 9.3 Auditability

For any accepted transaction, constraint (i) guarantees `ct_j` is a correct
exponential-ElGamal encryption of the true `v_j^out` under the `audit_pk`
tied to the true `asset_id` used (per (h)). Therefore the institution
holding the matching `audit_sk` can always recover `v_j^out` (subject to the
range bound, §9.4) for every note of its own asset, anywhere in the shared
pool, without needing any other party's cooperation. Conversely, an
institution without the matching `audit_sk` learns nothing from `ct_j`
beyond a semantically-secure ciphertext, per §9.2's DDH assumption. This is
an unforgeable property (soundness-backed), not a cooperative/trust-based
one.

### 9.4 Discrete-Log-Bounded Decryption

Exponential ElGamal requires solving a discrete log in `[0, 2^L)` to recover
a plaintext value. This bounds `L` to a practically invertible range
(recommend `L ≈ 40`–`48` bits depending on the asset's required precision
and expected transaction volume — larger `L` trades off audit decryption
cost against representable value range). Institutions performing continuous
auditing can maintain incremental baby-step-giant-step tables or simply
track running balances as new notes arrive, avoiding a cold search per
note.

### 9.5 Externally Verifiable Supply — the Most Important Open Question

This is the most significant finding of this review, and it is a genuine
tension in the design, not a bug with a clean fix.

Zcash's own multi-asset shielded design (Orchard ZSA, ZIP 226/227)
deliberately keeps each asset's **issuance and burn events public**, even
though transfers of that asset inside the pool are shielded, specifically
so that *anyone* — not just the issuer — can independently sum public
issuance/burn events and cross-check them against expectations, giving the
network an externally verifiable supply invariant per asset. This choice
was made with the explicit rationale that hiding everything removes the
network's ability to detect an inflation bug from outside.

That rationale was validated the hard way: a real inflation vulnerability
was found in deployed Orchard in 2026, serious enough to require an
emergency network-wide halt of shielded transactions. Because Orchard hides
transfer amounts, outside observers could not fully rule out that the bug
had already been exploited before the fix — the same property that gives
Orchard its privacy also made the *extent* of a soundness failure
unusually hard to bound from outside, purely because amounts (not asset
identity, in Orchard's case) are hidden.

This system hides *more* than Orchard's shielded transfers do — it hides
**asset identity itself**, not just amounts, inside `C_transfer`. That is
precisely the property that delivers the "large shared anonymity pool"
goal (§9.2), but it also means: **no one other than the issuing
institution itself — not the network, not other institutions, not an
independent auditor without that institution's cooperation — can compute
"how much of asset X exists inside the shielded pool right now" and check
it against Layer 1's issuance ledger.** A systemic bug in `C_transfer` that
broke value conservation (constraint (f)) for *all* assets at once — the
same general failure mode as the real Orchard bug — would not be
independently detectable by the network the way ZSA's public issuance map
is designed to make it detectable. Each institution retains the ability to
reconcile *its own* asset via `audit_sk_i` (§9.3), but this is opt-in,
requires the institution to actually run the reconciliation on an ongoing
basis, and — in a coordinated exploit that broke both (f) and (i)
simultaneously — could in principle be fooled at the same time as the
value-conservation check it is meant to backstop.

**This is presented as an open question for cross-examination, not a
settled recommendation**, because the two options genuinely trade off
against each other and against the stated goals:

1. **Keep full asset-hiding (as specified)** and treat per-institution
   audit-key reconciliation as a *mandatory, ongoing, first-class
   operational requirement* for every institution — not merely an optional
   compliance feature — since it is now the *only* inflation-detection
   mechanism available for that asset. This preserves the "large shared
   pool" goal in full but weakens the safety net relative to ZSA's
   design.
2. **Partially reveal asset identity** (e.g., an ZSA-style public per-note
   asset base, or a public per-asset running total maintained via a
   homomorphic accumulator anyone can check against the L1 issuance
   ledger) to restore network-wide, no-special-key-required inflation
   detection, at the cost of exactly the anonymity-set fragmentation
   analyzed in the high-level design (large pool becomes many per-asset
   pools again).
3. **A middle path this review proposes as a candidate improvement:**
   require each institution to periodically publish a small, separate
   zero-knowledge **"solvency proof"** — a proof that the sum of values the
   institution can decrypt via `audit_sk_i` across its own asset's notes to
   date is consistent with its Layer-1 net issuance figure — without
   revealing which specific notes are its own or any individual value.
   This restores something functionally similar to ZSA's externally
   verifiable supply invariant (anyone can check the institution's
   published proof, not just trust its say-so) on a per-asset,
   privacy-preserving basis, without reversing the asset-hiding decision
   that the shared-pool goal depends on. It adds a new circuit and a
   recurring off-chain publication process, both currently unspecified —
   this is flagged as a concrete direction for the next design iteration,
   not yet a specified mechanism.

Whichever direction is chosen, it should be chosen deliberately and
recorded here, not left as an implicit consequence of the asset-hiding
decision.

### 9.6 Trusted Setup Dependency

Groth16's soundness guarantee depends on the corresponding circuit's
structured reference string having had its toxic waste destroyed. A
compromised CRS for any of `C_transfer`, `C_shield`, `C_unshield`, or `C_L1`
allows forged proofs for *that* circuit — i.e., an inflation attack on the
associated layer, not an anonymity break. This is a hard operational
dependency requiring a multi-party ceremony; not addressed further in this
document beyond flagging it (§11.1).

---

## 10. Efficiency Analysis

Dominant per-transaction costs in `C_transfer` (order-of-magnitude,
`N_in = N_out = 2`, `D_pool = 32`, `D_reg = 20`, `L = 40`):

- 2 Merkle-path membership checks over the pool tree: `2 × D_pool = 64`
  Poseidon hash invocations.
- 1 registry membership check: `D_reg = 20` Poseidon hash invocations.
- Commitment/nullifier/key-derivation hashes: on the order of 6–8 Poseidon
  invocations.
- 2 owner-key derivations (`pk = ask · G`): 2 fixed-base scalar
  multiplications.
- 2 auditor-ciphertext constructions: 2 variable-base scalar multiplications
  (by `v`, ~`L` doublings each) + 2 fixed/variable-base multiplications (by
  randomness).
- Range checks: `4 × L = 160` boolean constraints (2 inputs + 2 outputs).

This is comparable in shape and order of magnitude to Zcash Orchard's
per-action circuit (dominated similarly by Merkle-path hashing plus a small
number of curve operations), which is known to be practically provable
(sub-second to low-single-digit-second proving time on commodity hardware,
depending on prover implementation) and cheaply verifiable (Groth16: 3
pairings, independent of circuit size). This supports the "efficient"
requirement on paper; concrete benchmarking is deferred to implementation.

`C_shield` / `C_unshield` are smaller (one-sided: only an output-note or
only an input-note set, plus the L1 side), so cheaper than `C_transfer`.

---

## 11. Explicitly Deferred Items

### 11.1 Trusted Setup Ceremony
Design and execution of the MPC ceremony(-ies) producing the CRS for each
circuit. Not specified here.

### 11.2 On-Chain Syscall Integration
Wiring Groth16 verification to Solana's `alt_bn128_*` syscalls. The
protocol above assumes a generic verification oracle.

### 11.3 Registry Governance
Who can add/remove institutions, rotate `audit_pk_i`/`issuance_pk_i`, and
how `registry_root` updates propagate and are agreed upon on-chain. Not
specified here — this document only requires that *some* `registry_root` is
publicly agreed at any given time.

### 11.4 Fee / Relayer Model
How gas/fees for shielded transactions are paid without deanonymizing the
sender (standard "relayer + fee note" pattern is a natural fit, not
specified in detail here).

### 11.5 Variable Input/Output Counts
v1 fixes `N_in = N_out = 2` per transaction with dummy-note padding.
Supporting larger transactions efficiently (e.g., via batching or
recursion) is future work.

### 11.6 Diversified / Stealth Addresses
v1 uses a single static `pk` per user. Repeated payments to the same user
are linkable via repeated `pk` values inside ciphertexts an observer cannot
decrypt, but this is worth strengthening with diversified addresses
(Sapling/Orchard-style) in a later revision.

### 11.7 Cross-Asset Operations
No swap/exchange functionality inside the shielded pool; all notes in a
single transaction must share one `asset_id` (§4.3(g)). Any cross-asset
functionality is out of scope for v1.

### 11.8 Key Compromise Response
Incident response for a compromised `audit_sk_i` (which deanonymizes that
institution's own historical flows to whoever obtained the key, but not
other institutions') — rotation procedure and its interaction with already-
issued ciphertexts is not specified here.

---

## 12. Summary of What Is and Isn't Novel Here

- **Novel / fully specified in this document:** the asset-hiding shared
  Layer-2 pool (§3–§4), the per-asset unforgeable audit-ciphertext mechanism
  (§4.4, §9.3), and the shield/unshield bridge tying a per-institution L1
  ledger to that shared pool (§6).
- **Adapted, not re-derived:** the Layer-1 confidential-account mechanics
  (§5), which are Zether with a different curve/proof backend and an
  explicit institutional audit ciphertext; correctness relies on citing
  Zether's own published security analysis for the parts unchanged in
  substance.
- **Deferred:** everything in §11.

### 12.1 Changelog: Soundness Review Pass

This revision incorporates findings from a dedicated soundness/efficiency
review conducted against the initial draft. Material changes:

1. Bound `asset_id_i` to `issuance_pk_i` and required registration to prove
   control of `issuance_sk_i` (§2.1, §2.2) — closes an asset-identifier
   squatting/front-running gap.
2. Required `value_base_i` to be computed canonically rather than
   registrant-supplied, and flagged the degenerate-generator failure mode
   it otherwise opens (§2.1, §4.3(h)/(i) note).
3. Folded `cm` into the nullifier derivation as defense-in-depth against
   Faerie-Gold/second-preimage-class attacks, and required an untruncated
   commitment hash (§3.2).
4. Removed the dummy-note asset-id exemption as an unnecessary special case
   (§3.3, §4.3(g)).
5. Added an explicit same-transaction nullifier-distinctness constraint,
   closing a same-note double-count path in the value-conservation equation
   (§4.3(c')).
6. Added an explicit range constraint on `public_delta`'s signed
   decomposition, closing a field-wraparound / under-constrained-circuit
   gap in value conservation (§4.3(f)).
7. Added a required transaction-binding signature, correcting an earlier
   over-simplification that assumed removing value-commitment machinery
   also removed the need for any binding/anti-malleability mechanism
   (§4.5).
8. Flagged Layer 1's inherent account-ciphertext concurrency/front-running
   liveness issue and required an epoch or nonce-equivalent mechanism,
   consistent with Zether's own documented mitigation (§5.3).
9. Added §9.5, identifying full asset-hiding's removal of any externally
   verifiable (non-issuer) supply invariant as the most significant open
   design question, and proposed a candidate per-institution "solvency
   proof" mechanism as a possible middle path.

None of these were fatal to the overall architecture; all are
incorporated as amendments above rather than requiring a redesign.