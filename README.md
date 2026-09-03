# Dooplex Agent Skills

Reusable agent skills for managing [Dooplex](https://dooplex.ai) creator accounts.

## Available skills

| Skill | Description |
| --- | --- |
| [`dooplex-cli`](skills/dooplex-cli/SKILL.md) | Manage an AI twin through the authenticated `@dooplex/cli`: configuration, prompts, knowledge, corpus, playbooks, conversations, channels, scheduled drops, content, and public pages. |

## Install

Install interactively for any supported agent:

```bash
npx skills add dooplex-ai/skills --skill dooplex-cli
```

Install globally for Claude Code:

```bash
npx skills add dooplex-ai/skills --skill dooplex-cli --global --agent claude-code --yes
```

Install globally for Codex:

```bash
npx skills add dooplex-ai/skills --skill dooplex-cli --global --agent codex --yes
```

The skill uses the public `@dooplex/cli` npm package and requires Node.js 20 or newer. Authenticate once before managing an account:

```bash
npx @dooplex/cli login
```

## Security

Authentication tokens are saved locally by `@dooplex/cli` and are never included in this repository. Review the skill before installation and keep `~/.dooplex/credentials.json` and `DOOPLEX_CLI_TOKEN` private.

## License

MIT
