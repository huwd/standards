# Standards Project — Plan

## Goal

Reduce cycle time on new project bootstrapping by maintaining a central, importable collection of standards. Agents starting a new repo can pull from here rather than rebuilding conventions from scratch each time.

## Problem being solved

Several personal projects (`learn_hanzi`, `audiobook-sync`, and others) have independently developed similar conventions for:

- Commit message format and content
- Branch naming and PR workflow
- TDD red-green-refactor cycle
- Verification loop (lint, tests, coverage, security scan)
- CI/CD pre-merge gates
- GitHub branch protection rulesets
- Dependabot configuration

Each new repo currently repeats this bootstrapping work. This repo captures the distilled, reusable version.

## Source repos

Standards have been extracted from:

- `../learn_hanzi` — Rails 8, RSpec, Rubocop, Brakeman, GitHub Actions CI, GitHub rulesets
- `../audiobook-sync` — Python 3.12, pytest, ruff, mypy, uv, ports-and-adapters architecture

---

## Repo structure (target)

```
standards/
  CLAUDE.md              # global standard — symlink to ~/.claude/CLAUDE.md

docs/
  plan.md                # this file

github/
  rulesets/
    protect_main.json    # require PR + status checks + conventional commits on default branch
    prevent_tag_deletion.json
  workflows/
    ci-rails.yml         # Rails CI template (lint, security, test, coverage)
    ci-python.yml        # Python CI template (ruff, mypy, pytest --cov)
  dependabot.yml         # weekly grouped dependabot for bundler + github-actions
```

---

## Standards captured

### Commits

- Conventional Commits format: `<type>[optional scope]: <description>`
- Types: `feat fix docs style refactor perf test build ci chore`
- Subject: max 50 chars, imperative mood, no trailing period
- Body: wrapped at 72 chars, explains _why_ (not what), notes alternatives considered
- Each commit is a self-contained logical unit
- History is tidied on feature branches before PR

### Branching

- Never commit directly to `main`
- Branch naming: `<type>/<issue-number>-<short-description>` (issue number optional)
- PR against `main`; branch is deleted after merge
- Ruleset enforces: no deletion of main, no force-push, PR required, conventional commit regex, required status checks

### TDD workflow

1. Write failing tests (red) — verify lint passes, tests fail — commit
2. Implement (green) — verify tests pass — commit
3. Refactor — tests still green — commit
4. Type-check (if typed language) — commit
5. Lint/format — commit

### Verification loop (pre-merge)

All of these must pass in CI before merge:

| Check         | Rails                     | Python                                                |
| ------------- | ------------------------- | ----------------------------------------------------- |
| Lint          | `bin/rubocop -f github`   | `uv run ruff check . && uv run ruff format --check .` |
| Type check    | —                         | `uv run mypy src/`                                    |
| Security scan | `bin/brakeman --no-pager` | —                                                     |
| Tests         | `bundle exec rspec`       | `uv run pytest`                                       |
| Coverage      | SimpleCov artifact upload | `uv run pytest --cov` artifact upload                 |

Coverage targets: ≥80% line, ≥75% branch.

### GitHub rulesets

Import `github/rulesets/protect_main.json` and `github/rulesets/prevent_tag_deletion.json` via the GitHub API or UI.

`protect_main` enforces:

- No deletion of the default branch
- No force-push
- PR required before merge
- Conventional commits regex on commit messages
- Required status checks must pass (CI jobs named in the ruleset)

### Dependabot

Weekly grouped updates (minor + patch together, major separate) for both the language package ecosystem and `github-actions`.

---

## Remaining work

- [x] Extract and generalise `ci-python.yml` workflow from `audiobook-sync` patterns
- [x] Extract and generalise `ci-rails.yml` workflow from `learn_hanzi`
- [x] Copy and generalise `dependabot.yml` template
- [x] Verify ruleset JSONs are importable as-is (actor IDs may need adjusting per org)
- [x] Document how to bootstrap a new repo using this collection
- [x] Confirm symlink setup
- [x] Add `standards/CLAUDE.md` reference back to this repo URL once it has a remote

---

## Bootstrap checklist for a new repo

When starting a new project, work through these in order:

1. `git init`, initial commit with `.gitignore` and `README.md`
2. Create `CLAUDE.md` (copy and adapt from `standards/CLAUDE.md`, add project-specific commands)
3. Push to GitHub, set repo to private if appropriate
4. Import GitHub rulesets from `github/rulesets/`
5. Copy `.github/dependabot.yml` (update `package-ecosystem` for the language)
6. Add CI workflow from `github/workflows/` (adapt job names to match ruleset `required_status_checks`)
7. Create first feature branch, open first PR — verify the full gate runs
