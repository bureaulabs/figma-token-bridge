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
  merge base that makes conflict detection possible.
- **Plan** (`figma-token-bridge.plan.json`, generated) — an ordered, readable list of
  operations you inspect before anything is written.

A session is: `status` → `plan` → read/edit the plan file → `apply`. The plan file
is the review surface, so review is diffable and PR-attachable rather than a
fleeting chat prompt.

## Usage

```
/figma-token-bridge <verb> [options]
```

New here? Run `/figma-token-bridge howto` for a guided tour of the model and verbs.

| Verb | Purpose |
|------|---------|
| `howto` | Explain how the skill works — model, verbs, a typical session, first-run setup. Reads and writes nothing. |
| `status` | Per-token: unchanged / changed-in-code / changed-in-figma / conflict / new. Read-only. |
| `plan` | Compute the three-way merge, write the plan file. No writes to code or Figma. |
| `apply` | Execute a plan, then advance the lockfile. |
| `adopt` | Record current state as the lockfile base (use `--init` on first run). |

Options: `--figma <url>`, `--only <glob>` (repeatable), `--resolve code|figma`
(for conflicts at apply time), `--force code|figma` (deliberate whole-side override
at apply time — overwrites the other side's changes too), `--init` (first-run adopt).

## Why a lockfile

Diff-and-confirm tools have no memory of prior state, so they can only ask which
side wins. With a committed base, the question becomes "what changed relative to
what we last agreed?" — answerable per token. Each `apply` advances the lock, so
your git history is a precise audit trail of every sync.

## Requirements

- The **Figma MCP server** connected in your agent (Claude Code, Cursor, etc.).
- A token source in your repo.
- Edit access to the Figma file for any run that writes to Figma.

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
