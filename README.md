# Liner Skills

Agent skills for working with the [Liner API](https://liner.com/developers).

Skills are plain markdown instructions that coding agents read and follow.
They work with Claude Code, Cursor, Codex, OpenCode, and other agents that
support the [Agent Skills](https://agentskills.io) format.

## Skills

### `migrate-to-liner`

Migrates an existing OpenAI-compatible LLM integration to the Liner Model API.

Before changing any code it audits every call site for parameters Liner handles
differently, and shows a cost comparison computed from your actual token
volume. It stops and reports rather than migrating a call site that would break.

```bash
npx skills add liner-engineering/skills --skill migrate-to-liner
```

Then ask your agent:

```
Migrate this project to the Liner Model API
```

You can also use it without installing anything. Point your agent at
[the skill file](skills/migrate-to-liner/SKILL.md) and ask it to follow the
instructions there.

You will need a Liner API key from [platform.liner.com](https://platform.liner.com).

## Contributing

Issues and pull requests are welcome. If a skill gives you a wrong or unsafe
result, please open an issue with the input that produced it.

## License

MIT. See [LICENSE](LICENSE).
