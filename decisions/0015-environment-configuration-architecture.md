# ADR-0015: Environment Configuration Architecture

## Status

Accepted

## Record Completeness

Complete

## Context

ADR-14 named the roles (`test / sandbox / staging / prod`) but left the *mechanism*
undecided, and the gap was actively costing correctness:

- `swarm/kernel/config/runtime.exs` fell back to `System.get_env("SWARM_DB_NAME",
  "swarm_dev")` outside `:test` — an unset env var silently connected to
  `swarm_dev`, which forced a second concept ("conditional-prod") purely to stop
  tooling from mutating that accidental target.
- `hive/docker-compose.yml` repeated the same `${SWARM_DB_NAME:-swarm_dev}`
  literal in two services — the same footgun at the deployment layer.
- `hive/docker-compose.yml` still `include:`s `../swarm/dev/docker-compose.yml`
  for the `postgres` service, which pins the plain `pgvector/pgvector:pg16`
  image. The live staging instance was hand-swapped (`docker run`, not compose)
  to a ParadeDB image carrying `pg_search` (ADR-0016, swarm). A bare `docker
  compose up` recreating `postgres` would silently revert that swap and drop the
  bm25 index — the operator has been working around this with `--no-deps`
  (`board/todo/fix-compose-postgres-drift.md`).
  Naming without fixing the *mechanism* just moves the lie, so this ADR treats
  the DB-name derivation and the compose drift as one change: both are the
  compose file trusting a hardcoded/defaulted literal instead of the declared
  environment.
- `swarm_prod` is documented (`docs/STATE.md`, this ADR's own predecessor) as
  the **staging** role, yet the literal name propagates the "prod" label into
  logs, scripts, and operator habit — including this project's own coder
  writing "prod-deploy" for a staging deploy.

## Decision

**One indicator, `SWARM_ENV ∈ {test, staging, prod}`, drives config end-to-end.**
Sandbox stays a role realized by disposable clones (`swarm_slice`,
`swarm_shadow`, ad hoc snapshots) — per ADR-14, "a clone is an instrument, not a
tier" — so it is deliberately **not** a `SWARM_ENV` value; sandbox DBs are always
named explicitly at the point of use, never derived.

**1. Kernel (`swarm/kernel/config/runtime.exs`) — derive, don't default:**

```text
explicit SWARM_DB_NAME  → wins outright (sandbox clones, ad hoc probes)
else config_env() == :test → "swarm_" <> (SWARM_ENV || "test")
else SWARM_ENV is set      → "swarm_" <> SWARM_ENV
else                        → raise (refuse to guess)
```

An explicit `SWARM_DB_NAME` always wins so sandbox work (a clone named anything)
never needs to fight the derivation. Outside `:test`, an unset `SWARM_ENV` (and
no explicit override) is a **hard error at boot** — this is what retires
"conditional-prod": there is no longer a silent target for it to guard against.

**2. Deployment (`hive/`) — layered env files, no literal in the compose file:**

```text
hive/env/base.env        committed   — cross-env constants (registry, ports, plugin/data dir conventions)
hive/env/<env>.env       committed   — per-stage overrides (test.env, staging.env, prod.env placeholder)
hive/secrets.env         git-ignored — unchanged pattern, real creds only
shell                     highest    — operator override, always wins (Compose semantics)
```

`docker-compose.yml` derives the DB name the same way the kernel does —
`SWARM_DB_NAME: swarm_${SWARM_ENV:?SWARM_ENV must be set to test|staging|prod}`
— so the **hard error fires even on a bare `docker compose up`**, not only
through the wrapper. `hive/scripts/compose` is the documented convenience path
that layers the three `--env-file` flags by `$SWARM_ENV` (Compose merges
repeated `--env-file` in order; real shell exports still win over all of them);
using it is a habit, not a safety boundary — the interpolation guard is.

The old single `hive/.env` / `.env.example` are retired in favor of `env/`; a
machine-specific value (a different `OLLAMA_MODELS_DIR` path, say) is set as a
real shell export before invoking the wrapper, per the workspace rule against
hardcoding machine-specific paths (`AGENTS.md`) — it was never a *stage* concern.

**3. Compose drift fix (folded in, one root cause):** `postgres:` is defined
directly in `hive/docker-compose.yml` (no longer only via the `swarm/dev`
`include:`), pinned to the ParadeDB image actually running
(`localhost:5000/paradedb-pg16:0.24.1`, `pg_search`+`pg_cron`+`pg_stat_statements`
preload — required, ParadeDB only writes preload into a *fresh* datadir), backed
by the existing `hive_pgdata` volume declared `external: true` (reconciling
compose ownership of the hand-swapped container is a separate live action, see
Consequences). `POSTGRES_DB` derives from `SWARM_ENV` the same way.

**4. Rename by recreate, not in-place ALTER.** `swarm_prod` → snapshot → restore
into `swarm_staging` → verify parity (row counts **and** the bm25 index rebuilds
clean) → repoint the kernel (`SWARM_ENV=staging`) → retain the old DB until
verified → drop. This is a live-data operation on the one internal instance;
gated per action-class (see Consequences), not bundled into the code change.

**5. `groot` wording.** The admin-role check in `hive/plugins/web_channel` is
**already** role-based (`GROOT_ROLE` / `is_groot` test realm-role membership, not
a username match) and lives entirely inside the private `hive/` repo — grep
confirms zero references in `swarm/`. The one leak is vocabulary: `docs/STATE.md`
(public) names the role literally. This ADR scrubs that wording to describe it
generically ("a role-based admin account"); parameterizing *which* username
holds the role via `hive/`-private config is real design work and stays with
`board/ideas/users-identity-privacy.md` (item 2) rather than being half-done
twice.

## Consequences

- The `swarm_dev` silent default and the "conditional-prod" guard concept it
  required both disappear — there is nothing left to guard against once nothing
  defaults there.
- `SWARM_ENV=prod` today resolves to `swarm_prod`, which does not exist as a
  role-consistent name until the rename lands — sequencing matters: ship the
  mechanism, then rename, so `staging` never resolves to a DB literally called
  `swarm_prod`.
- The compose drift fix means a future `docker compose up -d` (no `--no-deps`)
  reconciles `postgres` into compose ownership instead of reverting it — but
  the *first* application of that change against the already-hand-swapped
  container is itself a live action: snapshot first, verify `hive_pgdata` is
  reused (not re-initialized), confirmed on real infrastructure before
  declaring the drift fixed, not just the file diff.
- Sandbox DB names (`swarm_slice`, `swarm_shadow`, clones) stay literal by
  design — the `SWARM_DB_NAME` override path exists specifically so this ADR
  does not force sandbox tooling to invent fake `SWARM_ENV` values.
- Branch↔env promotion (does `main` mean staging or prod, branch-per-env vs
  overlay-per-env) is explicitly **not** decided here — deferred to the
  go-public ADR (`board/ideas/go-public-deployment.md`), per
  `board/ideas/environment-config.md`.
- `docs/STATE.md`'s repository table currently lists `hive/ git, PUBLIC`, which
  contradicts both root `AGENTS.md` (`hive/ PRIVATE`) and `hive/AGENTS.md`
  itself, and contradicts STATE's own prose elsewhere ("hive/plugins ... private
  repo"). Corrected as part of this ADR's docs audit — `hive/` is private.

## Verification

Decorrelated council (codex/gpt-5.5 + local llama3.3:70b) on the implemented mechanism, before
the live rename: both **SOUND-WITH-CAVEATS**. codex found two real edge cases, both fixed before
merge: (1) `System.get_env/1` returns `""` for an explicitly-set-but-blank var, which the original
`is_binary` check would have treated as a real value — an env `SWARM_DB_NAME=""` would have "won"
and `SWARM_ENV=""` would have derived the database name `swarm_`; runtime.exs now normalizes blank
to unset first. (2) The original `:test` branch let an ambient `SWARM_ENV` leak into a compile-time
`:test` run (e.g. `MIX_ENV=test SWARM_ENV=staging mix test` would have targeted `swarm_staging`),
weakening the "test is always hermetic" guarantee (ADR-14) — `:test` now ignores `SWARM_ENV`
entirely and always derives `swarm_test` unless `SWARM_DB_NAME` is explicit. Compose's own
`${VAR:-default}`/`${VAR:?err}` forms were verified empirically to already treat blank as unset
(POSIX colon-form semantics), so no compose-side fix was needed. `mix test` 333/0, credo --strict,
dialyzer, and `docker compose config` (all three `SWARM_ENV` values, with/without the offline
overlay) all re-verified green after the fix. Full council transcripts: `board/journal.md`.

**Live rename executed 2026-07-01** (operator-authorized, checkpointed): snapshot
`swarm_prod` (`pg_dump -Fc`, 54MB, `tmp/snapshots/swarm_prod_pre_rename_20260701.dump`) → `createdb
swarm_staging --template=template0` (the default `template1` carried a stale glibc collation-version
stamp from before the ADR-0016 image swap — an unrelated pre-existing infra quirk, worked around by
templating from `template0` instead of touching the shared `template1`/`postgres` catalogs) → `pg_restore
--no-owner` → verified exact parity (node 1626/1626, edge 1472/1472, content 925/925, chunk 8728/8728,
scope distribution identical, `schema_migrations` 21/21, `chunk_bm25` index present and a live `@@@`
query returns real results) → repointed the kernel (`SWARM_ENV=staging scripts/compose up -d --no-deps
kernel`) → live-verified (healthy, embed round-trip 1024-dim, public-scope search 0 hits / group-scope
10 hits — no-leak holds). `swarm_prod` is retained, untouched, unchanged, pending a burn-in period before
any drop (not done this session). **Deliberately deferred:** reconciling the hand-run `hive-postgres-1`
container into compose ownership — it carries no compose labels, so `docker compose up --no-deps postgres`
could hit a container-name conflict with unpredictable resolution; safer to leave the currently-healthy
container running as-is than risk it for a non-blocking cleanup (tracked in
`board/todo/fix-compose-postgres-drift.md`). **Known residual gap:** the pre-existing, git-ignored
`hive/.env` (not touched — outside the agent's write boundary) still contains a stale explicit
`SWARM_DB_NAME=swarm_prod`, which would win over the derivation on a **bare** `docker compose up` (though
not through `scripts/compose`, whose explicit `--env-file` flags replace `.env` entirely per Compose
semantics) — the operator should retire or update that file now that `env/` covers its purpose.

## Alternatives

- **Keep defaulting `swarm_dev`, just rename the constant** — rejected: renaming
  without removing the silent default leaves the same footgun under a new label.
- **In-place `ALTER DATABASE swarm_prod RENAME TO swarm_staging`** — rejected:
  no verification window, no retained rollback target, and it does not
  independently confirm the bm25 index survives a real restore path (which the
  eventual go-public migration will also need to trust).
- **A single flat `.env` with `SWARM_ENV` switching values inline (case-statement
  in compose)** — rejected: Compose has no native conditional block; a real
  case-statement would live in the wrapper script anyway, so layered files are
  simpler and let `diff env/staging.env env/prod.env` answer "what actually
  differs" directly.
- **Do the full `groot` → parameterized-username redesign now** — rejected as
  scope creep for this ADR: it overlaps `board/ideas/users-identity-privacy.md`
  (item 2), which has not yet had its own design pass; doing it twice risks a
  half-finished version landing here.
