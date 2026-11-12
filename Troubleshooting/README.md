# DB Sentinel | MySQL — Troubleshooting

Parte do **DB Sentinel Database Procedures Framework**: biblioteca técnica de procedimentos operacionais (SOPs) para administração de bancos de dados.

Esta subpasta cobre **diagnóstico e solução de problemas em MySQL**: **concorrência** (locks, bloqueios, deadlocks) e **desempenho de queries** (EXPLAIN, índices, slow log). Como identificar, onde olhar e como resolver. Os procedimentos são independentes de clientes, não contêm dados de ambientes reais e são **validados em laboratório** reproduzindo os cenários de verdade.

Os documentos são **didáticos**: escritos para analistas júnior e estagiários, com a saída real dos comandos de diagnóstico.

## Por onde começar

1. **Fundamentos** — entenda os tipos de lock e a caixa de ferramentas de diagnóstico.
2. Depois vá ao runbook do seu sintoma (bloqueio, deadlock ou DDL travado).

## Procedimentos

### Concorrência (locks)

| Procedimento | Sintoma que resolve | Status |
|---|---|---|
| [Locks e bloqueios: fundamentos e onde olhar](troubleshooting-locks-fundamentos.md) | Base conceitual + mapa "sintoma → onde olhar" | Validado |
| [Identificar e resolver bloqueios (lock waits)](troubleshooting-bloqueios-lock-waits.md) | "Uma query não termina" / erro `1205` | Validado |
| [Deadlocks](troubleshooting-deadlocks.md) | Erro `1213 Deadlock found` | Validado |
| [Metadata locks (DDL travado)](troubleshooting-metadata-locks.md) | `ALTER` preso em "Waiting for table metadata lock" | Validado |

### Desempenho de queries

| Procedimento | Sintoma que resolve | Status |
|---|---|---|
| [Queries lentas: EXPLAIN e índices](troubleshooting-queries-lentas-explain.md) | Query lenta por full table scan / falta de índice | Validado |
| [Slow query log](troubleshooting-slow-query-log.md) | Capturar quais queries estão demorando | Validado |
| [Achar queries lentas (performance_schema/sys)](troubleshooting-queries-lentas-performance-schema.md) | Descobrir as queries mais custosas sem o slow log | Validado |

> **Status:** *Validado* = cenário reproduzido e saída capturada em laboratório.

## A caixa de ferramentas (referência rápida)

| Ferramenta | Para quê |
|---|---|
| `sys.innodb_lock_waits` | Quem trava quem (comece sempre aqui) |
| `information_schema.innodb_trx` | Transações ativas e estado `LOCK WAIT` |
| `performance_schema.data_locks` | Os locks de linha concedidos/esperando |
| `performance_schema.metadata_locks` | Locks de metadados (DDL) |
| `SHOW ENGINE INNODB STATUS` (`-E`) | Último deadlock + transações |
| `SHOW PROCESSLIST` | Visão geral das conexões e estados |
| `KILL` / `KILL QUERY` | Encerrar o bloqueador |
| `EXPLAIN` / `EXPLAIN ANALYZE` | Plano e custo real de uma query |
| `sys.statement_analysis` | Queries mais custosas (por tempo total) |
| `sys.statements_with_full_table_scans` | Queries que varrem a tabela toda |
| Slow query log | Capturar queries acima de um tempo |

## Ambiente de homologação

| Item | Versão |
|---|---|
| Sistema operacional | Ubuntu 26.04.1 LTS |
| MySQL Server | 9.7.4 LTS |
| Virtualização | Oracle VirtualBox |

Cenários reproduzidos no nó primário do InnoDB Cluster do lab, em um schema de teste `tshoot` (tabela `contas`). Cada documento informa o cenário exato que usa.

## Padrão dos documentos

- Explicação do mecanismo antes do comando.
- Cenário reproduzível passo a passo (quem trava quem, e quando).
- **Saída real** das views de diagnóstico.
- Como resolver e como prevenir.
- Seções de observações e "não incluído", com links cruzados.

## Estrutura

```
Troubleshooting/
├── README.md                                   (este arquivo)
├── troubleshooting-locks-fundamentos.md
├── troubleshooting-bloqueios-lock-waits.md
├── troubleshooting-deadlocks.md
├── troubleshooting-metadata-locks.md
├── troubleshooting-queries-lentas-explain.md
├── troubleshooting-slow-query-log.md
└── troubleshooting-queries-lentas-performance-schema.md
```

## Como usar

1. Identifique o sintoma (query travada, erro 1213, ALTER preso).
2. Abra o procedimento correspondente e rode os comandos de diagnóstico na ordem.
3. Confirme o diagnóstico (quem trava quem) **antes** de agir.
4. Ao usar `KILL`, lembre que a transação da vítima é desfeita (`ROLLBACK`).

## Próximos temas (planejados)

- Conexões (`max_connections`, `too many connections`).
- Replicação do Group Replication (atraso, erros de applier).
- Recursos do servidor (CPU, memória, disco, `tmp` cheio).

## Aviso

Os procedimentos foram validados em laboratório. Teste em ambiente de homologação antes de aplicar em produção. `KILL` encerra conexões e desfaz transações — confirme o alvo antes.

## Mantenedor

DB Sentinel Consultoria em Tecnologia da Informacao Ltda.
