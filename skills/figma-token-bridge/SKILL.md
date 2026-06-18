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
  the file is remembered across sessions and `--figma` is needed only to override it,
  and **how this project reads Figma** in a `figmaRead` object. There are two read
  **strategies**, chosen once by `setup` (see [Setup](#setup)) and remembered here:
  `rest` (the Figma Variables REST API — Enterprise only) and `page` (a skill-managed
  page of variable-bound artefacts, read via the MCP server — works on any plan). Every
  section that touches Figma branches on `figmaRead.strategy`; the full schema is in
  [Lockfile format](#lockfile-format).
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
| `setup` | Interactively configure how this project reads Figma (`rest` vs `page`), bind the Figma file, generate the managed token page if needed, and write the initial lockfile. Runs automatically on first run; re-runnable anytime to switch strategy or regenerate the page. | lockfile (+ a Figma page in `page` strategy) |
| `status` | Show, per token, whether it is unchanged / changed-in-code / changed-in-figma / conflicted / new. Read-only. | nothing |
| `plan` | Compute the three-way merge and write `figma-token-bridge.plan.json`. Does not touch code or Figma. | plan file |
| `apply` | Execute an existing plan, then update the lockfile to the new agreed state. | code or Figma, lockfile |
| `adopt` | Write the current reconciled state to the lockfile without changing either side, recording the bound `figmaFile`. The lower-level baseline primitive that `setup` calls to write the first lock; also used to accept the current reality as a new base. | lockfile |

First run (no lockfile) triggers `setup` automatically. Thereafter a normal session is
`status` → `plan` → review the plan file → `apply`. There is no in-turn "apply? yes/no"
prompt: the plan file *is* the review surface, and `apply` is a deliberate separate
invocation. This makes review inspectable, diffable, and attachable to a PR rather than
ephemeral chat.

### howto

`/figma-token-bridge howto` is the built-in guide. It reads and writes nothing — it
just explains the skill in chat so a user can orient without opening this file. Use it
when the user is new, asks "how do I use this?", or types `howto` with no other verb.

Respond with a concise walkthrough, not a dump of this document:

1. **The model in one line** — a committed lockfile records the last agreed state, and
   each run reconciles code and Figma against it via three-way merge, so one-sided
   changes flow and two-sided changes surface as conflicts instead of being clobbered.
2. **The verbs** — `setup` (interactive first-run config: pick the read strategy, bind
   the file, write the first lock), `status` (read-only diff), `plan` (write the
   reviewable plan file), `apply` (execute a plan, advance the lock), `adopt` (set the
   lockfile base).
3. **A typical session** — `status` → `plan` → open `figma-token-bridge.plan.json` and
   edit it → `apply`.
4. **First run** — there's no lockfile yet, so the first verb auto-launches `setup`. It
   detects whether your plan supports the Variables REST API and picks a read strategy:
   `rest` (Enterprise) or `page` (the default for everyone else — a skill-managed token
   page). Setup writes the first lockfile. Mention this whenever no
   `figma-token-bridge.lock.json` exists.
5. **What it needs** — depends on the strategy. `rest` needs a Figma REST token
   (e.g. `FIGMA_TOKEN`) with variables-read scope (Enterprise). `page` needs only the
   Figma MCP server connected — **no token**. Either way, *writing* to Figma needs the
   MCP server, plus a token source in the repo and edit access to the Figma file.

Tailor the depth to the question: a bare `howto` gets the whole tour; "how do I
resolve a conflict?" gets the `--resolve` / `--force` explanation and little else.
`howto` never reads Figma or the codebase and never asks for a `--figma` URL.

### Options

- `--figma <url>` — Figma Design file or selection URL. The bound file is stored in the
  lockfile's `figmaFile` once set, so it is remembered across sessions and is **not**
  required on every run. Resolution order for any verb that reads Figma:
  1. `--figma <url>` if passed — wins, and is treated as a deliberate override.
  2. else the lockfile's `figmaFile`, if a lockfile exists.
  3. else (no lockfile yet) `setup` owns asking for it once and writes it into the first
     lockfile.
  When `--figma` overrides a different stored `figmaFile`, say so before reading — a
  changed file binding usually means a re-`adopt` (or re-`setup`), not a normal sync.
  Passing `--figma` that matches the stored value is a harmless no-op.
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
- `--init` — for `adopt`, allowed only when no lockfile exists yet. `setup` performs the
  same baseline write interactively; `adopt --init` is the explicit, non-interactive
  primitive. Re-run `setup` with no flags anytime to switch strategy or regenerate the
  managed token page.

## Preflight

Before any verb that touches Figma, check that the capability that verb needs is
actually available, and **stop with a specific, actionable message** if it isn't.
Never proceed into a read or write that will silently come up empty or fail
mid-operation. `howto` needs nothing and skips preflight entirely.

**No lockfile → run `setup`, don't error.** If a read/write verb (`status`/`plan`/
`apply`) is invoked and there is no `figma-token-bridge.lock.json`, the project hasn't been
configured yet. Launch the interactive [Setup](#setup) flow instead of erroring — setup
decides the read strategy, binds the file, and writes the first lockfile. Everything
below applies once a lockfile exists.

What a read needs depends on the lockfile's `figmaRead.strategy`; a Figma *write* is the
same in either strategy.

- **Read in `rest` strategy** needs a Figma REST token. Check the environment
  (`FIGMA_TOKEN`, or `FIGMA_ACCESS_TOKEN` / `FIGMA_PERSONAL_ACCESS_TOKEN`). If none is
  set, stop and request it:

  > **Figma token required (rest strategy).** This project reads the full variables
  > table via the Figma Variables REST API, which needs a personal access token with
  > **variables read** scope. Set it persistently so it survives shell reloads (zsh —
  > use `~/.zshenv`, which every shell sources, not `~/.zshrc`, which only interactive
  > ones do):
  > ```bash
  > echo 'export FIGMA_TOKEN="figd_your_token_here"' >> ~/.zshenv
  > source ~/.zshenv   # load it into the current shell
  > ```
  > For bash, use `~/.bashrc` (or `~/.profile` for login shells). See the README's
  > "Setting `FIGMA_TOKEN`" section for token creation and verification.

  Validate the token by probing the read (see [Figma side](#figma-side)). If the probe
  returns **401**, it's an auth failure (bad/expired token or missing scope) — say so
  and stop. If it returns **403**, the file lost Enterprise access; tell the user and
  suggest `/figma-token-bridge setup` to switch to the `page` strategy.

- **Read in `page` strategy** needs the **Figma MCP server** connected (it reads the
  managed token page via the server, not the REST API) — **no token required**. If the
  server isn't connected, stop with the MCP message below.

- **Write (`apply` writing to Figma, either strategy)** needs the **Figma MCP server**
  connected. If it isn't:

  > **Figma MCP server not configured.** This needs the Figma MCP server connected in
  > your agent. Connect it (and confirm `figma-use` is available), then re-run. A
  > read-only `status`/`plan` in `rest` strategy does not need the MCP server — only the
  > REST token above; `page` strategy needs the server for reads too.

Run the check that matches the verb and strategy, validate before doing real work (a
token that is present but rejected, or an MCP server that is configured but unreachable,
must be reported as such — presence is not the same as working), and distinguish the
failures plainly so the user knows whether to fix a token, a plan, an MCP connection, or
run setup. Surface the request once; do not loop.

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

The Figma snapshot is produced by whichever read strategy the lockfile records
(`figmaRead.strategy`); both strategies produce the same canonical rows, so everything
downstream — the merge, the plan, `apply` — is strategy-agnostic.

### Code sources (any codebase)

Detect, in priority order, and state which was used:

1. DTCG Design Tokens JSON (`**/tokens.json`, `**/*.tokens.json`)
2. Style Dictionary input JSON
3. CSS custom properties (`:root { --color-bg-primary: … }`)
4. A theme object in JS/TS (`theme.ts`, `tokens.ts`, exported nested object)

Modes come from whichever convention the source uses — nested `{ light, dark }`,
key suffixes (`.dark`), or sibling theme files (`light.json` / `dark.json`). Record
the detected mode convention in the plan so `apply` writes back in the same shape.

These detected code tokens (and their modes) are also the source `setup` uses to
**generate the managed token page** in `page` strategy — one artefact per token, a
section per mode (see [Figma side](#figma-side)).

### Figma side

How the Figma snapshot is read depends on `figmaRead.strategy` in the lockfile. Both
strategies normalize into the same canonical rows; they differ only in coverage and
fidelity. `setup` picks the strategy once (see [Setup](#setup)).

#### Strategy `rest` (Enterprise)

The highest-fidelity read. `GET /v1/files/:fileKey/variables/local` with a Figma
token enumerates **every local variable** — all collections, modes, per-mode values,
and alias (`VARIABLE_ALIAS`) targets — **independent of whether any node uses them**, so
a newly added or unused variable is included. This is the full variables table. It needs
a token in the environment (e.g. `FIGMA_TOKEN`) and is a Figma **Enterprise-plan**
endpoint. On `401` say it's an auth failure and stop; on `403` say the file isn't on
Enterprise (or the token lacks scope) and suggest `setup` to switch to `page`.

#### Strategy `page` (any plan, the default)

For non-Enterprise projects the snapshot is read from a **skill-managed page of
variable-bound artefacts** — the page whose node id is `figmaRead.tokensPageNodeId`.
Each token is *applied* to a node, and because `get_variable_defs` resolves the
variables bound to a node, applying every token to an artefact makes the whole set
readable through the MCP server without the REST API.

- **Artefact mapping** — `color` → a swatch (rectangle) with the variable bound to its
  fill; `number` → a frame whose bound dimension (width/height/spacing) uses the
  variable; `string`/type → a text node with the variable bound to its content or text
  style; `boolean` → a labeled toggle artefact (a node named for the token whose
  visible on/off state is driven by the variable). Each artefact's layer name is the
  token's canonical `key`.
- **Multi-mode** — every mode is represented as its own section (a frame per mode, named
  for the mode) so all per-mode values are present on one page at once; the mode names
  are recorded in `figmaRead.modes`.
- **How to read** — `get_metadata` on `tokensPageNodeId` to enumerate the per-mode
  sections and their artefact nodes, then `get_variable_defs` scoped to those nodes to
  resolve each bound variable's name→value per mode. Normalize into the same canonical
  rows (`key`, `type`, `modes`, `ref`, `hash`).
- **Honest caveats** — coverage is exactly "what's on the managed page": a variable with
  no artefact is invisible to this read (this is why the deletion guard below treats such
  absences as *unknown*, never deletions). Alias/`ref` fidelity is limited — node reads
  resolve values rather than exposing `VARIABLE_ALIAS` targets, so `ref` may be captured
  as a resolved value; flag relinks conservatively rather than guessing. Completeness is
  defined relative to the page the skill itself generates and maintains.

A bare `get_variable_defs` on an arbitrary selection remains available as an ad-hoc
spot-check, but it is not a strategy — it only sees the current selection.

**Load `figma-use` only before a write**, never for ordinary reads — with one
exception: `page` strategy's `setup`/`apply` use `figma-use` to *create and maintain*
the managed page, which is itself a write.

#### Never read a partial table as deletion

A token present in the lockfile but absent from the Figma read is only a real deletion
if the read was **complete**. Because a scoped or failed read can omit variables that
still exist, guard against false `flag`s — and "complete" means something different per
strategy:

- **`rest`** — only classify absent-in-figma tokens as `deleted in figma` when the
  snapshot came from a REST read that **succeeded and returned a non-empty table**.
- **`page`** — a token absent from the read is **not** a deletion just because it lacks
  an artefact; absences are *unknown*. A genuine page-mode deletion is only an artefact
  the skill previously maintained that is now gone from the page. When in doubt, treat
  page-coverage gaps as unknown and never auto-delete.
- In either strategy, if the read fails, returns empty, or coverage is uncertain,
  **abort the run** (or, for `status`, report "Figma read incomplete — coverage
  limited") rather than emitting deletions. A bad read must never advance the lockfile
  or produce delete/`flag` ops.

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

The table is strategy-agnostic, with one caveat: in `page` strategy the `deleted in
figma` row fires only when an artefact the skill maintained was genuinely removed from
the managed page, never when a variable merely lacks an artefact (see the deletion
guard above).

## Plan file format

`plan` writes `figma-token-bridge.plan.json`:

```json
{
  "figmaTokenBridgeVersion": 1,
  "figmaFile": "<url>",
  "strategy": "page",
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
`apply` executes exactly the ops present in the file. The `strategy` field records which
read strategy produced the plan, so `apply` knows whether it must also maintain the
managed token page (`page`) or only write variables (`rest`).

## Lockfile format

`figma-token-bridge.lock.json` is committed and holds the agreed base plus the project's
Figma binding and read strategy:

```json
{
  "figmaTokenBridgeVersion": 1,
  "figmaFile": "https://figma.com/design/<fileKey>/<name>",
  "figmaRead": {
    "strategy": "page",
    "tokensPageNodeId": "123:456",
    "modes": ["light", "dark"]
  },
  "tokens": {
    "color.bg.primary": {
      "type": "color",
      "modes": { "light": "#FFFFFF", "dark": "#0B0B0F" },
      "ref": null,
      "hash": "<contentHash>"
    }
  }
}
```

- `figmaRead.strategy` — `"rest"` or `"page"`. For `"rest"`, `tokensPageNodeId` and
  `modes` are omitted (the REST read needs neither). For `"page"`, both are present:
  `tokensPageNodeId` is the managed page, `modes` the mode sections represented on it.
- `tokens` — the agreed base, one canonical row per token (the same shape produced by
  reading; see [Reading and normalization](#reading-and-normalization)). This is the
  merge base the three-way compare uses.

## apply

1. Refuse if the plan's `base` lock hash no longer matches the on-disk lockfile
   (something changed since planning) — tell the user to re-`plan`.
2. If any `flag` ops remain and no matching `--resolve` was given, refuse and list
   them. Resolution is explicit, per the conflict's nature.
3. For Figma writes: load `figma-use`, then apply ops sequentially in this order —
   ensure collections/modes, `create` primitives, `create`/`relink` aliases, `set`
   values. Never parallelize writes. Never delete unless an op explicitly says so
   *and* the user passed an explicit delete resolution. In `page` strategy this same
   step must **also maintain the managed token page** so future reads stay complete: a
   newly created token gets a new artefact in every mode section; a changed value
   updates the artefact (and adds a mode section if a new mode appeared). The variables
   are still the source of truth — the page is their readable surface — so a designer's
   edit to a bound variable is captured automatically on the next read, no special
   handling needed.
4. For code writes: edit the detected source files in their native shape and mode
   convention. Rely on the user's VCS as the safety net; if the working tree is not
   under version control, say so before writing.
5. On success, rewrite `figma-token-bridge.lock.json` to the new agreed state and report the
   new lock hash. Persist `figmaFile` and `figmaRead` (the resolved file and strategy
   used for this run — refresh `figmaRead.modes` if a new mode was added) so the binding
   sticks. The lock now reflects reality; the next `status` is clean and needs no
   `--figma`.

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
- In `page` strategy, `--force code` must also reconcile the managed page — repair or
  regenerate artefacts so the forced state is exactly what future reads will see.

`--force` without a value, or with both sides somehow implied, is an error — never
guess a direction.

## Safety and reversibility

- The lockfile is the audit trail: every `apply` advances it, and git history shows
  exactly what each sync agreed to.
- Before writing to Figma, save a snapshot of current Figma state to
  `.figma-token-bridge/figma-<fileKey>-<timestamp>.json` so a bad apply can be reconstructed.
  In `page` strategy this snapshot also captures the managed page state (artefact node
  ids and values) so a bad page mutation is recoverable, not just the variables.
- A `--force` apply additionally snapshots the side being overwritten before writing,
  so an override is recoverable from disk in addition to VCS.
- Deletions and conflicts are never resolved without an explicit instruction.
- Capability gating is handled up front by **Preflight**, strategy-aware: a `rest` read
  needs a `FIGMA_TOKEN`; a `page` read needs the MCP server; any Figma write needs the
  MCP server; and a verb run with no lockfile launches `setup` instead of erroring. A
  missing capability stops the run with a specific message rather than failing
  mid-operation.

## Setup

`setup` is the interactive first-run configuration. It decides how this project reads
Figma, binds the file, generates the managed token page if needed, and writes the first
lockfile.

1. **Trigger.** Runs automatically when a read/write verb finds no lockfile (per
   [Preflight](#preflight)). Also invokable explicitly as `/figma-token-bridge setup` to
   reconfigure at any time — switch strategy, rebind the file, or regenerate the page.

2. **Resolve the Figma file.** Use `--figma <url>` if passed; otherwise ask once (there
   is no lockfile yet to read it from). Extract the `fileKey`.

3. **Detect read capability — by probe, not introspection.** A Figma token's scopes
   cannot be read from the token itself, so detection means *attempting* the REST read
   and branching on the result:
   - **No token in the environment** → choose **`page`**.
   - **Token present** → probe `GET /v1/files/:fileKey/variables/local`:
     - **200** → REST works → choose **`rest`**.
     - **403** → the token is valid but lacks `file_variables:read` scope, or the file
       isn't on an Enterprise org → choose **`page`** and tell the user why (offer
       `rest` later if they get Enterprise / add scope).
     - **401** → bad or expired token → report it; offer to fix the token and re-probe,
       or proceed with **`page`**.
     - anything else / empty → treat as `page`-eligible and report the ambiguity.

4. **`rest` branch.** Confirm `figmaRead = { "strategy": "rest" }`. No page generation.
   Proceed to the baseline write (step 6).

5. **`page` branch.** Confirm the **Figma MCP server is connected** (writing the page
   needs it; stop and request it if not). Read the code-side tokens and their modes (see
   [Code sources](#code-sources-any-codebase)). Offer to **generate the managed token
   page** from those tokens: a page of variable-bound artefacts, one artefact per token,
   a section per mode (see [Figma side](#figma-side)), created via `figma-use`. Capture
   the new page node id and the mode list as
   `figmaRead = { "strategy": "page", "tokensPageNodeId": "...", "modes": [...] }`.

6. **Write the baseline lockfile (this is `adopt --init`).** With no prior base every
   differing token would look like a conflict, so reconcile once — help the user settle
   divergence, or pick a side for the initial state — then write
   `figma-token-bridge.lock.json` with `figmaFile`, `figmaRead`, and the agreed `tokens`
   base. Subsequent runs use real three-way merge and reuse the stored file and strategy.

`setup` is the guided wrapper; the baseline write it performs is exactly `adopt --init`,
which remains available as the explicit, non-interactive primitive for users who already
know their strategy or are scripting.
