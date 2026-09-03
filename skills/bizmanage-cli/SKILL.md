---
name: bizmanage-cli
description: Use the bizmanage CLI to pull/push customizations (objects, fields, actions, reports, pages, backend scripts) between the local filesystem and any Bizmanage SaaS instance. Use when the user asks to log in to Bizmanage, initialize a Bizmanage project, pull or push customizations, validate metadata, check sync status, or troubleshoot bizmanage-cli commands. Source: https://github.com/bizmanagrcrm/bizmanage-cli
---

# Bizmanage CLI

A CLI for syncing customizations (objects, fields, actions, reports, pages, backend
scripts) between a local git repo and a Bizmanage instance (any `https://<subdomain>.bizmanagecrm.com`).
`src/` in the project is the source of truth: `pull` writes platform state into it,
you edit/commit in git, `push` deploys it back.

Install: `npm install -g bizmanage-cli` (or `npx bizmanage-cli <cmd>`).

## Core workflow

```bash
bizmanage login              # save API key, test auth (GET /restapi/ping)
bizmanage pull --init        # pull platform state, init project structure if needed
# edit files under src/, then:
git add src bizmanage.config.json
git commit -m "..."
bizmanage push                # validates, tests, then deploys changed files
```

Project layout after `pull`/`init`:
```
src/
├── objects/<view-name>/definition.json, fields/, actions/, data/
├── backend/ (backend-scripts)
├── reports/  (.sql + meta.json)
└── pages/    (.html + meta.json)
bizmanage.config.json   # commit this
.bizmanage/             # local cache/state — gitignore this (esp. file-hashes.json)
```

## Commands

- **`bizmanage login [-a, --alias <alias>]`** — prompts for instance URL + API key, tests auth, saves key. Use different `--alias` values for multiple environments/instances (e.g. `production`, `staging`, `local`).
- **`bizmanage logout [-a, --alias <alias>] [--all]`** — removes saved config for an alias, or `--all` for everything.
- **`bizmanage init [project-name] [-a <alias>] [--force]`** — scaffolds a new project (`src/objects|backend|reports|pages`, `bizmanage.config.json`, `.gitignore`, `.bizmanage/`).
- **`bizmanage create-view <name> [-p <path>] [--display-name <name>] [--force]`** — scaffolds `src/objects/<name>/definition.json` plus empty `fields/`, `actions/`, `data/`.
- **`bizmanage status [-p <path>] [--auth-only] [--changes-only]`** — shows saved aliases/instance URLs (masked keys) and local file change stats.
- **`bizmanage status-detail [-a <alias>] [--project] [--api]`** — detailed auth/connection/project info; `--api` tests endpoints.
- **`bizmanage clear-cache`** (aliases `rm-cache`, `cache-clear`) `[-p <path>]` — clears `.bizmanage/file-hashes.json` so the next pull/push treats all files as changed. Use after manual file edits outside the CLI or to force a full re-sync.
- **`bizmanage validate [-f <file>] [-p <path>] [-a, --all]`** — Zod schema + business-logic validation. Defaults to only changed files; `--all` checks everything; `-f` checks one file.
- **`bizmanage test [-a <alias>] [--ping-only] [--verbose]`** — tests auth/connection (`GET /restapi/ping`).
- **`bizmanage pull [options]`** — downloads customizations to local files.
- **`bizmanage push [options]`** — validates, runs `npm test`, then deploys local changes.

### Pull / Push shared options

Both support scoping to specific customization types and items:

- `--include <target>` — one or more of: `objects, fields, actions, reports, pages, scripts` (push also allows `data`). Comma-separated or repeatable.
- `--object <name>` / `--view <name>` (alias) — limit to matching tables/views.
- `--field <name>` — supports `table.field` or `table:field`.
- `--action <name>` — supports `table.action` or `table:action`.
- `--report <name>`, `--page <name>`, `--script <name>` — match by name.
- `-a, --alias <alias>` — which saved environment/instance to use.

Pull-only:
- `-o, --output <path>` (default `./src`)
- `-d, --delay <ms>` — throttle between API requests (default 0; use e.g. `100`–`500` for rate-limited instances)
- `--init` — also scaffold the project
- `-f, --force` — pull blocks by default if local changes exist (to avoid clobbering work); `-f` overwrites local files and clears the hash cache

Push-only:
- `-s, --source <path>` (default cwd)
- `--all` — push every file, not just changed ones
- `--skip-tests` — skip `npm test` before deploy
- `--skip-validation` — skip metadata validation

Examples:
```bash
bizmanage pull --include actions --object customers
bizmanage pull --field customers.status --delay 200
bizmanage push --include reports
bizmanage push --page /dashboards/team-overview --script recalculate-totals
bizmanage push --all --skip-tests            # dev-only fast full deploy
```

**Gotcha — the "changed files" cache is alias-agnostic:** by default, `push` only sends files whose content hash differs from `.bizmanage/file-hashes.json`. That cache is keyed by file content, not by which instance/alias you last pushed to — so pushing the same files to a *second* instance (e.g. deploying something already pushed to production, now onto a fresh local/staging instance) can silently skip files it thinks are "already pushed." If a target instance doesn't yet have files you know are new to it, use `--all` (optionally scoped with `--page`/`--action`/etc.) to force them through, then verify with a scoped `pull -f` back from that same alias and diff against git — a clean diff confirms the deploy actually matches.

## Verbosity / debugging

Flags stack in detail: `-v`/`--verbose` (INFO) → `-vv`/`--very-verbose`/`--debug` (DEBUG) → `-vvv`/`--extra-verbose` (TRACE) → `-vvvv`/`--bizmanage-debug` (logs every request/response/error). `--silent` shows only errors. `--log-timestamps` adds timestamps.

## Config storage

Auth configs live in `~/.bizmanage/config.json`, keyed by alias:
```json
{ "default": { "instanceUrl": "...", "apiKey": "..." }, "production": { ... } }
```
Never print or commit API keys; the CLI masks them in `status` output.

## Version control

Commit: `src/`, `bizmanage.config.json`.
Ignore: `.bizmanage/` (especially `.bizmanage/file-hashes.json`, a rebuildable local cache — committing it causes noisy diffs across teammates), `node_modules/`, `.env*`.

## Troubleshooting

- **Auth error** → `bizmanage logout && bizmanage login`.
- **`npm test` fails with "Missing script: test"** → the project has no `package.json` test script configured; this isn't a code failure, it's a missing script. Use `--skip-tests` rather than trying to "fix" a test suite that doesn't exist.
- **Validation failures** → run `bizmanage validate` (or `push --skip-tests`) to isolate whether it's validation vs. tests.
- **Sync/cache weirdness** (e.g. after manual edits outside the CLI, merge conflicts, or the alias-agnostic cache gotcha above) → `bizmanage clear-cache`, or a targeted `push --all` for just the affected files.
- **Pull blocked by local changes** → commit/push first, or `bizmanage pull -f` to force-overwrite (only after your local work is safely committed/pushed).
- **Deploy verification** → after a push, pull the same selection back with `-f` into the git working tree and check `git diff --stat` is empty. A clean diff is a reliable signal the platform now has exactly what you pushed.
- Per-command help: `bizmanage <command> --help`.
