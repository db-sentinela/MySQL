# MySQL Troubleshooting — Slow Query Log

O slow query log é a forma mais direta de **capturar** quais queries estão demorando em um ambiente real. Este documento mostra como ligar, ajustar e **ler** uma entrada — com saída real do lab.

> Pré-leitura: `troubleshooting-queries-lentas-explain.md` (depois de achar a query lenta aqui, é lá que você a conserta).

---

## O que é

Um arquivo (ou tabela) onde o MySQL registra toda query cujo tempo de execução passou de um limite (`long_query_time`). É a sua "câmera" para flagrar as queries pesadas em produção, sem ficar olhando o `PROCESSLIST` na sorte.

Estado inicial no lab (padrão): **desligado**, limite de 10s.

```sql
SELECT @@slow_query_log, @@long_query_time, @@log_output, @@slow_query_log_file;
```
```
| on_off | limite_seg | saida | arquivo                             |
|   0    | 10.000000  | FILE  | /var/lib/mysql/mysql-db-01-slow.log |
```

---

## Ligar e ajustar

Em tempo de execução (sem reiniciar), para uma janela de diagnóstico:

```sql
SET GLOBAL slow_query_log = ON;
SET GLOBAL long_query_time = 0.05;      -- registra queries acima de 50 ms
SET GLOBAL log_output = 'FILE';         -- 'FILE' (arquivo) ou 'TABLE' (mysql.slow_log)
```

> `long_query_time` aceita decimais (0.05 = 50 ms). O valor vale para **novas conexões**; a sessão que você quer medir deve reconectar depois de ligar.

Para valer **após reiniciar**, coloque no `my.cnf` (ex.: `/etc/mysql/mysql.conf.d/`):

```ini
[mysqld]
slow_query_log      = 1
slow_query_log_file = /var/log/mysql/mysql-slow.log
long_query_time     = 0.5
```

### Variáveis que valem conhecer

| Variável | Para quê |
|---|---|
| `long_query_time` | Limite em segundos (aceita decimais) |
| `log_queries_not_using_indexes` | Registra **toda** query sem índice, mesmo rápida (ótimo para caçar full scans; cuidado, gera volume) |
| `min_examined_row_limit` | Só loga se examinou pelo menos N linhas (corta ruído) |
| `log_output` | `FILE` ou `TABLE` (consulta via `SELECT * FROM mysql.slow_log`) |
| `log_slow_admin_statements` | Inclui `ALTER`, `ANALYZE` etc. |
| `log_slow_replica_statements` | Loga queries lentas aplicadas pela replicação |

---

## Lendo uma entrada

Rodamos uma query lenta de propósito (full scan em `status`, sem índice) e ela caiu no log. Entrada **real**:

```
# Time: 2026-11-11T21:57:33.878183Z
# User@Host: clusteradmin[clusteradmin] @ localhost [127.0.0.1]  Id:   748
# Query_time: 0.082898  Lock_time: 0.000003 Rows_sent: 1  Rows_examined: 500000
SET timestamp=1794434253;
SELECT COUNT(*) FROM tshoot.pedidos WHERE status='cancelado';
```

### O que cada campo diz

| Campo | Significado | Por que importa |
|---|---|---|
| `Query_time` | Tempo total da query | O que estourou o `long_query_time` |
| `Lock_time` | Tempo esperando lock | Alto aqui = problema de bloqueio, não de query |
| `Rows_sent` | Linhas devolvidas ao cliente | **1** |
| `Rows_examined` | Linhas que o MySQL **leu** para responder | **500000** |

**A razão de ouro:** `Rows_examined` muito maior que `Rows_sent` (aqui, 500.000 para devolver 1) é a assinatura de **falta de índice**. Peça o `EXPLAIN` dessa query e resolva (ver o doc de EXPLAIN).

> Contraste: a query indexada (`WHERE cliente_id=1234`), que examina 45 linhas em ~0,1 ms, **não apareceu** no log — ficou abaixo do limite. O log mostra só o que dói.

---

## Agregando o log: `pt-query-digest`

Em produção o log tem milhares de linhas. Não leia na mão — agregue por "forma" da query (digest) com o `pt-query-digest` (Percona Toolkit):

```bash
pt-query-digest /var/log/mysql/mysql-slow.log | less
```

Ele ranqueia as queries por **tempo total gasto** (as que mais pesam no conjunto), mostrando contagem, tempo médio e linhas examinadas. É assim que se acha "a query que está matando o banco".

> Alternativa nativa (sem instalar nada): `performance_schema`/`sys` — ver `troubleshooting-queries-lentas-performance-schema.md`.

---

## Cuidados

- **`long_query_time = 0` loga tudo** — útil por poucos minutos para um perfil completo, mas gera volume e I/O. Não deixe assim em produção.
- **O arquivo cresce.** Faça rotação (`logrotate`) e acompanhe o espaço em disco.
- **`log_output='TABLE'`** é prático para consultar com SQL, mas a tabela `mysql.slow_log` também cresce — limpe periodicamente (`TRUNCATE`).
- **Desligue depois do diagnóstico**, se você só ligou para uma investigação pontual:

```sql
SET GLOBAL slow_query_log = OFF;      -- volta ao estado anterior do lab
SET GLOBAL long_query_time = 10;
```

---

## Comandos úteis

```sql
-- Estado atual
SELECT @@slow_query_log, @@long_query_time, @@log_output, @@slow_query_log_file;

-- Ligar para diagnostico
SET GLOBAL slow_query_log=ON; SET GLOBAL long_query_time=0.5;

-- Ler (arquivo)
-- (no shell) sudo tail -f /var/lib/mysql/mysql-db-01-slow.log

-- Ler (tabela, se log_output=TABLE)
SELECT start_time, query_time, rows_examined, sql_text
FROM mysql.slow_log ORDER BY query_time DESC LIMIT 10;
```

---

## Não incluído neste documento

- Análise do plano das queries capturadas — `troubleshooting-queries-lentas-explain.md`.
- Caça às queries lentas sem o log, em tempo real — `troubleshooting-queries-lentas-performance-schema.md`.
- Instalação e uso avançado do Percona Toolkit.
