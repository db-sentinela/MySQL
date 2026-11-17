# DB Sentinel | MySQL — Tuning

Parte do **DB Sentinel Database Procedures Framework**: biblioteca técnica de procedimentos operacionais (SOPs) para administração de bancos de dados.

Esta subpasta cobre **tuning de performance em MySQL**: metodologia, buffer pool, memória, redo/durabilidade, conexões, e um deep dive nos recursos **pouco conhecidos** do otimizador. Os procedimentos são independentes de clientes, não contêm dados de ambientes reais e são **validados em laboratório** — com medições e experimentos reais.

> A regra de ouro: **meça, não chute.** Uma mudança por vez, com baseline e benchmark. Copiar `my.cnf` de blog quebra mais do que conserta.

## Por onde começar

1. **Fundamentos e metodologia** — como tunar sem chutar; a hierarquia de gargalos.
2. **Buffer pool** — o ajuste de maior impacto.
3. Os demais conforme o gargalo (memória, I/O, conexões, otimizador).

## Procedimentos

| Procedimento | Tema | Status |
|---|---|---|
| [Fundamentos e metodologia](tuning-fundamentos-metodologia.md) | Ciclo de tuning, gargalos, ferramentas | Validado |
| [InnoDB Buffer Pool](tuning-innodb-buffer-pool.md) | Hit ratio, sizing, warmup (dump/load) | Validado |
| [Memória e buffers](tuning-memoria-e-buffers.md) | Global × por conexão, temp tables, teto de RAM | Validado |
| [Redo log e durabilidade](tuning-redo-log-e-durabilidade.md) | `flush_log_at_trx_commit`, O_DIRECT, io_capacity | Validado |
| [Conexões e threads](tuning-conexoes-e-threads.md) | `max_connections`, pool, thread cache | Validado |
| [Índices e otimizador](tuning-indices-e-otimizador.md) | Histogramas, índices invisíveis, hints | Validado |
| [Diagnóstico e benchmark](tuning-diagnostico-e-benchmark.md) | sys schema, índices inúteis, sysbench | Validado |

> **Status:** *Validado* = medição/experimento reproduzido e saída capturada em laboratório.

## O que foi medido/experimentado no lab

- **Buffer pool:** hit ratio **99,8%**; achado — pool no **default de 128 MB** num servidor de ~7,6 GB (o maior ganho potencial).
- **Histogramas:** em coluna desbalanceada (`critico` = 0,04%), o `EXPLAIN` foi de `filtered 10%` (chute) para **0,10%** (real) — 100× mais preciso.
- **Índices invisíveis:** otimizador ignora o invisível (full scan) e passa a usar com `use_invisible_indexes=on` e depois `VISIBLE` (range, 508 linhas) — rollout seguro.
- **sys schema:** `innodb_buffer_stats_by_table`, `schema_unused_indexes` (com a ressalva "não usado ≠ inútil"), `schema_redundant_indexes`.
- **Config real:** `flush_log_at_trx_commit=1`, `O_DIRECT`, redo 100 MB, `max_connections=151`, `thread_cache=9`.

## Ferramentas e plugins

Para tuning de MySQL, as ferramentas de referência são externas: **sysbench** (benchmark), **Percona Toolkit** (`pt-query-digest`), **MySQLTuner**, **PMM/Grafana**, **innotop**. Não há plugin do diretório **específico de tuning MySQL**; os que ajudam no ciclo são o **Data** (SQL/exploração/dashboards das métricas) e o **Bench Claude** (relatórios de benchmark) — complementares, não substitutos.

## Ambiente de homologação

| Item | Valor |
|---|---|
| MySQL Server | 9.7.4 |
| RAM do host | ~7,6 GB |
| `innodb_buffer_pool_size` | 128 MB (default) |
| Dados de teste | ~42 MB (`pedidos` 500k, `eventos` 200k) |

Medições no primário do InnoDB Cluster do lab.

## Estrutura

```
Tuning/
├── README.md                              (este arquivo)
├── tuning-fundamentos-metodologia.md
├── tuning-innodb-buffer-pool.md
├── tuning-memoria-e-buffers.md
├── tuning-redo-log-e-durabilidade.md
├── tuning-conexoes-e-threads.md
├── tuning-indices-e-otimizador.md
└── tuning-diagnostico-e-benchmark.md
```

## Aviso

Procedimentos validados em laboratório. Teste em homologação antes de produção, **uma mudança por vez** e com benchmark. Variáveis de memória por conexão multiplicam pela quantidade de conexões — dimensione com a conta do teto de RAM.

## Mantenedor

DB Sentinel Consultoria em Tecnologia da Informacao Ltda.
