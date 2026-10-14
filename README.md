# Funx Skill

> [!TIP]
> **把这个地址甩给你的 AI，然后开始爽!!!!** 🚀
>
> **Paste this URL to your AI and go.**
>
> ```
> https://github.com/funx-site/skill
> ```
>
> 让 Claude Code / Cursor / 任意支持 Skill 的 AI 读取本仓库，它就学会用 `funx` CLI 帮你把 Worker 部署到 [Funx](https://funx.demox.site)（`https://<name>.funx.run`）。
>
> Point Claude Code, Cursor, or any skill-compatible agent at this repo. It learns how to deploy and manage Funx workers with the `funx` CLI.

An [Agent Skill](https://docs.claude.com/en/docs/claude-code/skills) that teaches AI coding agents (Claude Code, Claude Agent SDK, and other skill-compatible runtimes) how to deploy and manage [Funx](https://funx.demox.site) workers using the `funx` CLI.

Naming and layout mirror [demox-site/skill](https://github.com/demox-site/skill). This repo is the public companion to [`funx-site/cli`](https://github.com/funx-site/cli) (`@funx-site/cli`).

## What it does

When the user asks to write, run, or deploy a Funx worker, the agent loads this skill and follows the documented workflow: the Workers `fetch` contract, `funx dev`, `funx deploy --name`, and vars/secrets. Login stays with the human. The agent does not complete it.

Contract details live in the [funx-site/funx README](https://github.com/funx-site/funx) and the console at https://funx.demox.site.

There is no `funx-content-scan` skill. Funx does not publish a public blocklist API yet.

## Layout

```
skills/
  funx/
    SKILL.md                 # deploy/manage workers with the funx CLI
```

## Install

### Claude Code

Copy the skill into your skills directory:

```bash
# Personal (all projects)
cp -r skills/funx ~/.claude/skills/funx

# Or project-scoped
cp -r skills/funx .claude/skills/funx
```

Then invoke it with `/funx`, or let the agent trigger it when you mention deploying a worker to Funx.

### Other runtimes

Any skill-compatible runtime can load `skills/funx/SKILL.md`. Point your skills loader at the `skills/` directory, or paste https://github.com/funx-site/skill and ask the agent to read it.

## The funx CLI

This skill drives [`@funx-site/cli`](https://github.com/funx-site/cli):

```bash
npm install -g @funx-site/cli
funx --help
```

## License

MIT
