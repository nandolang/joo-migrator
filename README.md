# joo-migrator

SQL schema migrations for [joo](https://github.com/joolang/joo) projects:
versioned `up`/`down` migrations under `resources/migrations`, profile-aware
metadata, transactional batches, a cross-instance lock, and a checksum-verified
history ledger. Written 100% in joo.

Two modes, one engine:

- **Automatic** — import the package and call `Migrator.run()` at the top of
  `main()`; the schema is current before the application serves its first
  request.
- **Manual** — a standalone CLI (`up`, `down`, `status`, `validate`,
  `create`), fully decoupled from the application.

## Install

```
joo install github.com/joolang/joo-migrator
```

Database drivers are **provided by the consuming project** — the migrator is
dialect-aware but driver-agnostic. SQLite (`modernc.org/sqlite`, pure Go)
ships with the package as the built-in dev/test dialect; for the others add
the driver your project already uses:

```
joo install --go github.com/go-sql-driver/mysql
joo install --go github.com/lib/pq
```

## Quick start

### Automatic mode

```joo
import std.os.Os
import joolang.migrator.Migrator

pub class Main {

	pub static void main(args []string) {
		var result, err = Migrator.run()
		if (Exceptions.eval(err)) {
			log.error("migrations failed, refusing to start: {}", err)
			Os.exit(1) // orchestrators must see the failed boot
		}
		// ... start the application
	}
}
```

`Migrator.run()` reads the datasource the application itself uses
(`db.driver`/`db.url` from `resources/properties[-<env>].json` or the
`DB_DRIVER`/`DB_URL` env vars) and migrates `resources/migrations` under the
`JOO_ENV` profile. Overrides go through the fluent config:

```joo
Migrator.runWith(MigratorConfig.new()
	.dir("db/migrations")
	.profile("prod"))
```

### Manual mode (CLI)

```
migrator create add_users          # scaffold migrations/002_add_users/
migrator up                        # apply everything pending
migrator status                    # [applied] / [pending] / [skipped] / ...
migrator validate                  # tree + ledger checks, nothing executed
migrator down --steps 1            # revert the newest applied migration
migrator down --to 4               # revert everything above code 4
```

Every command accepts `--driver`, `--url`, `--dir`, `--profile`,
`--history-table`, `--lock-table` and `--lock-ttl-seconds`; unset flags fall
back to the same props/env resolution the automatic mode uses.

## Migration layout

Each migration is a folder named `NNN_name` (numeric code, underscore, name):

```
resources/migrations/
  000_db_init/
    up.sql             # required — the forward migration
    down.sql           # required — the revert (empty = consciously irreversible)
    migration.yml      # optional — execution metadata
  001_seed_dev_data/
    ...
```

- Codes order execution (ascending) and must be unique; gaps are fine.
- `up.sql` runs as a whole file — multiple statements are supported on every
  dialect (the MySQL URL automatically gains `multiStatements=true`).
- `down.sql` must exist; keep it **empty** only for a change that genuinely
  cannot be reverted — `down` refuses to "revert" those.

### Metadata (`migration.yml`, `.yaml` or `.json`)

```yaml
description: "seed data for local development"
profiles: [dev]        # empty/absent = runs under every profile
```

The active profile is `JOO_ENV` (the same value that picks
`properties-<env>.json`), overridable with `--profile`/`.profile(...)`. A
profile-scoped migration that was skipped (e.g. dev-only seeds in prod) stays
pending under that profile and is applied the first time a matching run
happens — execution order stays ascending within each run.

## Transactional semantics

| Dialect | Strategy |
|---|---|
| PostgreSQL / CockroachDB | **One transaction wraps the whole batch** — 10 migrations with the last failing leave the database untouched, ledger included. |
| SQLite | Same — one transaction per batch. |
| MySQL / MariaDB | **No transactional DDL** (every DDL statement commits implicitly). Migrations run sequentially; on failure the batch is compensated **best-effort** with each `down.sql` in reverse order, and every outcome is folded into the returned error. Write MySQL `down.sql` scripts to tolerate a partial apply (`DROP TABLE IF EXISTS ...`). |

Checksum drift — an applied migration whose `up.sql` changed on disk — fails
`up`, `down` and `validate`: create a new migration instead of editing an
applied one.

## Locking

Concurrent instances booting together migrate exactly once: a single-row lock
table (`JOO_MIGRATIONS_LOCK`) is taken before planning and released after the
run, on every path. The expiry deadline is app-computed (unix millis), so a
crashed holder frees itself after the TTL (15 min default) and comparisons
never depend on the database clock; expired locks are stolen with an
optimistic guard so two stealers cannot both win.

## Control tables

Created automatically (both names configurable):

- `JOO_MIGRATIONS_HISTORY` — code, id, checksum, applied_at, execution_ms,
  profile. Writes ride the batch transaction: a rolled-back batch leaves no
  ledger rows.
- `JOO_MIGRATIONS_LOCK` — the single-row lock.

## Configuration resolution

Explicit config (flags / `MigratorConfig`) > environment variables
(`DB_DRIVER`, `DB_URL`) > `resources/properties-<env>.json` >
`resources/properties.json`. The migrator opens its own deliberately tiny
pool — one connection, five-minute lifetime — and always closes it, so it can
never flood the database regardless of the application's pool settings.

## Supported databases

`postgres` / `postgresql` / `pgx` / `cockroach` / `cockroachdb`,
`mysql` / `mariadb`, `sqlite` / `sqlite3`. Adding a database = implementing
one `Dialect` class and registering it in `Dialects.forDriver` — the engine
and the stores never change.
