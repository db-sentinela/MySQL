# MySQL Troubleshooting — Identificar e resolver bloqueios (lock waits)

Runbook para "uma query não termina" / "o sistema travou" / erro `1205 Lock wait timeout exceeded`. Mostra como achar **quem** está travando e como liberar. Com `KILL` executado de verdade no lab.

> Pré-leitura: `troubleshooting-locks-fundamentos.md` (a caixa de ferramentas).

---

## Passo 1: confirmar que é bloqueio (e não lentidão)

Uma query "travada" pode estar lenta (faltando índice) ou **esperando lock**. Para saber, procure transações em `LOCK WAIT`:

```sql
SELECT trx_mysql_thread_id AS conn, trx_state, trx_started, LEFT(trx_query,40) AS query
FROM information_schema.innodb_trx
WHERE trx_state = 'LOCK WAIT';
```

Se aparecer alguém em `LOCK WAIT`, é bloqueio. Prossiga.

## Passo 2: achar quem trava quem

A consulta mais direta (schema `sys`):

```sql
SELECT waiting_pid, LEFT(waiting_query,40) AS query_espera,
       blocking_pid, LEFT(blocking_query,40) AS query_bloqueador, locked_table
FROM sys.innodb_lock_waits;
```

Saída **real** do lab (a conn 730 espera a 729):

```
+--------+------------------+------------+------------------+
| espera | ...              | bloqueador | query_bloqueador |
+--------+------------------+------------+------------------+
|    730 | UPDATE ...id=3   |        729 | DO SLEEP(30)     |
+--------+------------------+------------+------------------+
```

> **Cadeias de bloqueio:** às vezes o bloqueador também está esperando outro (A←B←C). Rode a consulta de novo olhando o `blocking_pid` até chegar na **raiz** — a transação que não espera ninguém. É essa que você precisa resolver.

## Passo 3: investigar o bloqueador antes de matar

Antes de `KILL`, entenda o que ele é. O caso mais comum e traiçoeiro é o **bloqueador ocioso**: uma transação que foi aberta, travou uma linha e **ficou parada** (aplicação que esqueceu de commitar, sessão humana no meio de um `BEGIN`). No `PROCESSLIST` ele aparece como `Sleep`, mas segura o lock.

```sql
-- transacoes abertas ha mais tempo (candidatas a vilao ocioso)
SELECT trx_mysql_thread_id AS conn, trx_state, trx_started,
       TIMESTAMPDIFF(SECOND, trx_started, NOW()) AS idade_seg,
       trx_rows_locked
FROM information_schema.innodb_trx
ORDER BY trx_started;
```

Decida com base nisto: uma transação aberta há muito tempo e `trx_state = RUNNING` mas sem query ativa é quase sempre segura de matar.

## Passo 4: resolver — `KILL` no bloqueador

Mate a **conexão** que segura o lock (o `blocking_pid`):

```sql
KILL 729;
```

Execução real e o que aconteceu:

```
KILL 729 enviado
linhas em innodb_lock_waits = 0        <- a espera sumiu
saida A (bloqueador morto): ERROR 2013 (HY000) ... Lost connection ...
saida B (liberado): (concluiu)
+----+---------+-------+
| id | titular | saldo |
|  3 | Carla   |   301 |        <- a transacao de B seguiu e commitou
+----+---------+-------+
```

**O que o `KILL` fez:**
- Derrubou a conexão 729. A transação dela foi **desfeita** (`ROLLBACK` automático) — tudo que ela não tinha commitado **se perdeu**.
- Ao liberar o lock, a conn 730 (que esperava) **seguiu na hora** e commitou.

> `KILL <id>` encerra a conexão inteira. Existe também `KILL QUERY <id>`, que mata **só a query atual** e mantém a conexão/transação — útil quando você quer interromper uma consulta sem desfazer a transação toda.

## Passo 5: validar

```sql
SELECT COUNT(*) AS esperas FROM sys.innodb_lock_waits;   -- deve ser 0
SELECT trx_mysql_thread_id, trx_state FROM information_schema.innodb_trx; -- ninguem em LOCK WAIT
```

---

## Cuidados ao matar

- `KILL` **desfaz** a transação da vítima. Se ela já tinha feito muito trabalho não commitado, o `ROLLBACK` pode demorar e gerar I/O.
- Mate a **raiz** da cadeia, não o último que reclamou — senão o bloqueio volta.
- Nunca mate as threads internas (ex.: `Group replication applier module`, `event_scheduler`). Elas aparecem no `PROCESSLIST`; mexa só nas conexões de aplicação.
- Em réplicas/secundários, não há escrita de app — bloqueio lá costuma ser do applier; não use `KILL`, investigue a replicação.

---

## Prevenção (onde o problema nasce)

- **Transações curtas:** faça `COMMIT`/`ROLLBACK` o quanto antes. Nada de abrir `BEGIN` e ir tomar café.
- **Nunca deixe transação aberta esperando input humano/aplicação.** É a causa número 1 do bloqueador ocioso.
- **Índices bons:** um `UPDATE ... WHERE coluna_sem_indice` varre e **trava muitas linhas** (ou a tabela). Com índice, trava só o necessário. Confirme com `EXPLAIN`.
- **Ordem consistente de acesso** às linhas/tabelas entre as transações (isso previne também deadlock).
- **Ajuste fino:** reduzir `innodb_lock_wait_timeout` faz as transações falharem mais rápido (bom para a app reagir), em vez de pendurar 50s.

---

## Comandos úteis

```sql
-- Quem trava quem (comece sempre por aqui)
SELECT * FROM sys.innodb_lock_waits\G

-- Transacoes abertas, da mais antiga para a mais nova
SELECT trx_mysql_thread_id, trx_state, trx_started, trx_rows_locked
FROM information_schema.innodb_trx ORDER BY trx_started;

-- Matar o bloqueador (conexao inteira) ou so a query
KILL <blocking_pid>;
KILL QUERY <blocking_pid>;
```

> Dica: no cliente `mysql` em modo batch, use o flag `-E` em vez do terminador `\G` para saída vertical.

---

## Não incluído neste documento

- Deadlocks (ciclo de espera) — `troubleshooting-deadlocks.md`.
- Metadata locks / DDL travado — `troubleshooting-metadata-locks.md`.
- Ajuste de isolamento de transação (`REPEATABLE READ` × `READ COMMITTED`) e seu efeito nos locks.
