# DoltLite for Primordia: Version-Controlled Data Analysis

**Status: analysis only — no code changes.** Researched 2026-07-12 against DoltLite v0.11.28.

This document analyzes whether [DoltLite](https://github.com/dolthub/doltlite) — a SQLite
fork that replaces the B-tree storage engine with a content-addressed prolly tree, giving
git-like commit/branch/merge/diff/reset semantics *inside* the database — could close
Primordia's data-safety gaps. Primordia's core risk model is that agents vibe-coding
changes for non-technical users can make **irrevocable data changes**, and today the
data layer has no undo at all.

The analysis is weighted toward three failure modes (in priority order):

1. **Agent destroys prod data** — an accepted change runs a bad migration or destructive
   write against production data.
2. **Data rollback parity** — deep rollback restores old code but keeps new data; there is
   no way to roll data back alongside code.
3. **Untested migrations** — schema migrations are only truly exercised against real prod
   data at deploy time.

**Bottom line up front:** DoltLite is a remarkably precise conceptual fit — it versions
data the same way Primordia already versions code, and it would turn "agent broke the
data" from an unrecoverable incident into `dolt_reset`. But at v0.11.x it is ~3.5 months
old, has no production-readiness claim, lacks the two SQLite features Primordia's
plumbing is built on (`VACUUM INTO` and WAL), and its JS binding mimics `node:sqlite`
rather than `bun:sqlite` with Bun support unconfirmed. The recommendation (§8) is a
phased approach: capture most of the safety value **now** with stock-SQLite changes that
are also the prerequisite refactors for DoltLite, run a validation spike, then adopt
DoltLite first for the highest-value/lowest-risk surface — data diff and data rollback —
rather than a wholesale storage-engine swap.

---

## 1. Where Primordia's data stands today

Primordia versions **code** with full git rigor: worktree per session, branch parentage
markers, blue/green slots, deep rollback from `primordia.productionHistory`. **Data**
gets none of that. The single SQLite file `.primordia-auth.db` is:

- **Untracked by git** (`.gitignore` ignores `*.db`, `*.db-shm`, `*.db-wal`), living
  one-copy-per-worktree, resolved as a bare relative path against `process.cwd()`
  (`lib/db/sqlite.ts:25`).
- **Moved only by copy-forward snapshots.** There are exactly five copy sites, all built
  on `VACUUM INTO`:
  | Site | Direction | Code |
  |---|---|---|
  | Session create | prod → new worktree | `lib/evolve-sessions.ts:908` |
  | Apply Updates hot-swap | prod → running preview | `upstream-sync` → `hotswap-db` via `lib/evolve-sessions.ts:672` |
  | Accept | **old prod slot → new prod slot** | `scripts/install.sh:650` |
  | Admin rollback | **current prod → rollback target** | `app/api/admin/rollback/route.ts:207` |
  | Emergency CLI rollback | current prod → previous slot | `scripts/rollback.ts:112` |
- **Migrated by ad-hoc boot-time code** — idempotent `CREATE TABLE IF NOT EXISTS` plus
  try/catch `ALTER TABLE` statements that run on every open (`lib/db/sqlite.ts:31-241`).
  There are no versioned migration files and no migration ledger beyond a few
  `instance_config` flags.
- **Deleted without archive** on reject (`manage/route.ts:652`) and on server-health
  worktree cleanup (`server-health/route.ts:243`) — the session's NDJSON log is archived,
  its DB snapshot is not.

Two structural consequences matter for the weighted failure modes:

**Accept discards preview data and re-runs migrations blind.** On accept, `install.sh`
overwrites the session worktree's DB with a fresh snapshot of the *old prod slot's* DB.
Everything the agent did to data during the preview — including the very schema
migration it developed and manually verified against the snapshot — is thrown away. The
schema change then re-executes against real prod data on the new slot's first boot, at
the only moment it has ever met that data. If the migration is wrong (drops a column,
corrupts a backfill, deletes rows), it executes against the live DB. A pre-migration
restore point *does* incidentally exist at that moment — the old slot's DB file is left
untouched by accept, and old slots accumulate indefinitely, so disk actually holds an
unmanaged lineage of point-in-time prod snapshots. But it exists by accident, not by
design: nothing surfaces it as a backup, no restore tooling targets it, and — see the
next paragraph — the tools that do touch it destroy it. This is failure modes 1 and 3
in a single mechanism.

**Rollback deliberately rolls data forward — and destroys the backup as it does so.**
Both rollback paths copy the *current* prod DB into the older slot before switching
traffic, so users stay logged in and no accounts vanish. That is the right default for
platform/auth data — but it means rollback can produce an incoherent pairing (old code +
new-schema data), and it is exactly wrong for the case where the reason you are rolling
back **is** the data damage. Worse: that `copyDb` step **overwrites the old slot's
pristine pre-damage DB with the damaged current data** — so the natural incident
response to "the deploy corrupted my data" (roll back) permanently destroys the only
pre-damage copy as its first action, with no archive taken. The other tool that touches
the incidental backup lineage, server-health oldest-worktree cleanup, likewise deletes
the slot's DB without archiving it (the session NDJSON log *is* archived; the DB is
not). The system keeps accidental backups and both of its management tools destroy
them. And there is no way to express "restore yesterday's `orders` table but keep
today's `sessions` table."

Finally, the agents themselves: evolve workers run with `bypassPermissions`
(`scripts/claude-worker.ts:246-247`), receive **no prompt guidance about the database**,
and are free to edit `lib/db/sqlite.ts` and execute arbitrary SQL. The only protection
is structural — they operate on a snapshot. That protection evaporates at accept time
for schema changes, which is precisely the weighted risk.

This asymmetry will get worse, not better. Today the DB holds platform data (auth,
roles, push subscriptions, events). The whole point of Primordia is that vibe-coded
features will create *user-app tables* — todo lists, inventories, customer records —
owned by users who cannot write a recovery script. For them, "the agent emptied my
table" must be a button, not a support incident.

## 2. What DoltLite actually provides

Verified against the repo README, release notes, and the `@dolthub/doltlite` npm package
(v0.11.28, 2026-07-09). Everything below is exposed through the unmodified SQLite C API
as SQL functions and virtual tables, so it works through any binding.

**Version control primitives:**

- `dolt_add()`, `dolt_commit()` (returns commit hash), `dolt_log`, `dolt_tag()`,
  `dolt_status`, `dolt_hashof()`.
- `dolt_branch()` / `dolt_checkout()` with **per-connection branch state** — different
  connections to the same file can sit on different branches simultaneously.
- `dolt_merge()`: three-way **row-level** merge; non-conflicting changes to different
  rows of the same table auto-merge. Conflicts surface in `dolt_conflicts` /
  `dolt_conflicts_<table>` tables (base/ours/theirs columns) and are resolved with
  SQL or `dolt_conflicts_resolve('--ours'|'--theirs', ...)`. Post-merge constraint
  violations (FK/unique/check) are recorded in `dolt_constraint_violations_<table>`
  rather than enforced inline, and `dolt_commit` refuses to proceed while any remain.
- `dolt_diff` / `dolt_diff_stat()` / `dolt_diff_summary()` / `dolt_schema_diff()` /
  per-table `dolt_diff_<table>` virtual tables. Diff cost is O(changes), not
  O(table size).
- **Undo:** `dolt_reset('--hard')` (discard uncommitted work), `dolt_revert(hash)`
  (new commit inverting an old one), cherry-pick, rebase.
- **Point-in-time reads:** `SELECT * FROM dolt_at_users('<commit|branch|tag>')`.
- `dolt_history_<table>` and `dolt_blame_<table>` — row-level history and "which commit
  last touched this row."
- Remotes: `dolt_push` / `dolt_fetch` / `dolt_pull` / `dolt_clone` over `file://` and
  `http://`, content-addressed (only missing chunks transfer). Bearer-JWT auth shipped
  in v0.11.28 — days old at time of writing.

**Storage:** single file, content-addressed chunk store with structural sharing (the
README's example: changing 1 row in a 10K-row table adds 1.9% to file size). History
accumulates until `dolt_gc()` (stop-the-world mark-and-sweep; `VACUUM` is aliased to
it).

**JS binding:** `@dolthub/doltlite` is a native N-API addon (prebuilds for Linux/macOS
x64+arm64, Windows x64) whose API is a drop-in for **`node:sqlite`**
(`DatabaseSync`/`StatementSync`), plus convenience wrappers (`doltCommit()`,
`doltMerge()`, `doltReset()`, `doltDiff()`, …). `bun add` is listed as an install
option, but tested Bun support is unconfirmed, and the API is *not* compatible with
`bun:sqlite`'s `Database`/`query()` shape — Primordia would need a thin adapter either
way. A WASM package also exists.

## 3. Fit against the weighted failure modes

### 3.1 Agent destroys prod data (highest weight)

This is where DoltLite is strongest, because the failure is a *data* mutation and
DoltLite makes data mutations reversible by construction.

- **Restore point before every dangerous moment.** A `dolt_commit` before accept-cutover
  and before boot-time migrations means any destructive write or migration executes on
  top of a named, immutable restore point *in the same file*. Recovery is
  `dolt_reset('--hard', <tag>)` — no hunting for the right slot snapshot, no losing
  post-cutover writes wholesale (they're commits too, so selective `dolt_revert` of just
  the bad change is possible while keeping good writes that came after).
- **Data diff as a review gate.** Primordia's session page already shows a "Files
  changed" git diff. DoltLite makes the equivalent **"Data changed"** panel almost free:
  `dolt_diff_stat()` between the session-start commit and the working set shows
  "orders: 3 rows modified, 1,240 rows deleted" *before* the user clicks Accept. For a
  non-technical user, "this change will delete 1,240 rows" is the single most valuable
  safety signal Primordia could show — it converts silent data damage into an informed
  decision. Nothing in stock SQLite provides this without building shadow-table
  infrastructure by hand.
- **Row-level forensics.** `dolt_history_<table>` and `dolt_blame_<table>` answer "what
  happened to my data and when" after the fact, which today is unanswerable.

Caveat: none of this constrains the agent *upfront* — an agent with `bypassPermissions`
could in principle run `dolt_reset` itself or edit history. DoltLite is an undo layer,
not a sandbox. (In practice agents don't know about or fight the versioning layer, and
the accept pipeline — not the agent — would own commit/tag creation. But prompt-level
guardrails and keeping VC operations out of the agent's vocabulary remain worth doing
regardless.)

### 3.2 Data rollback parity (second weight)

Today: code rolls back, data rolls forward, by copy. With DoltLite, rollback becomes a
*choice per table* instead of a single file-level copy direction:

- Deep rollback can `dolt_reset` the data to the tag created when the target slot was
  deployed — true code+data coherence — **or** keep data current, **or** mix: check out
  the old commit, then re-apply platform tables (`sessions`, `users`, `passkeys`) from
  the current branch with plain SQL, preserving the "nobody gets logged out" property
  that motivated today's copy-forward design while still restoring damaged user-app
  tables.
- The rollback UI can show `dolt_diff_stat` between "now" and each restore point, so an
  admin sees exactly what data a rollback would forfeit before pressing the button.
  Today that information does not exist.

This per-table selectivity is the capability gap stock SQLite genuinely cannot close:
file-level snapshots force all-or-nothing decisions, which is why current rollback picks
"nothing" (data-wise) and cannot help with data damage at all.

### 3.3 Untested migrations (third weight)

DoltLite reframes the migration problem the way git reframed deployment: **promote the
migrated data, don't re-run the migration.** In the target design, the preview branch's
DB is a DoltLite branch of prod's data; the agent's migration runs against it during the
session (as it already does today, against a snapshot); at accept, instead of discarding
that work and re-executing migrations blind, the pipeline `dolt_merge`s prod's interim
writes into the migrated branch (or vice versa) — three-way, row-level, with conflicts
surfaced rather than silently lost.

This is the most valuable *and* most speculative fit, for two reasons:

1. **Schema-change merging is undocumented in DoltLite.** Row-level merge is documented;
   what happens when the branches' *schemas* diverge (the exact migration case — one
   side added a column) is not. Dolt-the-parent-project aborts and asks you to align
   schemas first; DoltLite's README doesn't say, and its `dolt_schema_conflicts`
   equivalent doesn't appear to exist. Until validated empirically, assume migration
   promotion requires the merge to happen *before* the schema change diverges, or a
   re-run of the (already-rehearsed) migration on top of a merge — still a large
   improvement, since the migration was exercised against a *branch of the real data*
   and the pre-migration commit is a restore point.
2. Primordia's migrations are boot-time imperative code, not data operations DoltLite
   can see. Getting full value here means also adopting a real migration ledger (ordered
   migration files + a `schema_migrations` table) so "which migrations ran" is itself
   recorded data. That refactor is worth doing with or without DoltLite (§7).

Even in the weakest validated form — commit before migrations run, migration rehearsed
on branched real data, `dolt_reset` if boot-time execution fails — this failure mode
improves from "irreversible surprise" to "rehearsed, with a restore point."

### 3.4 Preview/prod divergence (deprioritized, noted for completeness)

Apply Updates currently discards all preview data via snapshot hot-swap. With DoltLite,
it becomes `dolt_pull`/`dolt_merge` of prod's data branch into the preview branch —
preview test data survives upstream syncs, and content-addressed transfer moves only
changed chunks. Nice, not urgent.

## 4. What adoption breaks or costs

The research surfaced concrete incompatibilities with Primordia's existing plumbing.
None are fatal; all are real work:

| Today | Under DoltLite | Impact |
|---|---|---|
| All 5 copy sites use `VACUUM INTO` | **`VACUUM INTO` errors** ("not supported for doltlite databases"); substitute is `dolt_clone('file://...')` to an empty destination, or push/pull | Every copy site rewritten; `dolt_clone` copies *history*, not a compacted single-version file — different size/semantics |
| `PRAGMA journal_mode=WAL` (`lib/db/sqlite.ts:29`) | **No WAL.** Writes serialize behind one exclusive file-level lock; reads stay concurrent | Fine for Primordia's actual load (low-QPS platform writes); loses concurrent-writer headroom for future write-heavy user apps |
| `bun:sqlite` built into Bun, zero deps | Native N-API addon, `node:sqlite`-shaped API, **Bun support unconfirmed** | Needs a compatibility adapter in `lib/db/`; needs an empirical Bun spike before anything else; new native dependency must pass the `bunfig.toml` Socket/minimum-release-age gates and adds a supply-chain surface |
| SQLite ~zero write overhead | ~1.3–1.4× batched writes, **~4× single-row autocommit writes** (their own benchmarks) | Primordia's writes are almost all single-statement autocommit (`lib/db/sqlite.ts` adapter); mitigable by batching, but real |
| WAL-file sidecar handling everywhere | No sidecars; instead in-file history growth until `dolt_gc()` (stop-the-world) | Simpler file handling; needs a GC schedule (server-health page is a natural home) |
| Stock file readable by any sqlite3 tooling | Prolly-tree format; interop is `ATTACH` + `INSERT INTO ... SELECT` | One-time import path exists; ecosystem tools (sqlite3 CLI, DB browsers) can't open the file |

Other documented divergences that touch Primordia lightly: `events` is a `STRICT`
table with indexes — per-table *row-level staging* is unimplemented on tables with
secondary indexes (whole-table staging still works, which is all Primordia needs);
non-INTEGER primary keys become `WITHOUT ROWID` (Primordia uses TEXT PKs widely — needs
a correctness pass); the PRAGMA support matrix is undocumented.

**Maturity is the overriding cost.** DoltLite launched ~late March 2026 and is at
v0.11.x with a very rapid release cadence — 10 releases in the last three weeks, and
recent changelogs are dominated by correctness fixes (GC safety under concurrent
workloads, index-seek bugs, unique-index delete bugs). The codebase is substantially
AI-generated (documented openly by DoltHub). Open issues include durability of GC's
rename on some platforms, no atomic commits across attached databases, and history
tables missing rows in edge cases. Test rigor is genuinely impressive (full SQLite TCL
regression suite, 5.7M-statement SQL logic corpus, differential oracle tests vs stock
SQLite and Dolt, ASan/UBSan), and DoltHub has a strong track record with Dolt itself —
but DoltHub makes no production-readiness claim, and neither should we. **Handing the
sole authoritative copy of user data to a 0.x storage engine to gain safety would be
trading a known risk for an unknown one.** This tension resolves in the phasing (§8):
adopt DoltLite first in roles where *it is not the only copy*.

## 5. The topology question

DoltLite offers two integration shapes, and the right one preserves Primordia's
existing structure:

**A. One shared DB, branch-per-session (rejected).** All worktrees open the same
DoltLite file; each preview connection checks out its session's data branch.
Maximally elegant, but it makes every preview server a writer to the production data
file through a young storage engine, funnels all writes through one exclusive file
lock across processes (multi-process semantics beyond the lock are unconfirmed), and
couples preview crashes to prod availability. Too much blast radius.

**B. Per-worktree files, history inside each file (recommended).** Keep today's
copy-per-worktree topology exactly — one DB file per slot, physically isolated — but
each file is a DoltLite file carrying its own commit history. Copies become
`dolt_clone`/`dolt_pull` instead of `VACUUM INTO`; accept commits + tags in the new
prod file; rollback is `dolt_reset` within the prod file (with per-table carve-outs)
rather than a cross-slot copy. Physical isolation (the thing that already works) is
retained; version control is added *within* each file. All of §3's benefits survive
this shape; only cross-process branch sharing is given up, and nothing in §3 needed it.

## 6. Alternatives with stock SQLite (the baseline to beat)

Per the agreed posture, DoltLite must beat the best stock-SQLite alternative, not the
status quo. What's achievable without it:

1. **Protect the restore points that already exist.** The old-slot DB files are already
   a point-in-time backup lineage (§1) — the pre-migration state survives every accept
   untouched. What's missing is not snapshot creation but snapshot *protection*: (a)
   admin/CLI rollback must archive the target slot's DB (one `VACUUM INTO` to a
   timestamped file) before `copyDb` clobbers it; (b) server-health cleanup must archive
   a slot's DB before `worktree remove` deletes it, exactly as it already does for the
   NDJSON log; (c) optionally, a timestamped archive at accept-cutover so restore points
   survive independently of slot lifecycle. Closes the worst of failure modes 1 and 3's
   *unrecoverability* at all-or-nothing granularity, for a few dozen lines across three
   files. **This should happen regardless of any DoltLite decision.**
2. **Archive session DBs like NDJSON logs** on reject (they're already archived
   for logs; the DB copy is currently just deleted).
3. **Migration ledger** (ordered migration files + `schema_migrations` table) replacing
   ad-hoc boot-time try/catch. Enables "which migrations will run on accept" display and
   makes migrations testable artifacts. Also a prerequisite for getting full DoltLite
   value (§3.3).
4. **SQLite's session extension** (`sqlite3session`) — the closest stock analog to
   DoltLite: records row-level changesets that can be inverted (undo) and diffed. But
   `bun:sqlite` does not expose it, changesets must be captured *while writes happen*
   (no retroactive diff), there is no branching/merging, and building UI-grade data diff
   on it is a from-scratch infrastructure project. This is the "best achievable
   alternative" for row-level undo, and it costs more engineering than DoltLite for
   less capability.
5. **Trigger-based audit shadow tables** — per-table `_history` tables maintained by
   triggers. Works, but must be generated and maintained for every table *including
   ones agents create dynamically*, doubles write cost, and reinventing diff/restore
   tooling on top is substantial.

The honest comparison: options 1–3 are cheap, engine-agnostic, and capture perhaps half
of the safety value (recoverability). What they cannot provide at any reasonable cost is
**row-level data diff for the accept gate, per-table selective rollback, and
merge-instead-of-discard data flows** — the capabilities that make data safety *elegant*
rather than merely possible. Those are DoltLite's genuine differentiation, and options
4–5 confirm that replicating them on stock SQLite costs more than adopting the engine.

## 7. Dependency-principle assessment

CLAUDE.md's design principles say minimal dependencies. DoltLite is a heavyweight one: a
native storage engine fork, 0.x, from a vendor whose flagship is a different product.
Points in its favor: Apache-2.0 (+ SQLite public domain), no runtime services, single
vendor with a decade of prolly-tree production experience in Dolt, and an API surface
that is plain SQL (low lock-in for *code*; the file format is the lock-in, mitigated by
`ATTACH`-based export back to stock SQLite). Points against: supply-chain surface of a
native addon, unconfirmed Bun support, and the maturity profile in §4.

Verdict: justified **only** for the capabilities that cannot be replicated cheaply
(data diff, selective rollback, data merge), **only** after a Bun spike passes, and
**only** phased so DoltLite is never the sole copy of user data until it has earned
trust in production-adjacent roles.

## 8. Recommendation: three phases

**Phase 0 — Stock-SQLite hardening (do now; no new dependencies).**
Stop destroying the incidental backups: archive the target slot's DB before rollback's
`copyDb` overwrites it and before server-health cleanup deletes it; archive (not
delete) session DBs on reject; add a timestamped prod archive at accept-cutover; adopt
a migration ledger. These are independently correct, close the unrecoverability hole
this week — mostly by *protecting restore points that already exist* rather than
building new machinery — and are the refactors DoltLite integration would require
anyway (single choke point for "copy the DB", migrations as data). Add a sentence to
the agent prompt telling agents the DB is snapshot-isolated and that destructive data
operations should be flagged in their summary.

**Phase 1 — Validation spike (timeboxed, throwaway).**
In a scratch worktree: `bun add @dolthub/doltlite`; run Primordia's schema + boot
migrations through it under Bun; measure write overhead on realistic event-insert load;
empirically answer the two documented unknowns — schema-divergent `dolt_merge`
behavior, and `dolt_clone`-as-`VACUUM INTO`-replacement semantics (size, speed, locking).
Kill criteria: binding fails under Bun; schema merge silently loses data; write
overhead breaks the preview UX.

**Phase 2 — DoltLite as the safety sidecar (first real adoption, low blast radius).**
Keep `bun:sqlite` as the live engine. At session start and at accept, mirror the DB
into a DoltLite file (ATTACH + `INSERT INTO ... SELECT`, committed per event) that
serves three read-only features: the **"Data changed" panel** on the session page
(diff of session-start vs pre-accept mirror), **restore points** browsable from
/admin/rollback with per-table restore (export back via ATTACH), and data history
forensics. The mirror is derived data — if DoltLite misbehaves, nothing of record is
lost. This ships the highest-weighted safety wins (visible data diff at the accept
gate; per-table restore) while the engine matures.

**Phase 3 — DoltLite as the engine (only after Phase 2 has run in production and
DoltLite reaches 1.0-grade stability).**
Swap `lib/db/` to the DoltLite binding behind the existing adapter seam
(`createSqliteAdapter` is already the single choke point); convert the five copy sites
to clone/pull; make accept commit+tag and rollback `dolt_reset` with platform-table
carve-outs; give user-app tables full branch-merge lifecycle per §3.3. Decide here, not
before, whether platform/auth tables move too or stay on `bun:sqlite` as a two-file
hybrid (open issue: no atomic commits across attached DBs).

## 9. Open questions

- Does `@dolthub/doltlite` actually load and behave under Bun? (Phase 1, blocking.)
- Schema-divergent merge semantics — documented nowhere; must be tested. (Phase 1.)
- `dolt_gc()` cadence and pause behavior at Primordia-realistic history sizes; who
  triggers it (server-health scheduler is the natural owner).
- Per-connection branch state across *processes* sharing one file — unconfirmed by
  upstream docs; Phase 2/3 as specified never relies on it, but worth clarifying with
  upstream.
- Whether the `events` table (highest write rate, STRICT + 3 indexes) should be
  versioned at all, or excluded as append-only telemetry to keep history growth and
  write overhead down.
- Upstream trajectory: auth for remotes shipped days ago; watch whether DoltHub
  commits to a stability guarantee / 1.0.
