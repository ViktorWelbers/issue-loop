# issue-loop

> An autonomous, label-driven issue queue for GitHub. Label an issue `loop`; on a
> schedule, Claude picks it up, opens a focused pull request, or asks you a precise
> question when it's blocked — then resumes from your answer on the next run.

`issue-loop` is a thin **orchestration layer** on top of Anthropic's official
[`claude-code-action`](https://github.com/anthropics/claude-code-action). The action
already does the hard part (running Claude unattended, auth, committing, opening PRs).
`issue-loop` adds the parts it doesn't have:

- a **queue** — work many issues over time from a single label,
- a **state machine** across runs (so re-runs resume instead of redoing),
- **human-in-the-loop** escalation (it @-mentions you and waits for your reply),
- **stack-agnostic** behavior — it reads *your* repo's `CLAUDE.md`/`CONTRIBUTING.md`
  and runs *your* tests; it assumes no particular language or framework,
- **cost guardrails** (issues per run, attempts, turn budget).

It is distributed as a **reusable workflow**, so adopting it is one job in one file.

## Quick start

1. **Add a secret** to your repo: `ANTHROPIC_API_KEY` (or `CLAUDE_CODE_OAUTH_TOKEN`
   from `claude setup-token` if you're on a Claude subscription).
2. **Add the workflow** — copy [`examples/issue-loop.yml`](examples/issue-loop.yml) to
   `.github/workflows/issue-loop.yml` in your repo and set `maintainer:` to your handle.
3. **Queue an issue** — add the `loop` label to any open issue.
4. **Run it** — wait for the daily schedule, or trigger it from the **Actions** tab
   (use the `dry_run` input first to preview what it would pick up).

## How it works

Each run is three stages:

| stage | does | uses an LLM? |
| --- | --- | --- |
| `select` | list `loop` issues, filter to *eligible*, cap, mark `loop:in-progress`, emit a matrix | no |
| `solve` | per issue: render a policy prompt → run `claude-code-action` → it implements + opens a PR | yes |
| `finalize` / `report` | reconcile labels from the result, post a fallback comment on a crash, clear stale state | no |

### State lives entirely in GitHub — nothing is written into your repo

| label | meaning |
| --- | --- |
| `loop` | **you** add this to opt an issue into the queue |
| `loop:in-progress` | claimed by the current run (auto-cleared) |
| `loop:pr-open` | a PR exists; excluded from auto-pickup |
| `loop:needs-human` | blocked; the maintainer was @-mentioned; skipped until you reply |
| `loop:error` | the run crashed; check the logs |

Progress and attempts are tracked from the bot's own dated issue comments (each ends
with a hidden `<!-- issue-loop:run ... -->` marker). The work itself lives on a
`loop/issue-<N>` branch and its PR. Re-runs continue that branch.

### Human-in-the-loop

When `issue-loop` can't proceed, it posts the **specific** question, @-mentions your
`maintainer`, and sets `loop:needs-human`. Reply in the issue. On the next run, because
your comment is newer than the bot's last one, the issue becomes eligible again and the
agent treats your reply as authoritative.

## Configuration

All inputs of the reusable workflow (`with:`):

| input | default | meaning |
| --- | --- | --- |
| `label` | `loop` | the queue label |
| `max_issues_per_run` | `1` | how many issues to work per run |
| `max_attempts` | `3` | give up (→ `loop:needs-human`) after this many runs on an issue |
| `maintainer` | `""` | handle to @-mention when blocked, e.g. `@you` |
| `model` | `""` | optional Claude model id; omit for the action default |
| `max_turns` | `40` | per-issue agent turn budget (cost cap) |
| `verify_command` | `""` | exact test command; omit to let the agent infer it |
| `issue` | `""` | target a single issue number (manual runs) |
| `dry_run` | `false` | select only; mutate nothing; run no agent |

Secrets (provide one): `anthropic_api_key` **or** `claude_code_oauth_token`.

## Security model

- Runs in **your** repo with the default `GITHUB_TOKEN` and the `permissions:` you
  grant in your calling workflow — **no GitHub App, no extra credentials**.
- The agent is instructed never to merge, never to push the default branch, never to
  force-push, and to act only on its one issue and branch.
- The runner + scoped token are the sandbox.

> **Note:** PRs opened by the default `GITHUB_TOKEN` do **not** trigger your other
> workflows (a GitHub anti-recursion rule). If you need the loop's PRs to run CI,
> use a GitHub App token — planned as an optional input.

## Cost note

If you authenticate with a **subscription** OAuth token, the loop draws down your
personal Claude rate limits (and large models drain them fast). Keep
`max_issues_per_run` and `max_turns` low to start, and watch your usage.

## Status

v1 is GitHub-Actions-native, implement-and-PR, sequential, human-merge. See
[CHANGELOG.md](CHANGELOG.md). Roadmap: GitHub App tokens, iterating on existing PRs
from review feedback, and a container/CLI core to run outside GitHub Actions.

## License

MIT — see [LICENSE](LICENSE).
