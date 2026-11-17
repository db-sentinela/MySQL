# MySQL Tuning — Diagnóstico e benchmark

Onde olhar para achar o que tunar, e como medir antes/depois de forma honesta. As views do `sys`, a caça a índices inúteis, e as ferramentas de benchmark. Com saída real do lab.

> Pré-leitura: `tuning-fundamentos-metodologia.md`.

---

## O schema `sys`: diagnóstico pronto

O `sys` traduz o `performance_schema` em views legíveis. As mais úteis para tuning:

### Índices nunca usados (candidatos a remover)

Índice que ninguém usa só **pesa** nas escritas e ocupa disco:

```sql
SELECT object_schema, object_name, index_name
FROM sys.schema_unused_indexes
WHERE object_schema NOT IN ('mysql','sys');
```

Saída real do lab:

```
tshoot   pedidos   idx_cliente
```

> ⚠️ **Cuidado na interpretação:** "não usado" é **desde o último restart**. No lab o `idx_cliente` aparece porque o servidor reiniciou e ninguém consultou por `cliente_id` ainda — mas o índice é útil. Só remova um índice "não usado" depois de observar um **período representativo** (dias/semanas com a carga real), idealmente deixando-o **invisível** antes de apagar.

### Índices redundantes/duplicados

```sql
SELECT table_schema, table_name, redundant_index_name, dominant_index_name
FROM sys.schema_redundant_indexes;
```

No lab: **nenhum** (bom). Um índice `(a)` é redundante se já existe `(a,b)` — o composto cobre o prefixo.

### Queries mais custosas e com full scan

```sql
SELECT * FROM sys.statement_analysis ORDER BY total_latency DESC LIMIT 10;
SELECT * FROM sys.statements_with_full_table_scans LIMIT 10;
SELECT * FROM sys.statements_with_temp_tables LIMIT 10;
```

(Detalhadas na subpasta `Troubleshooting` — são a ponte entre tuning de servidor e tuning de query.)

### O que está no buffer pool

```sql
SELECT object_schema, object_name, allocated, pages
FROM sys.innodb_buffer_stats_by_table ORDER BY pages DESC LIMIT 10;
```

Real no lab: `pedidos` 22,75 MiB / 1456 páginas lidera — mostra o working set real.

---

## Contadores-chave de `SHOW GLOBAL STATUS`

| Contador | O que indica |
|---|---|
| `Innodb_buffer_pool_reads` vs `..._read_requests` | Hit ratio (buffer pool pequeno?) |
| `Created_tmp_disk_tables` vs `Created_tmp_tables` | Temp tables indo para disco? |
| `Sort_merge_passes` | `sort_buffer_size` pequeno |
| `Select_full_join` / `Select_scan` | Falta de índice |
| `Threads_created` | Thread cache pequeno |
| `Table_open_cache_misses` | `table_open_cache` pequeno |
| `Innodb_row_lock_waits` / `_time` | Contenção de lock |

> Dica: tire um snapshot do `GLOBAL STATUS`, rode a carga, tire outro, e **compare os deltas** — os contadores são acumulados desde o boot.

---

## Benchmark: medir com honestidade

Tunar sem benchmark é fé. Ferramentas:

### `sysbench` (o padrão)

Gera carga OLTP reproduzível e mede throughput/latência:

```bash
# preparar dados
sysbench oltp_read_write --mysql-host=127.0.0.1 --mysql-user=app --mysql-password=... \
  --mysql-db=appdb --tables=10 --table-size=1000000 prepare

# rodar 60s com 16 threads
sysbench oltp_read_write --threads=16 --time=60 ... run
# -> transactions/s, latency p95, etc.
```

Rode **antes** e **depois** da mudança, com a **mesma** carga — é o único jeito de provar o ganho.

### `mysqlslap`

Benchmark simples embutido no cliente MySQL (bom para testes rápidos de uma query).

### Interpretação

- **Throughput** (transações/s) sobe? **Latência p95/p99** desce? Então ajudou.
- Olhe a **cauda** (p95/p99), não só a média — é ela que o usuário sente.
- Mude **uma** variável por benchmark.

---

## Ferramentas externas de diagnóstico

| Ferramenta | Para quê |
|---|---|
| **MySQLTuner** (`mysqltuner.pl`) | Script que lê status/variáveis e sugere ajustes. Ótimo ponto de partida (leia com senso crítico, não aplique cego). |
| **Percona Toolkit** (`pt-query-digest`) | Agrega o slow log e ranqueia as queries que mais pesam. |
| **Percona Monitoring and Management (PMM)** | Dashboards de métricas do MySQL em tempo real (Grafana). |
| **`innotop`** | "top" do InnoDB em tempo real. |

### E do lado do Claude Code

Não há um plugin **específico de tuning MySQL**, mas dois do diretório ajudam no ciclo:
- **Data** (Anthropic): escrever SQL, explorar os dados de `performance_schema`/`sys` e montar dashboards das métricas de performance.
- **Bench Claude**: rodar benchmarks e gerar relatórios em Markdown (orientado a bench-node/JS, mas útil para padronizar o relatório antes/depois).

Para o MySQL em si, as ferramentas externas acima (sysbench, Percona Toolkit, MySQLTuner, PMM) continuam sendo a base.

---

## Roteiro de uma análise de tuning

1. **Baseline:** `GLOBAL STATUS` + hit ratio + `sys.statement_analysis`.
2. **Gargalo:** memória (hit ratio), I/O (temp disk, checkpoint), query (full scans) ou lock?
3. **Hipótese + 1 mudança** (buffer pool, índice, histograma...).
4. **Benchmark** antes/depois (sysbench, mesma carga).
5. **Confirme o delta** nos contadores e na latência. Documente.

---

## Checklist

1. `sys.schema_unused_indexes` / `schema_redundant_indexes` revisados (com cautela na interpretação).
2. Contadores-chave do `GLOBAL STATUS` comparados por delta.
3. Mudanças validadas por **benchmark** (sysbench) com a mesma carga.
4. Olhar p95/p99, não só média.
5. Uma mudança por vez, documentada.

---

## Não incluído neste documento

- O básico de `EXPLAIN`/slow log/`sys` para achar queries lentas — subpasta `Troubleshooting`.
- Montagem de dashboards no PMM/Grafana.
- Perfis de carga sintéticos avançados no sysbench (Lua custom).
