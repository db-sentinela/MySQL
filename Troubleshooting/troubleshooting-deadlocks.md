# MySQL Troubleshooting — Deadlocks

O que é um deadlock, como o MySQL o resolve sozinho, e como **ler** o relatório para descobrir a causa e evitar que volte. Com um deadlock **provocado de verdade** no lab e o `SHOW ENGINE INNODB STATUS` real.

> Pré-leitura: `troubleshooting-locks-fundamentos.md` (bloqueio × deadlock).

---

## O que é

Um **deadlock** é um abraço mortal: duas transações seguram, cada uma, um lock que a outra quer. Nenhuma avança, e nenhuma pode liberar sem a outra liberar antes.

```mermaid
flowchart LR
    A["Transação A<br/>tem lock da linha 1<br/>quer a linha 2"] -->|espera| B
    B["Transação B<br/>tem lock da linha 2<br/>quer a linha 1"] -->|espera| A
```

Diferente de um bloqueio comum, o InnoDB **detecta o ciclo na hora** e desfaz uma das transações (a "vítima") para o sistema seguir. A vítima recebe:

```
ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction
```

**Deadlock não é bug — é esperado em sistemas concorrentes.** O que importa é a aplicação **tratar o 1213** e você reduzir a frequência.

---

## Como reproduzir (o cenário clássico de ordem cruzada)

Duas transações que tocam as mesmas linhas **em ordem oposta**:

| Tempo | Sessão A | Sessão B |
|---|---|---|
| t0 | `BEGIN; UPDATE contas ... WHERE id=1;` (trava **Ana**) | |
| t1 | | `BEGIN; UPDATE contas ... WHERE id=2;` (trava **Bruno**) |
| t2 | `UPDATE contas ... WHERE id=2;` → **espera Bruno** | |
| t3 | | `UPDATE contas ... WHERE id=1;` → **espera Ana** = ciclo! |
| t3 | (segue e commita) | **ERROR 1213** (vítima, rollback) |

No lab, a Sessão B foi a vítima (`ERROR 1213`) e a A commitou.

---

## Como ler o relatório: `LATEST DETECTED DEADLOCK`

O InnoDB guarda o **último** deadlock detectado. Veja (use `-E` para saída vertical):

```bash
mysql -E -e "SHOW ENGINE INNODB STATUS"
```

Saída **real** (anotada), seção do deadlock:

```
LATEST DETECTED DEADLOCK
------------------------
2026-11-11 21:45:22

*** (1) TRANSACTION:
TRANSACTION 10001, ACTIVE 4 sec starting index read
MySQL thread id 715, ... clusteradmin updating
UPDATE tshoot.contas SET saldo=saldo+1 WHERE id=2         <-- (1) ESTA PRESA querendo id=2

*** (1) HOLDS THE LOCK(S):
RECORD LOCKS ... index PRIMARY of table `tshoot`.`contas` ... lock_mode X locks rec but not gap
 ... 3: len 3; hex 416e61; asc Ana;;                      <-- (1) JA TEM o lock da linha "Ana" (id=1)

*** (1) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS ... lock_mode X ... waiting
 ... 3: len 5; hex 4272756e6f; asc Bruno;;                <-- (1) ESPERA o lock de "Bruno" (id=2)

*** (2) TRANSACTION:
TRANSACTION 10002, ACTIVE 3 sec ...
MySQL thread id 716, ... updating
UPDATE tshoot.contas SET saldo=saldo+1 WHERE id=1         <-- (2) ESTA PRESA querendo id=1

*** (2) HOLDS THE LOCK(S):
 ... asc Bruno;;                                          <-- (2) JA TEM "Bruno" (id=2)

*** (2) WAITING FOR THIS LOCK TO BE GRANTED:
 ... asc Ana;;                                            <-- (2) ESPERA "Ana" (id=1)

*** WE ROLL BACK TRANSACTION (2)                          <-- a VITIMA escolhida
```

### Decifrando

- **`*** (1)` e `*** (2)`** = as duas transações do ciclo.
- **`HOLDS THE LOCK(S)`** = o que cada uma **já tem**.
- **`WAITING FOR THIS LOCK`** = o que cada uma **quer** (e está presa).
- **`asc Ana` / `asc Bruno`** = o InnoDB imprime o conteúdo da linha travada em hexa + ASCII. Dá para **identificar exatamente qual linha** entrou no deadlock — aqui, Ana (id=1) e Bruno (id=2).
- **`lock_mode X locks rec but not gap`** = lock exclusivo de registro (o `X,REC_NOT_GAP` que você vê em `data_locks`).
- **`WE ROLL BACK TRANSACTION (2)`** = quem o InnoDB matou. Ele escolhe a vítima mais "barata" de desfazer (menos trabalho realizado).

Com isso você sabe **quais queries e quais linhas** colidiram — é por aí que se corrige o código.

---

## Tratar o 1213 na aplicação

A vítima não fez nada de errado; só perdeu no sorteio. A aplicação **deve capturar o erro 1213 e repetir a transação** (em geral 1–3 tentativas resolvem). Exemplo de lógica:

```
tentativas = 0
repita:
    tente { BEGIN; ...; COMMIT }
    se erro == 1213 e tentativas < 3: tentativas++; continue
    senão: propague o erro
```

Sem esse retry, o deadlock vira erro visível para o usuário.

---

## Prevenção

- **Ordem consistente de acesso:** se **todas** as transações mexerem nas linhas/tabelas na mesma ordem (ex.: sempre id crescente), o ciclo não se forma. Esta é a prevenção mais eficaz.
- **Transações curtas:** menos tempo segurando lock = menos janela para colidir.
- **Índices adequados:** `UPDATE`/`DELETE` sem índice travam faixas grandes e aumentam a chance de deadlock.
- **Isolamento menor:** `READ COMMITTED` gera menos gap locks que `REPEATABLE READ` (padrão) — reduz certos deadlocks, mas mude com cuidado (muda semântica).
- **Uma transação, um propósito:** evite acumular muitos updates não relacionados na mesma transação.

---

## Auditar deadlocks ao longo do tempo

O `SHOW ENGINE INNODB STATUS` só mostra o **último**. Para registrar **todos** no error log:

```sql
SET GLOBAL innodb_print_all_deadlocks = ON;   -- persista no my.cnf para valer sempre
```

Depois, procure no error log:

```bash
sudo grep -i -A 40 "DEADLOCK" /var/log/mysql/error.log
```

Assim você vê frequência, horários e padrões — essencial para atacar a causa.

---

## Comandos úteis

```bash
# Ver o ultimo deadlock
mysql -E -e "SHOW ENGINE INNODB STATUS"

# Logar todos os deadlocks
mysql -e "SET GLOBAL innodb_print_all_deadlocks = ON;"
sudo grep -i -A 40 "LATEST DETECTED DEADLOCK\|DEADLOCK" /var/log/mysql/error.log
```

---

## Não incluído neste documento

- Bloqueio simples (sem ciclo) e `KILL` — `troubleshooting-bloqueios-lock-waits.md`.
- Deep dive em gap locks / next-key locks e níveis de isolamento.
- Deadlocks envolvendo locks de metadados (DDL) — ver metadata locks.
