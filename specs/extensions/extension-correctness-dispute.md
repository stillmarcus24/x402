# Correctness and Dispute Extension

Status: Draft · Layer: downstream of settlement · Relates to: offer-and-receipt extension, issue #2887

## Motivation

x402 specifies how payment is offered, made, identified, and settled, and how a
receipt proves it. It does not specify what happens when the thing paid for was
**wrong** — a tool returns an incorrect result, a feed serves stale data, an answer
fails against ground truth. Payment correctness and *delivery* correctness are
different guarantees. Settlement proves money moved; it says nothing about whether the
counterparty earned it.

This extension defines a dispute path and a **bonded verdict**: a neutral adjudicator
stakes collateral that is slashed if its verdict is shown wrong, so a correctness claim
carries economic weight rather than reputation alone.

## Terms

- **Resolver** — the named source of ground truth a verdict is resolved against.
- **Verdict** — a signed, hash-chained statement (`CONFIRMED | REFUTED | PARTIAL |
  INSUFFICIENT_DATA`) over a disputed claim, bound to its resolver.
- **Correctness bond** — collateral held by the adjudicator, with a declared
  `per_verdict_max` slashable amount and a public on-chain balance.

## Normative requirements

1. A verdict **MUST** name its resolver and be independently re-verifiable: a third
   party recomputes the verdict hash from a published preimage and checks the signature
   against the adjudicator's key. (This reuses the receipt extension's signature model.)
2. An adjudicator offering bonded verdicts **MUST** publish, at a stable endpoint: the
   bond wallet, network, asset, `bonded_total`, `per_verdict_max`, live on-chain
   balance, custody type, and a running `slash_count` / `slashed_total`.
3. `per_verdict_max` **MUST NOT** exceed the on-chain balance divided by outstanding
   unslashed obligations; an adjudicator **MUST** report any unpaid or overdue slash
   obligation rather than advertise capacity it cannot cover.
4. A bond **MUST** declare whether a payout SLA is committed or best-effort; an
   uncommitted SLA **MUST NOT** be presented as a guarantee.
5. A dispute record **MUST** be append-only and reference the receipt (via
   `payment_identifier`) it disputes.
6. A bond **MAY** accept external contributions and **MAY** reinvest realized
   revenue into collateral; where it does, `per_verdict_max` scales with funded
   collateral, so the correctness guarantee compounds as capital is contributed
   rather than being fixed at issuance. Contribution and payout terms **MUST** be
   public, and the live collateral balance **MUST** remain independently checkable
   (requirement 2). This lets the party that bears a loss co-fund the guarantee that
   protects them, rather than requiring a single adjudicator to pre-commit all capital.

## Security considerations

- The bond caps the adjudicator's maximum lie-cost at `per_verdict_max`; consumers
  **MUST** treat a verdict's trust as bounded by that figure, not by the adjudicator's
  claims. A $1 `per_verdict_max` secures a verdict to $1, no more.
- Self-adjudication is disallowed: an adjudicator **MUST NOT** rule on a claim where it
  is the paid counterparty.
- Resolver drift: if the resolver's source changes after a verdict, the verdict is
  reported as unreproducible rather than silently re-resolved.

## Reference implementation (live)

StillOS Notary runs this today: bonded verdicts against named resolvers, Ed25519-signed
and hash-chained, with an on-chain correctness bond on Base (2-of-2 Gnosis Safe custody)
whose balance and slash history are public and re-verifiable. Endpoints: `/notary/bond`
(bond state), `/claim-verdict` (signed verdict), `/notary/verify?hash=` (recompute any
verdict or receipt). The bond starts small by design and is built to compound via
reinvestment and open contribution (requirement 6); the mechanism, not the current
amount, is what this extension standardizes.

Runnable conformance vectors (12 real receipts, each reproducing `SHA-256(preimage)`
cross-language byte-for-byte):
https://github.com/stillmarcus24/notary-x402-reference-kit/blob/main/conformance/dispute-extension-vectors.json
