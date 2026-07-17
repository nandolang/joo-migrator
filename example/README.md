# joo-migrator example

A minimal application consuming the migrator in **automatic mode**:
`Migrator.run()` at the top of `main()`, datasource from
`resources/properties.json`, one migration under `resources/migrations`.

## Build & run

```
joo install     # restores github.com/nandolang/joo-migrator into the global cache
joo run         # migrates example.db, then "starts" the app
```

A second `joo run` prints `schema is current` — the ledger
(`JOO_MIGRATIONS_HISTORY` inside `example.db`) already records `000_init`.
