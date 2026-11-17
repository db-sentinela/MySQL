# MySQL Tuning — Memória e buffers (global × por conexão)

O erro de tuning mais perigoso: aumentar buffers **por conexão** achando que ajuda — e estourar a RAM, derrubando o servidor por OOM/swap. Entenda a diferença e calcule o teto de memória. Valores reais do lab.

> Pré-leitura: `tuning-innodb-buffer-pool.md`.

---

## A distinção que salva o servidor

| Tipo | Alocação | Exemplos |
|---|---|---|
| **Global** | **Uma vez** para o servidor | `innodb_buffer_pool_size`, `innodb_log_buffer_size` |
| **Por conexão (sessão)** | **Multiplica** por cada conexão ativa | `sort_buffer_size`, `join_buffer_size`, `read_buffer_size`, `tmp_table_size` |

🔴 **O perigo:** `sort_buffer_size = 256M` com `max_connections = 151` pode querer **até ~38 GB** só de sort buffers. Buffer por conexão grande é uma bomba-relógio — cada conexão que ordena aloca aquilo.

---

## Memória por conexão (valores reais do lab)

| Variável | Valor no lab | Observação |
|---|---|---|
| `sort_buffer_size` | 256 KB | ✅ default sadio |
| `join_buffer_size` | 256 KB | ✅ só usado em joins sem índice |
| `read_buffer_size` | 128 KB | ✅ |
| `read_rnd_buffer_size` | 256 KB | ✅ |
| `tmp_table_size` / `max_heap_table_size` | 16 MB / 16 MB | teto de tabela temporária em memória |

Os defaults são **pequenos de propósito** — aumente **pontualmente** e só se o diagnóstico pedir (ex.: muitos `sort_merge_passes`). Não aumente "por garantia".

### Quando (e como) aumentar com segurança

Prefira aumentar **na sessão**, só para a query que precisa, em vez de global:

```sql
SET SESSION sort_buffer_size = 4*1024*1024;   -- só esta conexão
SELECT ... ORDER BY ...;
```

Sinais de que um buffer está pequeno (status):
- `Sort_merge_passes` alto → `sort_buffer_size` pequeno para as ordenações.
- `Select_full_join` alto → faltam índices (não é para aumentar `join_buffer`, é para indexar).

---

## O teto de memória (a conta que você deve fazer)

```
RAM usada ≈ buffer_pool + log_buffer + outros globais
          + (max_connections × buffers_por_conexão_no_pior_caso)
```

Deixe **folga para o SO**. Se a conta passa da RAM física, o servidor vai para **swap** (lentidão brutal) ou morre por **OOM**. Por isso: buffer pool generoso (global, previsível) + buffers por conexão **pequenos**.

---

## Tabelas temporárias (onde a memória vira disco)

Queries com `GROUP BY`, `DISTINCT`, `ORDER BY` complexo ou subqueries criam **tabelas temporárias**. Se cabem na RAM, rápido; se estouram `tmp_table_size`/`max_heap_table_size`, vão para **disco** — lento.

Mede-se por status (real no lab):

```sql
SHOW GLOBAL STATUS WHERE Variable_name IN ('Created_tmp_tables','Created_tmp_disk_tables');
-- lab: Created_tmp_tables=16922, Created_tmp_disk_tables=6
```

Razão `disk_tables / tmp_tables` baixa (aqui ~0,04%) é ótima. Se muitas vão para disco:
1. Primeiro, **otimize a query** (índice no `GROUP BY`/`ORDER BY`, menos colunas).
2. Só então considere subir `tmp_table_size`/`max_heap_table_size` (juntos).

### O engine TempTable (moderno)

No lab, `internal_tmp_mem_storage_engine = TempTable` (padrão atual, melhor que o antigo MEMORY — suporta `VARCHAR`/`BLOB` em RAM). O teto global de RAM dele é `temptable_max_ram` (1 GB no lab); ao estourar, usa arquivos mapeados em memória e depois disco.

---

## Outros buffers globais

| Variável | Lab | Papel |
|---|---|---|
| `innodb_log_buffer_size` | 64 MB | buffer do redo antes de ir ao disco; maior ajuda em escrita em rajada |
| `table_open_cache` | 4000 | descritores de tabela abertos (evita reabrir) |

---

## Checklist

1. **Buffers por conexão pequenos** (defaults); aumente **por sessão**, não global.
2. Faça a **conta do teto** de RAM (`pool + conexões × buffers`) com folga para o SO.
3. `Created_tmp_disk_tables` baixo; se alto, **otimize a query antes** de subir `tmp_table_size`.
4. `Sort_merge_passes`/`Select_full_join` altos → investigue (índice ou sort buffer da sessão).
5. Nunca `sort_buffer_size` gigante global — multiplica por conexão.

---

## Não incluído neste documento

- Buffer pool (global principal) — `tuning-innodb-buffer-pool.md`.
- Redo/undo log e flushing — `tuning-redo-log-e-durabilidade.md`.
- Perfil de memória detalhado via `performance_schema` (`memory_summary_*`).
