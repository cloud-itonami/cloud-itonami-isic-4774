# Security Policy

This project handles secondhand-goods listing, authentication and sale
decisions, including stolen-goods-reporting-relevant seller data. Treat
vulnerabilities as potentially high impact even when the demo data is
synthetic.

## Do Not Disclose Publicly

Report privately before opening public issues for:

- credential or verification-license-key exposure
- ResaleGovernor bypass (stolen-goods-reporting-gate, source-provenance-
  gate, counterfeit-flag-gate, condition-misrepresentation-gate)
- audit-ledger tampering
- over-disclosure beyond a subscriber contract's tier
- tenant isolation failures
- a sale confirming for a counterfeit-flagged or unauthenticated item
  through an undocumented path
- stolen-goods intake bypassing seller KYC verification

## Reporting

Use GitHub private vulnerability reporting when available for the
repository. If that is unavailable, contact the repository maintainers
through the cloud-itonami organization before publishing details.

Include:

- affected commit or version
- reproduction steps
- expected and actual behavior
- impact on listing/authentication data, governor enforcement or audit
  logging
- suggested fix, if known

## Production Guidance

- Store secrets and verification-license keys outside Git.
- Run governor tests before deployment.
- Export and review audit logs regularly.
- Use least privilege for listing agents, authenticators and service
  accounts.
- Alert on any stolen-goods-reporting-gate HOLD spike — it may indicate a
  coordinated fencing attempt.
