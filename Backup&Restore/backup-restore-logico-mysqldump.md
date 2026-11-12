# MySQL Backup & Restore — Backup lógico com mysqldump

O `mysqldump` é o canivete suíço do backup lógico: portátil, seletivo, presente em qualquer MySQL. Este documento mostra como fazer um backup **consistente e pronto para PITR**, e um restore — executados no lab.

> Pré-leitura: `backup-restore-fundamentos.md`.

---

## O comando que você deve usar

```bash
mysqldump --single-transaction --source-data=2 --set-gtid-purged=OFF \
  --databases bkp > /tmp/bkp.sql
```

Saída real (schema `bkp`, 5 linhas): arquivo de **2361 bytes**, com a coordenada de binlog embutida:

```
-- CHANGE REPLICATION SOURCE TO SOURCE_LOG_FILE='binlog.000006', SOURCE_LOG_POS=13383776;
```

### Cada opção, e por que importa

| Opção | O que faz | Por que importa |
|---|---|---|
| `--single-transaction` | Backup dentro de uma transação com snapshot | Consistente **sem travar** a aplicação (tabelas InnoDB) |
| `--source-data=2` | Grava a **posição do binlog** como comentário | É a âncora do **PITR** — de onde aplicar os binlogs depois |
| `--set-gtid-purged=OFF` | **Não** embute o estado de GTID no dump | Evita corromper o GTID ao restaurar no mesmo cluster vivo |
| `--databases bkp` | Dump do schema `bkp` (inclui `CREATE DATABASE`) | O restore recria o schema sozinho |

> `--source-data=2` = comentário (informativo). `--source-data=1` = comando **ativo** (bom para montar réplica, mas atenção ao restaurar em servidor normal). Em versões antigas o nome era `--master-data`.

### Variações úteis

```bash
# Instancia inteira
mysqldump --single-transaction --source-data=2 --set-gtid-purged=OFF \
  --all-databases > /tmp/full.sql

# So algumas tabelas
mysqldump --single-transaction bkp clientes pedidos > /tmp/tabelas.sql

# So a estrutura (sem dados)
mysqldump --no-data --databases bkp > /tmp/estrutura.sql

# Comprimindo na hora (economiza muito espaco)
mysqldump --single-transaction --all-databases | gzip > /tmp/full-$(date +%F).sql.gz
```

---

## Restore (teste real: acidente → recuperação)

Simulamos um acidente e restauramos. Sequência real no lab:

```bash
# 1) Backup feito acima (bkp.sql, 5 linhas)
# 2) ACIDENTE: alguem apaga a tabela
mysql -e "DROP TABLE bkp.clientes;"
#    -> SELECT agora falha: ERROR 1146 Table 'bkp.clientes' doesn't exist

# 3) RESTORE a partir do dump
mysql < /tmp/bkp.sql

# 4) Conferir
mysql -e "SELECT COUNT(*) FROM bkp.clientes;"
#    -> 5   (linhas de volta)
```

Resultado real: as **5 linhas voltaram**. Restore lógico = reexecutar o SQL do dump. Simples e confiável; a desvantagem aparece só em bases grandes (é mais lento que o físico).

> Restaurar um dump que contém `CREATE DATABASE`/`DROP TABLE` **sobrescreve** o que existir. Em produção, **restaure primeiro em um schema/instância de teste** e só promova depois de conferir.

---

## A pegadinha do GTID (leia antes de restaurar)

Com GTID ligado, o destino do restore muda a opção:

| Destino do restore | `--set-gtid-purged` | Por quê |
|---|---|---|
| **Mesmo cluster vivo** (recuperar tabela/dados) | `OFF` | Não reimportar estado de GTID (corromperia a replicação) |
| **Nó/cluster novo e vazio** (semear) | `ON` / `AUTO` | O novo precisa saber até onde o dump foi |

Errar aqui é uma das causas mais comuns de "restaurei e a replicação quebrou".

---

## Backup consistente + PITR: a dupla

O `--source-data=2` guardou `binlog.000006 / pos 13383776`. Isso quer dizer: "este dump representa o banco **até** esse ponto do binlog". Para chegar mais perto do presente (RPO baixo), você restaura o dump **e depois aplica os binlogs** a partir dessa posição — é o PITR, detalhado em `backup-restore-pitr.md`.

---

## Boas práticas

- **Rode em um secundário** e com `--single-transaction`.
- **Comprima** (`| gzip`) — dumps são texto e encolhem muito.
- **Guarde a coordenada de binlog** junto do dump (o `--source-data=2` já coloca no arquivo).
- **Agende** (cron) e **teste o restore** com regularidade.
- Para bases grandes, prefira o **MySQL Shell** (paralelo) — ver doc vizinho.

```bash
# Exemplo de cron (secundario), 02:30, comprimido
30 2 * * * dba  mysqldump --single-transaction --source-data=2 --set-gtid-purged=OFF \
  --all-databases | gzip > /backup/full-$(date +\%F).sql.gz
```

---

## Comandos úteis

```bash
# Backup consistente + coordenada de binlog
mysqldump --single-transaction --source-data=2 --set-gtid-purged=OFF --databases <db> > dump.sql

# Restore
mysql < dump.sql
# ou restaurar um .gz sem descompactar em disco
zcat dump.sql.gz | mysql

# Ver a coordenada guardada no dump
grep -i 'CHANGE REPLICATION SOURCE' dump.sql
```

---

## Não incluído neste documento

- Backup lógico paralelo/comprimido com MySQL Shell — `backup-restore-logico-mysql-shell.md`.
- Point-in-time recovery aplicando binlogs — `backup-restore-pitr.md`.
- `mysqlpump` (descontinuado) e detalhes de `--tab` (arquivos separados por tabela).
