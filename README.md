# joo-migrator

SQL schema migrations for [joo](https://github.com/nandolang/joo) projects.
Versioned `up`/`down` scripts under `resources/migrations`, metadata with
profile filtering, transactional batches, a cross-instance lock and a
checksum-verified history table. Written in joo.

Versão em português: [README.pt-br.md](README.pt-br.md).

There are two ways to run it, backed by the same engine:

- **Automatic**: import the package and call `Migrator.run()` at the top of
  `main()`. The schema is migrated before the application starts serving.
- **Manual**: a standalone CLI with `up`, `down`, `status`, `validate` and
  `create`, independent from the application.

## Install

```
joo install github.com/nandolang/joo-migrator
```

Database drivers come from the consuming project; the migrator picks the SQL
dialect from the driver name but does not bundle drivers itself. The one
exception is SQLite (`modernc.org/sqlite`, pure Go), which ships with the
package for dev/test use. For the rest, add whatever your project already
uses:

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

`Migrator.run()` reads the same datasource the application uses
(`db.driver`/`db.url` from `resources/properties[-<env>].json` or the
`DB_DRIVER`/`DB_URL` env vars) and migrates `resources/migrations` under the
`JOO_ENV` profile. To override any of that:

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
`--history-table`, `--lock-table` and `--lock-ttl-seconds`. Flags you don't
pass fall back to the same props/env resolution the automatic mode uses.

## Migration layout

Each migration is a folder named `NNN_name` (numeric code, underscore, name):

```
resources/migrations/
  000_db_init/
    up.sql             # required: the forward migration
    down.sql           # required: the revert (empty = irreversible on purpose)
    migration.yml      # optional: execution metadata
  001_seed_dev_data/
    ...
```

Codes set the execution order (ascending) and must be unique. Gaps are fine.
`up.sql` runs as a whole file, so multiple statements work on every dialect
(the MySQL URL gets `multiStatements=true` added automatically). `down.sql`
must exist; keep it empty only when the change genuinely cannot be reverted,
in which case `down` refuses to touch it.

### Metadata (`migration.yml`, `.yaml` or `.json`)

```yaml
description: "seed data for local development"
profiles: [dev]        # empty/absent = runs under every profile
```

The active profile is `JOO_ENV`, the same value that picks
`properties-<env>.json`. Override with `--profile` or `.profile(...)`. A
migration skipped by profile (say, dev-only seeds in prod) stays pending
under that profile and is applied on the first matching run; execution order
stays ascending within each run.

## Transaction semantics

| Dialect | Strategy |
|---|---|
| PostgreSQL / CockroachDB | One transaction wraps the whole batch. If migration 10 of 10 fails, the database (history included) stays as it was. |
| SQLite | Same, one transaction per batch. |
| MySQL / MariaDB | MySQL commits implicitly on every DDL statement, so a batch cannot be atomic there. Migrations run one by one; on failure the batch is compensated with each `down.sql` in reverse order, best effort, and the returned error reports what happened to each step. Write MySQL `down.sql` scripts to tolerate a partial apply (`DROP TABLE IF EXISTS ...`). |

Checksum drift (an applied migration whose `up.sql` changed on disk) fails
`up`, `down` and `validate`. Create a new migration instead of editing an
applied one.

## Locking

Instances that boot together migrate exactly once. A single-row lock table
(`JOO_MIGRATIONS_LOCK`) is taken before planning and released after the run,
on every path. The expiry deadline is computed by the app in unix millis, so
comparisons never depend on the database clock; a crashed holder frees itself
after the TTL (15 min default), and expired locks are stolen with an
optimistic guard so two stealers cannot both win.

## Control tables

Both created automatically, both names configurable:

- `JOO_MIGRATIONS_HISTORY`: code, id, checksum, applied_at, execution_ms,
  profile. Writes ride the batch transaction, so a rolled-back batch leaves
  no history rows behind.
- `JOO_MIGRATIONS_LOCK`: the single-row lock.

## Configuration resolution

Explicit config (flags / `MigratorConfig`) > environment variables
(`DB_DRIVER`, `DB_URL`) > `resources/properties-<env>.json` >
`resources/properties.json`. The migrator opens its own small pool (one
connection, five-minute lifetime) and always closes it, so it can't pile up
connections regardless of the application's own pool settings.

## Supported databases

`postgres` / `postgresql` / `pgx` / `cockroach` / `cockroachdb`,
`mysql` / `mariadb`, `sqlite` / `sqlite3`. Adding a database means writing
one `Dialect` class and registering it in `Dialects.forDriver`; the engine
and the stores stay untouched.
