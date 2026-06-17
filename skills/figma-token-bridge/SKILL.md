---
name: figma-token-bridge
description: "Reconcile design tokens between a codebase and Figma variables using a committed lockfile and a reviewable, re-runnable plan. Detects true conflicts via three-way merge against the last agreed state, rather than blindly overwriting one side. Invoke with /figma-token-bridge."
disable-model-invocation: true
metadata:
  mcp-server: figma
---

# figma-token-bridge

Reconcile design tokens between **code** and **Figma variables**.

figma-token-bridge does not "push" or "pull" blindly. It records the last agreed state in
a committed **lockfile** and reconciles changes against it, so a change made on one
side is applied, and a change made on *both* sides is surfaced as a conflict instead
of being silently overwritten.

Invoke with: `/figma-token-bridge <verb> [options]`

## Model: three nouns

- **Snapshot** — the canonical, normalized state of one side (code *or* Figma) at a
  moment in time. Produced by reading and normalizing; never written back.
- **Lockfile** — `figma-token-bridge.lock.json`, committed to the repo. The last state both
  sides agreed on. This is the merge base. Its presence is what makes conflict
  detection possible. It also records the project's bound Figma file (`figmaFile`), so
  the file is remembered across sessions and `--figma` is needed only to override it.
- **Plan** — `figma-token-bridge.plan.json`, a generated, ordered, human-readable list of
  operations (`create`, `set`, `relink`, `flag`). A plan is inspected and then
  executed as a separate step. Plans are deterministic: the same inputs produce the
  same plan, and re-running a plan that is already satisfied is a no-op.

The lockfile is the heart of the design. Without it, the only possible question is
"which side wins?" With it, the question becomes "what changed, relative to what we
last agreed?" — which is answerable per token.

## Verbs

| Verb | What it does | Writes |
|------|--------------|--------|
| `howto` | Explain how the skill works — the model, the verbs, a typical session, and first-run setup. Pure guidance; reads nothing and writes nothing. | nothing |
| `status` | Show, per token, whether it is unchanged / changed-in-code / changed-in-figma / conflicted / new. Read-only. | nothing |
| `plan` | Compute the three-way merge and write `figma-token-bridge.plan.json`. Does not touch code or Figma. | plan file |
| `apply` | Execute an existing plan, then update the lockfile to the new agreed state. | code or Figma, lockfile |
| `adopt` | Write the current reconciled state to the lockfile without changing either side, recording the bound `figmaFile`. Used to initialize the lock, or to accept the current reality as the new base. | lockfile |

A normal session is `status` → `plan` → review the plan file → `apply`. There is no
in-turn "apply? yes/no" prompt: the plan file *is* the review surface, and `apply` is
a deliberate separate invocation. This makes review inspectable, diffable, and
attachable to a PR rather than ephemeral chat.

### howto

`/figma-token-bridge howto` is the built-in guide. It reads and writes nothing — it
just explains the skill in chat so a user can orient without opening this file. Use it
when the user is new, asks "how do I use this?", or types `howto` with no other verb.

Respond with a concise walkthrough, not a dump of this document:

1. **The model in one line** — a committed lockfile records the last agreed state, and
   each run reconciles code and Figma against it via three-way merge, so one-sided
   changes flow and two-sided changes surface as conflicts instead of being clobbered.
2. **The verbs** — `status` (read-only diff), `plan` (write the reviewable plan file),
   `apply` (execute a plan, advance the lock), `adopt` (set the lockfile base).
3. **A typical session** — `status` → `plan` → open `figma-token-bridge.plan.json` and
   edit it → `apply`.
4. **First run** — there's no lockfile yet, so run `status`, reconcile once, then
   `/figma-token-bridge adopt --init` to write the first base. Mention this whenever no
   `figma-token-bridge.lock.json` exists.
5. **What it needs** — a Figma REST token (e.g. `FIGMA_TOKEN`) to read the full
   variables table, the Figma MCP server connected for writes, a token source in the
   repo, and edit access to the Figma file for any write.

Tailor the depth to the question: a bare `howto` gets the whole tour; "how do I
resolve a conflict?" gets the `--resolve` / `--force` explanation and little else.
`howto` never reads Figma or the codebase and never asks for a `--figma` URL.

### Options

- `--figma <url>` — Figma Design file or selection URL. The bound file is stored in the
  lockfile's `figmaFile` once set, so it is remembered across sessions and is **not**
  required on every run. Resolution order for any verb that reads Figma:
  1. `--figma <url>` if passed — wins, and is treated as a deliberate override.
  2. else the lockfile's `figmaFile`, if a lockfile exists.
  3. else (no lockfile yet, or lockfile has no `figmaFile`) ask once and remember it
     for the session; the first `adopt`/`apply` writes it into the lockfile.
  When `--figma` overrides a different stored `figmaFile`, say so before reading — a
  changed file binding usually means a re-`adopt`, not a normal sync. Passing `--figma`
  that matches the stored value is a harmless no-op.
- `--only <glob>` — restrict to matching token keys (e.g. `--only "color.*"`).
  Repeatable. Applies to every verb.
- `--resolve code|figma` — for `apply`, how to settle conflicts that the plan marked
  `flag`. With no `--resolve`, a plan containing conflicts refuses to apply and lists
  them. There is intentionally no default winner. `--resolve` only touches `flag` ops;
  it does not change one-sided changes the plan already routed correctly.
- `--force code|figma` — for `apply`, a deliberate override: make the named side
  authoritative for **every** in-scope token, including non-conflicting changes the
  other side made. This intentionally overwrites the losing side's one-sided changes,
  so it is a blunt instrument, distinct from `--resolve`. It must be passed explicitly,
  is never implied, and `plan`/`status` always show the per-token routing `--force`
  would override. Scope it with `--only` to limit the blast radius. See "Forced apply".
- `--init` — for `adopt`, allowed only when no lockfile exists yet.

## Preflight

Before any verb that touches Figma, check that the capability that verb needs is
actually available, and **stop with a specific, actionable message** if it isn't.
Never proceed into a read or write that will silently come up empty or fail
mid-operation. `howto` needs nothing and skips preflight entirely.

Two distinct capabilities, checked only when the verb needs them:

- **Read (any verb that reads Figma — `status`, `plan`, `apply`)** needs a Figma REST
  token. Check for it in the environment (`FIGMA_TOKEN`, or `FIGMA_ACCESS_TOKEN` /
  `FIGMA_PERSONAL_ACCESS_TOKEN`). If none is set, stop and request it:

  > **Figma token required.** This skill reads the full variables table via the Figma
  > REST API, which needs a personal access token with **variables read** scope. Set
  > one and re-run:
  > ```bash
  > export FIGMA_TOKEN=figd_xxx   # add to your shell profile to persist
  > ```
  > Note: the Variables REST API is a Figma **Enterprise-plan** feature. If a token is
  > set but the read returns 401/403, say whether it's an auth failure (bad/expired
  > token or missing scope) or a plan limitation (not on Enterprise), and stop.

- **Write (`apply` writing to Figma)** needs the **Figma MCP server** connected. If it
  isn't, stop before reading anything and request it:

  > **Figma MCP server not configured.** Writing to Figma needs the Figma MCP server
  > connected in your agent. Connect it (and confirm `figma-use` is available), then
  > re-run. A read-only `status`/`plan` does not need the MCP server — only the REST
  > token above.

Run the check that matches the verb, validate before doing real work (a token that is
present but rejected, or an MCP server that is configured but unreachable, must be
reported as such — presence is not the same as working), and distinguish the two
failures plainly so the user knows whether to fix a token, a plan, or an MCP
connection. Surface the request once; do not loop.

## Reading and normalization

Read each side into a snapshot of canonical token rows:

- `key` — dot-path canonical name (`color.bg.primary`). Figma `group/name` segments
  map to dots; confirm any non-trivial transform with the user before relying on it.
- `type` — `color` | `number` | `string` | `boolean`
- `modes` — map of mode name → value (a token is a value *per mode*, not one value)
- `ref` — if the token aliases another token, its canonical target key (else null)
- `hash` — stable content hash of `{type, modes, ref}` after normalization

The `hash` is what detection compares, so renames and reformatting that don't change
meaning don't register as drift, and a value change registers even if the formatting
looks similar (e.g. `#000` vs `#000000` normalize to the same hash; `#000` vs `#010101`
do not).

### Code sources (any codebase)

Detect, in priority order, and state which was used:

1. DTCG Design Tokens JSON (`**/tokens.json`, `**/*.tokens.json`)
2. Style Dictionary input JSON
3. CSS custom properties (`:root { --color-bg-primary: … }`)
4. A theme object in JS/TS (`theme.ts`, `tokens.ts`, exported nested object)

Modes come from whichever convention the source uses — nested `{ light, dark }`,
key suffixes (`.dark`), or sibling theme files (`light.json` / `dark.json`). Record
the detected mode convention in the plan so `apply` writes back in the same shape.

### Figma side

The Figma snapshot must cover **every local variable**, including ones not applied to
any frame or layer. That requires a full read of the variables table, which is the
**canonical read path**:

1. **REST Variables API (canonical).** `GET /v1/files/:fileKey/variables/local` with a
   Figma personal/CI token. This enumerates all local variable collections, their
   modes, per-mode values, and alias (`VARIABLE_ALIAS`) targets, **independent of
   whether any node uses them** — so a newly added or unused variable is included. This
   is the read the snapshot is built from. It needs a token in the environment
   (e.g. `FIGMA_TOKEN`) and is a Figma enterprise-plan endpoint; if the token is
   missing, say so and stop rather than silently falling back.

2. **MCP `get_variable_defs` (selection-only fallback).** This resolves only the
   variables **bound to a given node / the current selection**, and returns them as a
   flattened name→value map — it does **not** enumerate the table, and unused variables
   never appear. Use it only for a quick, explicitly scoped spot-check of a selection,
   never as the source for a full `status`/`plan`/`apply`. State plainly when a read
   came from this path that coverage is limited to the selection.

**Load `figma-use` only before a write**, never for reads.

#### Never read a partial table as deletion

A token present in the lockfile but absent from the Figma read is only a real
deletion if the read was **complete**. Because a scoped or failed read can omit
variables that still exist, guard against false `flag`s:

- Only classify absent-in-figma tokens as `deleted in figma` when the snapshot came
  from the canonical REST read **and** that read succeeded and returned a non-empty
  table. Otherwise treat absences as *unknown*, not deleted.
- If the REST read fails, returns empty, or only a selection-scoped fallback was
  available, **abort the run** (or, for `status`, report "Figma read incomplete —
  coverage limited") rather than emitting deletions. A bad read must never advance the
  lockfile or produce delete/`flag` ops.

## Three-way merge (the core algorithm)

For each token key, compare three states: **base** (from the lockfile), **code**, and
**figma**. Compare by `hash`, per mode.

| base | code | figma | classification | plan op |
|------|------|-------|----------------|---------|
| =    | =    | =     | unchanged | none |
| =    | ≠    | =     | changed in code | write code→figma (or figma→code on `apply --to code`) |
| =    | =    | ≠     | changed in figma | write figma→code (or code→figma) |
| =    | ≠    | ≠ (code=figma) | converged independently | none, advance lock |
| =    | ≠    | ≠ (code≠figma) | **conflict** | `flag` |
| absent | present | absent | new in code | `create` on figma |
| absent | absent | present | new in figma | `create` in code |
| present | absent | present | deleted in code | `flag` (never auto-delete) |
| present | present | absent | deleted in figma | `flag` (never auto-delete) |

The direction `apply` writes is determined by where the change originated, not by a
global "push/pull" mode. A run can therefore carry some code→figma and some
figma→code operations at once — each token flows from wherever it changed. Conflicts
and deletions never resolve automatically.

## Plan file format

`plan` writes `figma-token-bridge.plan.json`:

```json
{
  "figmaTokenBridgeVersion": 1,
  "figmaFile": "<url>",
  "base": "figma-token-bridge.lock.json@<lockHash>",
  "modeConvention": "nested",
  "ops": [
    { "op": "set", "key": "color.bg.primary", "mode": "dark",
      "to": "code", "from": "#0B0B0F", "was": "#000000" },
    { "op": "create", "key": "space.4", "target": "figma", "value": "4" },
    { "op": "relink", "key": "color.action", "ref": "color.brand.500", "target": "figma" },
    { "op": "flag", "key": "radius.lg", "reason": "conflict",
      "code": "12", "figma": "16", "base": "8" }
  ],
  "summary": { "set": 1, "create": 1, "relink": 1, "flag": 1 }
}
```

The plan is meant to be opened and read. A reviewer can delete ops they don't want;
`apply` executes exactly the ops present in the file.

## apply

1. Refuse if the plan's `base` lock hash no longer matches the on-disk lockfile
   (something changed since planning) — tell the user to re-`plan`.
2. If any `flag` ops remain and no matching `--resolve` was given, refuse and list
   them. Resolution is explicit, per the conflict's nature.
3. For Figma writes: load `figma-use`, then apply ops sequentially in this order —
   ensure collections/modes, `create` primitives, `create`/`relink` aliases, `set`
   values. Never parallelize writes. Never delete unless an op explicitly says so
   *and* the user passed an explicit delete resolution.
4. For code writes: edit the detected source files in their native shape and mode
   convention. Rely on the user's VCS as the safety net; if the working tree is not
   under version control, say so before writing.
5. On success, rewrite `figma-token-bridge.lock.json` to the new agreed state and report the
   new lock hash. Persist `figmaFile` (the resolved file used for this run) into the
   lockfile so the binding sticks. The lock now reflects reality; the next `status` is
   clean and needs no `--figma`.

### Forced apply (`--force`)

`--force code|figma` overrides the normal per-token routing and makes one side win
for every in-scope token. Use it only as a deliberate "this side is authoritative
right now" action — e.g. recovering from a badly drifted lockfile, or stamping a
known-good source over an experimental Figma file.

When `--force` is set:

- The named side's value wins for every in-scope token, including tokens the *other*
  side changed one-sidedly and tokens marked `flag`. The losing side's changes are
  overwritten. `--resolve` is ignored (force is the stronger statement).
- Before executing, restate the override plainly and quantify it — e.g. "`--force code`
  will overwrite N Figma-side change(s) and M conflict(s)" — using the plan's counts,
  so the cost is visible. Honor the same backup step (snapshot the side being
  overwritten first).
- Deletions are still never performed implicitly. `--force` overwrites values and
  relinks aliases; it does not delete tokens absent on the winning side unless the
  user also gives an explicit delete instruction.
- After applying, advance the lockfile as usual. The forced state becomes the new base.

`--force` without a value, or with both sides somehow implied, is an error — never
guess a direction.

## Safety and reversibility

- The lockfile is the audit trail: every `apply` advances it, and git history shows
  exactly what each sync agreed to.
- Before writing to Figma, save a snapshot of current Figma state to
  `.figma-token-bridge/figma-<fileKey>-<timestamp>.json` so a bad apply can be reconstructed.
- A `--force` apply additionally snapshots the side being overwritten before writing,
  so an override is recoverable from disk in addition to VCS.
- Deletions and conflicts are never resolved without an explicit instruction.
- Capability gating is handled up front by **Preflight**: a read needs a `FIGMA_TOKEN`,
  a Figma write needs the MCP server connected. Each is requested only when the verb
  needs it (a read-only `status`/`plan` never requires the MCP server), and a missing
  capability stops the run with a specific message rather than failing mid-operation.

## Initialization

First run, with no lockfile: there is no merge base, so every differing token would
look like a conflict. The first run also has no stored `figmaFile`, so pass `--figma`
once on this run. Run `status` to show the divergence, help the user reconcile once
(or pick a side for the initial state), then `/figma-token-bridge adopt --init` to write
the first lockfile — which records the `figmaFile` binding. Subsequent runs use real
three-way merge and reuse the stored file, so `--figma` is only needed to override.
