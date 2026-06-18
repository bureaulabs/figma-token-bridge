# figma-token-bridge

A Figma MCP skill that reconciles design tokens between your **codebase** and
**Figma variables** — without blindly overwriting either side.

Most token-sync tools ask "push or pull?" and overwrite the destination.
figma-token-bridge instead keeps a committed **lockfile** recording the last state both
sides agreed on, and reconciles changes against it. A token that changed on one
side flows to the other; a token that changed on *both* sides is surfaced as a
conflict instead of being silently clobbered.

Works with any codebase — DTCG tokens JSON, Style Dictionary, CSS custom
properties, or a JS/TS theme object.

## How it works

Three artifacts:

- **Snapshot** — normalized state of one side, read fresh each run.
- **Lockfile** (`figma-token-bridge.lock.json`, committed) — the last agreed state; the
  merge base that makes conflict detection possible. Also stores the bound Figma file
  and the **read strategy** (and, for the page strategy, the plugin-maintained
  token-page id), so both are remembered across sessions.
- **Plan** (`figma-token-bridge.plan.json`, generated) — an ordered, readable list of
  operations you inspect before anything is written.

First run launches an interactive [setup](#first-run-setup) before anything else.
Thereafter a session is: `status` → `plan` → read/edit the plan file → `apply`. The plan
file is the review surface, so review is diffable and PR-attachable rather than a
fleeting chat prompt.

## Usage

```
/figma-token-bridge <verb> [options]
```

New here? Run `/figma-token-bridge howto` for a guided tour of the model and verbs.

| Verb | Purpose |
|------|---------|
| `howto` | Explain how the skill works — model, verbs, a typical session, first-run setup. Reads and writes nothing. |
| `setup` | Interactively configure the read strategy, bind the Figma file, generate the token page if needed, write the first lockfile. Auto-runs on first run; re-runnable. |
| `status` | Per-token: unchanged / changed-in-code / changed-in-figma / conflict / new. Read-only. |
| `plan` | Compute the three-way merge, write the plan file. No writes to code or Figma. |
| `apply` | Execute a plan, then advance the lockfile. |
| `adopt` | Record current state as the lockfile base — the low-level baseline primitive `setup` calls (`--init` on first run). |
| `reset` | Clear this project's state (lockfile, plan, optionally snapshots / the Figma token page) and start over with a fresh `setup`. Confirms first. |

Options: `--figma <url>` (stored in the lockfile after the first run; only needed
again to override the bound file), `--only <glob>` (repeatable), `--resolve code|figma`
(for conflicts at apply time), `--force code|figma` (deliberate whole-side override
at apply time — overwrites the other side's changes too), `--init` (first-run adopt),
and for `reset`: `--purge` (also delete local snapshots), `--with-page` (also delete the
plugin-maintained Figma token page — which the plugin will recreate), `--yes` (skip the
confirmation prompt).

## First-run setup

On first run (no lockfile) the skill auto-runs an interactive setup: it asks for the
Figma file URL, detects whether your plan supports the Enterprise Variables REST API,
and picks one of two **read strategies** — remembered in the lockfile so you only do this
once:

- **`rest`** (Enterprise) — reads the full variables table via the Figma Variables REST
  API, the highest-fidelity read. Needs a `FIGMA_TOKEN` with variables-read scope.
- **`page`** (the default for everyone else) — reads tokens from a **Figma page of
  variable-bound artefacts maintained by the companion Token Sync plugin** (each token
  applied to a node, one section per collection × mode), through the MCP server. **No
  token required** — just the Figma MCP server connected and the plugin to keep the page
  current. The page is how non-Enterprise projects get a complete, mode-aware read
  without the REST API; on `setup` the skill discovers it by name and remembers its node
  id. The page shape is defined by the **Token Page Contract (v1)**, maintained in the
  Token Sync plugin repo, which the skill and plugin both honor.

Detection is by probe, not guesswork: setup tries the REST read and falls back to `page`
on a 403 (not Enterprise / missing scope) or when no token is set. Re-run
`/figma-token-bridge setup` anytime to switch strategy or re-discover the token page (e.g.
after upgrading to Enterprise).

## Why a lockfile

Diff-and-confirm tools have no memory of prior state, so they can only ask which
side wins. With a committed base, the question becomes "what changed relative to
what we last agreed?" — answerable per token. Each `apply` advances the lock, so
your git history is a precise audit trail of every sync.

## Requirements

What you need depends on the read strategy setup selects:

- **`page` strategy (default, any plan)** — the **Figma MCP server** connected in your
  agent (Claude Code, Cursor, etc.), plus the **Token Sync plugin** installed in Figma to
  build and maintain the token page. The MCP server is used for reading that page and for
  writing. **No `FIGMA_TOKEN` needed.**
- **`rest` strategy (Enterprise)** — a **Figma REST token** (e.g. `FIGMA_TOKEN`) with
  variables-read scope, to read the full variables table via the Enterprise-only
  Variables REST API. Writes still go through the MCP server.
- **Either strategy** — a token source in your repo, and edit access to the Figma file
  for any run that writes to Figma (in `page` strategy that also covers creating and
  maintaining the token page).

The skill **preflight-checks** these before doing any work, strategy-aware: a `rest` read
stops and asks for `FIGMA_TOKEN` if it's missing, a `page` read or any write stops and
asks for the MCP server if it isn't connected, and a verb run with no lockfile launches
setup rather than erroring — each only when that capability is actually needed.

### Setting `FIGMA_TOKEN` (zsh)

> Only needed if setup selected the **`rest`** (Enterprise) strategy. On the default
> **`page`** strategy there's no token to set — skip this section.

1. **Create the token** in Figma: avatar → **Settings** → **Security** → **Personal
   access tokens** → generate one with **Variables: read** scope (add **write** too if
   you'll write back). Copy it — it starts with `figd_` and is shown only once.

2. **Persist it for every shell.** Add the export to `~/.zshenv`, which zsh sources on
   *every* invocation — interactive terminals, non-interactive shells, and scripts/tools
   alike. That last part matters: agents and CLIs often run in non-interactive shells
   that never read `~/.zshrc`, so `~/.zshenv` is what makes the token reliably present
   on every reload.

   ```bash
   echo 'export FIGMA_TOKEN="figd_your_token_here"' >> ~/.zshenv
   ```

3. **Load it now** (new shells pick it up automatically going forward):

   ```bash
   source ~/.zshenv
   ```

4. **Verify** it's set:

   ```bash
   [ -n "$FIGMA_TOKEN" ] && echo "FIGMA_TOKEN is set" || echo "not set — open a new terminal"
   ```

> Notes
> - `~/.zshenv` covers shell-launched tools. **GUI apps** launched from the Dock (not a
>   terminal) don't read it — restart the app from a terminal, or set a launchd/login
>   environment, if your agent runs as a Dock app.
> - Prefer `~/.zshenv` over `~/.zshrc` here: `~/.zshrc` only runs for *interactive*
>   shells, so a token set there can be invisible to a tool running non-interactively.
> - Treat the token as a secret: don't commit it, and prefer a real secret manager over
>   a dotfile if your environment offers one.

## Install (project scope)

Packaged as a plugin so the skill is available to any compatible agent. Install it
into the **project**, not globally:

```bash
npx skills add bureaulabs/figma-token-bridge@figma-token-bridge -y
```

Or copy `skills/figma-token-bridge/` into your tool's skills directory. The
`.claude-plugin/`, `.cursor-plugin/`, and `.mcp.json` manifests let it load as a
plugin in each client.

### Why not a global install?

This skill declares a Figma MCP-server dependency (`metadata: mcp-server: figma` in
`SKILL.md`), and the MCP server is configured per project via `.mcp.json`. Installers
therefore scope the skill to the project and **reject a global install** — passing a
global flag (`-g` / `--global`) fails with:

```
figma-token-bridge → PromptScript: PromptScript does not support global skill installation
```

That's the installer refusing to place an MCP-dependent skill where its server isn't
available — not a problem with the skill itself. Project scope is the right home for a
tool that reconciles a specific repo against a specific Figma file.

If you truly need it on every project without re-installing, configure the Figma MCP
server globally in your agent's user settings and remove the `metadata.mcp-server`
line from `SKILL.md` so the skill no longer pins itself to project scope. Not
recommended — the dependency is real.

## Suggested .gitignore

```
.figma-token-bridge/          # local Figma snapshots (pre-write backups)
figma-token-bridge.plan.json  # generated; commit it only if you want plans in PRs
```

Keep `figma-token-bridge.lock.json` **committed** — it's the shared base.
