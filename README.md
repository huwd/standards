# standards

A central collection of reusable standards, templates, and conventions for personal projects. The goal is to reduce bootstrapping time on new repos and keep conventions consistent across them.

Standards are extracted from real projects, not invented here.

## What's in here

```
standards/
  CLAUDE.md              # Global Claude Code instructions — symlink to ~/.claude/CLAUDE.md

docs/
  plan.md                # Background, decisions, and remaining work

github/
  rulesets/
    protect_main.json    # Importable ruleset: PR required, no force-push, conventional commits, status checks
    prevent_tag_deletion.json
  workflows/
    ci-rails.yml         # Rails CI template: Brakeman, importmap audit, Rubocop, RSpec + coverage
    ci-python.yml        # Python CI template: ruff, mypy, pytest + coverage
  dependabot.yml         # Weekly grouped Dependabot updates for language ecosystem + github-actions
```

## Using the GitHub rulesets

Import via the GitHub UI: **Settings → Rules → Rulesets → Import**.

- `protect_main.json` — enforces PR before merge, no force-push, no branch deletion, conventional commit message format, and required status checks. Edit the `required_status_checks` array to match your CI job names before importing.
- `prevent_tag_deletion.json` — prevents tags being deleted or force-pushed.

Both have `bypass_actors: []` — no one bypasses the rules, including admins.

## Using the CI workflow templates

Copy the relevant file to `.github/workflows/ci.yml` in your project. Each file has numbered comments at the top listing what to adapt.

Key things to check:
- Job names must match the `required_status_checks` in your `protect_main` ruleset exactly
- Coverage thresholds are set in `pyproject.toml` / `.simplecov`, not in the workflow
- Action versions are pinned to mutable tags here — pin to full commit SHAs in production repos for supply-chain safety

## Using the Dependabot template

Copy to `.github/dependabot.yml` and set the `package-ecosystem` value for your language (`bundler`, `pip`, `npm`, `cargo`, etc.). The template sets up weekly grouped updates with a cooldown, keeping minor/patch together and majors separate.

## Using the Claude standard

`standards/CLAUDE.md` is the global Claude Code instruction file. To use it:

```bash
ln -sf <path-to-this-repo>/standards/CLAUDE.md ~/.claude/CLAUDE.md
```

It covers commit conventions, branching, TDD workflow, CI requirements, and GitHub repo setup. Project-specific `CLAUDE.md` files can extend or override it.

## Bootstrapping a new repo

1. `git init`, initial commit with `.gitignore` and `README.md`
2. Create `CLAUDE.md` (copy and adapt from `standards/CLAUDE.md`)
3. Push to GitHub
4. Import rulesets from `github/rulesets/`
5. Add `.github/dependabot.yml`
6. Add CI workflow from `github/workflows/` (adapt job names to match ruleset)
7. Create a feature branch, open the first PR — verify the full gate runs
