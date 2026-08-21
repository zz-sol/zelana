# Questions for Cat / the institutional team

We've redesigned the protocol around per-institution shielded pools
([high_level_design.md](high_level_design.md)), following the direction we
converged on: Zcash-style pools, one per institution, hidden TVL, repo/DvP
support. Before we commit to the detailed spec there are a few things we
need confirmed from the institutional side. Roughly ordered by how much
the answer would change the design — Q1 and Q2 matter most.

## Q1. What "hiding TVL of public assets" can actually mean

For a transparent asset like USDC, the strongest guarantee any scheme can
offer (ours and the wrapped-mint approach alike) is this: the public vault
balance of each institution's pool is visible at all times, because USDC
itself is transparent. What's hidden is every individual transfer, all
amounts and counterparties, all gross flows, and the gap between the vault
balance and the institution's true position (the outstanding bilateral
claims built up through inter-institution transfers). Each periodic net
settlement then reveals one net flow per counterparty pair per period.
Longer settlement periods leak less but carry more bilateral credit
exposure, and the cadence would be configured per institution pair.

Natively confidential assets (institution-issued bonds, deposit tokens)
don't have this limit — their TVL and supply are fully hidden from
everyone except the issuer.

Does this match what institutions expect when they say they want TVL of
public assets hidden? In particular, is periodic-net-flow visibility at
settlement acceptable, and is there a view on cadence (daily, weekly,
threshold-triggered)?

## Q2. Counterparty pairs are visible on inter-institution transfers

A cross-pool transfer necessarily touches both institutions' pool programs
on-chain, so third parties can see *that* A and B transacted and how often
— not the amount, not the instrument, not which clients. Is that
acceptable, or do institutions need the counterparty pair hidden too? If
the latter, we'd add scheduled zero-value dummy cross-pool transfers
and/or a routing layer. It's feasible, but it adds cost and operational
coordination, so we'd rather only build it if it's actually required.

## Q3. Receiving a transfer requires being online

Our design gates every incoming cross-pool transfer on the receiving
institution's signature. This is deliberate: it's the receiving
institution accepting the sender's IOU, and it doubles as real-time
exposure control — the receiver sees the incoming amount and can refuse
past its credit limit with that counterparty. It's analogous to accepting
a correspondent-banking credit. But it means an institution has to run an
online signing service to *receive* transfers. Is that acceptable
operationally? And would institutions rather approve per-transfer, or
pre-authorize limits (sign once per counterparty per period up to a cap)?

## Q4. Bilateral credit + periodic netting as the peg model

Inter-institution transfers don't move the underlying public asset per
transfer; they accrue bilateral claims that get net-settled periodically
in the base asset. (This is what makes the TVL hiding in Q1 possible.)
Between settlements, each institution carries unsecured bilateral exposure
to its counterparties, bounded by the limits from Q3 — economically much
like traditional bilateral netting, with non-performance at settlement
handled as a credit/legal event under the bilateral agreement rather than
by the protocol. Is that the right model, or do institutions expect
collateralized/pre-funded settlement? Note that collateral posted in a
public asset is visible, so pre-funding leaks more.

## Q5. Repo terms are enforced by contract, not by the protocol

In our design a repo's opening and closing legs are each atomic on-chain
DvPs — within a leg, both sides settle together or not at all, so there's
no principal risk. But whether the closing leg *happens at all* at
maturity is not protocol-enforced: if the cash taker defaults at term, the
cash giver keeps the collateral notes, per the repo agreement, exactly as
in traditional bilateral repo. Is contractual enforcement of the term
sufficient, or do institutions expect on-chain enforcement of the closing
leg (e.g. escrowing the collateral for the term)? On-chain term escrow is
buildable but locks the collateral — no rehypothecation during the term —
and adds protocol complexity.

## Q6. Who holds clients' spending keys

Within an institution's pool, notes are bearer-style: whoever holds a
note's spending key controls it. The institution's audit key sees amounts
but not client identities, and cannot move client funds (read-only audit,
as agreed). For an institution's own clients, who holds the spending keys
— the client directly, the institution as custodian, or both (2-of-2)?
This doesn't change the protocol, but it changes the wallet and
key-management deliverables, and what "the institution controls its own
pool" means in practice.

## Q7. Who runs the dummy traffic

Anonymity in a small pool is inflated by injecting zero-value transactions
that are indistinguishable from real ones. Someone has to generate that
traffic and pay its fees — most naturally the institution itself or a
relayer it runs — at a cadence that determines the privacy level. Are
institutions comfortable operating and paying for their own dummy traffic
as part of running a pool, or is this expected to be a service from the
protocol operator or a third party?
