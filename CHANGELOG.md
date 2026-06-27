# Changelog

## Unreleased

- Initial scaffold of `issue-loop`: a reusable GitHub Actions workflow that works a
  `loop`-labeled issue queue by delegating each per-issue solve to
  `anthropics/claude-code-action@v1`.
- Composite actions: `select` (eligibility + matrix), `render-prompt` (per-issue
  policy), `finalize` (label reconciliation + crash fallback), `report` (summary +
  stale-state cleanup).
- GitHub-native state machine (`loop`, `loop:in-progress`, `loop:pr-open`,
  `loop:needs-human`, `loop:error`); human-in-the-loop escalation; stack-agnostic
  policy prompt; cost guardrails.
