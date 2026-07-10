# Contributing

`cloud-itonami-isic-4774` accepts contributions to the OSS actor, governor
tests, documentation, examples and open business blueprint.

## Development

```bash
clojure -M:dev:test
clojure -M:lint
```

Keep changes small and include tests for governor, audit, store or
disclosure behavior.

## Rules

- Do not commit real seller/buyer PII, real item images, or real
  verification-license credentials.
- Keep production intake, authentication, sale and disclosure behind
  ResaleGovernor.
- Treat every new category or jurisdiction as high-risk: add tests for
  stolen-goods-reporting-gate, source-provenance-gate, counterfeit-flag-
  gate, condition-misrepresentation-gate and audit logging.
- Never fabricate a source-catalog entry or a verification-license record
  to expand apparent coverage.
- Document any new business-model or operator assumption in `docs/`.

## Pull Requests

PRs should describe:

- what behavior changed
- which governor invariant is affected
- how it was tested
- whether operator or certification docs need updates
