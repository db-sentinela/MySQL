# DB Sentinel | MySQL — Observabilidade com Percona Monitoring and Management (PMM 3)

Parte do **DB Sentinel Database Procedures Framework**: biblioteca técnica de procedimentos operacionais (SOPs) para administração e sustentação de bancos de dados em ambientes corporativos de missão crítica.

Esta subpasta documenta a **Observabilidade Centralizada com PMM 3.10** para MySQL, cobrindo desde a instalação do cliente e a adição de uma instância *standalone* até a instrumentação de um **InnoDB Cluster (Group Replication)** de alta disponibilidade. Todos os procedimentos — instalação, usuário de menor privilégio, telemetria de queries (QAN) e métricas de Group Replication — foram **validados e homologados no laboratório real** integrado ao PMM Server (`https://192.168.1.16:8443`).

---

## Procedimentos

| Procedimento | Descrição | Status |
|---|---|---|
| [01 — Instalação e Arquitetura do PMM Client](01-observabilidade-pmm-instalacao-e-cliente.md) | Arquitetura do PMM 3.10, instalação do agente, registro seguro, portas e validação de conectividade | Validado |
| [02 — Adicionando uma Instância MySQL (Single-Instance)](02-observabilidade-mysql-single-instance.md) | Usuário `pmm` de menor privilégio, Performance Schema, `pmm-admin add mysql` e Query Analytics (QAN) | Validado |
| [03 — Monitorando um InnoDB Cluster (Group Replication)](03-observabilidade-innodb-cluster-pmm.md) | Agrupamento (`--cluster=innodb_cluster_lab`), métricas `mysql_perf_schema_replication_group_*`, papéis e estados dos membros | Validado |
| [04 — Dashboards, Métricas-Chave do GR e QAN](04-observabilidade-dashboards-metricas-e-qan.md) | Navegação no Grafana, filtros corretos, matriz de alerta de Group Replication e análise de QAN | Validado |

---

## Diagrama da Arquitetura de Observabilidade

```
 ┌────────────────────────────────────────────────────────────────────────────────┐
 │                       PERCONA MONITORING & MANAGEMENT                          │
 │                        PMM SERVER 3.10 (192.168.1.16:8443)                     │
 │   ┌───────────────────────┐  ┌───────────────────────┐  ┌──────────────────┐   │
 │   │ VictoriaMetrics (TSDB)│  │  ClickHouse (QAN Data)│  │  Grafana 11+ UI  │   │
 │   └───────────▲───────────┘  └──────────▲────────────┘  └────────▲─────────┘   │
 └───────────────┼──────────────────────────┼────────────────────────┼────────────┘
                 │ Push Metrics (8443)      │ Query Telemetry        │ Dashboards
 ┌───────────────┴──────────────────────────┴────────────────────────┴────────────┐
 │ TOPOLOGIA DO LAB (InnoDB Cluster: innodb_cluster_lab | Env: lab)               │
 │                                                                                │
 │  ┌───────────────────────────┐ ┌───────────────────────────┐ ┌───────────────┐ │
 │  │   mysql-db-01 (PRIMARY)   │ │  mysql-db-02 (SECONDARY)  │ │ mysql-db-03   │ │
 │  │      192.168.1.20         │ │      192.168.1.25         │ │ (SECONDARY)   │ │
 │  │  • mysqld :3306           │ │  • mysqld :3306           │ │ 192.168.1.26  │ │
 │  │  • MySQL Router (6446/47) │ │                           │ │ • mysqld:3306 │ │
 │  │  [PMM AGENT]:             │ │  [PMM AGENT]:             │ │ [PMM AGENT]:  │ │
 │  │   ├─ mysqld_exporter      │ │   ├─ mysqld_exporter      │ │  ├─ mysqld_exp│ │
 │  │   ├─ mysql_perfschema_ag. │ │   ├─ mysql_perfschema_ag. │ │  ├─ perfschema│ │
 │  │   ├─ node_exporter        │ │   ├─ node_exporter        │ │  ├─ node_exp  │ │
 │  │   └─ vmagent (push)       │ │   └─ vmagent (push)       │ │  └─ vmagent   │ │
 │  └─────────────▲─────────────┘ └─────────────▲─────────────┘ └───────▲───────┘ │
 │                └──────────── Group Replication (3306) ───────────────┘         │
 └────────────────────────────────────────────────────────────────────────────────┘
```

---

## Mapa de Bolso do DBA (Onde Olhar / Comandos Essenciais)

| Recurso | O que inspecionar | Comando essencial | Ação |
|---|---|---|---|
| **Saúde dos Agentes** | pmm-agent e exporters | `sudo /usr/sbin/pmm-admin status` | Todos devem reportar `Running` (nenhum `Waiting`/`Failed`). |
| **Serviços Ativos** | Instâncias monitoradas | `sudo pmm-admin list` | Valide o IP/porta real (ex.: `192.168.1.20:3306`), sem duplicatas em `127.0.0.1`. |
| **Inventário Geral** | Todos os serviços (incl. outros SGBDs) | `sudo pmm-admin inventory list services` | Garante que MariaDB/PostgreSQL seguem preservados. |
| **Membros do Grupo** | Estado/papel no Prometheus | `curl -sk -u admin:*** '.../query?query=mysql_perf_schema_replication_group_member_info'` | Deve listar os 3 membros `ONLINE`, 1 `PRIMARY`. |
| **Tamanho do Cluster** | Membros ONLINE | `curl ... 'query=count(mysql_perf_schema_replication_group_member_info{member_state="ONLINE"})'` | Deve retornar `3`. |
| **Filtro de Dashboards** | Evitar nó fantasma | Seletor `Service Name` → `mysql-db-01/02/03` | **Nunca** filtrar por `pmm-server-*` (instância interna do PMM). |

---

## Ambiente de Homologação

| Nó | IP | Papel no Lab | Agentes PMM Ativos |
|---|---|---|---|
| **pmm-mon-01** | `192.168.1.16` | PMM Server 3.10 (Docker, HTTPS 8443) | Servidor Central (VictoriaMetrics, ClickHouse, Grafana) |
| **mysql-db-01** | `192.168.1.20` | InnoDB Cluster (PRIMARY) + MySQL Router | `mysqld_exporter`, `mysql_perfschema_agent`, `node_exporter`, `vmagent` |
| **mysql-db-02** | `192.168.1.25` | InnoDB Cluster (SECONDARY) | `mysqld_exporter`, `mysql_perfschema_agent`, `node_exporter`, `vmagent` |
| **mysql-db-03** | `192.168.1.26` | InnoDB Cluster (SECONDARY) | `mysqld_exporter`, `mysql_perfschema_agent`, `node_exporter`, `vmagent` |

- **SO:** Ubuntu 26.04.1 LTS (`resolute`)
- **MySQL Server:** 9.7.4 LTS
- **PMM:** 3.10.0 (Client & Server) — pacote `pmm-client 3.10.0-1.resolute`
- **Cluster Name (tag PMM):** `innodb_cluster_lab` | **Environment:** `lab`
- **Namespace de métricas GR:** `mysql_perf_schema_replication_group_*` (equivalente MySQL ao `wsrep_*` do Galera)

---

## Mantenedor

**DB Sentinel Consultoria em Tecnologia da Informação Ltda.**  
*Database Procedures Framework — Operational Standards & Best Practices.*
