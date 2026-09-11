# Operator Guide

This guide is for people who want to start an open business from
`cloud-itonami-isic-4774`.

## 1. Fork and Run

```bash
git clone https://github.com/cloud-itonami/cloud-itonami-isic-4774
cd cloud-itonami-isic-4774
kbb -M:dev:test
kbb -M:dev:run
```

The default demo uses entirely fictitious items, sellers and licenses.
Production listings must stay outside the repository and be injected
through a store adapter, and every authentication verdict must carry a
real, verifiable source citation.

## 2. Choose an Operating Mode

| Mode | Use when |
|---|---|
| Demo | validating the actor and governor contract |
| Self-host | one organization owns infrastructure and data |
| Managed tenant | an operator hosts for a customer |
| Certified operator | itonami.cloud has reviewed security and process controls |

## 3. Production Checklist

- replace demo items/sellers with real, KYC-verified seller records
- register a real `verification-license` for at least one authentication
  service before listing any authentication-required category
  (`resale.facts/authentication-required-categories`)
- register a real `verification-license` for a local law-enforcement
  secondhand-dealer reporting integration if your jurisdiction requires it
- configure Datomic Local, kotoba-server or an equivalent durable SSoT
- configure the LLM adapter through environment variables or secret manager
- define subscriber contract tenants/tiers and RBAC rules
- run `kbb -M:dev:test`
- run `kbb -M:lint`
- verify audit-ledger export
- document backup and restore
- document incident response
- document the buyer/seller dispute-handling SLA
- get written legal review for the jurisdictions you serve (secondhand-
  dealer/pawnbroker reporting statutes vary by state/country)

## 4. Sales Motion

Start with a narrow offer:

1. onboard one real authentication-service license
2. prove governed, tier-scoped disclosure end to end
3. run one authenticate → sale workflow in assisted mode (human-approved)
4. export the audit ledger for review
5. convert to a metered or subscription contract

Avoid claiming "全カテゴリの真贋保証" before your authentication-service
coverage actually spans the categories a customer needs — report coverage
honestly (`resale.facts/coverage`), never oversell.

## 5. Certification Requirements

itonami.cloud certification should require:

- passing tests and lint on the published version
- written data-flow diagram (intake → governor → authentication → sale →
  disclosure)
- backup/restore evidence
- incident contact and response window
- proof that production intake/authentication/sale/disclosure goes through
  ResaleGovernor
- proof that real seller/buyer PII is not stored in Git
- proof that a dispute channel exists and is human-reviewed
- customer-facing support and licensing terms

## 6. Operator Responsibilities

Operators are responsible for:

- lawful basis and secondhand-dealer licensing for each jurisdiction served
- local pawnbroker/secondhand-dealer reporting-law compliance
- secure infrastructure and tenant isolation
- honest source-catalog and verification-license maintenance
- human review workflow for high-value-sale and dispute-request operations
- data-retention policy
- security updates

The OSS project provides software and an operating blueprint. It does not
make an operator compliant by itself, and it does not license or endorse
any specific authentication vendor.
