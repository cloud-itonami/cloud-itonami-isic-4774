# Open Business Blueprint: cloud-itonami-isic-4774

This repository publishes an OSS business model for operating a
peer-to-peer/consignment secondhand-resale marketplace (eBay-secondhand /
Vestiaire Collective / GameStop trade-in class) on itonami.cloud, with
structural defenses against fenced-goods intake and counterfeit sales.

## Classification

- Repository name: `cloud-itonami-isic-4774`
- Primary classification: ISIC Rev.4 4774
- Activity: retail sale of second-hand goods (consignment/P2P resale)
- Served domain: item intake, authentication verdicts, sale confirmation,
  governed disclosure — never payment processing, custody, or shipping

The market-data actor (`cloud-itonami-isic-6311`) and the corporate-
compliance-intelligence actor (`cloud-itonami-isic-8291`) are the direct
templates: sealed advisor, independent governor, append-only ledger. This
is the 7th `:spec → real repo` promotion in `kotoba-lang/industry`'s
registry.

## Customer

Primary customers (contracted, licensed access only for the wholesale
disclosure surface — retail sellers/buyers interact through an operator's
own front end, not this repo directly):

- consignment/resale platform operators needing a governed intake+
  authentication+sale pipeline without building fraud/compliance controls
  themselves
- brand-protection teams needing counterfeit-flag audit trails
- local law enforcement / secondhand-dealer compliance integrations
  (via the `:licensed-secondhand-reporting-feed` seam)
- other `cloud-itonami-{ISIC}` blueprint operators who need a resale-
  marketplace capability (the `:resale-marketplace` wholesale pattern)

## Problem

Secondhand marketplaces (eBay, Poshmark, StockX, GameStop) each build their
own fraud/authenticity/compliance stack from scratch, and customers cannot
inspect the governance logic (why was this item listed, what backs this
authentication verdict, why did a sale require manual review). Fenced-
goods and counterfeit risk are usually bolted on
inconsistently rather than structurally guaranteed.

## Offer

Operators provide an OSS actor for secondhand resale:

- item intake with condition-claimed capture
- authentication verdicts for branded/high-value categories, source-cited
- structural stolen-goods-reporting gate (mirrors real US secondhand-
  dealer statutes: CA Business & Professions Code §21625 et seq., NY
  General Business Law Art. 5 §§60-70)
- structural counterfeit-flag gate (a sale cannot confirm without a real
  `:authentic` verdict for categories that require one)
- condition-misrepresentation gate (claimed vs. verified grade)
- governed, tier-scoped disclosure
- a buyer/seller dispute channel, always human-reviewed
- immutable audit ledger of every intake/authentication/sale/disclosure
  event

The core promise: ResaleAdvisor-LLM can draft intake normalization and
authentication proposals, but it cannot intake, authenticate, sell, or
disclose unless the independent ResaleGovernor allows it.

## Revenue

Operators can sell:

- per-listing or per-transaction fee (consignment commission)
- tiered subscriptions: `:tier/basic` (listing) → `:tier/pro` (+ condition/
  authentication detail) → `:tier/institutional` (+ seller-id/raw
  provenance, for compliance-integrated wholesale customers)
- wholesale API access to other `cloud-itonami-{ISIC}` blueprint operators
- managed hosting: monthly subscription per tenant
- authentication-service integration: connecting a real authenticator
- compliance package: audit export, dispute-handling SLA, security review

| Package | Customer | Price shape |
|---|---|---|
| Basic listing | small resale shop | per-listing fee |
| Pro tier | consignment platform | monthly platform fee + commission |
| Institutional tier | brand-protection / compliance team | monthly fee + usage |
| Fleet wholesale | other cloud-itonami operators | API metering |

## Unit Economics

Track these numbers for every operator:

- authentication-service integration hours per new category
- monthly infrastructure cost
- LLM cost per operation (intake / authenticate / sale / disclosure)
- dispute-handling hours per tenant
- gross margin after infrastructure and support
- churn and expansion revenue per contract tier

## Open Participation

Anyone may fork, self-host, submit patches, publish compatible source-
catalog extensions (real, citable sources only), or create a local
operator business. itonami.cloud should require certification before
listing an operator as a trusted provider.

## Marketplace Metadata

```edn
{:itonami.blueprint/id "cloud-itonami-isic-4774"
 :itonami.blueprint/name "Secondhand Resale Marketplace Actor"
 :itonami.blueprint/isic-rev4 "4774"
 :itonami.blueprint/domain :retail/secondhand-resale
 :itonami.blueprint/license "AGPL-3.0-or-later"
 :itonami.blueprint/operator-model :certified-open-business
 :itonami.blueprint/repo "https://github.com/cloud-itonami/cloud-itonami-isic-4774"
 :itonami.blueprint/status :public-oss
 :itonami.blueprint/required-technologies [:identity :forms :audit-ledger]
 :itonami.blueprint/optional-technologies [:dmn :bpmn]}
```

## Non-Negotiables

- Do not commit real seller/buyer PII or real verification-license
  credentials.
- Do not add a schema field for buyer payment credentials or seller bank
  details.
- Do not bypass the ResaleGovernor for production intake, authentication,
  sale or disclosure.
- Do not serve a disclosure to a tenant without an active, registered
  contract.
- Do not fabricate a source-catalog entry or a verification-license record
  to expand apparent coverage.
- Do not market an uncertified deployment as an itonami.cloud certified
  operator.
