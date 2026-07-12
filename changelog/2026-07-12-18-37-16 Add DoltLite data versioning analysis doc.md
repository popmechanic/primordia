# Add DoltLite data versioning analysis doc

## What changed

Added `docs/doltlite-data-versioning-analysis.md` — a deep analysis of whether
DoltLite (a SQLite fork with git-like commit/branch/merge/diff/reset semantics built
on a content-addressed prolly tree) could close Primordia's data-safety gaps for
vibe-coded changes. Analysis only; no code changes.

## Why

Primordia versions code with full git rigor but has no versioning for data at all:
the SQLite DB is untracked, moves only via `VACUUM INTO` copy-forward snapshots,
migrations run blind against real prod data at deploy time, and rollback restores
code while rolling data forward. An agent that damages production data leaves no
restore point. The doc inventories the current data lifecycle (all five copy sites,
the accept/rollback data flows, agent guardrail gaps), maps DoltLite v0.11.28's
verified capabilities onto the three highest-weighted failure modes (agent destroys
prod data, data rollback parity, untested migrations), documents concrete
incompatibilities (no `VACUUM INTO`, no WAL, `node:sqlite`-shaped binding with
unconfirmed Bun support, write overhead, 0.x maturity), compares against the best
achievable stock-SQLite alternatives, and recommends a three-phase path: stock-SQLite
hardening now, a Bun validation spike, DoltLite as a read-only safety sidecar (data
diff panel + per-table restore points), and only then consideration as the primary
engine.
