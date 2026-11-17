# DB Sentinel | MySQL

Parte do **DB Sentinel Database Procedures Framework**: biblioteca técnica de procedimentos operacionais (SOPs) para administração de bancos de dados.

Este repositório cobre **MySQL**. Os procedimentos são independentes de clientes, não contêm dados de ambientes reais e são **validados em laboratório** antes da publicação — cada um executa os comandos de verdade e usa a saída real como "resultado esperado".

## Seções

| Seção | Cobertura | Documentos |
|---|---|---|
| [InnoDB-Cluster](InnoDB-Cluster/) | Alta disponibilidade: instalação, failover/switchover, quórum, queda total, clone, start/stop, monitoração, Router HA | 10 + README |
| [Troubleshooting](Troubleshooting/) | Locks e bloqueios, deadlocks, metadata locks, queries lentas (EXPLAIN, slow log, performance_schema) | 7 + README |
| [Backup & Restore](Backup&Restore/) | Lógico (mysqldump, MySQL Shell), físico (CLONE, XtraBackup) e **PITR** | 5 + README |
| [Security](Security/) | Hardening, privilégios, autenticação, privilege escalation, TLS, SQL injection, auditoria | 7 + README |
| [Tuning](Tuning/) | Buffer pool, memória, redo/durabilidade, conexões, histogramas e índices invisíveis, benchmark | 7 + README |
| [Observability](Observability/) | Observabilidade com PMM 3: instalação do cliente, adição de instância, InnoDB Cluster (Group Replication) e dashboards/QAN | 4 + README |

Cada seção tem um `README.md` próprio, que indexa e explica seus documentos.

## Ambiente de homologação

| Item | Versão |
|---|---|
| Sistema operacional | Ubuntu 26.04.1 LTS |
| MySQL Server | 9.7.4 LTS |
| MySQL Shell | 26.7.1 |
| MySQL Router | 26.7.0 |
| Virtualização | Oracle VirtualBox |

Topologia: InnoDB Cluster de 3 nós (1 primário + 2 secundários) com MySQL Router. Cada procedimento informa as versões exatas que usa.

## Padrão dos documentos

Todo procedimento deste repositório segue o mesmo formato:

- Diagrama (topologia/fluxo) quando ajuda.
- Versões e pré-requisitos explícitos.
- Passos numerados, com comandos prontos para copiar.
- **Saída real** da execução em laboratório como "resultado esperado".
- Observações, troubleshooting e "não incluído", com links cruzados entre documentos.
- Didático: escrito para ser seguido por estagiários e analistas júnior.

## Estrutura

```
MySQL/
├── readme.md
├── InnoDB-Cluster/      (HA — 10 docs + README)
├── Troubleshooting/     (locks, deadlocks, queries lentas — 7 docs + README)
├── Backup&Restore/      (lógico, físico, PITR — 5 docs + README)
├── Security/            (hardening, vulnerabilidades, TLS — 7 docs + README)
├── Tuning/              (buffer pool, otimizador, benchmark — 7 docs + README)
└── Observability/       (PMM: instalação, add instância, InnoDB Cluster, dashboards — 4 docs + README)
```

## Como usar

1. Escolha a seção pelo seu objetivo e leia o `README.md` dela.
2. Leia a seção de versões e pré-requisitos do procedimento.
3. Execute os passos na ordem indicada e valide cada etapa antes de seguir.
4. Troque todas as senhas de exemplo antes de usar em qualquer ambiente real.

## Aviso

Os procedimentos foram validados em laboratório. Teste em ambiente de homologação antes de aplicar em produção. Operações destrutivas ou de segurança (recuperação de desastre, forçar TLS, política de senha) trazem seções de risco — leia antes de executar.

## Mantenedor

DB Sentinel Consultoria em Tecnologia da Informacao Ltda.
