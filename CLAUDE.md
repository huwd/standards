# CLAUDE.md — standards repo

This repo is a living collection of reusable standards, templates, and best practices for Huw's personal projects. It exists to reduce cycle time on new project bootstrapping and to keep conventions consistent across repos.

See `docs/plan.md` for the full plan and current status.

## Repo structure

```
standards/
  CLAUDE.md                    # global Claude standard — symlink to ~/.claude/CLAUDE.md
docs/
  plan.md                      # plan and progress for this repo
skills/
  SKILLS.md                    # skill authoring, compatibility, and symlink conventions
  dependabot-pr-review/        # portable Dependabot PR review skill
  do-release/                  # portable package release workflow skill
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

Cross-harness skills should be authored under `skills/` using the Agent Skills
directory format, then symlinked into each tool-specific skills directory; see
`skills/SKILLS.md`.

Global Claude config files in this repo are made available to all projects via
symlinks into `~/.claude/`. When adding a new global file, create the symlink
and commit the command here. Current symlinks:

```bash
# Global Claude standard
ln -sf ~/Projects/Personal/standards/standards/CLAUDE.md ~/.claude/CLAUDE.md

# Dependabot PR review skill
ln -sfn ~/Projects/Personal/standards/skills/dependabot-pr-review ~/.claude/skills/dependabot-pr-review
ln -sfn ~/Projects/Personal/standards/skills/dependabot-pr-review ~/.codex/skills/dependabot-pr-review
ln -sfn ~/Projects/Personal/standards/skills/dependabot-pr-review ~/.agents/skills/dependabot-pr-review

# Release workflow skill
ln -sfn ~/Projects/Personal/standards/skills/do-release ~/.claude/skills/do-release
ln -sfn ~/Projects/Personal/standards/skills/do-release ~/.codex/skills/do-release
ln -sfn ~/Projects/Personal/standards/skills/do-release ~/.agents/skills/do-release
```

## Commit standards

Follow the same conventions documented in `standards/CLAUDE.md`.

## Branching

Never commit directly to `main`. Branch naming: `<type>/<short-description>`.

```bash
git checkout -b docs/add-python-ci-template
gh pr create --base main
```
