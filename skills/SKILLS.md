# Skills

This directory is the canonical home for reusable agent skills that should work across Codex, Claude Code, and OpenCode where possible.

## Structure

Each skill lives in its own directory and uses the Agent Skills shape:

```text
skills/
  SKILLS.md
  <skill-name>/
    SKILL.md
    scripts/       # optional executable helpers
    references/    # optional supporting docs
    assets/        # optional templates or static resources
```

The directory name and the `name` frontmatter value in `SKILL.md` must match. Use lowercase letters, numbers, and single hyphens only:

```yaml
---
name: example-skill
description: Does a specific repeatable workflow. Use when the user asks for that workflow or mentions its inputs.
---
```

Keep shared skill frontmatter conservative for cross-harness compatibility:

- `name`
- `description`
- `license`, if useful
- `compatibility`, only when the skill has real environment requirements
- `metadata`, only for stable non-runtime notes

Avoid tool-specific features in shared skills unless the skill is explicitly marked as tool-specific. Put detailed references in `references/` instead of making `SKILL.md` huge.

## Compatibility

Shared skills should assume they may be loaded by different agent harnesses:

- **Codex** reads skills from the local Codex skills tree in this environment.
- **Claude Code** reads personal skills from `~/.claude/skills/`.
- **OpenCode** reads skills from `~/.config/opencode/skills/`, `~/.claude/skills/`, and the vendor-neutral `~/.agents/skills/`.

Write instructions in terms of ordinary files, commands, and repository state. Do not rely on a particular model, subagent, tool name, permission mode, or dynamic context feature unless the skill documents that requirement in `compatibility`.

## Symlink Strategy

Symlink whole skill directories, not only `SKILL.md`, so any `scripts/`, `references/`, or `assets/` travel with the skill.

For a skill at:

```text
~/Projects/Personal/standards/skills/<skill-name>
```

create links like:

```bash
# Vendor-neutral location supported by OpenCode and likely future harnesses
ln -sfn ~/Projects/Personal/standards/skills/<skill-name> ~/.agents/skills/<skill-name>

# Claude Code
ln -sfn ~/Projects/Personal/standards/skills/<skill-name> ~/.claude/skills/<skill-name>

# Codex
ln -sfn ~/Projects/Personal/standards/skills/<skill-name> ~/.codex/skills/<skill-name>

# OpenCode native location, optional if ~/.agents/skills is already used
ln -sfn ~/Projects/Personal/standards/skills/<skill-name> ~/.config/opencode/skills/<skill-name>
```

Create parent directories first when needed:

```bash
mkdir -p ~/.agents/skills ~/.claude/skills ~/.codex/skills ~/.config/opencode/skills
```

Restart the agent session after adding or changing symlinks. Some tools watch skill files live, but restart is the reliable baseline.

## Agents Are Separate

Do not put tool-specific subagent definitions in this directory. Claude Code and OpenCode use similar Markdown files for agents, but their frontmatter and runtime semantics differ enough that they should have separate wrappers.

Keep portable prompts or procedures here as skills. Keep harness-specific agent definitions in a separate `agents/` tree if and when this repo needs one.
