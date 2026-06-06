# CLAUDE.md — standards repo

This repo is a living collection of reusable standards, templates, and best practices for Huw's personal projects. It exists to reduce cycle time on new project bootstrapping and to keep conventions consistent across repos.

See `docs/plan.md` for the full plan and current status.

## Repo structure

```
standards/
  CLAUDE.md                    # global Claude standard — symlink to ~/.claude/CLAUDE.md
docs/
  plan.md                      # plan and progress for this repo
  review-dependabot-pr.md      # Dependabot PR review skill — symlink to ~/.claude/review-dependabot-pr.md
github/
  rulesets/                    # importable GitHub ruleset JSON files
  workflows/                   # reusable CI workflow templates
  dependabot.yml               # dependabot config template
```

## Working in this repo

This repo is primarily documentation and JSON templates. Changes are usually:

- **Distilling** — extracting a pattern that has proven useful in a concrete project
- **Generalising** — making a project-specific thing language/framework-agnostic
- **Adding** — new templates, rulesets, or workflow snippets

Before adding a standard, verify it against at least one real project. Don't invent conventions here — extract them.

Global Claude config files in this repo are made available to all projects via
symlinks into `~/.claude/`. When adding a new global file, create the symlink
and commit the command here. Current symlinks:

```bash
# Global Claude standard
ln -sf ~/Projects/Personal/standards/standards/CLAUDE.md ~/.claude/CLAUDE.md

# Dependabot PR review skill
ln -sf ~/Projects/Personal/standards/docs/review-dependabot-pr.md ~/.claude/review-dependabot-pr.md
```

## Commit standards

Follow the same conventions documented in `standards/CLAUDE.md`.

## Branching

Never commit directly to `main`. Branch naming: `<type>/<short-description>`.

```bash
git checkout -b docs/add-python-ci-template
gh pr create --base main
```
