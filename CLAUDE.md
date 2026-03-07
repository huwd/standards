# CLAUDE.md — standards repo

This repo is a living collection of reusable standards, templates, and best practices for Huw's personal projects. It exists to reduce cycle time on new project bootstrapping and to keep conventions consistent across repos.

See `docs/plan.md` for the full plan and current status.

## Repo structure

```
standards/
  CLAUDE.md          # global Claude standard — symlink to ~/.claude/CLAUDE.md
docs/
  plan.md            # plan and progress for this repo
github/
  rulesets/          # importable GitHub ruleset JSON files
  workflows/         # reusable CI workflow templates
  dependabot.yml     # dependabot config template
```

## Working in this repo

This repo is primarily documentation and JSON templates. Changes are usually:

- **Distilling** — extracting a pattern that has proven useful in a concrete project
- **Generalising** — making a project-specific thing language/framework-agnostic
- **Adding** — new templates, rulesets, or workflow snippets

Before adding a standard, verify it against at least one real project. Don't invent conventions here — extract them.

When updating `standards/CLAUDE.md`, also update the symlink target so changes take effect globally:

```bash
ln -sf <repo_location>/standards/standards/CLAUDE.md ~/.claude/CLAUDE.md
```

## Commit standards

Follow the same conventions documented in `standards/CLAUDE.md`.

## Branching

Never commit directly to `main`. Branch naming: `<type>/<short-description>`.

```bash
git checkout -b docs/add-python-ci-template
gh pr create --base main
```
