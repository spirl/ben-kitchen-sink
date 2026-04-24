# PR Style

## Titles

Imperative present tense, starting with a verb, capitalized, no period. Optional ticket in parentheses at end.

Good: `Add span ID and trace ID to GRPC request-related logs (FRY-208)`
Bad: `Adds span ID...` / `Added span ID...` / `This PR adds span ID...`

```
Integrate Kinesis in local Tilt environments
Add ability to configure Agent Attestation for clusters
Instrument BuildBuddy integration to github workflows
Remove ClickHouse-backed data from spirlctl and spirladm
```

## Description Sections

Check for `.github/pull_request_template.md` (or `.github/PULL_REQUEST_TEMPLATE/*.md`) before opening a PR. If found, use it — fill each section, don't substitute your own format. Standard sections:

- **Summary** — what the PR changes
- **Motivation** — link to issue, or motivation if no issue
- **Tests performed** — how tested (automated or manual); screenshots for user-visible changes
- **Public Release Notes** — customer-facing notes (bugs, features, breaking changes); "N/A" if none
- **Rollback Plan** — special rollback instructions; "N/A" if default (revert to previous commit) applies

## Labels

- **Target label** — component(s) affected, e.g. `target:spirl-agent`. GH Action adds these automatically; add manually if needed.
- **Type label** — must start with `type:`. Use `enhancement`, `bug-fix`, `ci`, `documentation`, `breaking`, `tests-only`, or similar. Types drive release note sections.
