# cloud-itonami-isic-4774

Open Business Blueprint for **ISIC Rev.4 4774**: retail sale of second-hand
goods — the eBay-secondhand / Vestiaire Collective / GameStop trade-in
class of business — published as an OSS business that any qualified
operator can fork, deploy, run, improve and sell.

A peer-to-peer/consignment resale marketplace: sellers submit items for
intake, the marketplace authenticates branded/high-value items, lists them,
and confirms sales — with structural defenses against the two risks unique
to secondhand retail: **fenced (stolen) goods** entering inventory, and
**counterfeit** items reaching a sale. Built on this workspace's
[`langgraph`](https://github.com/kotoba-lang/langgraph) StateGraph runtime
(portable `.cljc`, supervised superstep loop, interrupts, Datomic/in-mem
checkpoints) — the same actor pattern as
[`cloud-itonami-isic-6311`](https://github.com/cloud-itonami/cloud-itonami-isic-6311)
and [`cloud-itonami-isic-8291`](https://github.com/cloud-itonami/cloud-itonami-isic-8291).

> **Why an actor layer at all?** A ResaleAdvisor-LLM is great at
> normalizing seller intake, drafting authentication verdicts, and
> proposing subscriber column sets — but it has **no notion of secondhand-
> dealer reporting duties, counterfeit/IP law, condition-grading
> integrity, or a subscriber's disclosure tier**. Letting it write or
> confirm directly invites fenced-goods intake from an unverified seller,
> a counterfeit item reaching a sale, a misrepresented condition grade, or
> over-disclosure beyond a contract's tier. This project seals the
> ResaleAdvisor-LLM into a single node and wraps it with an independent
> **ResaleGovernor**, a human **review workflow**, and an immutable
> **audit ledger**.

## Scope (deliberately narrow — read this before anything else)

This actor **mediates listing, authentication and sale decisions**. It
never touches buyer payment credentials or seller bank details — there is
no field anywhere in this schema for payment processing (see
`docs/adr/0001-architecture.md`). Authentication provenance is limited to
a real, citable public reference (`src/resale/facts.cljc`: USPTO trademark
search, for brand-registration lookup only) or an operator-registered
`:licensed-authentication-service` / `:licensed-secondhand-reporting-feed`
— every authentication verdict must resolve to one of these, never a bare
"the LLM inferred it".

## Consuming this actor from another blueprint

`:disclosure/query` (an item's listing, columns limited to your contract
tier) is the governed read surface. It always runs through the
ResaleGovernor's licensed-disclosure check — there is no bypass.

See [`docs/DESIGN.md`](docs/DESIGN.md) for the full architecture and
[`docs/adr/0001-architecture.md`](docs/adr/0001-architecture.md) for the
decision record. See [`docs/business-model.md`](docs/business-model.md) and
[`docs/operator-guide.md`](docs/operator-guide.md) to start this as an open
business on itonami.cloud.

## Open business

This repository is not only source code. It is a public, forkable business
model:

| Layer | What is open |
|---|---|
| OSS core | Actor runtime, ResaleGovernor, governed disclosure, audit ledger |
| Business blueprint | Customer, offer, pricing, unit economics, sales motion |
| Operator playbook | How to fork, deploy, support and sell the service |
| Trust controls | Governance, security reporting, policy tests, audit requirements |

## The core contract

```
request + injected role/tenant/phase context
        │
        ▼
   ┌───────────────┐    proposal      ┌──────────────────────┐
   │ ResaleAdvisor  │ ───────────────▶│ ResaleGovernor        │  (independent system)
   │ -LLM (sealed)  │  draft + source │  stolen-goods ·        │
   └───────────────┘   citation       │  counterfeit · condition│
                                       └──────────────────────┘
                                              │
                                   commit / publish only if allowed
                                              ▼
                                    append-only audit ledger
```

**Single invariant**: ResaleAdvisor-LLM never intakes, authenticates,
sells, or discloses a record the ResaleGovernor would reject.

## Run

```bash
clojure -M:dev:test   # governor contract · store parity · phases · facts
clojure -M:dev:run    # 8-operation demo through one OperationActor
clojure -M:lint
```

## Non-Negotiables

- Do not commit real seller/buyer PII or real verification-license
  credentials.
- Do not add a schema field for buyer payment credentials or seller bank
  details.
- Do not bypass the ResaleGovernor for production intake, authentication,
  sale or disclosure.
- Do not serve a disclosure without an active, registered contract.
- Do not fabricate a source-catalog entry or a verification-license
  record.

License: AGPL-3.0-or-later.
