# MySQL Troubleshooting — Locks e bloqueios: fundamentos e onde olhar

A base para investigar "o banco travou", "a query não termina", "deu deadlock". Explica os tipos de lock do InnoDB e, principalmente, **onde olhar** — com saída real de um bloqueio reproduzido em laboratório.

> Esta é a porta de entrada da subpasta. Os runbooks específicos (bloqueios, deadlocks, metadata locks) partem daqui.

---

## Por que existe lock

Vários clientes mexem nos mesmos dados ao mesmo tempo. Para ninguém ler/escrever "pela metade", o InnoDB **tranca** (lock) o que uma transação está mexendo até ela terminar (`COMMIT`/`ROLLBACK`). Isso é normal e desejável. O problema aparece quando uma transação **segura** o lock tempo demais e outra fica **esperando** — ou quando duas se travam em círculo (deadlock).

### Bloqueio × Deadlock (não confunda)

| | **Bloqueio (lock wait)** | **Deadlock** |
|---|---|---|
| O que é | Uma transação espera outra liberar o lock | Duas (ou mais) se esperam em círculo |
| O MySQL resolve sozinho? | Não — espera até o `innodb_lock_wait_timeout` (padrão 50s) | **Sim** — detecta o ciclo e mata uma vítima na hora |
| Erro | `1205 Lock wait timeout exceeded` (se estourar o tempo) | `1213 Deadlock found ...` (imediato, na vítima) |
| Runbook | `troubleshooting-bloqueios-lock-waits.md` | `troubleshooting-deadlocks.md` |

---

## Tipos de lock no InnoDB

| Lock | O que tranca | Exemplo |
|---|---|---|
| **Row lock — `S` (shared)** | Uma linha, para leitura | `SELECT ... LOCK IN SHARE MODE` / `FOR SHARE` |
| **Row lock — `X` (exclusive)** | Uma linha, para escrita | `UPDATE`, `DELETE`, `SELECT ... FOR UPDATE` |
| **Record / Gap / Next-key** | A linha, o "vão" entre linhas, ou ambos | Protege contra inserção fantasma em faixas |
| **Intention — `IS`/`IX`** | A tabela (sinaliza intenção de travar linhas) | Automático, aparece como `TABLE ... IX` |
| **Metadata lock (MDL)** | A *estrutura* da tabela | `DDL` (ALTER) espera transações abertas — ver doc próprio |

No exemplo real abaixo você verá um `IX` na tabela e um `X,REC_NOT_GAP` na linha — exatamente esses tipos.

---

## Onde olhar — a caixa de ferramentas

Reproduzimos um bloqueio real: a transação da **conn 708** travou a linha da Ana (`UPDATE ... WHERE id=1`) e segurou; a **conn 709** tentou alterar a mesma linha e ficou esperando. Veja o que cada ferramenta mostra.

### 1. `SHOW PROCESSLIST` / `information_schema.PROCESSLIST` — visão rápida

Quem está rodando o quê, e em que estado:

```sql
SELECT ID, TIME, STATE, LEFT(INFO,45) AS query
FROM information_schema.PROCESSLIST
WHERE INFO IS NOT NULL;
```

```
+------+-----+----------------------------+-----------------------------------------------+
| conn | seg | estado                     | query                                         |
+------+-----+----------------------------+-----------------------------------------------+
|  708 |   6 | User sleep                 | DO SLEEP(15)                                  |  <- quem segura
|  709 |   4 | updating                   | UPDATE tshoot.contas SET saldo=saldo+100 ...  |  <- quem espera
+------+-----+----------------------------+-----------------------------------------------+
```

O `STATE` é a primeira pista. `updating` parado + `TIME` subindo = provável espera por lock.

### 2. `information_schema.innodb_trx` — transações ativas

Mostra o estado da transação. Procure `LOCK WAIT`:

```sql
SELECT trx_mysql_thread_id AS conn, trx_state, trx_started,
       trx_rows_locked, LEFT(trx_query,30) AS query
FROM information_schema.innodb_trx;
```

```
| conn | estado    | inicio              | linhas_trav | query                          |
| 709  | LOCK WAIT | 2026-11-11 21:44:29 | 1           | UPDATE tshoot.contas SET saldo |  <- esperando
| 708  | RUNNING   | 2026-11-11 21:44:27 | 1           | DO SLEEP(15)                   |  <- rodando/segurando
```

### 3. `performance_schema.data_locks` — os locks em si

Quem tem (`GRANTED`) e quem quer (`WAITING`) cada lock:

```sql
SELECT ENGINE_TRANSACTION_ID AS trx, OBJECT_NAME AS tabela, INDEX_NAME AS idx,
       LOCK_TYPE AS tipo, LOCK_MODE AS modo, LOCK_STATUS AS status, LOCK_DATA AS dado
FROM performance_schema.data_locks;
```

```
| trx  | tabela | idx     | tipo   | modo          | status  | dado |
| 9952 | contas | NULL    | TABLE  | IX            | GRANTED | NULL |
| 9956 | contas | NULL    | TABLE  | IX            | GRANTED | NULL |
| 9952 | contas | PRIMARY | RECORD | X,REC_NOT_GAP | GRANTED | 1    |  <- 9952 TEM o lock da linha 1
| 9956 | contas | PRIMARY | RECORD | X,REC_NOT_GAP | WAITING | 1    |  <- 9956 QUER a linha 1
```

Leia assim: a trx `9952` **tem** (`GRANTED`) o lock `X` da linha de PK = 1; a `9956` **quer** (`WAITING`) a mesma linha. `LOCK_DATA = 1` é o valor da chave travada.

### 4. `sys.innodb_lock_waits` — "quem trava quem" (a mais prática)

A view do schema `sys` já entrega a relação pronta:

```sql
SELECT waiting_pid, waiting_query, blocking_pid, blocking_query, locked_table
FROM sys.innodb_lock_waits;
```

```
| esperando_conn | query_esperando          | bloqueador_conn | tabela            |
| 709            | UPDATE tshoot.contas ... | 708             | `tshoot`.`contas` |
```

**Esta é a consulta que você vai rodar primeiro na vida real:** ela diz direto que a conn **709 está parada por causa da 708**.

### 5. `performance_schema.metadata_locks` — para DDL travado

Quando o problema é um `ALTER` preso ("Waiting for table metadata lock"), os locks não aparecem em `data_locks`, e sim aqui. Ver `troubleshooting-metadata-locks.md`.

### 6. `SHOW ENGINE INNODB STATUS` — o relatório completo

Traz a seção **LATEST DETECTED DEADLOCK** (último deadlock) e a lista de transações. Use o flag `-E` (vertical) no cliente — o terminador `\G` pode falhar em modo batch:

```bash
mysql -E -e "SHOW ENGINE INNODB STATUS"
```

Detalhado no doc de deadlocks.

---

## As variáveis que controlam a espera

| Variável | Padrão | O que faz |
|---|---|---|
| `innodb_lock_wait_timeout` | 50 (s) | Quanto uma transação espera por um **row lock** antes de falhar com `1205` |
| `lock_wait_timeout` | 31536000 (s) | Timeout dos **metadata locks** (DDL) — por padrão, praticamente infinito |
| `innodb_print_all_deadlocks` | OFF | Se `ON`, grava **todo** deadlock no error log (ótimo para auditar) |

```sql
SELECT @@innodb_lock_wait_timeout, @@lock_wait_timeout, @@innodb_print_all_deadlocks;
```

---

## Mapa rápido: sintoma → onde olhar

| Sintoma | Primeiro comando |
|---|---|
| "Uma query não termina" | `sys.innodb_lock_waits` (quem trava quem) |
| "Quero ver transações presas" | `information_schema.innodb_trx` (estado `LOCK WAIT`) |
| "Deu erro 1213 (deadlock)" | `SHOW ENGINE INNODB STATUS` → `LATEST DETECTED DEADLOCK` |
| "Meu ALTER não sai do lugar" | `performance_schema.metadata_locks` |
| "Erro 1205 lock wait timeout" | `sys.innodb_lock_waits` + `innodb_trx` (achar o segurador) |
| "Visão geral do que está rodando" | `SHOW PROCESSLIST` |

---

## Ambiente de teste usado

Schema `tshoot`, tabela `contas(id PK, titular, saldo)`, 3 linhas (Ana/Bruno/Carla), no primário do InnoDB Cluster do lab. Todos os exemplos acima foram capturados reproduzindo os bloqueios de verdade.

---

## Próximos documentos

- Identificar e resolver bloqueios (com `KILL`): `troubleshooting-bloqueios-lock-waits.md`
- Deadlocks: `troubleshooting-deadlocks.md`
- Metadata locks (DDL travado): `troubleshooting-metadata-locks.md`
