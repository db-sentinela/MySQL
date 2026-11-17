# 04 — Dashboards do PMM, Métricas-Chave do Group Replication e QAN

Visualizar métricas no Grafana sem entender os filtros e as peculiaridades do PMM Server leva a interpretações falsas e alarmes indevidos. Este documento mostra como navegar nos painéis do **PMM 3.10** para MySQL/InnoDB Cluster, configurar os filtros corretamente (evitando o nó fantasma do próprio PMM) e interpretar as métricas de Group Replication e do **Query Analytics (QAN)**.

> Pré-leitura: `03-observabilidade-innodb-cluster-pmm.md`.

---

## O que é

Dashboards pré-construídos relevantes para o InnoDB Cluster:
1. **MySQL Instance Summary:** saúde da instância — CPU, Buffer Pool, threads, commits, I/O e QPS.
2. **MySQL Group Replication Summary:** saúde do grupo — papéis dos membros (`PRIMARY`/`SECONDARY`), estados (`ONLINE`/`RECOVERING`/...), atraso de aplicação e filas.
3. **PMM Query Analytics (QAN):** perfil de carga em tempo real, agrupando consultas em *digests* e ordenando por **tempo de carga total**.

---

## Por que importa

1. **Eliminar o "nó fantasma" (`pmm-server-*`):** o PMM Server roda em container com sua própria base interna. Por ordenação alfabética, o Grafana às vezes pré-seleciona essa instância interna no filtro `Service Name` — e o DBA acaba analisando o próprio servidor de monitoramento achando que é o banco de produção.
2. **MTTR menor:** saber exatamente quais painéis olhar reduz o tempo de diagnóstico de 30 min para 2 min.
3. **Caça à query vilã (QAN):** correlaciona picos de CPU com a consulta exata, mostrando a query parametrizada, o p95 de latência e a contagem de execuções.

---

## Como reproduzir

### 1. Acesso à interface web
- **URL:** `https://192.168.1.16:8443`
- **Usuário:** `admin`  — **Senha:** `Admin#2026`

### 2. Ajustar os filtros no topo do dashboard
Em **"MySQL Instance Summary"** ou **"MySQL Group Replication Summary"**:
- **Environment:** `lab`
- **Cluster:** `innodb_cluster_lab`
- **Service Name:** `mysql-db-01`, `mysql-db-02` ou `mysql-db-03`
- **Interval:** `1m` ou `5m`

### 3. Consultar métricas essenciais via API (scripts/checagens rápidas)
```bash
# 3A: Quantos membros do grupo estao ONLINE (o "tamanho do cluster")
curl -sk -u 'admin:Admin#2026' -G 'https://192.168.1.16:8443/prometheus/api/v1/query' \
  --data-urlencode 'query=count(mysql_perf_schema_replication_group_member_info{member_state="ONLINE"})'

# 3B: Papel/estado de cada membro
curl -sk -u 'admin:Admin#2026' -G 'https://192.168.1.16:8443/prometheus/api/v1/query' \
  --data-urlencode 'query=mysql_perf_schema_replication_group_member_info'

# 3C: Atraso de aplicacao de replicacao por no
curl -sk -u 'admin:Admin#2026' -G 'https://192.168.1.16:8443/prometheus/api/v1/query' \
  --data-urlencode 'query=mysql_perf_schema_replication_group_worker_lag_in_seconds'
```

---

## O que você vê (saída real)

### Membro do grupo com papel e estado (métrica de GR):
```json
mysql_perf_schema_replication_group_member_info{
  service_name="mysql-db-01", member_host="mysql-db-01",
  member_role="PRIMARY", member_state="ONLINE"
}
```

### Os três serviços reportando sob o mesmo cluster:
```text
mysql_up{service_name="mysql-db-01", cluster="innodb_cluster_lab"}  →  1
mysql_up{service_name="mysql-db-02", cluster="innodb_cluster_lab"}  →  1
mysql_up{service_name="mysql-db-03", cluster="innodb_cluster_lab"}  →  1
```

---

## Métricas-chave e matriz de alerta do InnoDB Cluster

| Condição | Métrica | Severidade | Significado |
|---|---|---|---|
| Perda de membro | `count(mysql_perf_schema_replication_group_member_info{member_state="ONLINE"}) < 3` | 🔴 ALTA | Nó caiu ou partição de rede — risco de quórum |
| Membro não-ONLINE | `member_state != "ONLINE"` (RECOVERING/UNREACHABLE/OFFLINE) | 🟠 MÉDIA | Nó fora de sincronia |
| Zero/dois primários | `count(...member_role="PRIMARY") != 1` (single-primary) | 🔴 ALTA | Failover em curso ou anomalia |
| Atraso de replicação | `mysql_perf_schema_replication_group_worker_lag_in_seconds > N` | 🟠 MÉDIA | Secundário acumulando backlog |
| Buffer pool pequeno | `rate(mysql_global_status_innodb_buffer_pool_reads[5m])` alto | 🟡 BAIXA | Leituras indo ao disco (ver subpasta Tuning) |

> **GR ≠ Galera:** se você trouxe alertas do repositório MariaDB, troque `mysql_global_status_wsrep_*` por `mysql_perf_schema_replication_group_*`. O conceito de `wsrep_cluster_size` vira **contagem de membros `ONLINE`**.

---

## Lições

- **Como interpretar o QAN:** foque na coluna **Load %**. Uma query de 1 ms executada 500 mil vezes/min pesa muito mais no banco do que uma query lenta rodada uma vez por dia. O QAN ordena por carga total — ataque o topo.
- **Leia o papel, não o nome.** Num InnoDB Cluster, qual nó é `PRIMARY` muda com failover/switchover. Monte painéis e alertas sobre `member_role`/`member_state`, não sobre "o db-01 é o primário".
- **Correlacione camadas:** um pico de latência no QAN + aumento de `Threads_running` no Instance Summary + I/O no Node Summary conta a história completa do incidente.

---

## Pegadinhas

- **A armadilha do `pmm-server-*` no filtro.** Nunca analise um gráfico sem conferir o seletor `Service Name`. Se estiver em `pmm-server-postgresql` (a base interna do container PMM), você verá o próprio servidor de monitoramento, não o seu cluster MySQL.
- **Contadores cumulativos exigem `rate()`.** Métricas como leituras do buffer pool ou bytes enviados são acumuladas desde o boot. Alertar sobre o valor absoluto dispara para sempre; use `rate(metrica[5m])`.
- **Resoluções `hr`/`mr`/`lr`.** O PMM coleta a mesma métrica em várias resoluções (high/medium/low). Ao contar séries (ex.: `count(mysql_up)`), o número pode ser maior que a quantidade de nós por causa das múltiplas resoluções/jobs — conte por `service_name` (`count by (service_name) (...)`) para ter o número real de instâncias.
