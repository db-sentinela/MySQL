# MySQL InnoDB Cluster — Modo Multi-Primary

Como mudar o cluster do modo **Single-Primary** (um nó escreve) para **Multi-Primary** (todos escrevem), o que ganha, e os riscos — principalmente **conflitos de escrita**. Com experimento real no lab, incluindo um conflito provocado de verdade.

> Pré-leitura: `innodb-cluster-fundamentos-alta-disponibilidade.md` e `innodb-cluster-failover-switchover.md`.

> ⚠️ **Recomendação honesta:** fique em **Single-Primary** salvo um motivo forte. Multi-Primary só vale se a **aplicação foi desenhada para ele** (particionamento de escrita, tratamento de conflito). Caso contrário, ele traz mais dor (conflitos) do que ganho.

---

## Single-Primary × Multi-Primary

| | **Single-Primary** (padrão) | **Multi-Primary** |
|---|---|---|
| Quem escreve | 1 nó (PRIMARY); os outros são R/O | **Todos** os nós (R/W) |
| Conflito de escrita | Impossível (só um escreve) | **Possível** — precisa ser tratado |
| Ganho | Simples, previsível | Escrita distribuída **se** a app particiona |
| Risco | Baixo | Conflitos, PK/auto_increment, deadlocks de certificação |
| Indicação | 95% dos casos | App específica, ciente dos conflitos |

Multi-Primary **não** multiplica a capacidade de escrita magicamente: toda escrita ainda precisa ser **certificada** por todos os nós. Ele ajuda quando escritas de nós diferentes tocam **dados diferentes**.

---

## Como alternar o modo

### Para Multi-Primary

```bash
mysqlsh --uri clusteradmin@mysql-db-01:3306 --js \
  --execute="dba.getCluster().switchToMultiPrimaryMode()"
```

Saída **real** do lab:

```
Switching cluster 'labCluster' to Multi-Primary mode...
Instance 'mysql-db-02:3306' was switched from SECONDARY to PRIMARY.
Instance 'mysql-db-03:3306' was switched from SECONDARY to PRIMARY.
The cluster successfully switched to Multi-Primary mode.
```

Agora **todos** são PRIMARY R/W (status real):

```
topologyMode = Multi-Primary   status = OK
mysql-db-01:3306   PRIMARY   R/W
mysql-db-02:3306   PRIMARY   R/W
mysql-db-03:3306   PRIMARY   R/W
```

E `@@read_only` virou `0` nos três nós — todos aceitam escrita.

### De volta para Single-Primary

```bash
mysqlsh --uri clusteradmin@mysql-db-01:3306 --js \
  --execute="dba.getCluster().switchToSinglePrimaryMode('clusteradmin@mysql-db-01:3306')"
```

Saída real (db-01 volta a ser o único PRIMARY):

```
Instance 'mysql-db-02:3306' was switched from PRIMARY to SECONDARY.
Instance 'mysql-db-03:3306' was switched from PRIMARY to SECONDARY.
The cluster successfully switched to Single-Primary mode.
```

O argumento define **qual** nó fica como PRIMARY. A troca é **online** — sem downtime.

---

## Escrevendo em nós diferentes (o que o modo entrega)

No lab, com o cluster em Multi-Primary, inserimos a partir de **cada** nó:

```bash
mysql -h mysql-db-01 ... -e "INSERT INTO mpdemo.t(origem) VALUES('escrito-no-db-01')"
mysql -h mysql-db-02 ... -e "INSERT INTO mpdemo.t(origem) VALUES('escrito-no-db-02')"
mysql -h mysql-db-03 ... -e "INSERT INTO mpdemo.t(origem) VALUES('escrito-no-db-03')"
```

Lendo de qualquer nó, **tudo replicou** (saída real do db-03):

```
+----+------------------+
| id | origem           |
+----+------------------+
|  2 | escrito-no-db-01 |
|  3 | escrito-no-db-02 |
|  4 | escrito-no-db-03 |
+----+------------------+
```

Escritas de nós diferentes, todas aplicadas e consistentes.

---

## 🔴 O risco central: conflito de escrita

Se **dois nós** alteram a **mesma linha** ao mesmo tempo, o Group Replication detecta na **certificação** e **desfaz uma** das transações. Provamos no lab — duas transações concorrentes no `id=2`, uma no db-01 e outra no db-02:

```
-- db-01 (A): commitou sem erro
-- db-02 (B): ERROR 1180 (HY000): Got error 149 - 'Lock deadlock; Retry transaction' during COMMIT
valor final de id=2: conflito-A
```

A transação A venceu; a B foi **rejeitada no commit** e teria que ser **repetida** pela aplicação. 

**Consequências práticas:**
- A aplicação **precisa tratar** esse erro (como um deadlock: capturar e repetir a transação).
- O erro aparece **no commit**, não no `UPDATE` — a app pode ter feito trabalho que será desfeito.
- Em Single-Primary isso **não existe** (só um nó escreve) — é o preço do Multi-Primary.

---

## Outros cuidados do Multi-Primary

### Toda tabela precisa de PRIMARY KEY
O Group Replication **exige** chave primária (ou única não-nula) em toda tabela para rastrear conflitos. Uma tabela sem PK recusa a escrita:

```
ERROR 3098 (HY000): The table does not comply with the requirements by an external plugin.
```

### auto_increment sem colisão
Para dois nós não gerarem o mesmo `id` ao inserir em paralelo, o GR ajusta automaticamente `auto_increment_increment` e `auto_increment_offset` por nó. Efeito: os IDs podem ter **saltos** (não são mais perfeitamente sequenciais). A aplicação não deve depender de IDs contíguos.

### O Router em Multi-Primary
Com todos os nós PRIMARY, a porta RW (6446) entrega o **primeiro disponível** (`first-available`), e a RO (6447) distribui. Para espalhar escrita de verdade entre os nós, a aplicação precisa de uma estratégia (ex.: porta RW Split 6450, ou conexões dirigidas) — e, de novo, tratar conflitos.

---

## Quando (e quando NÃO) usar Multi-Primary

**Pode fazer sentido** quando:
- A aplicação **particiona** as escritas (cada nó mexe em dados distintos — por tenant, por região).
- Você precisa de escrita local em topologia geograficamente distribuída e aceita o custo de conflito.

**Fique em Single-Primary** quando:
- A aplicação não foi desenhada para conflitos (a imensa maioria).
- Você quer simplicidade e previsibilidade — failover automático já dá a HA.

---

## Checklist

1. Entenda que Multi-Primary **não** é "mais escrita de graça" — toda escrita ainda é certificada por todos.
2. **Toda tabela com PRIMARY KEY** (obrigatório).
3. A aplicação **trata conflito** (erro no commit) com retry, como um deadlock.
4. A aplicação **não depende de IDs contíguos** (auto_increment com saltos).
5. Teste a troca (`switchToMultiPrimaryMode`/`switchToSinglePrimaryMode`) em homologação — é online e reversível.
6. Na dúvida, **Single-Primary**.

---

## Comandos úteis

```bash
# Para Multi-Primary
mysqlsh --uri clusteradmin@mysql-db-01:3306 --js --execute="dba.getCluster().switchToMultiPrimaryMode()"

# De volta para Single-Primary (define o PRIMARY)
mysqlsh --uri clusteradmin@mysql-db-01:3306 --js \
  --execute="dba.getCluster().switchToSinglePrimaryMode('clusteradmin@mysql-db-01:3306')"

# Ver o modo atual
mysqlsh --uri clusteradmin@mysql-db-01:3306 --js \
  --execute="print(dba.getCluster().status().defaultReplicaSet.topologyMode)"
```

---

## Não incluído neste documento

- Estratégias de particionamento de escrita na aplicação.
- Multi-Primary em topologia geo-distribuída (latência de certificação).
- Ajuste de `group_replication_auto_increment_increment`.
