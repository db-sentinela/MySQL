# MySQL Tuning — Fundamentos e metodologia

Antes de mexer em qualquer variável: **como** tunar sem chutar. A metodologia, a hierarquia de gargalos e as ferramentas de medição. Base para todo o resto da subpasta.

> ⚠️ A regra número 1 do tuning: **meça, não chute.** Copiar `my.cnf` da internet quebra mais do que conserta. Mude **uma coisa por vez** e meça o efeito.

---

## O ciclo de tuning

```mermaid
flowchart LR
    B["1. Baseline<br/>(medir o estado atual)"] --> I["2. Identificar o gargalo"]
    I --> C["3. Mudar UMA coisa"]
    C --> M["4. Medir de novo"]
    M --> B
```

Sem **baseline**, você não sabe se melhorou. Sem medir **um gargalo real**, você otimiza o que não importa. A maior parte do ganho vem de **poucos** ajustes certos — não de dezenas de variáveis.

---

## A hierarquia dos gargalos (onde está a dor)

Ataque nesta ordem — o de cima dá o maior retorno:

| Nível | Gargalo | Onde tratar |
|---|---|---|
| 1 | **Queries e índices ruins** | O maior ofensor. `EXPLAIN`, índices, histogramas |
| 2 | **Memória (buffer pool)** | `innodb_buffer_pool_size` — ler do disco é 1000× mais lento que da RAM |
| 3 | **I/O de disco** | redo, flush, `innodb_io_capacity`, tipo de disco |
| 4 | **Contenção (locks)** | ver subpasta Troubleshooting |
| 5 | **CPU / conexões** | `max_connections`, thread cache, thread pool |

> Uma query sem índice varrendo 500 mil linhas não se conserta com mais RAM — se conserta com um índice. **Primeiro a query, depois a máquina.**

---

## As perguntas que guiam

1. **O quê está lento?** Uma query específica, ou o servidor todo?
2. **O recurso saturado é qual?** CPU, disco, memória ou lock?
3. **Qual a meta?** Latência (resposta rápida) ou throughput (muitas transações)?
4. **Mudou o quê recentemente?** Volume de dados, carga, uma query nova?

---

## As ferramentas de medição

| Ferramenta | Para quê |
|---|---|
| `SHOW GLOBAL STATUS` | Contadores do servidor (hit ratio, temp tables, threads) |
| `performance_schema` / `sys` | Queries custosas, índices não usados, buffer por tabela |
| `SHOW ENGINE INNODB STATUS` | Estado interno do InnoDB (I/O, buffer, locks) |
| `EXPLAIN` / `EXPLAIN ANALYZE` | Plano e custo de uma query (ver Troubleshooting) |
| SO: `top`, `iostat`, `vmstat` | Saber se o gargalo é CPU, disco ou memória **no host** |

### Exemplo de leitura: hit ratio do buffer pool (real no lab)

```sql
SELECT ROUND(100*(1 - (
  SELECT VARIABLE_VALUE FROM performance_schema.global_status WHERE VARIABLE_NAME='Innodb_buffer_pool_reads') / (
  SELECT VARIABLE_VALUE FROM performance_schema.global_status WHERE VARIABLE_NAME='Innodb_buffer_pool_read_requests')),2) AS hit_ratio_pct;
-- -> 99.81   (99%+ e bom; abaixo de ~95% indica buffer pool pequeno)
```

---

## O estado do lab (baseline real)

| Métrica | Valor | Leitura |
|---|---|---|
| `innodb_buffer_pool_size` | **128 MB** (default!) | 🔴 minúsculo para um servidor com ~7,6 GB de RAM |
| Dados (todos os schemas) | 34,5 MB | cabem no pool → por isso hit ratio 99,8% |
| `innodb_flush_log_at_trx_commit` | 1 | durabilidade total (ACID) |
| `innodb_flush_method` | O_DIRECT | ✅ evita cache duplo do SO |
| `innodb_redo_log_capacity` | 100 MB | pequeno para carga de escrita alta |
| `max_connections` | 151 | default |
| `thread_cache_size` | 9 | baixo |

O baseline já aponta o **maior ganho potencial**: o buffer pool no default de 128 MB. Hoje o hit ratio é alto só porque a base é pequena (34 MB); com dados maiores que 128 MB, esse default viraria o gargalo nº 1. Detalhe em `tuning-innodb-buffer-pool.md`.

---

## Erros clássicos (não faça)

- **Copiar `my.cnf` de blog** sem entender — cada workload é diferente.
- **Mexer em 10 variáveis de uma vez** — você não sabe qual ajudou/piorou.
- **Super-dimensionar buffers por conexão** (`sort_buffer_size` gigante) — multiplica por conexão e estoura a RAM (ver memória).
- **Otimizar a máquina antes da query** — índice resolve o que RAM nenhuma resolve.
- **Tunar sem baseline** — sem antes/depois, é fé, não engenharia.

---

## Documentos desta subpasta

1. Fundamentos e metodologia (este)
2. `tuning-innodb-buffer-pool.md` — o ajuste de maior impacto
3. `tuning-memoria-e-buffers.md` — memória global × por conexão
4. `tuning-redo-log-e-durabilidade.md` — I/O, flush, o trade-off ACID
5. `tuning-conexoes-e-threads.md`
6. `tuning-indices-e-otimizador.md` — histogramas, índices invisíveis, hints (pouco conhecidos)
7. `tuning-diagnostico-e-benchmark.md` — sys schema, índices não usados, sysbench
