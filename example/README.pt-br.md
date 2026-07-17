# Exemplo do joo-migrator

Uma aplicação mínima consumindo o migrator em **modo automático**:
`Migrator.run()` no início do `main()`, datasource em
`resources/properties.json`, uma migração em `resources/migrations`.

English version: [README.md](README.md).

## Build e execução

```
joo install     # restaura github.com/nandolang/joo-migrator no cache global
joo run         # migra o example.db e "sobe" a aplicação
```

Um segundo `joo run` imprime `schema is current` — o histórico
(`JOO_MIGRATIONS_HISTORY` dentro do `example.db`) já registra a `000_init`.
