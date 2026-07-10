# Governance

`cloud-itonami-isic-4774` is an OSS open-business blueprint. Governance
covers both code and the operator model.

## Maintainers

Maintainers may merge changes that preserve these invariants:

- ResaleAdvisor-LLM cannot directly intake, authenticate, sell or resolve a
  dispute.
- ResaleGovernor remains independent of the advisor.
- hard governor violations (stolen-goods-reporting-gate, source-
  provenance-gate, counterfeit-flag-gate, condition-misrepresentation-gate,
  licensed-disclosure) cannot be overridden by human approval.
- a dispute/correction request never auto-resolves, at any rollout phase.
- a high-value sale always reaches a human, regardless of confidence.
- every commit, hold and disclosure event is auditable.
- no schema field exists for buyer payment credentials or seller bank
  details — this actor mediates decisions, it never touches money.
- real seller/buyer PII and real verification-license credentials stay
  outside Git.

## Decision Records

Architecture decisions live in `docs/adr/`. Changes to the trust model,
storage contract, disclosure scope, public business model, operator
certification or license should add or update an ADR.

## Operator Governance

Anyone may fork and operate independently. itonami.cloud certification is a
separate trust mark and should require security, audit, support and
data-flow review.

Certified operators can lose certification for:

- bypassing governor checks
- disclosing data to an uncontracted party
- confirming a sale for a counterfeit-flagged or unauthenticated required-
  category item
- intaking high-value/flagged-category items from unverified sellers
- misrepresenting certification status
- failing to respond to security incidents or buyer/seller disputes
- hiding material changes to customer-facing operation
