---
name: funx
description: Deploy and manage Funx workers with the funx CLI (@funx-site/cli). Use when the user wants to write, locally run, or deploy a Worker that exports fetch, or to set vars and secrets on https://<name>.funx.run. Covers funx dev, funx deploy --name, and human-only login. Triggers include "funx", "funx.run", "deploy a worker", "funx dev", and "funx deploy".
---

# Funx CLI

The `funx` CLI deploys a Cloudflare Workers-style worker and serves it at `https://<name>.funx.run`.

Command reference: https://github.com/funx-site/cli (`@funx-site/cli`).
Worker contract: https://github.com/funx-site/funx#writing-a-worker and the console at https://funx.demox.site.
If `funx --help` disagrees with this file, follow the installed CLI and those two docs.

This skill is only for the user's own workers. Do not deploy or operate the platform (funx-api, router, runtime, certificates, STS, or deploy keys). Those stay in funx-site/funx for internal agents.

## Setup

The package name is planned as `@funx-site/cli`. It is **not published to npm yet**. The CLI requires Node.js >= 20. That is the CLI engine. The worker runtime still exposes Node 18 globals (see the table below); the CLI does not run on Node 18.

Until it is on npm, install from https://github.com/funx-site/cli with `npx` or `npm link`:

```bash
npx github:funx-site/cli --help

# or clone and link the bin
git clone https://github.com/funx-site/cli
cd cli && npm install && npm link
```

After it is published:

```bash
npm i -g @funx-site/cli
```

Verify it before relying on it:

```bash
funx --help
```

If `funx` is not found, use the `npx` or `npm link` form above for every command below. If a flag in this skill is rejected, `--help` wins.

## Authentication (human only)

Deploy, list, delete, vars, and secrets need a logged-in user. **The agent cannot log in.**

1. If a command says the user is not logged in, or `funx --help` shows a login command, **ask the human to sign in themselves**.
2. The console is https://funx.demox.site. If `funx --help` lists `login`, the human runs `funx login` in their own terminal (in Claude Code, suggest they type `! funx login`). Do not run it unattended and do not wait on a browser prompt.
3. Do not ask for, type, or store passwords, tokens, session keys, or cloud credentials. Do not set operator environment variables to "help" login succeed.
4. After the human confirms they are logged in, continue with dev, deploy, or binding commands.

`funx dev` is local. It does not need a login.

## Writing a worker

Same contract as Cloudflare Workers. Prefer this shape:

```js
export default {
  async fetch(request, env, ctx) {
    return Response.json({ path: new URL(request.url).pathname, hi: env.GREETING });
  },
};
```

| Workers | Funx |
| --- | --- |
| `export default { fetch(request, env, ctx) }` | yes (ESM `.mjs`/`.js`, or CommonJS `module.exports = { fetch }`) |
| `addEventListener('fetch', e => e.respondWith(...))` | yes (vars/secrets also exposed as globals) |
| `Request`, `Response`, `Headers`, `URL`, streams, `crypto.subtle`, `Response.json` | Node 18 globals |
| `env.VAR`, `env.SECRET` | `funx var set` / `funx secret put` (strings) |
| `ctx.waitUntil(p)` | awaited before the response finishes, max 5 s |
| `ctx.passThroughOnException()` | no-op; a thrown exception becomes 500 |
| `wrangler.toml` `main = "src/index.js"` | honoured as the entry |
| `.dev.vars` | read by `funx dev` only; never packaged |
| KV, D1, R2, Durable Objects, Cron, `request.cf` | not available |

Entry resolution: `package.json` `"main"`, then `wrangler.toml` `main`, then `index.mjs` / `index.js` / `worker.js` / `src/index.js`. The packager writes `funx.json`. Do not add `funx.json`, `funx_entry.js`, `funx_entry.cjs`, or `scf_bootstrap` yourself.

The older `exports.handler = async (event) => ({ statusCode, headers, body })` form still runs. New workers should use `fetch`.

Public URL after deploy: `https://<name>.funx.run`.

### Name rules

`<name>` is 3–32 characters of `a-z`, `0-9`, and `-`, and must not start or end with `-`. Reserved: `api`, `auth`, `www`, `admin`, `app`, `console`, `status`, `mail`.

### What gets packaged

`funx deploy` zips the worker directory.

- `node_modules` is included as-is. Keep the zip **≤ 6 MB**.
- Never packaged: `.env*`, `.dev.vars`, `*.pem`, `*.key`, `.git`, and private key filenames. Do not commit `.dev.vars`.
- Limits from the Funx README: request ≤ 1 MB, response ≤ 6 MB, user function timeout 10 s. Re-check https://github.com/funx-site/funx if a deploy error quotes a different limit.

## Local dev

```bash
funx dev [dir|file] [--port 8787] [--var KEY=VALUE ...]
```

- Default URL is `http://localhost:8787`. It reloads when files in the directory change.
- `.dev.vars` in the worker directory supplies local vars and secrets (`KEY=value`, optional quotes, optional `export`).
- `--var KEY=VALUE` overrides `.dev.vars` for that process.
- No cloud account is required. Do not put real production secrets in the repo; `.dev.vars` stays local.

```bash
funx dev ./worker --port 8787 --var GREETING=hello
```

## Deploy

```bash
funx deploy <dir> --name <name>
```

`<dir>` is the worker directory (the one that contains the entry file). `--name` is required. Deploying the same name again updates that worker. On success the CLI prints the public URL. Report that URL to the user.

```bash
funx deploy ./worker --name hello
# https://hello.funx.run
```

Check the name rules and the 6 MB zip limit before deploying. Point `deploy` at the worker directory, not a parent folder full of unrelated dependencies.

## Vars and secrets

Bindings show up as `env.NAME` inside `fetch`. Names are letters, digits, and `_`, not starting with a digit, at most 64 characters. Do not use platform prefixes (`FUNX_`, `SCF_`, `TENCENTCLOUD_`, and similar). Each value is at most 2048 bytes; all bindings together must fit in 4 KB.

Plain vars (values are not secret; the user can see them again):

```bash
funx var set <name> <KEY> <value>
funx var delete <name> <KEY>
```

Secrets (the value is never printed back):

```bash
funx secret list <name>          # names only
funx secret delete <name> <KEY>
funx secret put <name> <KEY>     # reads the value from stdin
```

**Do not put a secret on the command line, in chat, or in a committed file.** Ask the human to run `funx secret put` in their own terminal so the value never enters the transcript. `funx secret list` is safe: it prints names only.

For local runs, write the same keys into `.dev.vars` (gitignored) or pass `--var`. `.dev.vars` is not uploaded.

## Managing workers

```bash
funx list
funx delete <name>
funx disable <name>    # that worker's public URL returns 403
funx enable <name>     # undo disable
```

`funx delete` removes the worker. Confirm with the user before deleting. Do not invent extra flags; if the installed CLI prompts, ask the human to answer it.

Only manage workers the logged-in user owns. Stop if a command requires platform credentials.

## Typical workflow

1. Confirm the human is logged in before any cloud command. `funx dev` can start before that.
2. Write `export default { async fetch(request, env, ctx) { ... } }`.
3. `funx dev <dir>` and hit `http://localhost:8787`.
4. Check the name and that the directory will zip to ≤ 6 MB.
5. `funx deploy <dir> --name <name>`.
6. Report `https://<name>.funx.run`.
7. Set plain config with `funx var set`. Ask the human to run `funx secret put` for anything secret.

## Troubleshooting

- **Not logged in / auth error** — stop. Ask the human to sign in at https://funx.demox.site or to run `funx login` themselves if `funx --help` lists it.
- **Bad name** — 3–32 of `a-z`, `0-9`, `-`, no leading or trailing hyphen, not a reserved label.
- **Entry file not found** — add `index.js` / `index.mjs` / `worker.js` / `src/index.js`, or set `package.json` `"main"` / `wrangler.toml` `main` to a file that exists.
- **Package too large** — zip must be ≤ 6 MB. `node_modules` counts.
- **`funx dev` exits immediately** — the entry threw while loading, or the directory path is wrong.
- **Command hangs on login** — the human still has to finish sign-in. Do not keep waiting.
- **`funx: command not found`** — `@funx-site/cli` is not on npm yet. Use `npx` or `npm link` from https://github.com/funx-site/cli. After publish: `npm i -g @funx-site/cli`. The CLI needs Node.js >= 20.
- **A flag in this skill is rejected** — run `funx --help` and follow https://github.com/funx-site/cli.

## Out of scope

Do not run or document platform commands: `router-deploy`, `cert-rotate`, `runtime-build`, `runtime-deploy`, `runtime-secrets`, `api-build`, `keygen`, `jwt-keygen`, `user-hash`, `set-origin`, `rotate-key`. Do not request STS keys or a deploy key. Funx has no public blocklist API; do not invent a content-scan call.
