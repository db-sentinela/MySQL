# MySQL Tuning — Redo log, flushing e durabilidade

O coração da escrita do InnoDB: o redo log e as decisões de **flush**. Aqui mora o trade-off clássico entre **durabilidade (ACID)** e **performance de escrita**. Config real do lab.

> Pré-leitura: `tuning-fundamentos-metodologia.md`.

---

## Como o InnoDB escreve (resumo)

1. A transação altera páginas no **buffer pool** (memória) e grava no **redo log** (sequencial, rápido).
2. No `COMMIT`, conforme a durabilidade, o redo é **sincronizado** no disco.
3. Em background, as páginas "sujas" do buffer pool são escritas nos arquivos de dados (**flush**), no ritmo do `innodb_io_capacity`.

O redo log permite recuperar transações confirmadas após uma queda. Por isso o tamanho dele e a política de flush definem **quão rápido** e **quão seguro** o banco escreve.

---

## A variável mais crítica: `innodb_flush_log_at_trx_commit`

Define o que acontece com o redo **a cada commit**:

| Valor | Comportamento | Durabilidade | Performance |
|---|---|---|---|
| **1** (lab, padrão) | Escreve **e** sincroniza (fsync) no disco a cada commit | 🟢 **Total (ACID)** — não perde nada | Mais lenta |
| **2** | Escreve no SO a cada commit, fsync a cada ~1s | 🟡 Perde ~1s **só se o SO/host cair** | Mais rápida |
| **0** | Escreve+fsync a cada ~1s (não no commit) | 🔴 Perde ~1s se o **mysqld** cair | Mais rápida |

**No lab está `1`** — correto para um ambiente que não pode perder transações (financeiro, pedidos). Em um cluster com Group Replication, a durabilidade também é reforçada pela replicação síncrona entre os nós.

> Quando baixar para `2`: cargas de escrita massiva onde perder 1s em um crash de host é aceitável (ex.: ingestão de dados reprocessáveis). **Nunca** baixe sem entender o risco de perda.

---

## `innodb_flush_method`

Como o InnoDB escreve nos arquivos. No lab: **`O_DIRECT`** (✅).

| Método | Efeito |
|---|---|
| `O_DIRECT` | Ignora o cache do SO para os **dados** → evita **cache duplo** (o buffer pool já cacheia). Recomendado em Linux com buffer pool grande. |
| `fsync` | Usa o cache do SO (cache duplo, desperdiça RAM). |

`O_DIRECT` é a escolha certa quando o buffer pool é generoso.

---

## Redo log capacity

No lab: **`innodb_redo_log_capacity = 100 MB`**.

```ini
[mysqld]
innodb_redo_log_capacity = 2G    # exemplo para carga de escrita alta
```

- Redo **maior** = menos flushes forçados, escrita mais suave em rajadas → melhor throughput de escrita. Custo: recovery pós-crash um pouco mais longo e mais disco.
- Redo **pequeno** (como os 100 MB do lab) = o InnoDB precisa fazer *checkpoint* com frequência sob carga, o que **limita a escrita**. Para um servidor de escrita intensa, 1–4 GB é comum.
- No MySQL 8.0.30+/9.x, use `innodb_redo_log_capacity` (substitui o antigo par `innodb_log_file_size` × `innodb_log_files_in_group`).

Monitorar pressão de checkpoint: `SHOW ENGINE INNODB STATUS` (seção LOG) e `Innodb_checkpoint_age`.

---

## `innodb_io_capacity` (ritmo do flush de background)

Diz ao InnoDB quantas operações de I/O por segundo o disco aguenta, para dosar o flush de páginas sujas. No lab: **10000 / 20000 (max)** — valores de **SSD/NVMe**.

| Disco | `innodb_io_capacity` sugerido |
|---|---|
| HDD 7200rpm | 100–200 |
| SSD SATA | 1000–5000 |
| NVMe | 10000+ |

> Valor alto demais em disco lento faz o InnoDB tentar flush agressivo e **competir** com as queries. Ajuste ao disco real.

---

## `innodb_doublewrite`

No lab: **ON** (✅). Protege contra *torn pages* (página escrita pela metade num crash): o InnoDB escreve primeiro numa área doublewrite, depois no destino. Deixe **ON** salvo em filesystems que garantem escrita atômica. É barato e evita corrupção.

---

## Resumo: durabilidade × performance

```mermaid
flowchart LR
    S["Segurança máxima<br/>trx_commit=1, doublewrite ON"] <--> P["Performance de escrita<br/>trx_commit=2, redo maior"]
```

O lab está no lado **seguro** (ACID total) — adequado para dados críticos. Para escrita massiva reprocessável, mover o ponteiro em direção à performance é válido, **conscientemente**.

---

## Checklist

1. `innodb_flush_log_at_trx_commit` = **1** para dados críticos (lab ✅); `2` só com risco entendido.
2. `innodb_flush_method` = **O_DIRECT** em Linux com buffer pool grande (lab ✅).
3. `innodb_redo_log_capacity` dimensionado à carga de escrita (lab 100 MB → pequeno para escrita intensa).
4. `innodb_io_capacity` casado com o disco real (lab em valores de SSD).
5. `innodb_doublewrite` = **ON** (lab ✅).

---

## Não incluído neste documento

- Buffer pool — `tuning-innodb-buffer-pool.md`.
- `innodb_adaptive_flushing`, `innodb_lru_scan_depth` e tuning fino de flush.
- Interação redo/durabilidade com a durabilidade do Group Replication.
