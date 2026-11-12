# DB Sentinel | MySQL — Backup & Restore

Parte do **DB Sentinel Database Procedures Framework**: biblioteca técnica de procedimentos operacionais (SOPs) para administração de bancos de dados.

Esta subpasta cobre **backup e restore em MySQL** — o seguro mais crítico de qualquer banco. Lógico, físico e **PITR** (point-in-time recovery). Os procedimentos são independentes de clientes, não contêm dados de ambientes reais e são **validados em laboratório**: cada técnica foi executada de verdade, incluindo restaurar backups e **desfazer um desastre** por PITR.

> **Um backup que nunca foi restaurado não é um backup — é uma esperança.** Todo procedimento aqui inclui o restore, não só o backup.

## Por onde começar

1. **Fundamentos** — tipos de backup, RPO/RTO, GTID/binlog, regra 3-2-1.
2. Escolha a técnica (lógico ou físico) pela matriz de decisão.
3. **PITR** — leia antes de precisar; é o que salva de um erro humano.

## Procedimentos

| Procedimento | Tipo | Status |
|---|---|---|
| [Fundamentos e estratégia](backup-restore-fundamentos.md) | Conceitual | Referência |
| [Backup lógico com mysqldump](backup-restore-logico-mysqldump.md) | Lógico | Validado |
| [Backup lógico com MySQL Shell](backup-restore-logico-mysql-shell.md) | Lógico (paralelo) | Validado |
| [Backup físico (CLONE / XtraBackup / a frio)](backup-restore-fisico.md) | Físico | Validado |
| [Point-in-Time Recovery (PITR)](backup-restore-pitr.md) | Recuperação | Validado |

> **Status:** *Validado* = backup **e** restore executados e conferidos em laboratório. *Referência* = material de fundamentação.

## O que foi provado no lab

- **mysqldump:** dump consistente com coordenada de binlog; `DROP TABLE` acidental → restore trouxe os dados de volta.
- **MySQL Shell:** dump paralelo (500 mil linhas, 22,6 MB → 5,9 MB zstd); `DROP SCHEMA` → `loadDump` restaurou 500.000 linhas em 6 s.
- **CLONE físico:** snapshot a quente de 136 MB; instância descartável subiu do backup e serviu os dados (500k linhas).
- **XtraBackup:** backup a quente `completed OK!` (133 MiB), compatível com o server 9.7.
- **PITR:** `DELETE` sem `WHERE` **desfeito** — full restaurado + binlog reaplicado até o instante anterior ao desastre.

## Matriz de decisão

| Situação | Caminho |
|---|---|
| Base pequena, portátil, seletiva | `mysqldump` |
| Base média/grande, rápido e paralelo | MySQL Shell `util.dumpInstance` |
| Base grande, restore rápido | Físico (CLONE / XtraBackup) |
| Voltar a um instante exato / desfazer erro | **PITR** |
| Recuperar uma tabela apagada | Lógico (restaura em schema de teste) |

## Ambiente de homologação

| Item | Valor |
|---|---|
| MySQL Server | 9.7.4 |
| `log_bin` / `binlog_format` | ON / ROW |
| `gtid_mode` | ON |
| Retenção de binlog | 30 dias |
| Plugin CLONE | ACTIVE |
| Percona XtraBackup | 9.7.1 |

Os backups devem rodar preferencialmente em um **secundário** do InnoDB Cluster (ver subpasta `InnoDB-Cluster/`).

## Estrutura

```
Backup-Restore/
├── README.md                                 (este arquivo)
├── backup-restore-fundamentos.md
├── backup-restore-logico-mysqldump.md
├── backup-restore-logico-mysql-shell.md
├── backup-restore-fisico.md
└── backup-restore-pitr.md
```

## Regras de ouro

1. **Teste o restore** com a mesma frequência do backup (fire drill).
2. **3-2-1:** 3 cópias, 2 mídias, 1 fora do local.
3. **Rode em um secundário** e com backup consistente (`--single-transaction` ou ferramenta a quente).
4. **Guarde a coordenada de binlog/GTID** junto do backup — é a ponte para o PITR.
5. **Atenção ao GTID** no restore: `OFF` para o mesmo cluster, `ON`/replace para semear um novo.

## Aviso

Procedimentos validados em laboratório. Teste em homologação antes de produção. Operações de restore **sobrescrevem dados** — restaure primeiro em instância/schema de teste e só então promova.

## Mantenedor

DB Sentinel Consultoria em Tecnologia da Informacao Ltda.
