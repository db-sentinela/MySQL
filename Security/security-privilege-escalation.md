# MySQL Security — Privilege escalation (do banco para o sistema)

Como um atacante com acesso a uma conta privilegiada pode sair do banco e **ler/escrever arquivos do SO** — e como o `secure_file_priv` e companhia impedem isso. Com simulação real do privilégio `FILE`.

> Pré-leitura: `security-usuarios-privilegios.md`. ⚠️ Conteúdo ofensivo em laboratório, para fins **defensivos**.

---

## O vetor mais clássico: o privilégio `FILE`

O privilégio `FILE` permite que uma conta leia e escreva **arquivos no disco do servidor** com as permissões do usuário `mysql`, via:

- `LOAD_FILE('/caminho')` — lê um arquivo e devolve o conteúdo numa query.
- `SELECT ... INTO OUTFILE '/caminho'` — escreve o resultado de uma query num arquivo.
- `LOAD DATA INFILE` — carrega um arquivo para dentro de uma tabela.

Com `FILE` e um `secure_file_priv` mal configurado, isso vira:
- **Leitura de segredos:** `/etc/passwd`, chaves, arquivos de config com senhas.
- **RCE (execução remota):** escrever um *webshell* na pasta de um servidor web, ou um cron job.

### O freio: `secure_file_priv`

Esta variável limita **onde** `LOAD_FILE`/`OUTFILE` podem tocar:

| Valor | Efeito |
|---|---|
| vazio (`''`) | 🔴 **sem limite** — lê/escreve em qualquer lugar (muito perigoso) |
| `NULL` | ✅ desativa `FILE` por completo |
| um diretório (ex.: `/var/lib/mysql-files/`) | ✅ restringe a esse diretório |

No lab está em **`/var/lib/mysql-files/`** (bom).

---

## Simulação real (no lab)

O `clusteradmin` tem `FILE`. Com `secure_file_priv = /var/lib/mysql-files/`:

```sql
-- escrever um arquivo via SQL (dentro do diretorio permitido)
SELECT 'dados sensiveis do banco' INTO OUTFILE '/var/lib/mysql-files/poc.txt';

-- ler de volta (dentro do permitido) -> FUNCIONA
SELECT LOAD_FILE('/var/lib/mysql-files/poc.txt');
-- -> 'dados sensiveis do banco'

-- tentar ler FORA (/etc/passwd) -> BLOQUEADO
SELECT LOAD_FILE('/etc/passwd');
-- -> NULL   (bloqueado pelo secure_file_priv)
```

**Interpretação:** o mecanismo `FILE` funciona (escreveu e leu dentro do diretório), mas o `secure_file_priv` **impediu** a leitura de `/etc/passwd`. Se o `secure_file_priv` estivesse vazio, aquele último `LOAD_FILE` devolveria o conteúdo de `/etc/passwd` — e um `INTO OUTFILE '/var/www/html/shell.php'` poderia plantar um webshell. **É essa a diferença entre config segura e comprometimento.**

---

## Outros vetores de escalonamento

### `local_infile` (lado cliente)
Com `local_infile = ON`, um **servidor malicioso** (ou um MITM) pode mandar o **cliente** enviar arquivos locais dele via `LOAD DATA LOCAL INFILE`. No lab está `OFF` (✅). Mantenha assim; ligue só quando uma ferramenta exigir (ex.: `util.loadDump`) e desligue depois.

### `SUPER` / `SYSTEM_VARIABLES_ADMIN`
Permite mudar variáveis globais em runtime — inclusive **baixar `secure_file_priv`?** Não (é read-only), mas pode, por exemplo, desligar o log de auditoria, alterar `init_connect`, mexer na replicação. Dê só a quem administra.

### UDF maliciosa (`INSERT` em `mysql.func`)
Com `INSERT` em `mysql` + `FILE`, um atacante pode escrever uma biblioteca `.so` no diretório de plugins e registrar uma **UDF** que executa comandos do SO. Defesa: não dar `FILE` + escrita em `mysql`, e `--plugin-dir` com permissão restrita.

### `secure_file_priv` + backups/replicação
Cuidado com scripts que rodam como root e leem arquivos gerados por `OUTFILE`; valide caminhos.

---

## Como se defender

- [ ] **`FILE` só para quem precisa** (quase nunca para apps). Audite: `SELECT GRANTEE FROM information_schema.user_privileges WHERE PRIVILEGE_TYPE='FILE';`
- [ ] **`secure_file_priv` nunca vazio** — um diretório restrito ou `NULL`.
- [ ] **`local_infile = OFF`** (ligue só pontualmente).
- [ ] **`SUPER`/admin dinâmicos** restritos a contas administrativas.
- [ ] **Permissões do SO:** o usuário `mysql` não deve ter escrita em diretórios web, cron, etc.
- [ ] **AppArmor/SELinux** confinando o `mysqld` (no lab o AppArmor já limita o `mysqld` a `/var/lib/mysql` — camada extra que, por exemplo, barrou escrita de clone em `/backups`).

---

## Auditoria rápida

```sql
-- Quem tem FILE
SELECT GRANTEE FROM information_schema.user_privileges WHERE PRIVILEGE_TYPE = 'FILE';

-- Config dos freios
SHOW GLOBAL VARIABLES WHERE Variable_name IN ('secure_file_priv','local_infile','plugin_dir');
```

---

## Não incluído neste documento

- Exploração detalhada de UDF e cadeias de RCE (fora de escopo defensivo).
- Escalonamento via replicação/`repl` user.
- Hardening de SELinux/AppArmor passo a passo.
