# PR Style

## Titles

Imperative, verb-first, capitalized, no period. Optional ticket in parens at end.

Good: `Add span ID and trace ID to GRPC request-related logs (FRY-208)`
Bad: `Adds span ID...` / `Added span ID...` / `This PR adds span ID...`

```
Integrate Kinesis in local Tilt environments
Add ability to configure Agent Attestation for clusters
Instrument BuildBuddy integration to github workflows
Remove ClickHouse-backed data from spirlctl and spirladm
```

## Description Sections

Check `.github/pull_request_template.md` (or `.github/PULL_REQUEST_TEMPLATE/*.md`) before opening; use it, don't substitute format.

## Draft by Default

Create as draft (`gh pr create --draft`); omit only if user explicitly asks.

## Labels

- **Target label** — component affected, e.g. `target:spirl-agent`. Added by GH Action; add manually if needed.
- **Type label** — prefix `type:`. Values: `enhancement`, `bug-fix`, `ci`, `documentation`, `breaking`, `tests-only`. Drives release notes.
