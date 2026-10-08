# AGENTS.md

This repo packages a single Agent Skill. It contains no application code.

```
README.md                                             landing page and install instructions
.github/skills/bootc-conversion/SKILL.md              the conversion procedure
.github/skills/bootc-conversion/references/           loaded on demand by the skill
```

When editing:

- `SKILL.md` is the procedure; `references/bootc-reference.md` is the manual. Keep facts
  in the reference and cite them from the skill rather than duplicating.
- Reference files must stay **inside** the skill folder and be linked from `SKILL.md`
  with relative paths. A file that is never referenced is never loaded, and a path that
  escapes the skill folder breaks once the folder is copied somewhere else.
- The skill directory name must match the `name:` in the frontmatter.
- `description:` is the only text an agent sees when deciding whether to load the skill.
  Update it whenever the skill's scope changes. Max 1024 characters.
