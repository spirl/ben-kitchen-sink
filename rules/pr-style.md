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

Check for `.github/pull_request_template.md` (or `.github/PULL_REQUEST_TEMPLATE/*.md`) before opening. If found, use it — fill each section, don't substitute your own format. Standard sections:

- **Summary** — what the PR changes
- **Motivation** — link to issue, or motivation if no issue
- **Tests performed** — how tested (automated or manual); screenshots for user-visible changes
- **Public Release Notes** — customer-facing notes (bugs, features, breaking changes); "N/A" if none
- **Rollback Plan** — special rollback instructions; "N/A" if default applies

## Draft by Default

Always create as draft (`gh pr create --draft`); omit only if user explicitly asks for ready-for-review.

## Labels

- **Target label** — component affected, e.g. `target:spirl-agent`. GH Action adds automatically; add manually if needed.
- **Type label** — must start with `type:`. Use `enhancement`, `bug-fix`, `ci`, `documentation`, `breaking`, `tests-only`, or similar. Types drive release note sections.
