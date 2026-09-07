# liner-engineering/skills

Agent skills shared across the team. Compatible with Claude Code, Cursor, Codex, and any agent following the [Agent Skills spec](https://agentskills.io/specification).

## Install

```bash
npx skills add liner-engineering/skills --list                                  # see what's here
npx skills add liner-engineering/skills                                         # pick skills + agents interactively
npx skills add liner-engineering/skills --skill example-skill -a claude-code -y # one skill, one agent
```

## Add a skill

1. Create `skills/<name>/SKILL.md`. `name` must equal the directory name: lowercase kebab-case, max 64 chars.
2. `description` says what it does and when to use it (max 1024 chars). Nothing else is required.
3. Optional support files go in `scripts/`, `references/`, `assets/` next to `SKILL.md`.
4. Check discovery: `npx skills add . --list`

Start from `skills/example-skill/`. How the CLI discovers and installs skills: [docs/research/npx-skills-add-repo-format.md](docs/research/npx-skills-add-repo-format.md).
