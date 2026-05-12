# NetBox Skills — Maintenance Guide

## Repo Structure

```
skills/
  netbox/              # Hub skill — entry point, decision trees
    references/        # Shared references (product catalog, version matrix)
  netbox-*/            # 15 specialized skills
    SKILL.md           # Core instructions (< 500 lines)
    references/        # Detailed docs loaded on demand
    scripts/           # Optional executable code
commands/              # Agent commands (setup-mcp, build-plugin)
.claude-plugin/        # Claude Code plugin metadata
.cursor-plugin/        # Cursor plugin metadata
```

## Skill Conventions

### Frontmatter Format

Every SKILL.md must have exactly three frontmatter fields:

```yaml
---
name: skill-directory-name
description: >
  One-paragraph description. Use when [trigger conditions].
license: Apache-2.0
---
```

- `name` must match the directory name exactly
- No `metadata`, `version`, `compatibility`, or other fields
- Description should include "Use when..." trigger conditions

### Content Guidelines

- **500-line limit** for SKILL.md — move details to `references/`
- Start with a "When to Use This Skill" or decision tree section
- Include a Quick Reference section near the top
- Use relative links between skills: `[skill-name](../skill-name/SKILL.md)`
- Reference files use progressive disclosure — only loaded when needed

### Writing Style

- Direct, imperative tone — "Use X when Y" not "You might want to use X"
- Code examples over prose explanations
- Tables for quick lookups, prose for workflows
- No marketing language — technical accuracy first

## Adding a New Skill

1. Create `skills/<name>/SKILL.md` with the three required frontmatter fields
2. Add a `references/` directory if the skill needs supplementary docs
3. Add the skill to the hub (`skills/netbox/SKILL.md`) — both decision tree and index table
4. Add the skill to `README.md` skills table
5. Add to `AGENTS.md` skills table
6. Verify `name` field matches directory name

## Adding a Command

1. Create `commands/<name>.md` with `name` and `description` frontmatter
2. Add to the commands table in `README.md`
3. Include clear step-by-step guidance with example invocations

## Cross-References

- Skills reference each other via relative paths: `../skill-name/SKILL.md`
- Reference files within a skill: `references/filename.md`
- Shared references live in `skills/netbox/references/`
