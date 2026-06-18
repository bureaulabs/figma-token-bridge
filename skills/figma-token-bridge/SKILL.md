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
  `rest` (the Figma Variables REST API — Enterprise only) and `page` (a
  **plugin-maintained** page of variable-bound artefacts, read via the MCP server —
  works on any plan). Every section that touches Figma branches on `figmaRead.strategy`;
  the full schema is in [Lockfile format](#lockfile-format).
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
| `setup` | Interactively configure how this project reads Figma (`rest` vs `page`), bind the Figma file, discover/adopt the plugin-maintained token page (`page` strategy), and write the initial lockfile. Runs automatically on first run; re-runnable anytime to switch strategy or re-discover the page. | lockfile |
| `status` | Show, per token, whether it is unchanged / changed-in-code / changed-in-figma / conflicted / new. Read-only. | nothing |
| `plan` | Compute the three-way merge and write `figma-token-bridge.plan.json`. Does not touch code or Figma. | plan file |
| `apply` | Execute an existing plan, then update the lockfile to the new agreed state. | code or Figma, lockfile |
| `adopt` | Write the current reconciled state to the lockfile without changing either side, recording the bound `figmaFile`. The lower-level baseline primitive that `setup` calls to write the first lock; also used to accept the current reality as a new base. | lockfile |
| `reset` | Clear this project's figma-token-bridge state (lockfile, plan, and optionally snapshots / the managed Figma page) so the next run starts from scratch with a fresh `setup`. Confirms before deleting. | deletes local state (+ Figma page with `--with-page`) |

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
   lockfile base), `reset` (clear state and start over).
3. **A typical session** — `status` → `plan` → open `figma-token-bridge.plan.json` and
   edit it → `apply`.
4. **First run** — there's no lockfile yet, so the first verb auto-launches `setup`. It
   detects whether your plan supports the Variables REST API and picks a read strategy:
   `rest` (Enterprise) or `page` (the default for everyone else — read from a
   plugin-maintained token page). Setup writes the first lockfile. Mention this whenever
   no `figma-token-bridge.lock.json` exists.
5. **What it needs** — depends on the strategy. `rest` needs a Figma REST token
   (e.g. `FIGMA_TOKEN`) with variables-read scope (Enterprise). `page` needs the Figma
   MCP server connected and the **Token Sync plugin** to maintain the token page — **no
   token**. Either way, *writing* to Figma needs the MCP server, plus a token source in
   the repo and edit access to the Figma file.

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
  primitive. Re-run `setup` with no flags anytime to switch strategy or re-discover the
  plugin-maintained token page.
- `--purge` — for `reset`, also delete the `.figma-token-bridge/` snapshot directory
  (pre-write backups). Off by default so reset keeps the backups as a safety net.
- `--with-page` — for `reset` in `page` strategy, also delete the **plugin-maintained**
  token page in Figma. Destructive and needs the MCP server; the page is snapshotted
  first and the deletion is confirmed separately. Note the page is shared with the Token
  Sync plugin, which will recreate it on its next run.
- `--yes` — for `reset`, skip the confirmation prompt (for scripting). Never assumed.

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

- `key` — dot-path canonical name (`color.bg.primary`), the join key between code, the
  lockfile, and the page. The Figma variable name maps to the key by swapping the group
  separator `/` for `.` (`color/bg/primary` ↔ `color.bg.primary`) and nothing else: the
  rest is verbatim, case preserved, and a key segment must not itself contain `.` or `/`.
  Confirm any richer transform with the user before relying on it.
- `type` — `color` | `number` | `string` | `boolean`
- `modes` — map of mode name → value (a token is a value *per mode*, not one value)
- `ref` — if the token aliases another token, its canonical target key (else null)
- `hash` — stable content hash of `{type, modes, ref}` after normalization

**Value normalization** (both sides apply it before hashing so cosmetic differences
don't read as drift):

- **color** → `#RRGGBB`, or `#RRGGBBAA` when alpha < 1, uppercase hex.
- **number** → JS number with trailing zeros trimmed (`4.0` → `4`).
- **string** → verbatim.
- **boolean** → `true` / `false`.

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

In `page` strategy these detected code tokens are also what the skill **fills in** on the
plugin-maintained page — for a token present in code but absent from the page, `apply`
creates the Figma variable and adds its artefact (see [Figma side](#figma-side) and
[apply](#apply)). The skill does not generate the page itself; the Token Sync plugin owns
it.

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

For non-Enterprise projects the snapshot is read from a **plugin-maintained page of
variable-bound artefacts**, governed by the **Token Page Contract (v1)** — the shared
spec the **Token Sync plugin** (the page's primary owner) and this skill both honor.
The contract is the source of truth; the operative rules are mirrored here, but defer to
it on any discrepancy. The plugin owns the page; the skill *reads* it and *fills in*
code-only tokens (see [apply](#apply)).

**Rule 0 — every artefact carries a live binding to the real `Variable`.**
`get_variable_defs` is node-scoped: it returns only variables *actually bound to a node's
properties*. A swatch painted a static color, or text that merely spells out a hex value,
exposes no variable and is invisible. The artefact is the token *applied* to a node, not
a picture of it. Decorative labels/captions are allowed but never read.

- **Page identity & discovery** — exactly one managed page per file, named
  `🔸 Tokens` (discovery relies on the name, because MCP tools see node
  names/structure, not plugin-private data). The plugin also stamps a shared marker
  (`figma_token_bridge` namespace, `managedPage=1`). `setup` resolves the page once and
  records its node id as `figmaRead.tokensPageNodeId`; later runs target that id and
  re-discover by name if it goes stale.
- **Mode sections** — Figma modes are per-collection, so the page is organized by
  **(collection × mode)**: one section frame per pair, named exactly
  `"<Collection> / <Mode>"` (e.g. `Brand / Dark`), with the explicit mode set for that
  collection so every artefact inside *resolves* at that mode. A section holds only
  artefacts for its collection's variables. `figmaRead.modes` records the union of mode
  names seen (informational); the section names carry the authoritative collection+mode
  pairing.
- **Artefact → binding (one per variable per section, layer-named the canonical key):**

  | Variable type | Artefact | Bound property |
  |---|---|---|
  | `COLOR` | Rectangle (swatch) | fill paint color |
  | `FLOAT` | Frame | `width` |
  | `STRING` | Text node | `characters` |
  | `BOOLEAN` | Frame container + child | child's `visible` (binding is reported even when hidden; the named container stays enumerable) |

  Identity is **(canonical key, section)** — upserts match by layer name *within a
  section*; never a second artefact for the same key in the same section.
- **How to read** — (1) resolve the page (`tokensPageNodeId`, else by name); (2)
  `get_metadata` on the page to enumerate section frames (parse `"<Collection> / <Mode>"`)
  and their artefact children (names = canonical keys); (3) for each section,
  `get_variable_defs` scoped to its nodes resolves each bound variable's `name → value`
  *at that section's mode*; (4) normalize into canonical rows `{key, type, modes, ref,
  hash}` — key from the layer name, mode from the section, value per
  [§ normalization](#reading-and-normalization). This feeds the three-way merge unchanged.
- **Coverage & caveats** — coverage is exactly "what's on the page": a variable with no
  artefact is *unknown*, never a deletion (the deletion guard below). Alias/`ref` fidelity
  is best-effort — `get_variable_defs` resolves the *value*, not the `VARIABLE_ALIAS`
  target, so resolved values are authoritative and relinks are flagged conservatively.
- **Scope: variables only** — the contract (and this skill) cover `COLOR` / `FLOAT` /
  `STRING` / `BOOLEAN` variables. Figma **styles** (paint/text/effect/grid) are out of
  scope and live in a separate region outside the `"<Collection> / <Mode>"` sections;
  never enumerate a style tile as a token artefact.

A bare `get_variable_defs` on an arbitrary selection remains available as an ad-hoc
spot-check, but it is not a strategy — it only sees the current selection.

**Load `figma-use` only before a write**, never for ordinary reads — with one exception:
the skill's `page`-strategy **fill-in** write (see [apply](#apply)) uses `figma-use` to
create a code-only variable and add its artefact. The plugin remains the page's primary
owner; the skill only contributes the missing pieces.

#### Never read a partial table as deletion

A token present in the lockfile but absent from the Figma read is only a real deletion
if the read was **complete**. Because a scoped or failed read can omit variables that
still exist, guard against false `flag`s — and "complete" means something different per
strategy:

- **`rest`** — only classify absent-in-figma tokens as `deleted in figma` when the
  snapshot came from a REST read that **succeeded and returned a non-empty table**.
- **`page`** — a token absent from the read is **not** a deletion just because it lacks
  an artefact; absences are *unknown*. A genuine page-mode deletion is only an artefact
  that was present in the lockfile base and is now gone from the page (the Token Sync
  plugin deletes an artefact when its variable is removed in Figma). When in doubt, treat
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
figma` row fires only when an artefact that was in the base is genuinely gone from the
managed page, never when a variable merely lacks an artefact (see the deletion guard
above).

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
  `tokensPageNodeId` is the plugin-maintained page (discovered by name on `setup`),
  and `modes` is the informational union of mode names seen across its
  `"<Collection> / <Mode>"` sections (the section names carry the authoritative
  collection+mode pairing).
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
   *and* the user passed an explicit delete resolution. In `page` strategy the write is a
   **fill-in contribution** to the plugin-maintained page, per the Token Page Contract:
   for a token present in code but absent from the page, create the underlying Figma
   variable in the right collection and type, then **upsert** its bound artefact —
   identity is `(canonical key, section)`, so add one artefact per `(collection, mode)`
   section, layer-named the key, ensuring the section exists per the contract. Upserts
   are idempotent and must never duplicate, reorder, or clobber the plugin's artefacts.
   The binding is the source of truth — no shared hash is required — so the plugin picks
   up the new variable natively on its next sync and the two writers converge. A
   designer's edit to a bound variable is captured automatically on the next read.
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
- In `page` strategy, `--force code` writes the forced values onto the Figma variables
  and upserts their artefacts on the plugin-maintained page (same `(key, section)`
  identity as a normal fill-in) so the forced state is what future reads see — without
  clobbering or reordering the plugin's other artefacts.

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
Figma, binds the file, discovers/adopts the plugin-maintained token page (`page`
strategy), and writes the first lockfile.

1. **Trigger.** Runs automatically when a read/write verb finds no lockfile (per
   [Preflight](#preflight)). Also invokable explicitly as `/figma-token-bridge setup` to
   reconfigure at any time — switch strategy, rebind the file, or re-discover the page.

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

5. **`page` branch.** Confirm the **Figma MCP server is connected** (stop and request it
   if not). **Discover the plugin-maintained token page** named `🔸 Tokens` in the
   bound file (via `get_metadata`), and record its node id. If no such page exists, the
   Token Sync plugin hasn't run yet — tell the user to run it once to create the page
   (even an empty one), then re-run `setup`; the skill does **not** generate the page
   itself. With the page found, read its `"<Collection> / <Mode>"` sections to capture
   the union of mode names, and set
   `figmaRead = { "strategy": "page", "tokensPageNodeId": "...", "modes": [...] }`. Any
   token present in code but absent from the page is filled in later by `apply` (see
   [apply](#apply)), not during setup.

6. **Write the baseline lockfile (this is `adopt --init`).** With no prior base every
   differing token would look like a conflict, so reconcile once — help the user settle
   divergence, or pick a side for the initial state — then write
   `figma-token-bridge.lock.json` with `figmaFile`, `figmaRead`, and the agreed `tokens`
   base. Subsequent runs use real three-way merge and reuse the stored file and strategy.

`setup` is the guided wrapper; the baseline write it performs is exactly `adopt --init`,
which remains available as the explicit, non-interactive primitive for users who already
know their strategy or are scripting.

## Reset

`reset` clears this project's figma-token-bridge state so it can start over from a clean
`setup`. Use it to abandon a bad baseline, switch to a fresh Figma file, or wipe a setup
that picked the wrong strategy.

It is destructive, so **always confirm before deleting** unless `--yes` is passed: list
exactly what will be removed (and warn that the lockfile is committed, so removal shows
up as a tracked deletion recoverable from git history). Then remove, scaled by flags:

- **Default** — delete `figma-token-bridge.lock.json` (the binding, strategy, and base) and
  `figma-token-bridge.plan.json` if present. The `.figma-token-bridge/` snapshots are kept
  as a safety net.
- **`--purge`** — additionally delete the `.figma-token-bridge/` snapshot directory.
- **`--with-page`** — additionally delete the **plugin-maintained** token page in Figma
  (`page` strategy only). This needs the MCP server connected; snapshot the page first
  (per [Safety](#safety-and-reversibility)) and confirm this Figma-side deletion
  *separately* from the local-state deletion — losing the page is not recoverable from
  git. Note the page is **shared** with the Token Sync plugin, which will recreate it on
  its next run, so this rarely makes sense; without the flag, `reset` never touches Figma
  and the page is left in place.

`reset` only removes figma-token-bridge's own artefacts — it never edits your code tokens
or (absent `--with-page`) anything in Figma. After a reset there is no lockfile, so the
next verb auto-launches `setup` exactly as on a true first run.
