# MySQL InnoDB Cluster — Desligamento e inicialização ordenados

Como **parar** e **subir** o ambiente inteiro (ou um nó só) sem gerar crash, perda de quórum ou sustos. Baseado em comportamento observado no lab.

> Pré-leitura: `innodb-cluster-fundamentos-alta-disponibilidade.md` e `innodb-cluster-recuperacao-queda-total.md` (a inicialização do ambiente inteiro usa o reboot de lá).

---

## A regra que mais economiza plantão

Existem dois tipos de "parar um nó", e eles são opostos:

| | Parada **limpa** (`systemctl stop mysql`) | **Crash** (kill -9, queda de energia) |
|---|---|---|
| O que o GR faz | Saída **graciosa**: o nó avisa e sai do grupo | O nó **some**; os outros o veem `UNREACHABLE` |
| Efeito no quórum | Grupo **encolhe** de forma ordenada, mantém quórum enquanto houver maioria | Pode **derrubar o quórum** se a maioria some junto |
| Risco | Baixo | Alto (NO_QUORUM, crash recovery) |

**Sempre pare com `systemctl stop mysql`.** Evidência real do lab — parando **limpo** um secundário, o cluster seguiu gravável:

```
status = OK_NO_TOLERANCE_PARTIAL
mysql-db-01:3306   role=PRIMARY     status=ONLINE
mysql-db-02:3306   role=SECONDARY   status=ONLINE
mysql-db-03:3306   role=SECONDARY   status=(MISSING)

Router 6446 (RW) -> mysql-db-01  read_only=0   <- aplicacao continua escrevendo
```

O nó parado vira `(MISSING)`, mas a maioria (2 de 3) mantém o quórum e a escrita.

---

## Caso 1 (o mais comum): manutenção em UM nó

Não derrube o ambiente para mexer em um nó. Faça só nele:

```bash
# 1. (se for o PRIMARY) passe o papel para outro antes, sem downtime:
mysqlsh --uri clusteradmin@<nó-vivo>:3306 --js \
  --execute="dba.getCluster().setPrimaryInstance('clusteradmin@<outro-nó>:3306')"

# 2. pare o nó de forma limpa
sudo systemctl stop mysql

# 3. faça a manutenção (SO, disco, configuração...)

# 4. suba o nó — ele rejunta sozinho (start_on_boot = ON)
sudo systemctl start mysql
```

Valide que ele voltou `ONLINE` (ver `innodb-cluster-monitoracao-healthcheck.md`). Enquanto o nó esteve fora, o cluster ficou em `OK_NO_TOLERANCE_PARTIAL` — operando, mas sem folga; por isso faça a manutenção **rápido** e um nó **de cada vez**.

> Se for mexer no **PRIMARY**, faça o switchover do passo 1 primeiro. Assim a aplicação nem percebe (evita o failover forçado).

---

## Caso 2: desligar o AMBIENTE INTEIRO (ordenado)

Para manutenção geral (ex.: desligar as VMs, parada elétrica programada).

### Ordem de desligamento

```
1. Pausar a aplicação (ou parar o Router)  -> para de chegar escrita nova
2. Parar os SECUNDÁRIOS, um a um           -> sudo systemctl stop mysql
3. Parar o PRIMÁRIO por último             -> sudo systemctl stop mysql
4. Parar o Router                          -> sudo systemctl stop mysqlrouter
```

**Por que o primário por último:** parando os secundários antes, o primário termina como o nó **mais atualizado**. Anote qual foi o último a parar (o primário) — ele será a **semente** natural na hora de subir.

> ⚠️ Mesmo um desligamento 100% limpo deixa o cluster em **queda total** (todos `OFFLINE`). O Group Replication **não** forma grupo sozinho depois disso — é proposital, para evitar split-brain. Ou seja: subir o ambiente **sempre** passa pelo `rebootClusterFromCompleteOutage()`. O desligamento ordenado não evita o reboot; ele só garante que o reboot seja **tranquilo** (GTIDs em dia, sem crash recovery).

### Ordem de inicialização

```
1. Ligar os hosts/VMs
2. Garantir o mysqld de pie nos 3 nós      -> sudo systemctl start mysql ; systemctl is-active mysql
3. Reconstruir o grupo a partir da semente -> rebootClusterFromCompleteOutage() no ex-primário
4. Subir o Router                          -> sudo systemctl start mysqlrouter
5. Validar e religar a aplicação
```

Comando do passo 3 (detalhado em `innodb-cluster-recuperacao-queda-total.md`):

```bash
mysqlsh --uri clusteradmin@<ex-primário>:3306 --js \
  --execute="dba.rebootClusterFromCompleteOutage()"
```

Se você não anotou qual era o primário, compare o `gtid_executed` dos 3 e use o de maior conjunto como semente (ver runbook de queda total).

---

## O Router no roteiro

- O Router **não guarda dados**; parar/subir é seguro a qualquer momento.
- Suba o Router **depois** que o cluster já tem quórum (passo 4). Se subir antes, ele fica sem backend e as conexões falham até o cluster voltar.
- Verifique que voltou a escutar:

```bash
systemctl is-active mysqlrouter
sudo ss -ltnp | grep -E ':(6446|6447|6450|8443)'
```

---

## Checklist

**Desligar:** aplicação pausada → secundários parados (limpo) → primário parado por último (anotado) → Router parado.

**Subir:** mysqld ativo nos 3 → `rebootClusterFromCompleteOutage()` na semente → `status = OK` com 3 `ONLINE` → Router ativo e escutando → aplicação religada.

---

## Comandos úteis

```bash
# Parar/subir um no (limpo)
sudo systemctl stop mysql
sudo systemctl start mysql

# Switchover antes de mexer no primario
mysqlsh --uri clusteradmin@<no-vivo>:3306 --js \
  --execute="dba.getCluster().setPrimaryInstance('clusteradmin@<outro-no>:3306')"

# Subir o ambiente inteiro (apos todos os mysqld de pe)
mysqlsh --uri clusteradmin@<ex-primario>:3306 --js \
  --execute="dba.rebootClusterFromCompleteOutage()"

# Router
sudo systemctl stop mysqlrouter
sudo systemctl start mysqlrouter
```

---

## Não incluído neste documento

- Automação do roteiro (scripts/Ansible) de parada e subida.
- Janela de manutenção com drain de conexões no Router.
- Atualização de versão (rolling upgrade) — tema próprio.
