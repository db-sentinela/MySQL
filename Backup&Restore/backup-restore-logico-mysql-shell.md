# MySQL Backup & Restore — Backup lógico com MySQL Shell (dump paralelo)

O `mysqldump` é ótimo, mas single-thread. Para bases médias/grandes, o **MySQL Shell** faz backup lógico **em paralelo e comprimido** — muito mais rápido. Este documento mostra o `util.dumpSchemas`/`util.loadDump` com execução real no lab (tabela de 500 mil linhas).

> Pré-leitura: `backup-restore-fundamentos.md` e `backup-restore-logico-mysqldump.md`.

---

## Por que usar o Shell em vez do mysqldump

| | `mysqldump` | MySQL Shell (`util.*`) |
|---|---|---|
| Paralelismo | 1 thread | **N threads** |
| Compressão | só via pipe externo | **nativa** (zstd/gzip) |
| Formato | 1 arquivo `.sql` gigante | arquivos por tabela, em **chunks** |
| Restore | reexecuta SQL (lento) | `loadDump` **paralelo** |
| Retomar | não | sim (resume) |

Para bases a partir de alguns GB, a diferença é enorme.

---

## Backup: `util.dumpSchemas` / `util.dumpInstance`

```bash
mysqlsh --uri clusteradmin@mysql-db-01:3306 --js \
  --execute="util.dumpSchemas(['tshoot'], '/tmp/shelldump', {threads:4})"
```

Saída **real** (tabela `tshoot`, 2 tabelas, 500 mil linhas):

```
1 schemas will be dumped and within them 2 tables, 0 views.
Schemas dumped: 1        Tables dumped: 2
Uncompressed data size: 22.61 MB
Compressed data size: 5.91 MB        <- zstd: ~4x menor
Rows written: 500003
Dump duration: 00:00:00s
```

Variantes:

```bash
# Instancia inteira (equivale ao --all-databases)
util.dumpInstance('/tmp/full', {threads:4})

# Apenas algumas tabelas
util.dumpTables('tshoot', ['pedidos'], '/tmp/dump_pedidos', {threads:4})
```

### Como é o dump por dentro

O Shell gera **vários arquivos** (não um `.sql` único) — por isso carrega em paralelo. Conteúdo real de `/tmp/shelldump` (14 arquivos, 5.8M):

```
@.json  @.sql  @.post.sql  @.done.json     <- metadados do dump
tshoot.json  tshoot.sql                    <- DDL do schema
tshoot@contas.json  tshoot@contas.sql      <- DDL por tabela
tshoot@contas@@0.tsv.zst                   <- DADOS (chunk, comprimido zstd)
tshoot@contas@@0.tsv.zst.idx               <- indice do chunk
tshoot@pedidos.json  tshoot@pedidos.sql
tshoot@pedidos@@0.tsv.zst  ...
```

Os dados ficam em `.tsv.zst` (TSV comprimido), fatiados em chunks para o load paralelo.

---

## Restore: `util.loadDump`

```bash
mysqlsh --uri clusteradmin@mysql-db-01:3306 --js \
  --execute="util.loadDump('/tmp/shelldump', {threads:4})"
```

### ⚠️ A pegadinha: `local_infile`

O `loadDump` carrega os dados via `LOAD DATA LOCAL`, que exige `local_infile = ON` no servidor. No lab o padrão é **OFF**, e o load falhou até ligarmos:

```sql
SET GLOBAL local_infile = ON;     -- necessario para o loadDump
```

Depois disso, o restore **real** (após apagar o schema de propósito):

```
Executing schema DDL - done
Executing table DDL - done
Loading data - done
2 chunks (500.00K rows, 22.61 MB) for 2 tables in 1 schemas were loaded in 6 sec
   (avg throughput 3.55 MB/s, 78.49K rows/s)
0 warnings were reported during the load.
```

Verificação: `tshoot.pedidos` voltou com **500.000** linhas. (Dica de segurança: volte `local_infile = OFF` depois.)

> O `loadDump` **recria** os schemas/tabelas do dump. Para carregar em um destino diferente, use opções como `schema`/`excludeSchemas` ou carregue em uma instância separada.

---

## GTID no Shell

Os utilitários tratam GTID de forma explícita:

- O dump grava o `gtidExecuted` nos metadados (`@.json`).
- No load, controle com `updateGtidSet` (`off` | `append` | `replace`) e `skipBinlog`. Para **carregar em um cluster vivo sem bagunçar o GTID**, a escolha padrão já é segura; para **semear um destino novo**, use `updateGtidSet:'replace'`. Leia a doc antes de usar em cluster de produção — a lógica de GTID é a mesma pegadinha do mysqldump.

---

## Boas práticas

- **Ajuste `threads`** ao número de vCPUs do host de backup.
- **Rode em um secundário.**
- **`local_infile = ON`** no destino do `loadDump` (e desligue depois).
- Para bases grandes, `util.dumpInstance` + `util.loadDump` é o caminho lógico mais rápido disponível sem ferramenta paga.

---

## Comandos úteis

```bash
# Dump
mysqlsh --uri clusteradmin@<host>:3306 --js --execute="util.dumpInstance('/backup/full', {threads:4})"

# Load (lembre do local_infile=ON no destino)
mysqlsh --uri clusteradmin@<host>:3306 --js --execute="util.loadDump('/backup/full', {threads:4})"

# So uma tabela
mysqlsh --uri clusteradmin@<host>:3306 --js --execute="util.dumpTables('db',['tabela'],'/backup/t',{threads:4})"
```

---

## Não incluído neste documento

- Backup físico (CLONE/XtraBackup) — `backup-restore-fisico.md`.
- PITR — `backup-restore-pitr.md`.
- `util.copyInstance` (copiar direto de um servidor para outro, sem arquivo intermediário).
