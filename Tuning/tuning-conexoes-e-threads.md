# MySQL Tuning — Conexões e threads

Conexão demais mata o MySQL tanto quanto conexão de menos trava a aplicação. Como dimensionar, por que um **pool de conexões na aplicação** quase sempre é a resposta certa, e os ajustes de thread. Config real do lab.

> Pré-leitura: `tuning-fundamentos-metodologia.md`.

---

## O mito do "aumenta o max_connections"

Quando a app dá *"Too many connections"*, a reação errada é subir `max_connections` sem pensar. Cada conexão consome memória (buffers por conexão) e CPU (troca de contexto). **Milhares** de conexões ativas fazem o servidor gastar mais tempo gerenciando threads do que executando queries.

A resposta certa quase sempre é: **pool de conexões na aplicação** (HikariCP, PgBouncer-like, pool do framework) que reusa poucas conexões, em vez de abrir uma por request.

---

## `max_connections`

No lab: **151** (default).

```sql
SHOW GLOBAL STATUS LIKE 'Max_used_connections';   -- pico historico de conexoes
SHOW GLOBAL STATUS LIKE 'Threads_connected';       -- conectadas agora (lab: 4)
```

Dimensione pelo **pico real** (`Max_used_connections`) com folga, não por chute. Se o pico é 40, `max_connections=151` sobra; se bate no teto, primeiro veja se é a app abrindo conexão demais (falta de pool) antes de subir.

> Lembre do teto de memória: cada conexão pode alocar buffers por conexão. `max_connections` alto + buffers grandes = RAM estourada (ver `tuning-memoria-e-buffers.md`).

---

## Thread cache

Abrir uma thread por conexão nova custa. O `thread_cache_size` guarda threads ociosas para reusar.

No lab: **`thread_cache_size = 9`** (baixo). Diagnóstico:

```sql
SHOW GLOBAL STATUS WHERE Variable_name IN ('Threads_created','Connections');
-- lab: Threads_created=7
```

- `Threads_created` crescendo rápido = thread cache pequeno (muita criação de thread). Suba `thread_cache_size`.
- Com um **pool na aplicação** (conexões persistentes), a criação de threads é rara e o thread cache importa menos.

Valor típico: algo próximo do `Max_used_connections` (ou dezenas, não milhares).

---

## `wait_timeout` (conexões ociosas)

No lab: **28800 s (8h)**. Conexões ociosas ocupam slot e memória por até 8h.

```ini
[mysqld]
wait_timeout = 600        # fecha conexao ociosa apos 10 min
interactive_timeout = 600
```

Baixar ajuda a liberar conexões "esquecidas" por apps sem pool — mas se a app usa pool persistente, não deixe baixo demais (o pool reclamaria de conexões mortas). Ajuste junto com o pool.

---

## `table_open_cache`

No lab: **4000**. Descritores de tabela abertos (evita reabrir arquivos a cada acesso).

```sql
SHOW GLOBAL STATUS LIKE 'Table_open_cache_misses';
```
Muitos `misses` → suba `table_open_cache`. Relevante em bancos com **muitas tabelas**.

---

## Thread pool (cargas muito concorrentes)

Por padrão o MySQL usa **uma thread por conexão**. Sob **milhares** de conexões concorrentes, isso degrada. O **thread pool** (plugin no MySQL Enterprise; disponível no Percona Server) multiplexa muitas conexões em poucas threads, mantendo a CPU eficiente.

- Community (como o lab): não tem thread pool nativo → a defesa é o **pool na aplicação** + `max_connections` sensato.
- Percona Server / Enterprise: avaliar o thread pool se houver milhares de conexões simultâneas.

---

## Checklist

1. **Pool de conexões na aplicação** — a medida nº 1 (poucas conexões reusadas).
2. `max_connections` pelo pico real (`Max_used_connections`) com folga, não por chute.
3. `thread_cache_size` adequado se `Threads_created` cresce rápido.
4. `wait_timeout` menor para matar conexões ociosas (coordenado com o pool).
5. `table_open_cache` se houver muitos `Table_open_cache_misses`.
6. Thread pool (Percona/Enterprise) só para concorrência extrema.

---

## Não incluído neste documento

- Memória por conexão e o teto de RAM — `tuning-memoria-e-buffers.md`.
- Configuração do MySQL Router como pool/roteamento — subpasta InnoDB-Cluster.
- Tuning de thread pool do Percona Server em detalhe.
