# Global Claude Standard

This file applies to all of Huw's personal projects. It is symlinked from `~/.claude/CLAUDE.md`.

Source of truth: `standards/standards/CLAUDE.md`

Project-specific `CLAUDE.md` files extend or override these defaults.

---

## Commit standards

Format: `<type>[optional scope]: <description>`

**Types**: `feat` `fix` `docs` `style` `refactor` `perf` `test` `build` `ci` `chore`

**Subject line**:

- Max 50 characters
- Imperative mood ("Add feature" not "Added feature")
- No trailing period
- Breaking changes: `feat!:` or `BREAKING CHANGE:` footer

**Body** (required for anything non-trivial):

- Separated from subject by a blank line
- Wrapped at 72 characters
- Answer three questions:
  - Why is this change necessary?
  - How does it address the issue?
  - What side effects does it have?
- Capture the _why_ — the code shows _how_, but rationale is hard to reconstruct later
- Note alternatives considered if choosing approach A over B

**Structure**:

- Each commit is a self-contained logical unit — avoid needing "and" in the subject
- Commits should tell a coherent narrative through history
- Tidy feature branch history before opening a PR

References: [Conventional Commits](https://www.conventionalcommits.org/), [GDS Git conventions](https://gds-way.digital.cabinet-office.gov.uk/standards/source-code/working-with-git.html#commits), [Joel Chippindale — Telling Stories Through Your Commits](https://blog.mocoso.co.uk/posts/talks/telling-stories-through-your-commits/)

---

## Branching

Never commit directly to `main`. Always work on a branch and open a PR.

Branch naming: `<type>/<issue-number>-<short-description>` (issue number optional if no tracker)

```bash
git checkout -b feat/42-add-user-auth
git push -u origin feat/42-add-user-auth
gh pr create --base main
```

Valid prefixes match commit types: `feat/` `fix/` `chore/` `ci/` `test/` `docs/` `refactor/`

---

## Development workflow (TDD)

Follow a strict red-green-refactor cycle. Commit between each step.

1. **Red** — write failing tests that define the expected behaviour. Verify lint passes, tests fail on the new cases. Commit and push.
2. **Green** — write minimal code to make tests pass. Verify tests pass. Commit and push.
3. **Refactor** — clean up while keeping tests green. Commit and push.
4. **Type-check** (typed languages) — add type annotations, verify type checker passes. Commit.
5. **Lint/format** — run linter and formatter, fix issues. Commit.

---

## Verification loop

Before a PR can merge, all of the following must pass in CI:

- **Lint** — style and formatting checks
- **Type checking** — where the language supports it (mypy, TypeScript, etc.)
- **Security scan** — where tooling exists (brakeman for Rails, etc.)
- **Tests** — full test suite
- **Coverage** — minimum 80% line coverage, 75% branch coverage

Run the full local check before pushing (see project-specific `CLAUDE.md` for the exact commands).

---

## Mutation testing

Run after achieving the coverage targets, to find tests that pass for the
wrong reasons. See `docs/mutation-testing.md` for the full guide.

- Tool (Python): `mutmut` — configure in `setup.cfg`
- Mutate core logic modules only; exclude infrastructure boundaries
- Target ≥ 65% overall kill rate on core modules
- Not a blocking CI gate — run via `workflow_dispatch` for on-demand snapshots
- Annotate equivalent mutations with `do_not_mutate` rather than chasing them

---

## CI/CD (GitHub Actions)

- CI runs on `pull_request` and `push` to `main`
- All jobs must pass before merge (enforced via GitHub ruleset `required_status_checks`)
- Job names in the workflow must match the names listed in the branch protection ruleset
- Pin action versions to a full commit SHA (not a mutable tag) for supply-chain safety
- Upload coverage as an artifact on every run

---

## GitHub repo setup

On a new repo, apply these in order:

1. Import `protect_main.json` ruleset — enforces PR, no force-push, conventional commits regex, required status checks
2. Import `prevent_tag_deletion.json` ruleset
3. Add `.github/dependabot.yml` — weekly grouped updates for the package ecosystem and `github-actions`

Ruleset JSON files live in `standards/github/rulesets/`.

---

## Security

- Never commit secrets, tokens, or credentials — use environment variables or volume-mounted files
- Never log credential values
- Sensitive config files should be `chmod 600` or `700`
- If a tool produces `.env` files, add them to `.gitignore` before the first commit

---

## This standards repo

Full plan and bootstrap checklist: `standards/docs/plan.md`
