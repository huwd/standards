---
name: dependabot-pr-review
description: Review Dependabot pull requests in a repository. Use when asked to assess, triage, comment on, approve, merge, or decide what to do with one Dependabot PR, all open Dependabot PRs, or dependency update PRs.
---

# Dependabot PR Review

Review Dependabot PRs and give a clear verdict: what changed upstream, what could break, what the package touches in this codebase, and whether to merge, verify, investigate, or hold.

## Choose a Mode

- **Single-PR mode**: the user pasted or named one PR. Review that PR only.
- **Audit mode**: the user asks about all open Dependabot PRs, pending dependency updates, or says something like "check dependabot". Discover PRs with the GitHub API first, falling back to `gh`; do not ask for URLs.

If ambiguous, default to audit mode.


## GitHub Access Strategy

Prefer direct GitHub API calls over `gh` when available. Some sandboxed environments, such as sbx, attach GitHub credentials to `api.github.com` requests at the network layer without exposing a token to the agent. Do not ask the user for a token and do not print, read, or persist secrets.

First, ping the API:

```bash
curl -fsS https://api.github.com/user | jq -r .login
```

If this returns a login, use the API command set below. Make API requests with `curl` and GitHub headers, but without an `Authorization` header unless the environment already provides a safe wrapper. If the ping fails with 401/403/404 or network/auth errors, fall back to the `gh` command set.

Use `jq` for JSON parsing where available. If `jq` is unavailable, use `gh` fallback rather than ad hoc parsing for non-trivial responses.

### API command set

Determine the repository from the local remote URL:

```bash
git remote get-url origin
```

Normalize the result to `OWNER/REPO` before calling the API.

List open Dependabot PRs:

```bash
# Paginated: increment page=1,2,... until no results.
curl -fsS "https://api.github.com/repos/<OWNER>/<REPO>/pulls?state=open&per_page=100&page=1" \
  -H "Accept: application/vnd.github+json" \
  -H "X-GitHub-Api-Version: 2022-11-28" \
| jq -r ".[] | select(.user.login == \"dependabot[bot]\") | [.number, .title, .html_url, .created_at, .head.ref, .head.sha] | @tsv"
```

Fetch PR metadata, issue body/comments, and changed files:

```bash
curl -fsS "https://api.github.com/repos/<OWNER>/<REPO>/pulls/<NUMBER>" \
  -H "Accept: application/vnd.github+json" \
  -H "X-GitHub-Api-Version: 2022-11-28"

curl -fsS "https://api.github.com/repos/<OWNER>/<REPO>/issues/<NUMBER>" \
  -H "Accept: application/vnd.github+json" \
  -H "X-GitHub-Api-Version: 2022-11-28"

curl -fsS "https://api.github.com/repos/<OWNER>/<REPO>/pulls/<NUMBER>/files?per_page=100" \
  -H "Accept: application/vnd.github+json" \
  -H "X-GitHub-Api-Version: 2022-11-28"
```

Fetch the PR diff:

```bash
curl -fsS "https://api.github.com/repos/<OWNER>/<REPO>/pulls/<NUMBER>" \
  -H "Accept: application/vnd.github.v3.diff" \
  -H "X-GitHub-Api-Version: 2022-11-28"
```

Check CI using the PR head SHA:

```bash
curl -fsS "https://api.github.com/repos/<OWNER>/<REPO>/commits/<HEAD_SHA>/check-runs" \
  -H "Accept: application/vnd.github+json" \
  -H "X-GitHub-Api-Version: 2022-11-28" \
| jq -r ".check_runs[] | [.name, .status, .conclusion] | @tsv"

curl -fsS "https://api.github.com/repos/<OWNER>/<REPO>/commits/<HEAD_SHA>/status" \
  -H "Accept: application/vnd.github+json" \
  -H "X-GitHub-Api-Version: 2022-11-28"
```

Check for an existing review marker:

```bash
curl -fsS "https://api.github.com/repos/<OWNER>/<REPO>/issues/<NUMBER>/comments?per_page=100" \
  -H "Accept: application/vnd.github+json" \
  -H "X-GitHub-Api-Version: 2022-11-28" \
| jq -r ".[].body" | rg "dependabot-audit:v1"
```

Post a review comment only after explicit user approval:

```bash
jq -Rs "{body: .}" /tmp/dep-review-<NUMBER>.md \
| curl -fsS -X POST "https://api.github.com/repos/<OWNER>/<REPO>/issues/<NUMBER>/comments" \
    -H "Accept: application/vnd.github+json" \
    -H "X-GitHub-Api-Version: 2022-11-28" \
    -H "Content-Type: application/json" \
    --data-binary @-
```

Merge a safe PR with a merge commit:

```bash
jq -n "{merge_method: \"merge\"}" \
| curl -fsS -X PUT "https://api.github.com/repos/<OWNER>/<REPO>/pulls/<NUMBER>/merge" \
    -H "Accept: application/vnd.github+json" \
    -H "X-GitHub-Api-Version: 2022-11-28" \
    -H "Content-Type: application/json" \
    --data-binary @-
```

Request a Dependabot rebase:

```bash
printf "%s" "@dependabot rebase" \
| jq -Rs "{body: .}" \
| curl -fsS -X POST "https://api.github.com/repos/<OWNER>/<REPO>/issues/<NUMBER>/comments" \
    -H "Accept: application/vnd.github+json" \
    -H "X-GitHub-Api-Version: 2022-11-28" \
    -H "Content-Type: application/json" \
    --data-binary @-
```

### gh fallback command set

Use these commands only when the API ping fails or API access is unavailable:

```bash
# Determine repo
gh repo view --json nameWithOwner -q .nameWithOwner

# List open Dependabot PRs
gh pr list --author "app/dependabot" --state open \
  --json number,title,url,createdAt,headRefName,labels \
  --limit 50

# Fetch PR metadata, checks, and diff
gh pr view <NUMBER> --repo <OWNER/REPO> --json title,body,url,files,headRefName,createdAt,labels
gh pr checks <NUMBER> --repo <OWNER/REPO>
gh pr diff <NUMBER> --repo <OWNER/REPO>

# Check for an existing review marker
gh pr view <NUMBER> --repo <OWNER/REPO> --json comments --jq '.comments[].body' | rg 'dependabot-audit:v1'

# Post a review comment after explicit user approval
gh pr comment <NUMBER> --repo <OWNER/REPO> --body-file /tmp/dep-review-<NUMBER>.md

# Merge with a merge commit
gh pr merge <NUMBER> --repo <OWNER/REPO> --merge

# Request Dependabot rebase
gh pr comment <NUMBER> --repo <OWNER/REPO> --body "@dependabot rebase"
```

## Audit Workflow

1. Determine the repo using the selected command set: API first if the ping succeeded, otherwise `gh` fallback.

2. List open Dependabot PRs using the selected command set.

If none are open, say so and stop. If the number of open PRs is at or near `open-pull-requests-limit` in `.github/dependabot.yml`, call out queue saturation because it can block new updates, including security PRs.

3. Analyze each PR with the single-PR workflow. Fetch independent PRs in parallel where the harness allows.

4. Produce a consolidated report:

- Preamble: `Reviewed N open Dependabot PRs in <repo>.`
- Summary table, sorted by `Merge`, `Verify`, `Investigate`, `Hold`; within each bucket, oldest first. Security PRs go first regardless of bucket.
- Details section, one compact subsection per PR.
- Overall recommendation grouped by verdict, with related package families called out as sets.

Use this table shape:

```markdown
| # | Package | Bump | Type | Age | Verdict | Why |
|---|---------|------|------|-----|---------|-----|
| [#123](url) | package-name | 1.2.3 -> 1.2.4 | patch | 7d | Merge | dev-only patch |
```

Keep the `Why` column concrete and short. Do not add columns; put extra details below.

## Single-PR Workflow

### 1. Gather PR Details

Fetch PR metadata, CI status, changed files, and diff using the selected command set: API first if the ping succeeded, otherwise `gh` fallback.

Also read `.github/dependabot.yml`. Confirm a cooldown such as `cooldown: 7` exists for the relevant ecosystem. For routine updates, missing cooldown blocks auto-merge and should be flagged to @huwd.

Extract for each package:

- package name
- old version -> new version
- direct vs transitive dependency
- manifest or workspace touched
- dependency type, such as production, development, test, optional, peer, or toolchain
- bump type: patch, minor, major, or pre-1.0 minor, which should be treated as major-equivalent unless the ecosystem has stronger guarantees
- whether the PR is advisory-driven, from PR body, labels, CVE, GHSA, or security language

For grouped PRs, assess every package. If one package requires escalation or hold, the whole PR inherits that concern.

### 2. Apply Hard Gates

CI is a hard gate:

- Required CI failed: do not merge. Comment with the failing job and block reason.
- Required CI pending: wait; do not merge on partial green.
- Advisory-driven PR with CI passing: merge on the advisory fast path. Do not wait for cooldown.
- Advisory-driven PR with CI failing: block and tag @huwd.

Routine updates require cooldown. If cooldown is absent, do not merge automatically; ask for or raise a PR to add it and tag @huwd.

### 3. Review Upstream Changes

Read changelog, release notes, migration guide, or commit titles for the exact version range. Try sources in this order when relevant to the ecosystem:

1. PR body links from Dependabot.
2. GitHub releases or tags for the source repository.
3. Repository changelog files such as `CHANGELOG.md`, `HISTORY.md`, or package-specific changelogs.
4. Package registry metadata, such as npm, RubyGems, PyPI, crates.io, or equivalent.
5. Package diff tooling as a last resort, such as `npm diff`, `bundle info`, or ecosystem equivalent.

Prioritize findings in this order:

- breaking changes: removed or renamed APIs, changed defaults, changed return types, dropped runtime support, ESM/CJS or export-map changes, peer dependency tightening
- deprecations
- security fixes
- notable bug fixes or features relevant to this repo

If no changelog is findable, say so explicitly. Do not invent release notes.

### 4. Check Codebase Impact

Search actual usage before deciding risk. Prefer ecosystem-aware search, then broad text search:

- manifests and lockfiles for dependency scope
- imports, requires, configuration files, initializers, build config, CI config, and runtime entry points
- package family siblings that should move together
- transitive changes in lockfile diffs

For runtime or major updates, CI passing is not enough. Cross-reference breaking changes against actual usage and state either `affected` with file paths and required fix, or `not used here` with the search basis.

Flag extra scrutiny for auth, cryptography, network/HTTP, payment, database, framework/runtime, build system, deployment, and LLM/API client packages.

For related package families, avoid merging one PR while siblings remain stale or unreviewed. Common examples include React/React DOM, React Router packages, TypeScript/ESLint packages, Vitest/Playwright packages, Tailwind/plugin pairs, Storybook packages, Cloudflare/Wrangler packages, and similar ecosystem families discovered from the repo.

### 5. Assign a Verdict

Use these exact verdicts:

- **Merge**: CI passes, cooldown/advisory rule is satisfied, changelog is clean, codebase impact is low or well understood.
- **Verify**: likely safe, but a specific runtime/manual check is needed that CI may not cover.
- **Investigate**: human judgment is needed because risk or compatibility is unclear.
- **Hold**: breaking changes, failed/pending CI, missing cooldown for routine updates, ignore-rule violation, workspace split, package-family mismatch, or code changes needed first.

Risk guide:

| Factor | Lower risk | Higher risk |
|--------|------------|-------------|
| Bump | patch on stable version | major or pre-1.0 minor |
| Scope | dev/test/tooling only | production runtime |
| Usage | isolated or unused | widespread or hot path |
| Changelog | bug fixes only | API/default/runtime changes |
| Package | formatter/types | auth/crypto/network/framework/deploy |
| Lockfile | small, expected diff | broad transitive churn |
| Family | standalone or complete set | partial sibling bump |
| Security | no advisory | advisory, prioritize once CI passes |

Do not recommend running the full test suite as the main action; CI owns that. Instead, name the specific thing CI may not catch, such as runtime config, deployment behavior, generated types, browser hydration, local dev server behavior, external service compatibility, or deprecation warnings.

## Output Format

For one PR:

```markdown
## Dependabot Review: `<package>` (<old> -> <new>)

### Bump Type
[patch/minor/major/pre] - [one line about risk]

### What Changed
[Breaking changes first, then deprecations, security fixes, relevant bug fixes/features. If quiet: "No breaking changes or deprecations found."]

### Breaking Changes in This Codebase
[Only include if applicable: affected file plus concrete fix. Otherwise omit.]

### Codebase Impact
[Grouped list of touched areas, not an exhaustive file dump.]

### Recommendation
[Merge / Verify / Investigate / Hold] - [1-3 sentences with the reason and any specific check.]
```

For audit mode, keep each PR detail to roughly 15-25 lines and put the summary table first.

## Posting Findings to PRs

Always ask before posting. Never comment automatically.

- Single PR: `Want me to post this review as a comment on PR #<number>? (yes / no)`
- Audit mode: `Want me to post each PR review as a comment on its PR? (yes / no / selective)`

Use the selected command set. For API mode, write the comment to `/tmp/dep-review-<NUMBER>.md` and use the API comment command from the API command set. For `gh` fallback, use `--body-file` so markdown survives shell quoting:

```bash
gh pr comment <NUMBER> --repo <OWNER/REPO> --body-file /tmp/dep-review-<NUMBER>.md
```

Before posting, check for an existing review marker using the selected command set. For `gh` fallback:

```bash
gh pr view <NUMBER> --repo <OWNER/REPO> --json comments --jq '.comments[].body' | rg 'dependabot-audit:v1'
```

If a prior marker exists, ask whether to skip or repost. Default to skipping if the user does not specify.

Use this comment shape:

```markdown
## Dependabot review

**Verdict:** <Merge / Verify / Investigate / Hold>

<one-line reason>

<details>
<summary>Full review</summary>

<full per-PR review>

</details>

<!-- dependabot-audit:v1 -->
```

Do not add attribution or a generated-by signature. If commenting fails for one PR, report it and continue with the rest.

## Merge Behavior

When merging a safe PR, use a merge commit rather than squash so the Dependabot commit remains visible for audit history. Use the selected command set. For `gh` fallback:

```bash
gh pr merge <NUMBER> --repo <OWNER/REPO> --merge
```

After merging one Dependabot PR in a batch, remaining Dependabot PRs may need rebasing. Request it with the selected command set, then re-check CI before any further merge. For `gh` fallback:

```bash
gh pr comment <NUMBER> --repo <OWNER/REPO> --body "@dependabot rebase"
```

## Explicit Non-Goals

- Do not merge major version bumps automatically.
- Do not merge PRs with failed or pending required CI.
- Do not treat missing cooldown as acceptable for routine updates.
- Do not perform speculative compatibility analysis when changelog evidence suggests a breaking change; escalate or hold with concrete concerns.
- Do not post comments without explicit user approval.
