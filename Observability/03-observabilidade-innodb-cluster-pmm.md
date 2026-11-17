# 03 — Monitorando um InnoDB Cluster (Group Replication) no PMM

Quando os nós MySQL deixam de ser instâncias isoladas e formam um **InnoDB Cluster de Alta Disponibilidade** (Group Replication), o monitoramento precisa correlacionar os nós sob um mesmo cluster, acompanhar os papéis (PRIMARY/SECONDARY), os estados dos membros e a saúde da replicação de grupo. Este documento mostra como instrumentar os 3 nós e validar a telemetria de Group Replication no PMM 3.

> Pré-leitura: `01-observabilidade-pmm-instalacao-e-cliente.md` e `02-observabilidade-mysql-single-instance.md`.

---

## O que é

No PMM 3, monitorar um InnoDB Cluster agrega:

1. **Agrupamento lógico (`--cluster`):** ao adicionar o serviço em **cada** nó com `--cluster=innodb_cluster_lab`, o PMM unifica as séries no dashboard **"MySQL Group Replication Summary"** e nos filtros de cluster do Grafana.
2. **Métricas de Group Replication (`mysql_perf_schema_replication_group_*`):** o `mysqld_exporter` lê o `performance_schema` do MySQL e expõe os contadores do grupo — o **equivalente MySQL ao `wsrep_*`** do Galera. No lab foram coletadas **21 séries** por nó, incluindo:
   - `mysql_perf_schema_replication_group_member_info` (host, papel e **estado** de cada membro).
   - `mysql_perf_schema_replication_group_worker_lag_in_seconds` (atraso de aplicação).
   - `mysql_perf_schema_replication_group_worker_apply_time_seconds` / `_transport_time_seconds` / `_rep_delay_seconds`.
3. **Todos os nós são iguais:** diferente do Galera (que tem árbitro `garbd`), em um InnoDB Cluster os 3 nós rodam `mysqld` e participam do grupo — logo, **os 3 recebem serviço MySQL** no PMM.

---

## Por que importa

1. **Visibilidade de quórum e failover:** o PMM mostra, por nó, o `member_state` (`ONLINE`/`RECOVERING`/`UNREACHABLE`/`OFFLINE`) e o `member_role` (`PRIMARY`/`SECONDARY`). Uma troca de primário (failover) ou a queda de um membro fica imediatamente visível.
2. **Diagnóstico de atraso de replicação:** os contadores `worker_lag_in_seconds` e `apply_time` apontam o nó que está acumulando backlog antes que ele vire gargalo.
3. **Correlação com a camada de aplicação:** com os 3 nós sob a mesma tag `cluster`, dá para comparar carga, QPS e latência entre primário e secundários no mesmo painel.

---

## Como reproduzir

### 1. Garantir o usuário `pmm` acessível na rede privada
Criado **uma vez no primário** (replica para todos pelo Group Replication) — ver doc 02. O host `'pmm'@'192.168.1.%'` cobre a conexão de cada nó ao seu próprio IP.

### 2. Adicionar o serviço em CADA nó, com a tag de cluster
Repita em `mysql-db-01` (192.168.1.20), `mysql-db-02` (192.168.1.25) e `mysql-db-03` (192.168.1.26), trocando o IP/nome:

```bash
# No mysql-db-01:
sudo pmm-admin add mysql \
  --service-name=mysql-db-01 --host=192.168.1.20 --port=3306 \
  --username=pmm --password='Pmm#2026' \
  --cluster=innodb_cluster_lab --query-source=perfschema --environment=lab

# No mysql-db-02:  --service-name=mysql-db-02 --host=192.168.1.25 ...
# No mysql-db-03:  --service-name=mysql-db-03 --host=192.168.1.26 ...
```

### 3. Validar os serviços e as métricas de grupo (API do PMM)
```bash
# Quantos servicos MySQL do cluster estao UP:
curl -sk -u 'admin:Admin#2026' -G 'https://192.168.1.16:8443/prometheus/api/v1/query' \
  --data-urlencode 'query=mysql_up{cluster="innodb_cluster_lab"}'

# Papel e estado de cada membro do grupo:
curl -sk -u 'admin:Admin#2026' -G 'https://192.168.1.16:8443/prometheus/api/v1/query' \
  --data-urlencode 'query=mysql_perf_schema_replication_group_member_info{service_name="mysql-db-01"}'
```

---

## O que você vê (saída real)

### 1. Os três serviços reportando `mysql_up=1` sob o mesmo cluster:
```text
mysql_up{service_name="mysql-db-01", cluster="innodb_cluster_lab"}  →  1
mysql_up{service_name="mysql-db-02", cluster="innodb_cluster_lab"}  →  1
mysql_up{service_name="mysql-db-03", cluster="innodb_cluster_lab"}  →  1
```

### 2. Membro do grupo com papel e estado (métrica de GR):
```json
mysql_perf_schema_replication_group_member_info{
  service_name="mysql-db-01",
  member_host="mysql-db-01",
  member_role="PRIMARY",
  member_state="ONLINE"
}
```

### 3. Inventário de métricas de Group Replication coletadas (por nó):
```text
mysql_perf_schema_replication_group_member_info
mysql_perf_schema_replication_group_worker_apply_time_seconds
mysql_perf_schema_replication_group_worker_lag_in_seconds
mysql_perf_schema_replication_group_worker_rep_delay_seconds
mysql_perf_schema_replication_group_worker_transport_time_seconds
...  (21 series por no)
```

**Análise das evidências:** os três nós (`mysql-db-01/02/03`) reportam sob `cluster="innodb_cluster_lab"`; o `mysql-db-01` aparece como `PRIMARY`/`ONLINE`; os contadores de atraso de replicação de grupo estão disponíveis para alertas.

---

## Lições

- **Group Replication ≠ Galera no namespace de métricas.** Dashboards e alertas de MariaDB usam `mysql_global_status_wsrep_*`. O InnoDB Cluster usa **`mysql_perf_schema_replication_group_*`**. Ao reaproveitar alertas entre os repositórios, troque o namespace.
- **Padronize os nomes de serviço** (`mysql-db-01`, `-02`, `-03`) com prefixo uniforme — é o que o Grafana usa como chave de filtro.
- **Isolamento entre SGBDs:** adicionar/remover serviços MySQL no cliente **não afeta** os outros clusters do PMM (MariaDB/Galera, PostgreSQL/Patroni). Confirme com `sudo pmm-admin inventory list services`.

---

## Pegadinhas

- **`connect: connection refused` no `127.0.0.1:3306`.** Se o serviço for adicionado com `--host=127.0.0.1` mas o usuário `pmm` só existir para a rede privada (ou vice-versa), o exporter falha. Alinhe o `--host` do `add` com o host do `CREATE USER`. No lab usamos `--host=<ip do no>` e `'pmm'@'192.168.1.%'`.
- **Troca de primário muda o papel, não o serviço.** Após um failover/switchover, o `member_role` de um nó muda de `SECONDARY` para `PRIMARY` nas métricas — isso é esperado e **não** exige re-adicionar o serviço. Monitore o `member_role` em vez de fixar qual nó é primário.
- **Não confunda com o Router.** O MySQL Router (camada de conexão) não é um nó de banco; ele não entra como serviço MySQL. Para observá-lo, monitore o **host** dele (node_exporter) e, se exposto, seu endpoint de métricas — tema fora deste documento.
