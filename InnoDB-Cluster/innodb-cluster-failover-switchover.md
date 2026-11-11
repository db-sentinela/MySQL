# MySQL InnoDB Cluster — Failover automático e Switchover planejado

Runbook para os dois eventos que trocam quem é o PRIMARY: o **failover** (não planejado, automático) e o **switchover** (planejado, por comando). Inclui uma **simulação executada no lab** com a saída real, para você ver acontecendo antes de precisar na vida real.

> Pré-leitura: `innodb-cluster-fundamentos-alta-disponibilidade.md` (seção *Failover × Switchover*).

---

## Resumo de uma linha

- **Failover**: o PRIMARY caiu → o cluster elege outro sozinho. Você **observa e valida**, não dispara.
- **Switchover**: você quer trocar o PRIMARY (manutenção, rebalanceamento) → dispara com `setPrimaryInstance()`. Sem falha, sem perda.

Em ambos, a aplicação **não troca a string de conexão**: ela fala com o Router, e o Router reaprende quem é o PRIMARY.

---

## Parte A — Failover automático (simulação)

### A.0 Baseline

Confirme o PRIMARY atual e deixe uma "sonda" para provar continuidade de escrita. Escreva **pela porta RW do Router** (6446), que é por onde a aplicação escreve:

```bash
mysql -h 127.0.0.1 -P 6446 -u <user> -p -e "
  CREATE DATABASE IF NOT EXISTS halab;
  CREATE TABLE IF NOT EXISTS halab.probe(
    id INT AUTO_INCREMENT PRIMARY KEY, host VARCHAR(64), note VARCHAR(64),
    ts TIMESTAMP DEFAULT CURRENT_TIMESTAMP);
  INSERT INTO halab.probe(host,note) SELECT @@hostname,'baseline';
  SELECT id,host,note FROM halab.probe;"
```

Saída real (a escrita caiu no PRIMARY, `mysql-db-01`):

```
+----+-------------+----------+
| id | host        | note     |
+----+-------------+----------+
|  1 | mysql-db-01 | baseline |
+----+-------------+----------+
```

### A.1 Derrubar o PRIMARY

Simule a queda parando o serviço MySQL **no PRIMARY** (aqui, `mysql-db-01`):

```bash
sudo systemctl stop mysql
```

> Importante: paramos o **serviço `mysql`**, não a VM. O **host** `mysql-db-01` continua de pé — e, com ele, o MySQL Router e o binário do MySQL Shell. Por isso conseguimos administrar daqui mesmo durante a "queda".

### A.2 Observar a eleição

O Shell só está instalado no `mysql-db-01`, mas podemos rodá-lo **a partir de lá apontando para um nó vivo** (`mysql-db-02`):

```bash
mysqlsh --uri clusteradmin@mysql-db-02:3306 --js \
  --execute="print(JSON.stringify(dba.getCluster().status().defaultReplicaSet, null, 2))"
```

Saída real (resumida) alguns segundos após a queda:

```
status  = OK_NO_TOLERANCE_PARTIAL   | primary = mysql-db-02:3306
mysql-db-01:3306   role=SECONDARY   status=(MISSING)
mysql-db-02:3306   role=PRIMARY     status=ONLINE
mysql-db-03:3306   role=SECONDARY   status=ONLINE
```

**Lendo o resultado:**
- O cluster **elegeu `mysql-db-02` como novo PRIMARY** sozinho. Ninguém mandou.
- `mysql-db-01` aparece `(MISSING)`: o cluster sabe que ele existe no metadata, mas não o enxerga agora.
- `OK_NO_TOLERANCE_PARTIAL`: o cluster **continua operando** (há quórum: 2 de 3), mas **sem folga** — se cair mais um, vira perda de quórum.

### A.3 Provar que a aplicação continua escrevendo

A prova real de HA: escrever pela mesma porta 6446, **sem mudar nada**, durante a queda.

```bash
mysql -h 127.0.0.1 -P 6446 -u <user> -p -e "
  INSERT INTO halab.probe(host,note) SELECT @@hostname,'durante-failover';
  SELECT id,host,note FROM halab.probe;"
```

Saída real:

```
+----+-------------+------------------+
| id | host        | note             |
+----+-------------+------------------+
|  1 | mysql-db-01 | baseline         |   <- dado de antes, replicado, intacto
|  2 | mysql-db-02 | durante-failover |   <- escrita nova caiu no NOVO primary
+----+-------------+------------------+
```

E a leitura (6447) passa a responder só o secundário sobrevivente:

```
mysql-db-03   1
mysql-db-03   1
mysql-db-03   1
```

Ou seja: **zero mudança na aplicação, a escrita migrou sozinha para o `mysql-db-02`, nenhum dado perdido.** Isso é o failover entregando o que promete.

> Se a **primeira** conexão logo após a queda falhar, repita. O Router leva alguns segundos para reconvergir a topologia. É esperado, não é bug.

---

## Parte B — Retorno do nó (auto-rejoin)

Religue o MySQL no nó que caiu:

```bash
sudo systemctl start mysql
```

Espere ~30s e confira. Saída real:

```
status  = OK   | primary = mysql-db-02:3306
mysql-db-01:3306   role=SECONDARY   status=ONLINE
mysql-db-02:3306   role=PRIMARY     status=ONLINE
mysql-db-03:3306   role=SECONDARY   status=ONLINE
```

**O `mysql-db-01` voltou sozinho como SECONDARY**, aplicou o que perdeu via GTID (incremental) e o cluster voltou a `OK` (com folga de novo). Repare: ele **não** reassume o papel de PRIMARY automaticamente — quem está PRIMARY (`mysql-db-02`) continua. Se você quer o `mysql-db-01` de volta como PRIMARY, isso é um switchover (Parte C).

---

## Parte C — Switchover planejado

Troca de PRIMARY **por decisão sua**, sem falha. Útil para: devolver o papel ao nó preferido após um failover, liberar um nó para manutenção, ou rebalancear carga.

```bash
mysqlsh --uri clusteradmin@mysql-db-02:3306 --js \
  --execute="dba.getCluster().setPrimaryInstance('clusteradmin@mysql-db-01:3306')"
```

Validação real logo depois:

```
primary = mysql-db-01:3306   status = OK
mysql-db-01:3306   role=PRIMARY     mode=R/W   status=ONLINE
mysql-db-02:3306   role=SECONDARY   mode=R/O   status=ONLINE
mysql-db-03:3306   role=SECONDARY   mode=R/O   status=ONLINE
```

E o Router acompanhou:

```
6446 (RW) ->  mysql-db-01   0     <- escrita de volta ao db-01
6447 (RO) ->  mysql-db-02   1
              mysql-db-03   1
```

**O switchover é transparente e sem perda**: o GR só promove o novo PRIMARY depois que ele está 100% em dia, e o antigo PRIMARY vira SECONDARY de forma limpa.

---

## Checklist de validação (failover ou switchover)

1. `dba.getCluster().status()` → `status = OK` (ou `OK_NO_TOLERANCE_PARTIAL` se ainda falta um nó voltar).
2. Exatamente **um** PRIMARY `R/W`.
3. Router **6446** responde o PRIMARY com `read_only = 0`.
4. Router **6447** responde secundário(s) com `read_only = 1`.
5. A sonda (`halab.probe`) mantém os dados antigos e aceita novos.

---

## Observações

- **Quem é escolhido no failover:** por padrão, o secundário mais à frente em GTID (e você pode influenciar com *pesos* de eleição — tema avançado). No lab, com todos em dia, a eleição foi imediata.
- **Consistência pós-failover:** com `group_replication_consistency = BEFORE_ON_PRIMARY_FAILOVER` (padrão do lab), o novo PRIMARY só atende depois de aplicar o backlog — a aplicação não lê dado velho após a troca.
- **Tempo de indisponibilidade de escrita:** da queda até a 6446 voltar a aceitar escrita são poucos segundos (eleição + reconvergência do Router). A leitura pela 6447 segue nos secundários o tempo todo.
- **`OK_NO_TOLERANCE_PARTIAL` não é erro:** é o cluster avisando "estou operando, mas sem folga". Resolva trazendo o nó faltante de volta.
- **Switchover exige cluster saudável:** todos os nós precisam estar `ONLINE` e em dia. Não é ferramenta de emergência — para emergência existe failover (automático) e os runbooks de quórum/queda total.

---

## Troubleshooting

- **A 6446 recusa escrita logo após a queda:** reconvergência do Router em curso. Aguarde alguns segundos e repita.
- **O nó não volta como SECONDARY após `start`:** verifique `member_state` nele (`sudo mysql -e "SELECT member_state FROM performance_schema.replication_group_members;"`); se `RECOVERING`, está sincronizando (espere); se `OFFLINE`/`ERROR`, veja `/var/log/mysql/error.log` e, se preciso, `cluster.rejoinInstance('clusteradmin@<nó>:3306')`.
- **`setPrimaryInstance` falha:** normalmente porque algum nó não está `ONLINE`/em dia. Rode `cluster.status()` e resolva os `instanceErrors` antes.

---

## Comandos úteis

```bash
# Quem é o PRIMARY agora
mysqlsh --uri clusteradmin@mysql-db-01:3306 --js \
  --execute="print(dba.getCluster().status().defaultReplicaSet.primary)"

# Simular failover (no PRIMARY)
sudo systemctl stop mysql      # derruba
sudo systemctl start mysql     # religa (volta como SECONDARY)

# Switchover planejado
mysqlsh --uri clusteradmin@<nó-vivo>:3306 --js \
  --execute="dba.getCluster().setPrimaryInstance('clusteradmin@mysql-db-01:3306')"
```

---

## Não incluído neste documento

- Pesos de eleição (`memberWeight`) para controlar quem vira PRIMARY.
- Perda de quórum (quando a maioria cai) — `innodb-cluster-perda-de-quorum.md`.
- Recuperação de queda total — `innodb-cluster-recuperacao-queda-total.md`.
