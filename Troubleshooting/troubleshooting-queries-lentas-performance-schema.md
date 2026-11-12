# MySQL Troubleshooting — Achar queries lentas (performance_schema / sys)

Como descobrir as queries mais custosas **sem ligar o slow log e sem ler arquivo**, consultando o que o MySQL já mede sozinho. Com saída real do lab.

> Pré-leitura: `troubleshooting-queries-lentas-explain.md`. Complementa o `troubleshooting-slow-query-log.md`.

---

## Por que usar performance_schema / sys

- Já vem **ligado** por padrão (MySQL 8.0+). Não precisa habilitar nada nem reiniciar.
- Agrega as queries por **digest** (a "forma" da query, com os valores virando `?`), então você vê o **custo acumulado** de cada tipo de query — não uma linha por execução.
- Tudo por **SQL**, de qualquer cliente. O schema `sys` entrega isso em views prontas e legíveis (tempos como "72.98 ms", não picossegundos).

Pense assim: o **slow log** é a câmera que grava; o **performance_schema** é o painel que já mostra o resumo em tempo real.

---

## A view que você vai usar primeiro: `sys.statement_analysis`

Ranqueia as queries pelo tempo. Saída **real** do lab (tabela `pedidos`, 500 mil linhas):

```sql
SELECT LEFT(query,42) AS query, exec_count, total_latency, avg_latency,
       rows_examined_avg, full_scan
FROM sys.statement_analysis
WHERE db='tshoot' AND query LIKE '%pedidos%'
ORDER BY total_latency DESC LIMIT 6;
```

```
+--------------------------------------------+-------+---------------+-------------+-----------+-----------+
| query                                      | execs | total_latency | avg_latency | rows_exam | full_scan |
+--------------------------------------------+-------+---------------+-------------+-----------+-----------+
| SELECT COUNT(*) FROM `pedidos` WHERE ST... |     3 | 208.09 ms     | 69.36 ms    |    500000 | *         |
| SELECT COUNT(*) FROM `pedidos` WHERE `v... |     1 | 72.98 ms      | 72.98 ms    |    500000 | *         |
| SELECT COUNT(*) FROM `pedidos` WHERE `c... |     2 | 551.52 us     | 275.76 us   |        45 |           |
+--------------------------------------------+-------+---------------+-------------+-----------+-----------+
```

### Como ler (e priorizar)

- **`full_scan = *`** → a query faz full table scan. Candidata imediata a índice.
- **`rows_examined` (médio) enorme** → confirma o desperdício. Aqui, 500.000 lidas contra 45 da versão indexada.
- **`total_latency`** (tempo somado de todas as execuções) → é por onde priorizar. A query de `status` não é a mais lenta por execução, mas rodou 3× e **lidera o tempo total** — é a que mais pesa no servidor.
- **`avg_latency`** → tempo médio por execução.

A versão indexada (`WHERE cliente_id = ?`) aparece lá embaixo: `275 us`, 45 linhas, **sem** `full_scan`. É o alvo: transformar as de cima em algo parecido.

---

## Views irmãs (cada "cheiro" tem a sua)

```sql
-- Queries que fazem full table scan
SELECT query, exec_count, rows_examined_avg
FROM sys.statements_with_full_table_scans
WHERE db='tshoot' ORDER BY exec_count DESC;
```

Saída real:

```
| query                               | execs | rows_exam |
| SELECT COUNT(*) FROM `pedidos` WHERE ST... | 3 | 500000 |
| SELECT COUNT(*) FROM `pedidos` WHERE `v... | 1 | 500000 |
```

Outras views úteis do `sys`:

| View | Mostra queries que... |
|---|---|
| `sys.statements_with_full_table_scans` | varrem a tabela toda (falta de índice) |
| `sys.statements_with_sorting` | fazem ordenação custosa (`filesort`) |
| `sys.statements_with_temp_tables` | criam tabelas temporárias (ainda pior se em disco) |
| `sys.statements_with_errors_or_warnings` | retornam erro/aviso |
| `sys.statement_analysis` | visão geral, ranqueada por tempo |

---

## A fonte crua: `events_statements_summary_by_digest`

As views `sys` leem desta tabela do `performance_schema`. Útil quando você quer colunas específicas:

```sql
SELECT LEFT(DIGEST_TEXT,40) AS digest, COUNT_STAR AS execs,
       ROUND(SUM_TIMER_WAIT/1e12,3) AS seg_total,
       SUM_ROWS_EXAMINED AS rows_exam, SUM_ROWS_SENT AS rows_sent
FROM performance_schema.events_statements_summary_by_digest
WHERE SCHEMA_NAME='tshoot'
ORDER BY SUM_TIMER_WAIT DESC LIMIT 10;
```

> Detalhe que pega todo mundo: o `SCHEMA_NAME` do digest é o **database default da conexão** no momento da execução. Se a app conecta sem `USE`/database default e usa nomes qualificados (`tshoot.pedidos`), o `SCHEMA_NAME` fica `NULL`. Nesse caso, filtre por `DIGEST_TEXT LIKE '%pedidos%'` em vez de por `SCHEMA_NAME`.

Tempos no `performance_schema` cru estão em **picossegundos** (divida por `1e12` para segundos). O `sys` já faz isso para você.

---

## Queries lentas rodando AGORA

As views acima são histórico agregado. Para o que está travando **neste instante**:

```sql
-- conexoes ativas ordenadas pelo tempo da query atual
SELECT ID, USER, TIME, STATE, LEFT(INFO,60) AS query
FROM information_schema.PROCESSLIST
WHERE COMMAND <> 'Sleep' AND INFO IS NOT NULL
ORDER BY TIME DESC;

-- versao amigavel do sys (ja ignora conexoes internas/ociosas)
SELECT conn_id, user, current_statement, statement_latency
FROM sys.session
WHERE current_statement IS NOT NULL
ORDER BY statement_latency DESC;
```

`TIME`/`statement_latency` alto = query rodando há muito tempo. Combine com o `EXPLAIN` dela para entender o porquê.

---

## Zerando as estatísticas

Os números são acumulados desde o último boot (ou reset). Para medir uma janela limpa:

```sql
TRUNCATE performance_schema.events_statements_summary_by_digest;
```

Isso **não** apaga dados do banco — só zera os contadores de estatística de queries. Rode, reproduza a carga, e leia de novo.

---

## Roteiro

1. `sys.statement_analysis` ordenado por `total_latency` → as queries que mais pesam.
2. Marque as de `full_scan = *` e `rows_examined` alto.
3. Para cada uma, rode `EXPLAIN` e crie índice (ver doc de EXPLAIN).
4. `TRUNCATE` o digest, rode a carga de novo e confirme a melhora.

---

## Comandos úteis

```sql
-- Top por tempo total
SELECT * FROM sys.statement_analysis ORDER BY total_latency DESC LIMIT 10;

-- Full scans / filesort / temp tables
SELECT * FROM sys.statements_with_full_table_scans LIMIT 10;
SELECT * FROM sys.statements_with_sorting LIMIT 10;
SELECT * FROM sys.statements_with_temp_tables LIMIT 10;

-- Rodando agora
SELECT * FROM sys.session WHERE current_statement IS NOT NULL ORDER BY statement_latency DESC;

-- Zerar estatisticas de query
TRUNCATE performance_schema.events_statements_summary_by_digest;
```

---

## Não incluído neste documento

- Corrigir as queries encontradas (índices/plano) — `troubleshooting-queries-lentas-explain.md`.
- Captura histórica em arquivo — `troubleshooting-slow-query-log.md`.
- Instrumentação avançada do performance_schema (consumers/instruments, `events_statements_history_long`).
