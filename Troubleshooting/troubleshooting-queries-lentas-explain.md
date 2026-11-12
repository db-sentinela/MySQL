# MySQL Troubleshooting — Queries lentas: EXPLAIN e índices

A causa número 1 de query lenta: **examinar linhas demais** por falta de índice. Este documento ensina a ler o `EXPLAIN`, reconhecer um *full table scan* e corrigir com índice — com medição real de antes/depois no lab (500 mil linhas).

> Pré-leitura: `troubleshooting-locks-fundamentos.md` (contexto geral de diagnóstico).

---

## A ideia central: linhas examinadas × linhas retornadas

Uma query é lenta quase sempre porque **lê muito mais linhas do que devolve**. Se para devolver 45 linhas o MySQL precisa varrer 500.000, o problema não é "o banco está lento" — é falta de índice. O `EXPLAIN` mostra exatamente isso **antes** de você rodar a query.

Ambiente do teste: tabela `tshoot.pedidos`, **500.000 linhas**, 10.000 clientes. Query de exemplo:

```sql
SELECT COUNT(*), SUM(valor) FROM tshoot.pedidos WHERE cliente_id = 1234;
```

---

## Lendo o `EXPLAIN` (formato tradicional)

```sql
EXPLAIN FORMAT=TRADITIONAL SELECT COUNT(*) FROM pedidos WHERE status='cancelado';
```

**Sem índice** (coluna `status`), saída real:

```
type: ALL                <-- "ALL" = FULL TABLE SCAN (le a tabela inteira)
possible_keys: NULL      <-- nenhum indice utilizavel
key: NULL                <-- nenhum indice usado
rows: 498669             <-- estimativa de linhas lidas: ~500 mil
filtered: 10.00          <-- so ~10% passam no WHERE (o resto foi lido a toa)
Extra: Using where
```

**Com índice** (coluna `cliente_id`, após criar `idx_cliente`):

```
type: ref                <-- "ref" = acesso por indice (otimo)
possible_keys: idx_cliente
key: idx_cliente         <-- o indice que sera usado
key_len: 5
ref: const
rows: 45                 <-- vai ler ~45 linhas, nao 500 mil
filtered: 100.00         <-- quase tudo que le, aproveita
Extra: Using index
```

### A coluna `type` é a mais importante

Da pior para a melhor:

| `type` | Significado | Veredito |
|---|---|---|
| `ALL` | Full table scan — lê a tabela toda | 🔴 ruim em tabela grande |
| `index` | Varre o índice inteiro | 🟠 melhor que ALL, mas ainda varre |
| `range` | Faixa de um índice (`BETWEEN`, `>`, `IN`) | 🟢 bom |
| `ref` | Busca por valor em índice não único | 🟢 bom |
| `eq_ref` / `const` | Uma linha via chave única/PK | 🟢 ótimo |

Regra de bolso: **`type: ALL` em tabela grande = índice faltando.** Olhe também `key` (deve apontar um índice) e `rows` (deve ser pequeno perto do total).

---

## Medindo de verdade: `EXPLAIN ANALYZE`

O `EXPLAIN ANALYZE` **executa** a query e mostra o tempo real e as linhas realmente lidas.

**Sem índice** (saída real):

```
-> Aggregate: count(0), sum(pedidos.valor)  (actual time=76.5..76.5 rows=1 loops=1)
    -> Filter: (pedidos.cliente_id = 1234)  (actual time=1.37..76.4 rows=45 loops=1)
        -> Table scan on pedidos  (cost=50244 rows=498669) (actual time=0.139..61.5 rows=500000 loops=1)
                                                                                   ^^^^^^ leu 500.000 linhas
```

**Com índice** (saída real):

```
-> Aggregate: count(0), sum(pedidos.valor)  (actual time=0.127..0.127 rows=1 loops=1)
    -> Index lookup on pedidos using idx_cliente (cliente_id = 1234)  (actual time=0.119..0.122 rows=45 loops=1)
                                                                                          ^^^ leu 45 linhas
```

### O resultado, lado a lado

| | Sem índice | Com índice |
|---|---|---|
| Acesso | `Table scan` | `Index lookup` |
| Linhas lidas | **500.000** | **45** |
| Tempo | **~76,5 ms** | **~0,13 ms** |

Mesmo resultado, **~600× mais rápido**, só por criar o índice certo. Em uma tabela de milhões de linhas e sob concorrência, essa diferença é a fronteira entre "o sistema voa" e "o sistema caiu".

---

## Criando o índice

```sql
ALTER TABLE pedidos ADD INDEX idx_cliente (cliente_id);
-- ou
CREATE INDEX idx_cliente ON pedidos (cliente_id);
```

Depois, **confirme com `EXPLAIN`** que a query passou a usar o índice (`key` deixou de ser `NULL`).

### Boas práticas de índice

- **Indexe as colunas do `WHERE`, `JOIN` e `ORDER BY`** que filtram muito (alta seletividade). `cliente_id` (10.000 valores) é ótimo; `status` (3 valores) é fraco — um índice ali ajuda pouco.
- **Índice composto** segue a ordem das colunas (regra do "prefixo mais à esquerda"): um índice `(a, b)` serve para `WHERE a=...` e `WHERE a=... AND b=...`, mas **não** para `WHERE b=...` sozinho.
- **`Using index`** no `Extra` = *covering index* (o índice já tem tudo que a query pede, nem toca na tabela). É o melhor cenário.
- **Índice não é de graça:** cada índice pesa em `INSERT`/`UPDATE`/`DELETE` e ocupa disco. Crie os que resolvem queries reais, não "por via das dúvidas".
- **Cuidado com função na coluna:** `WHERE DATE(criado_em)='...'` **não usa** índice em `criado_em`. Prefira `WHERE criado_em >= '...' AND criado_em < '...'`.

---

## Roteiro de diagnóstico

1. Pegue a query lenta (do slow log ou do `sys` — ver docs vizinhos).
2. Rode `EXPLAIN` nela. Olhe `type`, `key` e `rows`.
3. `type: ALL` / `key: NULL` / `rows` enorme → falta índice.
4. Crie o índice na(s) coluna(s) do filtro seletivo.
5. Rode `EXPLAIN` de novo (deve virar `ref`/`range`) e `EXPLAIN ANALYZE` para confirmar o ganho.

---

## Comandos úteis

```sql
EXPLAIN SELECT ...;                      -- plano (padrao: formato em arvore)
EXPLAIN FORMAT=TRADITIONAL SELECT ...;   -- colunas classicas (type/key/rows)
EXPLAIN FORMAT=JSON SELECT ...;          -- detalhado (custo, uso de indice)
EXPLAIN ANALYZE SELECT ...;              -- EXECUTA e mostra tempo e linhas reais

SHOW INDEX FROM tabela;                  -- indices existentes
ALTER TABLE tabela ADD INDEX idx (coluna);
```

> No cliente `mysql` em modo batch, use o flag `-E` para ver o `EXPLAIN` tradicional em formato vertical.

---

## Não incluído neste documento

- Slow query log (como capturar as queries lentas) — `troubleshooting-slow-query-log.md`.
- Achar as piores queries em produção — `troubleshooting-queries-lentas-performance-schema.md`.
- Otimização avançada: índices parciais, `FORCE INDEX`, histogramas, reescrita de query.
