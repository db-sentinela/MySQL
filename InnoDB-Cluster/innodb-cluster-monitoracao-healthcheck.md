# MySQL InnoDB Cluster — Monitoração e Healthcheck

O que olhar, com quais comandos, para saber se o cluster está **saudável** — e os primeiros sinais de que algo vai mal. Todas as saídas abaixo são reais do lab.

> Pré-leitura: `innodb-cluster-fundamentos-alta-disponibilidade.md` (papéis, estados, quórum).

---

## O que é "saudável" (os 4 sinais vitais)

1. **Todos os nós `ONLINE`** — nenhum `RECOVERING`, `UNREACHABLE`, `ERROR` ou `(MISSING)`.
2. **Exatamente um PRIMARY** — nem zero (sem escrita), nem dois (anomalia).
3. **Filas e conflitos baixos/zerados** — o cluster está acompanhando a carga.
4. **Router roteando** — 6446 no PRIMARY, 6447 nos secundários.

---

## 1) Membros, papel e estado

A consulta que você mais vai rodar (via socket local, em qualquer nó):

```bash
sudo mysql -t -e "
  SELECT member_host AS host, member_role AS role, member_state AS state
  FROM performance_schema.replication_group_members ORDER BY role, host;"
```

Saída real (saudável):

```
+-------------+-----------+--------+
| host        | role      | state  |
+-------------+-----------+--------+
| mysql-db-01 | PRIMARY   | ONLINE |
| mysql-db-02 | SECONDARY | ONLINE |
| mysql-db-03 | SECONDARY | ONLINE |
+-------------+-----------+--------+
```

Qualquer `state` diferente de `ONLINE`, ou qualquer `role` fora de "1 PRIMARY + N SECONDARY", é motivo de olhar mais fundo.

## 2) Atraso de replicação (lag) e conflitos

O "lag" aqui não é o do replication clássico; use as **filas** como indicador:

```bash
sudo mysql -t -e "
  SELECT m.member_host AS host,
         s.count_transactions_in_queue AS cert_queue,
         s.count_transactions_remote_in_applier_queue AS apply_queue,
         s.count_conflicts_detected AS conflicts
  FROM performance_schema.replication_group_member_stats s
  JOIN performance_schema.replication_group_members m USING(member_id)
  ORDER BY host;"
```

Saída real (saudável):

```
+-------------+------------+-------------+-----------+
| host        | cert_queue | apply_queue | conflicts |
+-------------+------------+-------------+-----------+
| mysql-db-01 |          0 |           0 |         0 |
| mysql-db-02 |          0 |           0 |         0 |
| mysql-db-03 |          0 |           0 |         0 |
+-------------+------------+-------------+-----------+
```

- **`apply_queue` subindo e não voltando a zero** = um secundário não está dando conta de aplicar (disco lento, carga). Atenção.
- **`conflicts > 0** = em Single-Primary deveria ser sempre 0. Diferente de zero é sinal de problema (ou de escrita indevida em secundário).

## 3) Healthcheck em uma linha

Bom para alerta rápido / cron:

```bash
sudo mysql -N -e "
  SELECT CONCAT('online=', SUM(member_state='ONLINE'), '/', COUNT(*),
                ' primary=', SUM(member_role='PRIMARY'))
  FROM performance_schema.replication_group_members;"
```

Saída real (saudável): `online=3/3 primary=1`

**Saudável = `online=3/3 primary=1`.** Qualquer outra coisa (`online=2/3`, `primary=0`, `primary=2`) dispara investigação.

## 4) Visão do AdminAPI e o `statusText`

```bash
mysqlsh --uri clusteradmin@mysql-db-01:3306 --js \
  --execute="print(JSON.stringify(dba.getCluster().status(), null, 2))"
```

O campo mais importante é o `status` do replicaset:

| status | Significado | Ação |
|---|---|---|
| `OK` | Saudável, com folga (tolera 1 falha) | Nada |
| `OK_NO_TOLERANCE` | Operando, mas **sem folga** (só o mínimo de nós) | Trazer nó(s) de volta |
| `OK_NO_TOLERANCE_PARTIAL` | Operando sem folga, com nó(s) `(MISSING)` | Investigar o nó faltante |
| `NO_QUORUM` | **Escrita bloqueada** — perdeu a maioria | Runbook de perda de quórum |
| `OFFLINE` / erro ao obter | Queda total | Runbook de queda total |

Saída real (resumo, saudável): `status=OK primary=mysql-db-01:3306`.

---

## 5) Lado do Router

O cluster pode estar `OK` e a aplicação falhar por causa do Router. Cheque:

```bash
# Servico e portas
systemctl is-active mysqlrouter
sudo ss -ltnp | grep -E ':(6446|6447|6450|8443)'

# Quem responde cada porta (prova de roteamento)
mysql -h 127.0.0.1 -P 6446 -u <user> -p -N -e "SELECT @@hostname, @@read_only;"   # PRIMARY, read_only=0
mysql -h 127.0.0.1 -P 6447 -u <user> -p -N -e "SELECT @@hostname, @@read_only;"   # SECONDARY, read_only=1
```

Esperado: **6446** responde o PRIMARY com `read_only=0`; **6447** responde um secundário com `read_only=1`.

---

## 6) Serviços e logs

```bash
# Servicos
systemctl is-active mysql mysqlrouter

# Logs do MySQL
sudo tail -n 50 /var/log/mysql/error.log
sudo journalctl -u mysql -n 50 --no-pager

# Logs do Router
sudo tail -n 50 /etc/mysqlrouter/log/mysqlrouter.log
sudo journalctl -u mysqlrouter -n 50 --no-pager
```

No `error.log`, procure por linhas com `[Repl]` e `Group Replication` — mudanças de membro, expulsões e eleições aparecem ali.

---

## 7) Sinais de alerta (o que vigiar)

| Sintoma | O que provavelmente é | Para onde ir |
|---|---|---|
| Um membro em `RECOVERING` por muito tempo | Sincronizando devagar ou travado | Logs do nó; `cluster.status()` com `instanceErrors` |
| Membro `UNREACHABLE` | Nó caiu / rede | Runbook de failover ou de quórum |
| `primary=0` | Sem PRIMARY — failover em curso ou NO_QUORUM | Failover / quórum |
| `primary=2` | Anomalia (split-brain?) | Runbook de quórum (seção split-brain) |
| `apply_queue` crescente | Secundário não acompanha | Recursos do nó (CPU/disco) |
| `status=OK_NO_TOLERANCE*` | Rodando sem folga | Trazer nó faltante |
| Escrita "pendura" sem erro | NO_QUORUM | Runbook de perda de quórum |

---

## 8) Script de healthcheck pronto

Salve como `cluster-health.sh` e rode por cron/monitor. Retorna código 0 (OK) ou 1 (alerta):

```bash
#!/usr/bin/env bash
# Healthcheck InnoDB Cluster - rode em um no do cluster
set -euo pipefail
EXPECTED_NODES=3

read online total primary < <(sudo mysql -N -e "
  SELECT SUM(member_state='ONLINE'), COUNT(*), SUM(member_role='PRIMARY')
  FROM performance_schema.replication_group_members;" | tr '\t' ' ')

status="OK"
[ "${online:-0}" -ne "$EXPECTED_NODES" ] && status="ALERTA: online=$online/$EXPECTED_NODES"
[ "${primary:-0}" -ne 1 ] && status="ALERTA: primary=$primary (esperado 1)"

echo "[$(date '+%F %T')] online=$online/$total primary=$primary -> $status"
[[ "$status" == OK ]] || exit 1
```

> Para rodar sem `sudo` em um monitor, crie um usuário só-leitura com `SELECT` em `performance_schema` e use `-u monitor -p` no lugar de `sudo mysql`.

---

## Rotina sugerida

- **A cada minuto (automático):** healthcheck de uma linha (`online=N/N primary=1`).
- **A cada 5 min:** filas/conflitos e portas do Router.
- **Diário (olho humano):** `cluster.status()` completo e varredura nos logs por eleições/expulsões.

---

## Não incluído neste documento

- Integração com Prometheus/Grafana (exporter do MySQL) e dashboards.
- Alertas via `group_replication` + eventos do `performance_schema` em tempo real.
- Métricas de SO (disco, rede) que antecedem problemas de quórum.
