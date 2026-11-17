# MySQL Tuning — InnoDB Buffer Pool

O **único ajuste mais importante** do MySQL. O buffer pool é a memória onde o InnoDB guarda dados e índices; o que está nele é lido da RAM (nanossegundos), o que não está vem do disco (milissegundos — 1000× mais lento). Com medição real do lab.

> Pré-leitura: `tuning-fundamentos-metodologia.md`.

---

## O que é e por que importa

Toda leitura/escrita do InnoDB passa pelo buffer pool. Se a **working set** (os dados quentes) cabe nele, o banco praticamente não toca o disco para leitura. Se não cabe, o MySQL fica trocando páginas com o disco o tempo todo — e aí nenhuma outra tunagem salva.

---

## Medindo: o hit ratio

```sql
SELECT ROUND(100*(1 - (
  SELECT VARIABLE_VALUE FROM performance_schema.global_status WHERE VARIABLE_NAME='Innodb_buffer_pool_reads') / (
  SELECT VARIABLE_VALUE FROM performance_schema.global_status WHERE VARIABLE_NAME='Innodb_buffer_pool_read_requests')),2) AS hit_ratio_pct;
```

- `Innodb_buffer_pool_read_requests` = leituras pedidas (lógicas).
- `Innodb_buffer_pool_reads` = as que **tiveram que ir ao disco** (físicas).
- **Hit ratio** = 1 − reads/read_requests.

No lab: **99,81%** (`reads` 1176 de 142262 requests). Acima de ~99% é ótimo; **abaixo de ~95% é sinal de buffer pool pequeno** para o working set.

> ⚠️ Hit ratio alto num banco **pequeno** não prova que o pool está bem dimensionado — prova só que a base cabe. O teste real é com o volume de produção.

---

## Dimensionando

Regra prática para um **servidor dedicado** a MySQL:

> **`innodb_buffer_pool_size` ≈ 50–75% da RAM** (deixando o resto para o SO, conexões e outros buffers).

O lab mostra o problema clássico:

| Item | Valor |
|---|---|
| RAM do host | ~7,6 GB |
| `innodb_buffer_pool_size` | **128 MB (default)** |
| Dados | 34,5 MB |

O pool está no **default de 128 MB** — minúsculo. Hoje funciona só porque a base é de 34 MB. Num servidor dedicado desses, o correto seria algo como **4–5 GB**:

```ini
[mysqld]
innodb_buffer_pool_size = 5G
```

Ou em runtime (é ajustável sem restart, mas redimensionar é custoso — faça em janela):

```sql
SET GLOBAL innodb_buffer_pool_size = 5368709120;   -- 5 GB
```

> Não dê **toda** a RAM ao pool. Sobrou pouco para o SO e para os buffers por conexão → swap → desastre. 50–75% é o ponto seguro.

---

## Buffer pool instances (concorrência)

Em pools grandes, divida em várias instâncias para reduzir contenção interna (cada uma com seu mutex):

```ini
[mysqld]
innodb_buffer_pool_instances = 8     # para pools >= 1-8 GB
```

- Regra: ~1 instância por GB, até 8–16. O lab tem **1** (default, ok para 128 MB).
- Só vale a pena com pool a partir de ~1 GB. Cada instância tem no mínimo o `chunk_size` (128 MB no lab).

---

## Warmup: dump e load (pouco explorado, muito útil)

Quando o MySQL reinicia, o buffer pool começa **vazio** → as primeiras queries são lentas (tudo vem do disco) até "esquentar". O MySQL resolve isso salvando/restaurando a lista de páginas quentes:

```sql
-- no lab, ambos ja estao ON (bom):
innodb_buffer_pool_dump_at_shutdown = ON   -- salva a lista ao desligar
innodb_buffer_pool_load_at_startup  = ON   -- recarrega ao subir
```

Forçar manualmente (ex.: antes de um pico previsto):

```sql
SET GLOBAL innodb_buffer_pool_dump_now = ON;   -- salva agora
SET GLOBAL innodb_buffer_pool_load_now = ON;   -- recarrega agora
-- acompanhar:
SHOW STATUS LIKE 'Innodb_buffer_pool_load_status';
```

Isso salva **só a lista de páginas** (IDs), não os dados — o load relê do disco, mas de forma sequencial e dirigida. Reduz muito o tempo de "esquentar" após um restart/failover.

---

## O que está no pool agora (diagnóstico real)

A view `sys.innodb_buffer_stats_by_table` mostra o que está cacheado:

```sql
SELECT object_schema, object_name, allocated, data, pages
FROM sys.innodb_buffer_stats_by_table ORDER BY pages DESC LIMIT 6;
```

Saída real:

```
tshoot   pedidos   22.75 MiB   20.84 MiB   1456 pages
tshoot   eventos    6.69 MiB    6.09 MiB    428 pages
...
```

Útil para ver **quais tabelas dominam** a memória — e decidir se o working set cabe.

---

## Checklist

1. `innodb_buffer_pool_size` = 50–75% da RAM (servidor dedicado) — **o nº 1**.
2. Hit ratio monitorado (meta > 99%); queda = pool pequeno.
3. `innodb_buffer_pool_instances` para pools grandes (>1 GB).
4. `dump_at_shutdown`/`load_at_startup` **ON** (warmup automático).
5. Não dar RAM demais ao pool (deixe para SO + conexões).

---

## Não incluído neste documento

- Memória por conexão e temp tables — `tuning-memoria-e-buffers.md`.
- `innodb_old_blocks_pct`/`time` (resistência a varreduras que poluem o pool).
- NUMA e `innodb_numa_interleave` em servidores grandes.
