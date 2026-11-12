# claude-skills

Claude Code skills published to [Satellite](https://github.com/kcontreras-csm/satellite)'s skills store
(**Settings > Skills**). Satellite installs them into `~/.claude/skills/<id>`, where Claude Code picks them up.

## Add a skill

Create a top-level folder named with the skill's id (lowercase letters, digits and hyphens), containing a
`SKILL.md` and any files it needs:

    my-skill/
      SKILL.md
      reference/notes.md

`SKILL.md` starts with YAML frontmatter giving the skill's name, description and version:

```markdown
---
name: My Skill
description: What it does and when Claude should use it.
version: 1.0.0
---

# My Skill
…instructions…
```

Push to `main`. The **Skills** GitHub Action runs `scripts/generate_skills.py`, which validates every skill and
regenerates `skills.json` — the index Satellite reads, listing each skill with its files, their git blob SHAs
(downloads are verified against them) and sizes. Don't edit `skills.json` by hand. Raise a skill's `version` to
publish an update; Satellite then offers **Update** for it.

## Locally

    python3 scripts/generate_skills.py           # validate and write skills.json
    python3 scripts/generate_skills.py --check   # validate only; fail if skills.json is out of date

Limits: up to 300 skills, 100 files per skill, 2 MB per file, 20 MB per skill. The repository must be public
(Satellite fetches raw files with no token).
