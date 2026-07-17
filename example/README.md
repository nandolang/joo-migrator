# joo-migrator example

A minimal application consuming the migrator in **automatic mode**:
`Migrator.run()` at the top of `main()`, datasource from
`resources/properties.json`, one migration under `resources/migrations`.

## Status

The `deps.joo` entry assumes the package's published module path
(git-as-registry). Until the package is published to a git remote,
`joo install` cannot restore it — the example exists as the reference consumer
layout and becomes buildable with the first published tag.
