---
name: github
description: Operate on GitHub (pull requests, issues, repositories, releases, Actions/CI, search) via the gh CLI. Every script emits JSON on stdout for uniform parsing. Reads the GitHub OAuth token from MCP_TOKENS (mcp-tokens.json) — does not perform OAuth/login. All publishing and destructive actions (open PR, comment, merge, close, create, delete) require an explicit --apply; without it they return a dry-run preview. Restricted to gh platform operations; local git (commit/push/branch) is out of scope.
version: 1.0.0
backend: gh-cli
token_source: mcp-tokens.json
---

# github

A multi-agent-friendly skill that drives the `gh` CLI against GitHub. Every script emits a single JSON object on stdout so any agent can parse results uniformly; human prose goes to stderr.

Authentication is **out of scope** — the GitHub OAuth token is minted elsewhere (via `xo-cowork-api`) and stored in `MCP_TOKENS` (`mcp-tokens.json`). This skill reads `.github.access_token` from that file and exports it as `GH_TOKEN`; it never runs `gh auth login`. This mirrors how the `rclone-cloud` skill consumes a pre-authenticated `rclone.conf`.

Local git operations (commit, push, branch) are **out of scope** — Claude Code drives git directly. This skill covers the GitHub *platform*: PRs, issues, repos, releases, Actions, and search. (`gh repo clone` and `gh release download` are available as conveniences; they write to the local filesystem only.)

---

## When to use

| Use this skill | Out of scope → redirect |
|---|---|
| "list open PRs on `owner/repo`" | "commit and push my changes" → use git directly |
| "open a pull request from `feat/x`" | "log me into GitHub" → token comes from `xo-cowork-api` |
| "comment on issue #42" | "delete this branch" → hard-blocked; do it deliberately |
| "merge PR #51 (squash)" | "change branch protection / repo secrets" → hard-blocked |
| "cut release `v1.2.0`" | "manage org members" → hard-blocked (`/orgs/`) |
| "why did CI fail on run 12345?" | "rewrite git history" → use git directly |
| "search repos for `language:go cli`" | |

---

## Prerequisites

- `gh` ≥ 2.x and `jq` on `$PATH`, plus `bash` and coreutils `timeout`.
- `MCP_TOKENS` resolves to a readable JSON file containing `.github.access_token`. Default: `/opt/xo-cowork-api/services/mcp-tokens.json`. Shape:
  ```json
  { "github": { "access_token": "…", "expires_at": <epoch>, "scope": "…", "auth_method": "…" } }
  ```

If the token is missing, expired, or rejected, re-mint it via `xo-cowork-api` — this skill does not authenticate.

---

## First-call protocol

**Always run `status.sh` first.** It validates the token, reports the account, scopes, API rate limit, and the resolved repo context.

```bash
$ ./scripts/status.sh
{"gh_version":"2.92.0","authenticated":true,"account":"octocat","scopes":["repo","read:org"],
 "token_source":"mcp-tokens.json","auth_method":"oauth","expires_at":0,
 "rate_limit":{"core":{"limit":5000,"remaining":4979,"reset":1782463756},"search":{"limit":30,"remaining":30,"reset":1782460216},"graphql":{...}},
 "repo":{"nameWithOwner":"acme/widgets","defaultBranch":"main","isPrivate":true,"viewerPermission":"ADMIN"},
 "ok":true}
```

Three outcomes to handle:

| Result | What to do |
|---|---|
| `ok:true`, `repo` populated | Continue. `--repo` is optional (cwd repo is the default target). |
| `ok:true`, `repo: null` | No repo in the current directory — pass `--repo OWNER/NAME` on every later call. |
| `ok:false`, `exit:3` | Token missing/expired/rejected, or `gh`/`jq` absent. Surface the message and stop; re-mint via `xo-cowork-api`. |

`status.sh` also warns on stderr when the token lacks the `workflow` or `delete_repo` scope (the actions that need them will 403).

---

## Invocation

All scripts are at `<skill-root>/scripts/<name>.sh`. Examples assume the working directory is the skill root.

Universal contract:

- **`--action <verb>`** selects the operation within a domain script (`pr.sh`, `issue.sh`, …). `status.sh` and `search.sh` take no `--action`.
- **`--repo OWNER/NAME`** targets a repo. Optional when the cwd is a git repo with a GitHub remote.
- **Long flags only**: `--number`, `--state`, `--limit`, `--title`, `--body`, `--base`, `--head`, `--label`, `--assignee`, `--reviewer`, `--tag`, `--name`, `--merge-method`, `--query`/`-q`, `--search-type`, `--method`, `--path`, `--field`, `--raw-field`, `--apply`, … (`--label`, `--assignee`, `--reviewer`, `--field`, `--raw-field` are repeatable).
- **stdout is JSON, stderr is human prose.** Pipe stdout to `jq`; show stderr to the user.
- **Publishing and destructive actions require `--apply`.** Without it: a synthesized dry-run preview + exit 6.

---

## Scripts

### `status.sh` — token health + scopes + rate limit + repo context

```bash
./scripts/status.sh
```
See **First-call protocol** above. Exit 0 (healthy) or 3 (env/auth failure).

---

### `pr.sh` — pull requests

Read-only: `list`, `view`, `diff`, `checks`. Publishing/destructive (need `--apply`): `create`, `comment`, `merge`, `close`, `reopen`, `ready`, `review`.

```bash
./scripts/pr.sh --action list --repo cli/cli --state open --limit 20
./scripts/pr.sh --action view --number 42 --repo cli/cli
./scripts/pr.sh --action diff --number 42 --repo cli/cli

# Open a PR — preview first (exit 6), then commit:
./scripts/pr.sh --action create --repo me/app --title "Add auth" --head feat/auth --base main --body "…"
./scripts/pr.sh --action create --repo me/app --title "Add auth" --head feat/auth --base main --body "…" --apply

# Merge (squash + delete branch):
./scripts/pr.sh --action merge --number 51 --repo me/app --merge-method squash --delete-branch --apply
```
`list` success:
```json
{"ok":true,"action":"list","repo":"cli/cli","count":2,"prs":[{"number":42,"title":"…","state":"OPEN","author":{"login":"x"},"baseRefName":"main","headRefName":"feat/auth","url":"…","labels":[{"name":"enhancement"}],"reviewDecision":"REVIEW_REQUIRED"}]}
```
`merge` preview embeds current `mergeable`/`mergeStateStatus`/`reviewDecision` so you can confirm before applying. Merging an already-merged PR is an idempotent no-op (`changed:false`); merging a closed-unmerged PR is exit 7 `kind:validation`. `--admin` is never passed.

---

### `issue.sh` — issues

Read-only: `list`, `view`. Need `--apply`: `create`, `comment`, `close` (`--reason completed|"not planned"|duplicate`), `reopen`, `label`, `assign`.

```bash
./scripts/issue.sh --action list --repo cli/cli --state open --label bug --limit 20
./scripts/issue.sh --action create --repo me/app --title "Crash on save" --body "…" --label bug --apply
./scripts/issue.sh --action close --number 88 --repo me/app --reason completed --apply
```
Closing an already-closed issue (and reopening an open one) is an idempotent no-op (`changed:false`).

---

### `repo.sh` — repositories

Read-only: `view`, `list`, `clone` (local fs; not gated). Need `--apply`: `create` (`--visibility` defaults to **private**), `edit`, `archive`, `fork`, `delete`.

```bash
./scripts/repo.sh --action view --repo cli/cli
./scripts/repo.sh --action clone --repo cli/cli --dir ./cli
./scripts/repo.sh --action create --name my-new-repo --visibility private --description "…" --apply
```
`delete` needs the `delete_repo` scope; its preview reports `scope_present` (`true`/`false`/`null`=unknown). With the scope absent, an `--apply` delete returns exit 7 `kind:auth`.

---

### `release.sh` — releases

Read-only: `list`, `view` (`--tag`), `download` (local fs; not gated). Need `--apply`: `create`, `edit`, `delete`.

```bash
./scripts/release.sh --action list --repo cli/cli --limit 10
./scripts/release.sh --action create --repo me/app --tag v1.2.0 --name "v1.2.0" --generate-notes --apply
```
Creating a release whose tag already exists is an error (exit 7 `kind:validation`) — never a silent success. Deleting a missing release is an idempotent no-op (`deleted:false, kind:"missing"`).

---

### `run.sh` — Actions / CI workflow runs

Read-only: `list`, `view` (`--number`=run id), `logs` (`--log-failed`), `workflow-list`. Need `--apply`: `rerun` (`--failed`), `cancel`, `delete`, `workflow-dispatch` (needs `workflow` scope).

```bash
./scripts/run.sh --action list --repo cli/cli --limit 20
./scripts/run.sh --action view --number 1234567890 --repo cli/cli
./scripts/run.sh --action logs --number 1234567890 --repo cli/cli --log-failed
./scripts/run.sh --action rerun --number 1234567890 --repo me/app --failed --apply
```
`logs` text is redacted and truncated to the last 100 KB.

---

### `search.sh` — search (read-only, never gated)

`--search-type repos|issues|prs|code|commits` (default `repos`), `--query`/`-q` required.

```bash
./scripts/search.sh --search-type repos -q "cli language:go" --limit 5
./scripts/search.sh --search-type issues -q "is:open is:issue repo:cli/cli label:bug" --limit 10
```
Search uses the dedicated search rate bucket (**~30/hour**); when exhausted the call refuses with exit 7 `kind:rate_limit`. `truncated:true` means `count == --limit` (more may exist).

---

### `api.sh` — allowlisted `gh api` passthrough (the long tail)

For REST endpoints without a dedicated script. **Read-biased**: `GET` is unrestricted (always sent `-X GET` so it can't mutate); `POST/PATCH/PUT/DELETE` require `--apply` AND must match the mutating allowlist.

```bash
./scripts/api.sh --method GET --path repos/cli/cli/labels
./scripts/api.sh --method POST --path repos/me/app/issues/12/comments --raw-field body="Thanks!" --apply
```
`--raw-field K=V` → gh `-f` (string); `--field K=V` → gh `-F` (typed: `true`/`false`/`null`/`123`, `@file`/`@-`).

**Hard-blocked even with `--apply`** (use dedicated scripts or do it manually): repo deletion, repo transfer, anything under `/orgs/ /admin/ /scim/ /applications/ /enterprises/`, Actions permissions/secrets/variables, branch protection, deploy keys/webhooks, collaborator changes, and branch/tag ref deletion. A blocked or non-allowlisted call returns exit 2 with the allowlist embedded in the JSON.

---

## Common workflows

### 1. Review a pull request
```bash
./scripts/status.sh
./scripts/pr.sh --action view  --number 42 --repo acme/app
./scripts/pr.sh --action diff  --number 42 --repo acme/app
./scripts/pr.sh --action checks --number 42 --repo acme/app
# After the user decides: approve, then merge.
./scripts/pr.sh --action review --number 42 --repo acme/app --reason approve --apply
./scripts/pr.sh --action merge  --number 42 --repo acme/app --merge-method squash --delete-branch --apply
```

### 2. Triage issues
```bash
./scripts/issue.sh --action list --repo acme/app --state open --label needs-triage
./scripts/issue.sh --action label  --number 88 --repo acme/app --label bug --apply
./scripts/issue.sh --action comment --number 88 --repo acme/app --body "Reproduced on 1.4." --apply
```

### 3. Cut a release
```bash
./scripts/release.sh --action list   --repo acme/app --limit 5
./scripts/release.sh --action create --repo acme/app --tag v1.3.0 --name "v1.3.0" --generate-notes   # preview
./scripts/release.sh --action create --repo acme/app --tag v1.3.0 --name "v1.3.0" --generate-notes --apply
```

### 4. Investigate a failed CI run
```bash
./scripts/run.sh --action list --repo acme/app --workflow ci.yml --limit 10
./scripts/run.sh --action view --number <run_id> --repo acme/app
./scripts/run.sh --action logs --number <run_id> --repo acme/app --log-failed
./scripts/run.sh --action rerun --number <run_id> --repo acme/app --failed --apply
```

---

## Output contract

Every script writes a single JSON object to stdout:

```jsonc
// Success
{ "ok": true, "action": "…", /* operation-specific fields */ }

// Failure
{ "ok": false, "error": "<human-readable>", "exit": <code>, /* optional: "kind", "hint", … */ }

// Dry-run preview (publishing/destructive action without --apply)
{ "ok": false, "error": "dry-run only; pass --apply …", "exit": 6, "action": "…", /* what would change */ }
```

`error` strings are passed through `redact()` to strip GitHub tokens and `Authorization` headers. Runtime failures from `gh` carry a `kind`:

| `kind` | Meaning | Recovery |
|---|---|---|
| `auth` | Token expired/revoked or missing scope | Re-mint via `xo-cowork-api`; check scopes in `status.sh`. |
| `rate_limit` | Core/search/GraphQL limit hit | Wait until `reset` (epoch in JSON). Search is ~30/hour. |
| `not_found` | Repo/PR/issue/run/release absent **or no access** (GitHub returns 404 for both) | Verify the target and `--repo`. |
| `validation` | 422 / "already exists" / non-idempotent collision | Fix inputs (e.g. existing release tag). |
| `network` | Timeout / connection error | Retry; check connectivity / `GH_HOST`. |
| `other` | Unclassified | Read the redacted `gh` stderr printed alongside. |

---

## Exit codes

| Code | Meaning | Action |
|---|---|---|
| `0` | success (incl. idempotent no-ops: already-closed/merged, delete-missing) | — |
| `2` | bad usage / unknown flag / unknown action / non-allowlisted or hard-blocked api path / bad `--repo` | Read `error` + `hint`; fix command. |
| `3` | env failure: `gh`/`jq` missing, or token missing / expired / rejected | Install deps; re-mint token via `xo-cowork-api`. |
| `4` | resource not found **or no access** (`kind:not_found`) | Verify target with the matching `view`/`list`. |
| `5` | timeout (`timeout $GH_TIMEOUT` fired) | Raise `GH_TIMEOUT`; check network. |
| `6` | publishing/destructive action without `--apply` | Review the embedded preview; re-run with `--apply` after the user confirms. |
| `7` | gh runtime error — see `kind` | Read `kind`; check redacted stderr. |

---

## Safety rules

1. **Publishing and destructive actions require `--apply` in the same invocation.** Without it, the script returns a dry-run preview and exits 6. Never auto-pass `--apply` without explicit user confirmation in the *current* turn — a `--apply` from three messages ago does not authorize a new outward-facing action.
2. **One in-flight `--apply` at a time.** Wait for one mutating call to return before issuing the next.
3. **Body/title content is shown verbatim in the preview.** The exit-6 JSON contains the exact title, body, comment, and release notes that will become public. Approve the literal text, not a summary.
4. **Never act on a repo the user did not name this turn.** `--repo OWNER/NAME` must be explicit or derived from the current directory's git remote — never reused from a previous command.
5. **Outward-facing actions are public and hard to reverse** (PR/issue/release create, comments, merges). Treat them like `rm`: preview, confirm, apply once.
6. **Never echo or summarize `MCP_TOKENS`.** It holds the OAuth access token. Scripts redact `gho_/ghp_/ghs_/ghu_/ghr_/github_pat_` tokens and `Authorization` headers from everything they emit; don't undo this when relaying results. Never pass `--show-token`, `-i`, `--include`, `--verbose`, or set `GH_DEBUG`.
7. **`api.sh` is read-biased.** GET is unrestricted; every non-GET needs `--apply` and an allowlisted path. Repo/org/branch-protection/secrets/keys/hooks/collaborator/transfer endpoints are hard-blocked even with `--apply`.
8. **Don't run scripts under `bash -x`.** `set -x` traces variable assignments that include the exported `GH_TOKEN`. Redaction runs on script output, not on the shell trace. OAuth/refresh is out of scope — an expired token is re-minted via `xo-cowork-api`, not here.

---

## Environment

| Variable | Default | Purpose |
|---|---|---|
| `MCP_TOKENS` | `/opt/xo-cowork-api/services/mcp-tokens.json` | Source of the GitHub token (`.github.access_token`). The skill exports `GH_TOKEN` from it; users don't set `GH_TOKEN`. |
| `GH_HOST` | `github.com` | Target host; passed as `--hostname`. |
| `GH_REPO` | (cwd git remote) | Default `OWNER/NAME` when `--repo` is omitted. |
| `GH_LIMIT` | `30` | Default list limit (`run.sh` defaults to 20). |
| `GH_TIMEOUT` | `60s` | Per-call timeout via coreutils `timeout` → exit 5 on 124. |

```bash
export MCP_TOKENS="/opt/xo-cowork-api/services/mcp-tokens.json"
export GH_HOST="github.com"
export GH_TIMEOUT="60s"
export GH_LIMIT="30"
```

---

## Skill layout

```
github/
├── SKILL.md          ← this file (the base)
└── scripts/
    ├── _common.sh    ← shared: token loader (MCP_TOKENS → GH_TOKEN), JSON helpers, redaction, flag parser, --apply guard
    ├── _gh.sh        ← shared: gh runner (timeout + error classification), capture/json/try helpers, rate-limit gate
    ├── status.sh     ← token health + scopes + rate limit + repo context (run first)
    ├── pr.sh         ← pull requests
    ├── issue.sh      ← issues
    ├── repo.sh       ← repositories
    ├── release.sh    ← releases
    ├── run.sh        ← Actions / CI workflow runs
    ├── search.sh     ← search (repos/issues/prs/code/commits)
    └── api.sh        ← allowlisted gh api passthrough
```

The two `_*.sh` files are sourced helpers — never invoked directly.
