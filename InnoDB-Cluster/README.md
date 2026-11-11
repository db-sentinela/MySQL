# DB Sentinel | MySQL — InnoDB Cluster

Parte do **DB Sentinel Database Procedures Framework**: biblioteca técnica de procedimentos operacionais (SOPs) para administração de bancos de dados.

Esta subpasta cobre **MySQL InnoDB Cluster** — alta disponibilidade com Group Replication, MySQL Shell (AdminAPI) e MySQL Router. Os procedimentos são independentes de clientes, não contêm dados de ambientes reais e são validados em laboratório antes da publicação.

Os documentos são **didáticos**: escritos para serem seguidos por estagiários e analistas júnior, explicando o *porquê* de cada passo, não só o comando.

## Por onde começar

Se você nunca mexeu neste cluster, leia nesta ordem:

1. **Fundamentos** — entenda quórum, GTID, papéis e failover.
2. **Instalação** — como o cluster foi montado (topologia e servidores).
3. **Failover e switchover** — o dia a dia da troca de primário.
4. Os runbooks de desastre (**queda total**, **perda de quórum**) — para a emergência.

## Procedimentos

### Base conceitual

| Procedimento | Descrição | Status |
|---|---|---|
| [Fundamentos de Alta Disponibilidade](innodb-cluster-fundamentos-alta-disponibilidade.md) | Quórum, GTID, Group Replication, Single-Primary, papéis, failover × switchover e o papel do Router | Conceitual |

### Instalação

| Procedimento | Descrição | Status |
|---|---|---|
| [InnoDB Cluster Single-Primary com MySQL Router](innodb-cluster-single-primary-router.md) | Cluster de 3 nós com um primário (escrita) e dois secundários (leitura), failover automático e porta única de RW e RO via Router | Validado |

### Operação

| Procedimento | Descrição | Status |
|---|---|---|
| [Failover automático e Switchover planejado](innodb-cluster-failover-switchover.md) | Troca de primário na queda (automática) e planejada (`setPrimaryInstance`), com simulação real | Validado |
| [Desligamento e inicialização ordenados](innodb-cluster-start-stop-ordenado.md) | Parar e subir o ambiente (ou um nó) sem crash nem perda de quórum | Validado |
| [Substituição e recuperação de nó (clone)](innodb-cluster-substituicao-no-clone.md) | Remover e reprovisionar um nó do zero com `recoveryMethod:'clone'` | Validado |

### Recuperação de desastre

| Procedimento | Descrição | Status |
|---|---|---|
| [Recuperação de queda total](innodb-cluster-recuperacao-queda-total.md) | Cluster inteiro caído (`rebootClusterFromCompleteOutage`) | Validado |
| [Perda de quórum](innodb-cluster-perda-de-quorum.md) | Maioria caiu, escrita bloqueada (`forceQuorumUsingPartitionOf`) — inclui riscos de split-brain | Validado |

### Proteção e observabilidade

| Procedimento | Descrição | Status |
|---|---|---|
| [Backup e Restore](innodb-cluster-backup-restore.md) | Backup lógico (dump/Shell), restore e a pegadinha de GTID | Validado |
| [Monitoração e Healthcheck](innodb-cluster-monitoracao-healthcheck.md) | O que vigiar, consultas prontas, `statusText` e script de healthcheck | Validado |
| [Router redundante e VIP](innodb-cluster-router-ha-vip.md) | Alta disponibilidade da camada do Router (múltiplos Routers, VIP/keepalived) | Referência |

> **Status:** *Validado* = executado e conferido em laboratório. *Conceitual* = material de fundamentação. *Referência* = config/arquitetura verificadas, com padrões de produção documentados.

## Ambiente de homologação

| Item | Versão |
|---|---|
| Sistema operacional | Ubuntu 26.04.1 LTS |
| MySQL Server | 9.7.4 LTS |
| MySQL Shell | 26.7.1 |
| MySQL Router | 26.7.0 |
| Virtualização | Oracle VirtualBox |

Topologia: 3 nós (`mysql-db-01` PRIMARY inicial, `mysql-db-02` e `mysql-db-03` SECONDARY) + MySQL Router. Detalhes na [instalação](innodb-cluster-single-primary-router.md). Cada procedimento informa as versões exatas que usa.

## Padrão dos documentos

Todo procedimento desta subpasta segue o mesmo formato:

- Topologia/fluxo em diagrama quando ajuda.
- Versões e pré-requisitos explícitos.
- Passos numerados, com comandos prontos para copiar.
- **Saída real** da execução em laboratório como "resultado esperado".
- Seção de observações, troubleshooting e "não incluído", com links cruzados entre os documentos.

## Estrutura

```
InnoDB-Cluster/
├── README.md                                            (este arquivo)
├── innodb-cluster-fundamentos-alta-disponibilidade.md
├── innodb-cluster-single-primary-router.md
├── innodb-cluster-failover-switchover.md
├── innodb-cluster-start-stop-ordenado.md
├── innodb-cluster-substituicao-no-clone.md
├── innodb-cluster-recuperacao-queda-total.md
├── innodb-cluster-perda-de-quorum.md
├── innodb-cluster-backup-restore.md
├── innodb-cluster-monitoracao-healthcheck.md
└── innodb-cluster-router-ha-vip.md
```

## Como usar

1. Leia a seção de versões e pré-requisitos do procedimento.
2. Execute os passos na ordem indicada.
3. Valide cada etapa (o "resultado esperado") antes de seguir para a próxima.
4. Troque todas as senhas de exemplo antes de usar em qualquer ambiente real.
5. Nos runbooks de desastre, confirme **qual** cenário você tem (o fluxograma em *Recuperação de queda total* separa queda total de perda de quórum).

## Aviso

Os procedimentos foram validados em laboratório. Teste em ambiente de homologação antes de aplicar em produção. Operações de desastre (reboot de cluster, forçar quórum, clone) alteram o estado do cluster — leia as seções de risco antes de executar.

## Mantenedor

DB Sentinel Consultoria em Tecnologia da Informacao Ltda.
