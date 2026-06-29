# Tekkstudio Claude Code Skills

Claude Code slash commands published alongside our build series blog posts.
Each skill scaffolds a complete, production-ready architecture from a single command.

---

## Available skills

| Skill | Blog post | What it builds |
|-------|-----------|----------------|
| [`/build-chat-widget`](./build-chat-widget.md) | Build Series — Post 1 | Multi-tenant AI chat widget: AppSync + Lambda + DynamoDB + Bedrock KB |

---

## Install a skill

```bash
curl -o ~/.claude/commands/build-chat-widget.md \
  https://raw.githubusercontent.com/tekkstudio/skills/main/build-chat-widget.md
```

Then run it in any Claude Code session:

```
/build-chat-widget
```

---

## How skills work

Each file is a Claude Code slash command — a markdown file that Claude reads as instructions
when you invoke it. Skills generate complete project scaffolding (IaC, Lambda handlers, widget
files, scripts, README) directly into your working directory. Nothing is deployed automatically;
you run the deploy commands the skill produces.

More skills added with each post in the series.
