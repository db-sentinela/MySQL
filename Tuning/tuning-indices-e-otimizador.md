# MySQL Tuning — Índices e otimizador (recursos pouco conhecidos)

Deep dive nos recursos que a maioria não usa: **histogramas**, **índices invisíveis**, **optimizer hints**, **colunas invisíveis** e o `optimizer_switch`. Cada um com experimento real no lab.

> Pré-leitura: `tuning-fundamentos-metodologia.md` e, na subpasta Troubleshooting, o doc de `EXPLAIN`/índices (o básico).

---

## 1. Histogramas — estatística de distribuição (muito subutilizado)

O otimizador, sem ajuda, **assume distribuição uniforme** dos valores. Quando os dados são **desbalanceados**, ele erra feio a estimativa e escolhe o plano errado. O **histograma** dá a distribuição real — sem precisar de índice.

### Experimento real

Tabela `eventos` (200 mil linhas) com distribuição desbalanceada:

```
tipo      linhas    %
normal    189922    95.0
erro       10002     5.0
critico       76     0.04
```

`EXPLAIN` da consulta `WHERE tipo='critico'` (um valor **raro**):

```sql
-- ANTES do histograma
EXPLAIN SELECT COUNT(*) FROM eventos WHERE tipo='critico';
--   rows: 200000   filtered: 10.00     <- chute cego: estimaria ~20.000 linhas

-- cria o histograma
ANALYZE TABLE eventos UPDATE HISTOGRAM ON tipo WITH 16 BUCKETS;

-- DEPOIS do histograma
EXPLAIN SELECT COUNT(*) FROM eventos WHERE tipo='critico';
--   rows: 200006   filtered: 0.10      <- real ~0,04%: estima ~200 linhas
```

**100× mais preciso.** O otimizador passa a saber que `critico` é raro — e decide corretamente entre índice, join order, etc.

### Quando usar

- Colunas com **distribuição desigual** usadas em `WHERE`/`JOIN`, **sem índice** (ou onde índice não vale a pena, como colunas de baixa cardinalidade).
- Tipos: `singleton` (poucos valores distintos — foi o caso) ou `equi-height` (muitos valores).

```sql
-- ver/remover
SELECT * FROM information_schema.column_statistics WHERE table_name='eventos';
ANALYZE TABLE eventos DROP HISTOGRAM ON tipo;
```

> Histograma **não** se atualiza sozinho a cada escrita (diferente das stats de índice). Recrie após mudanças grandes na distribuição.

---

## 2. Índices invisíveis — rollout e rollback seguros

Um índice **invisível** existe e é mantido (atualizado nas escritas), mas o **otimizador o ignora**. Serve para testar a remoção de um índice sem apagá-lo, ou introduzir um novo com segurança.

### Experimento real

```sql
-- cria INVISIVEL
ALTER TABLE pedidos ADD INDEX idx_valor (valor) INVISIBLE;

-- (1) EXPLAIN padrao: IGNORADO
EXPLAIN SELECT COUNT(*) FROM pedidos WHERE valor BETWEEN 10 AND 11;
--   type: ALL    key: NULL    rows: 499323     <- full scan

-- (2) TRUQUE: testar o indice invisivel só nesta sessao
SET SESSION optimizer_switch='use_invisible_indexes=on';
EXPLAIN SELECT COUNT(*) FROM pedidos WHERE valor BETWEEN 10 AND 11;
--   type: range  key: idx_valor  rows: 508      <- usado, sem expor aos outros

-- (3) libera para todos
ALTER TABLE pedidos ALTER INDEX idx_valor VISIBLE;
--   type: range  key: idx_valor  rows: 508
```

### Dois usos de ouro

- **Antes de DROPAR** um índice suspeito: torne-o **invisível**, observe se algo piora por dias, e só então apague. Reverter é instantâneo (`VISIBLE`), sem recriar o índice (que em tabela grande é caro).
- **Novo índice em produção**: crie invisível, teste com `use_invisible_indexes=on` na sua sessão, e torne visível quando confiar.

---

## 3. Optimizer hints — forçar o plano quando o otimizador erra

Comentários especiais `/*+ ... */` que orientam o otimizador para **uma query específica** (melhor que variáveis globais):

```sql
-- forcar/proibir um indice
SELECT /*+ INDEX(pedidos idx_cliente) */ ... FROM pedidos WHERE cliente_id=1234;
SELECT /*+ NO_INDEX(pedidos idx_valor) */ ...;

-- ordem de join
SELECT /*+ JOIN_ORDER(a, b) */ ... FROM a JOIN b ...;

-- limitar tempo de execucao (ms) de uma query
SELECT /*+ MAX_EXECUTION_TIME(1000) */ ... ;   -- aborta se passar de 1s

-- ligar um optimizer_switch so nesta query
SELECT /*+ SET_VAR(optimizer_switch='use_invisible_indexes=on') */ ...;
```

Prefira **hints por query** a mudar `optimizer_switch` global — o efeito fica contido e documentado no SQL. Use com parcimônia: o otimizador acerta na maioria dos casos; hint é para a exceção.

---

## 4. Colunas invisíveis (MySQL 8.0.23+)

Colunas que existem mas **não aparecem** em `SELECT *` — só se nomeadas explicitamente. Úteis para adicionar uma coluna sem quebrar apps que fazem `SELECT *` ou `INSERT` sem lista de colunas:

```sql
ALTER TABLE pedidos ADD COLUMN auditoria JSON INVISIBLE;
SELECT * FROM pedidos;            -- nao traz 'auditoria'
SELECT id, auditoria FROM pedidos; -- traz quando pedida
```

---

## 5. Estatísticas de índice (a base de tudo)

O otimizador decide pelos **índices stats**. Se estão velhas (após muita escrita), ele erra:

```sql
ANALYZE TABLE pedidos;     -- recalcula as estatisticas
```

- `innodb_stats_persistent = ON` (padrão) guarda as stats em disco (estáveis entre restarts).
- `innodb_stats_auto_recalc = ON` recalcula após ~10% de mudança.
- Sintoma de stats ruins: `EXPLAIN` com `rows` muito diferente do real → rode `ANALYZE TABLE`.

---

## 6. `optimizer_switch` — ligar/desligar otimizações

Uma coleção de flags do otimizador (MRR, ICP, BKA, hash join, etc.):

```sql
SELECT @@optimizer_switch\G
-- exemplos:
SET SESSION optimizer_switch='use_invisible_indexes=on';
SET SESSION optimizer_switch='block_nested_loop=off';
```

Mexa só com motivo e **na sessão** primeiro. Raramente é a solução — índice e histograma resolvem mais.

---

## Checklist

1. **Histograma** em colunas desbalanceadas sem índice — ganho enorme de estimativa.
2. **Índice invisível** para testar remoção/adição sem risco (+ `use_invisible_indexes=on`).
3. **Optimizer hints** por query para a exceção em que o otimizador erra.
4. `ANALYZE TABLE` quando o `EXPLAIN` estima errado (stats velhas).
5. Colunas invisíveis para evolução de schema sem quebrar `SELECT *`.

---

## Não incluído neste documento

- O básico de `EXPLAIN`/índices — subpasta `Troubleshooting`.
- Índices funcionais, descendentes e multi-valued (JSON).
- Hash join e materialização de subqueries em detalhe.
