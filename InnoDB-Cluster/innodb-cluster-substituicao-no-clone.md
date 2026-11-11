# MySQL InnoDB Cluster — Substituição e recuperação de nó (clone)

Como tirar um nó problemático do cluster e colocá-lo de volta **reprovisionado do zero** com *clone* — o mesmo mecanismo da instalação, aqui usado para **conserto**. Inclui execução real no lab, com o transcript do clone.

> Pré-leitura: `innodb-cluster-fundamentos-alta-disponibilidade.md` (incremental × clone).

---

## Quando usar clone para consertar um nó

Use quando o nó **não consegue voltar sozinho** por recuperação incremental:

- Tem **transações errantes** (executou algo que o cluster nunca viu).
- Está **atrás demais** ou com binlogs/GTIDs purgados (não dá para repor só o que falta).
- Teve **corrupção** de dados/datadir.
- É **hardware novo** substituindo um nó morto (mesmo hostname/IP).

O clone **apaga o datadir do nó** e copia um snapshot físico de um membro saudável. É destrutivo **só para aquele nó** (que é uma réplica — os dados vêm de volta do primário), e é seguro para o cluster.

> Para um nó que só caiu e está em dia, você **não** precisa de clone — ele volta incremental sozinho (ver runbook de failover). Clone é para quando o incremental não serve.

---

## Procedimento

Há dois caminhos, dependendo de o nó ainda estar no metadata do cluster:

- **Nó ainda listado (mesmo que `(MISSING)`/`ERROR`):** `cluster.rejoinInstance(..., {recoveryMethod:'clone'})`.
- **Trocar/reprovisionar do zero:** `removeInstance` e depois `addInstance(..., {recoveryMethod:'clone'})`. É o caminho mais limpo e o que demonstramos abaixo.

### Passo 1: remover o nó do cluster

```bash
mysqlsh --uri clusteradmin@mysql-db-01:3306 --js \
  --execute="dba.getCluster().removeInstance('clusteradmin@mysql-db-03:3306')"
```

Saída real:

```
The instance will be removed from the InnoDB Cluster.
* Waiting for instance 'mysql-db-03:3306' to synchronize with the primary...
* Instance 'mysql-db-03:3306' is attempting to leave the cluster...
The instance 'mysql-db-03:3306' was successfully removed from the cluster.
```

Status fica com os nós restantes:

```
status = OK_NO_TOLERANCE
mysql-db-01:3306   PRIMARY     ONLINE
mysql-db-02:3306   SECONDARY   ONLINE
```

> Se o nó estiver **morto** e o `removeInstance` não conseguir falar com ele, use `cluster.removeInstance('...', {force:true})` para tirá-lo só do metadata.

> Se o que você vai trocar é o **hardware**: suba a VM nova com o **mesmo hostname/IP**, MySQL instalado e configurado (server_id/report_host, `/etc/hosts`), o usuário `clusteradmin`, e só então faça o `addInstance` abaixo.

### Passo 2: re-adicionar forçando clone

```bash
mysqlsh --uri clusteradmin@mysql-db-01:3306 --js \
  --execute="dba.getCluster().addInstance('clusteradmin@mysql-db-03:3306', {recoveryMethod:'clone'})"
```

Transcript **real** do lab (resumido):

```
Clone based recovery selected through the recoveryMethod option
Validating instance configuration at mysql-db-03:3306...  Instance configuration is suitable.
Clone based state recovery is now in progress.
NOTE: A server restart is expected to happen as part of the clone process.
NOTE: mysql-db-03:3306 is being cloned from mysql-db-01:3306
 ** Stage DROP DATA:  Completed        <- apaga o datadir do no
 ** Stage FILE COPY:  Completed        <- copia os arquivos do primario
 ** Stage PAGE COPY:  Completed
 ** Stage REDO COPY:  Completed
NOTE: mysql-db-03:3306 is shutting down...
* Waiting for server restart... ready  <- o no reinicia sozinho
 ** Stage FILE SYNC:  Completed
 ** Stage RESTART:    Completed
* Clone process has finished: 75.23 MB transferred in 3 sec (25.08 MB/s)
The instance 'mysql-db-03:3306' was successfully added to the cluster.
```

As **etapas do clone**, em português:
1. **DROP DATA** — o nó destino apaga os próprios dados.
2. **FILE COPY / PAGE COPY / REDO COPY** — copia os arquivos físicos do InnoDB do nó fonte.
3. **RESTART** — o MySQL reinicia sozinho para assumir os dados clonados.
4. **FILE SYNC** — ajustes finais; o nó entra no grupo e termina de aplicar o que mudou durante a cópia.

Status final, saudável de novo:

```
status = OK
mysql-db-01:3306   PRIMARY     ONLINE
mysql-db-02:3306   SECONDARY   ONLINE
mysql-db-03:3306   SECONDARY   ONLINE
```

---

## O restart demorou / deu timeout

Em VMs lentas, o restart pode passar do timeout do Shell. O MySQL geralmente volta sozinho. Se o Shell mostrar algo como `Waiting for server restart... timeout` e `Lost connection`:

```bash
# confirme que o MySQL voltou no no
ssh dba@mysql-db-03 "systemctl is-active mysql"

# reconcilie o metadata e confira
mysqlsh --uri clusteradmin@mysql-db-01:3306 --js --execute="dba.getCluster().rescan()"   # responda Y se perguntar
mysqlsh --uri clusteradmin@mysql-db-01:3306 --js --execute="print(JSON.stringify(dba.getCluster().status().defaultReplicaSet, null, 2))"
```

Opcional (antes do `addInstance`), aumentar a espera do restart:

```js
shell.options["dba.restartWaitTimeout"] = 300
```

---

## Checklist

1. `removeInstance` (ou `{force:true}` se o nó estiver morto) tirou o nó.
2. `addInstance(..., {recoveryMethod:'clone'})` concluiu com `successfully added`.
3. `status = OK` com todos os nós `ONLINE`.
4. Nenhum `instanceErrors` no `cluster.status()`.

---

## Comandos úteis

```bash
# Nó ainda no metadata, reprovisionar sem remover
mysqlsh --uri clusteradmin@mysql-db-01:3306 --js \
  --execute="dba.getCluster().rejoinInstance('clusteradmin@mysql-db-03:3306', {recoveryMethod:'clone'})"

# Remover a forca (no morto)
mysqlsh --uri clusteradmin@mysql-db-01:3306 --js \
  --execute="dba.getCluster().removeInstance('clusteradmin@mysql-db-03:3306', {force:true})"

# Reprovisionar do zero
mysqlsh --uri clusteradmin@mysql-db-01:3306 --js \
  --execute="dba.getCluster().addInstance('clusteradmin@mysql-db-03:3306', {recoveryMethod:'clone'})"
```

---

## Não incluído neste documento

- Substituição de hardware com hostname/IP **diferentes** (exige reconfigurar `report_host`, `/etc/hosts` e, às vezes, recriar o nó no metadata).
- Clone a partir de um secundário específico (`cloneDonor`).
- Ajuste fino de `group_replication_clone_threshold` (quando o cluster decide clone sozinho).
