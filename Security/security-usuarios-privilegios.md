# MySQL Security — Usuários e privilégios (least privilege)

O princípio mais importante de segurança de banco: **cada conta deve ter o mínimo de privilégio que o trabalho dela exige**. Assim, uma credencial vazada causa o mínimo de estrago. Com auditoria real do lab.

> Pré-leitura: `security-fundamentos-hardening.md`.

---

## Por que "least privilege" é tudo

Se a conta da aplicação só precisa de `SELECT/INSERT/UPDATE` em um schema, mas tem `ALL ON *.*`, então um **SQL injection** nessa app não vaza uma tabela — vaza o **servidor inteiro**, e com `FILE` pode até ler arquivos do SO. O privilégio excessivo transforma um bug pequeno em desastre.

---

## Auditoria real: quem tem privilégio perigoso

Consulta de auditoria (rode periodicamente):

```sql
SELECT GRANTEE, GROUP_CONCAT(PRIVILEGE_TYPE ORDER BY PRIVILEGE_TYPE) AS privs_globais
FROM information_schema.user_privileges
WHERE PRIVILEGE_TYPE IN ('SUPER','FILE','GRANT OPTION','SHUTDOWN','CREATE USER','PROCESS','RELOAD')
GROUP BY GRANTEE;
```

Saída **real** do lab:

```
'clusteradmin'@'%'          CREATE USER, FILE, PROCESS, RELOAD, SHUTDOWN, SUPER
'mysql.session'@'localhost' SHUTDOWN, SUPER
'root'@'localhost'          CREATE USER, FILE, PROCESS, RELOAD, SHUTDOWN, SUPER
```

**Análise:**
- `root@localhost` e `mysql.session` — esperado (contas de sistema).
- **`clusteradmin@'%'`** — este é o ponto de atenção. É um usuário **administrativo do cluster** com `ALL PRIVILEGES ... WITH GRANT OPTION` e host curinga `'%'`. Na documentação original de instalação isso é explicitamente marcado como **"em produção, restrinja os privilégios do clusteradmin"**. É o nosso exemplo vivo de conta superprivilegiada.

### Os privilégios que exigem cuidado redobrado

| Privilégio | Por que é perigoso |
|---|---|
| `ALL PRIVILEGES` | Dá tudo; quase nunca é necessário para uma app |
| `WITH GRANT OPTION` | A conta pode **dar privilégios a outras** (escalonar) |
| `FILE` | Ler/escrever arquivos do SO (ver privilege escalation) |
| `SUPER` / `SYSTEM_VARIABLES_ADMIN` | Mudar variáveis globais, matar conexões, desligar replicação |
| `SHUTDOWN` | Derrubar o servidor |
| `CREATE USER` | Criar/alterar contas |
| `PROCESS` | Ver queries de **todos** (pode vazar dados em trânsito) |
| host `'%'` | Conecta de qualquer lugar |

---

## Como aplicar least privilege

### Conta de aplicação (o padrão correto)

```sql
-- SO o necessario, SO no schema da app, SO do host da app
CREATE USER 'app'@'10.0.0.%' IDENTIFIED BY 'SenhaForte!2026';
GRANT SELECT, INSERT, UPDATE, DELETE ON appdb.* TO 'app'@'10.0.0.%';
-- nada de ALL, nada de *.*, nada de %
```

### Roles (agrupar privilégios — MySQL 8.0+)

Em vez de repetir grants em cada conta, use **roles**:

```sql
CREATE ROLE 'app_rw';
GRANT SELECT, INSERT, UPDATE, DELETE ON appdb.* TO 'app_rw';
GRANT 'app_rw' TO 'app'@'10.0.0.%';
SET DEFAULT ROLE 'app_rw' TO 'app'@'10.0.0.%';
```

Vantagem: revisa/ajusta a role uma vez, vale para todas as contas.

### Revisar e cortar excesso

```sql
-- o que uma conta realmente tem
SHOW GRANTS FOR 'app'@'10.0.0.%';

-- cortar o que sobra
REVOKE FILE ON *.* FROM 'suspeito'@'%';
REVOKE GRANT OPTION ON *.* FROM 'suspeito'@'%';
```

> **`clusteradmin` em produção:** mantenha-o só para administração do cluster (acesso restrito, host conhecido, idealmente via bastion), e dê às aplicações contas próprias e mínimas. Nunca use o `clusteradmin` como usuário da aplicação.

---

## Consultas de auditoria úteis

```sql
-- Contas com host curinga
SELECT user, host FROM mysql.user WHERE host = '%';

-- Contas anonimas (devem ser ZERO)
SELECT user, host FROM mysql.user WHERE user = '';

-- Quem pode dar privilegios (GRANT OPTION)
SELECT DISTINCT GRANTEE FROM information_schema.user_privileges WHERE PRIVILEGE_TYPE = 'GRANT OPTION';

-- Privilegios por schema de uma conta
SELECT * FROM information_schema.schema_privileges WHERE GRANTEE = "'app'@'%'";

-- Todos os grants de todos (revisao geral)
SELECT GRANTEE, PRIVILEGE_TYPE FROM information_schema.user_privileges ORDER BY GRANTEE;
```

---

## Checklist

1. **Nenhuma** conta anônima; **nenhuma** conta de app com `ALL`/`*.*`.
2. Apps com privilégio mínimo, schema específico, host específico.
3. `WITH GRANT OPTION`, `FILE`, `SUPER`, `SHUTDOWN` só para quem realmente administra.
4. `clusteradmin`/contas admin isoladas, host restrito, fora das mãos da aplicação.
5. Auditoria periódica com as consultas acima.

---

## Não incluído neste documento

- Senhas, expiração e política — `security-autenticacao-senhas.md`.
- Como o `FILE` vira leitura/escrita de arquivos do SO — `security-privilege-escalation.md`.
- Grants dinâmicos (`*_ADMIN`) em detalhe e auditoria contínua automatizada.
