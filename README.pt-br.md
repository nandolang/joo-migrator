# joo-migrator

Migrações de schema SQL para projetos [joo](https://github.com/nandolang/joo).
Scripts `up`/`down` versionados em `resources/migrations`, metadados com
filtro por profile, lotes transacionais, lock entre instâncias e tabela de
histórico com verificação de checksum. Escrito em joo.

English version: [README.md](README.md).

Há duas formas de rodar, sobre o mesmo engine:

- **Automática**: importe o pacote e chame `Migrator.run()` no início do
  `main()`. O schema é migrado antes de a aplicação começar a atender.
- **Manual**: um CLI independente da aplicação, com `up`, `down`, `status`,
  `validate` e `create`.

## Instalação

```
joo install github.com/nandolang/joo-migrator
```

Os drivers de banco vêm do projeto consumidor; o migrator escolhe o dialeto
SQL pelo nome do driver, mas não embute drivers. A exceção é o SQLite
(`modernc.org/sqlite`, Go puro), que acompanha o pacote para dev/teste. Para
os demais, adicione o que o seu projeto já usa:

```
joo install --go github.com/go-sql-driver/mysql
joo install --go github.com/lib/pq
```

## Começo rápido

### Modo automático

```joo
import std.os.Os
import joolang.migrator.Migrator

pub class Main {

	pub static void main(args []string) {
		var result, err = Migrator.run()
		if (Exceptions.eval(err)) {
			log.error("migrations failed, refusing to start: {}", err)
			Os.exit(1) // o orquestrador precisa ver o boot falho
		}
		// ... a aplicação sobe aqui
	}
}
```

O `Migrator.run()` lê o mesmo datasource que a aplicação usa
(`db.driver`/`db.url` de `resources/properties[-<env>].json` ou das env vars
`DB_DRIVER`/`DB_URL`) e migra `resources/migrations` sob o profile do
`JOO_ENV`. Para sobrescrever qualquer parte:

```joo
Migrator.runWith(MigratorConfig.new()
	.dir("db/migrations")
	.profile("prod"))
```

### Modo manual (CLI)

```
migrator create add_users          # cria migrations/002_add_users/
migrator up                        # aplica tudo que está pendente
migrator status                    # [applied] / [pending] / [skipped] / ...
migrator validate                  # valida árvore + histórico, sem executar
migrator down --steps 1            # reverte a migração aplicada mais recente
migrator down --to 4               # reverte tudo acima do código 4
```

Todos os comandos aceitam `--driver`, `--url`, `--dir`, `--profile`,
`--history-table`, `--lock-table` e `--lock-ttl-seconds`. Flags omitidas
caem na mesma resolução props/env do modo automático.

## Estrutura das migrações

Cada migração é uma pasta `NNN_nome` (código numérico, underscore, nome):

```
resources/migrations/
  000_db_init/
    up.sql             # obrigatório: a migração
    down.sql           # obrigatório: a reversão (vazio = irreversível de propósito)
    migration.yml      # opcional: metadados de execução
  001_seed_dev_data/
    ...
```

Os códigos definem a ordem de execução (crescente) e devem ser únicos.
Buracos na numeração são permitidos. O `up.sql` roda como arquivo inteiro,
então múltiplos statements funcionam em todos os dialetos (a URL do MySQL
ganha `multiStatements=true` automaticamente). O `down.sql` precisa existir;
deixe-o vazio somente quando a mudança realmente não tem volta — nesse caso
o `down` se recusa a tocá-la.

### Metadados (`migration.yml`, `.yaml` ou `.json`)

```yaml
description: "seed de dados para desenvolvimento local"
profiles: [dev]        # vazio/ausente = roda em qualquer profile
```

O profile ativo é o `JOO_ENV`, o mesmo valor que escolhe o
`properties-<env>.json`. Sobrescreva com `--profile` ou `.profile(...)`. Uma
migração pulada por profile (ex.: seeds de dev em prod) continua pendente
naquele profile e é aplicada na primeira execução compatível; a ordem de
execução continua crescente dentro de cada run.

## Semântica de transação

| Dialeto | Estratégia |
|---|---|
| PostgreSQL / CockroachDB | Uma transação envolve o lote inteiro. Se a migração 10 de 10 falhar, o banco (histórico incluso) fica como estava. |
| SQLite | Igual: uma transação por lote. |
| MySQL / MariaDB | O MySQL commita implicitamente a cada DDL, então o lote não tem como ser atômico. As migrações rodam uma a uma; em caso de falha o lote é compensado com cada `down.sql` em ordem reversa, em melhor esforço, e o erro retornado relata o que aconteceu com cada passo. Escreva os `down.sql` de MySQL tolerantes a aplicação parcial (`DROP TABLE IF EXISTS ...`). |

Drift de checksum (migração aplicada cujo `up.sql` mudou no disco) faz `up`,
`down` e `validate` falharem. Crie uma migração nova em vez de editar uma já
aplicada.

## Lock

Instâncias que sobem juntas migram exatamente uma vez. Uma tabela de lock de
linha única (`JOO_MIGRATIONS_LOCK`) é tomada antes do planejamento e liberada
depois da execução, em qualquer caminho. O prazo de expiração é calculado
pela aplicação em unix millis, então as comparações nunca dependem do relógio
do banco; um detentor que morreu se libera sozinho após o TTL (15 min por
padrão), e locks expirados são roubados com guarda otimista, então dois
ladrões nunca vencem ao mesmo tempo.

## Tabelas de controle

Criadas automaticamente, nomes configuráveis:

- `JOO_MIGRATIONS_HISTORY`: code, id, checksum, applied_at, execution_ms,
  profile. As escritas vão na transação do lote, então um lote revertido não
  deixa linhas de histórico para trás.
- `JOO_MIGRATIONS_LOCK`: o lock de linha única.

## Resolução de configuração

Config explícita (flags / `MigratorConfig`) > variáveis de ambiente
(`DB_DRIVER`, `DB_URL`) > `resources/properties-<env>.json` >
`resources/properties.json`. O migrator abre um pool próprio e pequeno (uma
conexão, tempo de vida de cinco minutos) e sempre o fecha, então não acumula
conexões independente do pool da aplicação.

## Bancos suportados

`postgres` / `postgresql` / `pgx` / `cockroach` / `cockroachdb`,
`mysql` / `mariadb`, `sqlite` / `sqlite3`. Adicionar um banco significa
escrever uma classe `Dialect` e registrá-la em `Dialects.forDriver`; o engine
e os stores não mudam.
