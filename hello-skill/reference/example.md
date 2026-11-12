# Example reference file

A skill can ship extra files alongside its `SKILL.md` — reference docs, scripts, templates. They install into
`~/.claude/skills/hello-skill/` next to the `SKILL.md`, and the generator lists every one in `skills.json` with
its git blob SHA so Satellite verifies the download.
