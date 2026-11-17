# MySQL Security — Autenticação e senhas

Como o MySQL autentica, por que contas sem senha e senhas fracas são um buraco, e como forçar uma política. Com a auditoria real do lab e a **simulação do `validate_password`**.

> Pré-leitura: `security-fundamentos-hardening.md`.

---

## Plugins de autenticação

Cada conta tem um **plugin** que valida a senha. No lab (auditoria real), **todas** usam `caching_sha2_password`:

```
user                     plugin                   senha
app                      caching_sha2_password    definida
clusteradmin             caching_sha2_password    definida
root@localhost           caching_sha2_password    ** VAZIA **
...
```

| Plugin | Avaliação |
|---|---|
| `caching_sha2_password` | ✅ Padrão moderno (SHA-256, mais forte). Exige conexão segura para enviar a senha em texto. |
| `mysql_native_password` | ⚠️ Legado (SHA-1). Evite; descontinuado. |
| `auth_socket` | Login pelo usuário do SO (sem senha de rede) — bom para contas administrativas locais. |

> Migrar uma conta para o plugin forte: `ALTER USER 'x'@'h' IDENTIFIED WITH caching_sha2_password BY 'NovaSenha';`

---

## 🔴 Achado real: `root@localhost` sem senha

A auditoria mostrou **`root@localhost` com `authentication_string` VAZIA** — senha em branco (herança do `mysqld --initialize-insecure` usado para montar o lab). Consulta para achar isso:

```sql
SELECT user, host FROM mysql.user WHERE authentication_string = '';
```

**Risco:** qualquer um que alcance o socket/localhost entra como root sem senha. Correção:

```sql
ALTER USER 'root'@'localhost' IDENTIFIED BY 'SenhaRootForte!2026';
```

> Mesmo que o root só aceite conexão local, senha em branco é inaceitável. E confirme que **não existe `root@'%'`** (root remoto).

---

## Simulação: política de senha com `validate_password`

Sem política, o MySQL aceita **qualquer** senha. Demonstração real no lab:

```sql
-- SEM politica: senha fraca passa
CREATE USER seg_weak@'%' IDENTIFIED BY '123';     -- ACEITO (!)
```

Instale a política (é um componente nativo):

```sql
INSTALL COMPONENT 'file://component_validate_password';
```

A política padrão (lida do lab após instalar):

```
validate_password.policy                MEDIUM
validate_password.length                8
validate_password.mixed_case_count      1
validate_password.number_count          1
validate_password.special_char_count    1
validate_password.check_user_name       ON
```

Agora a senha fraca é **rejeitada** (saída real):

```sql
CREATE USER seg_weak@'%' IDENTIFIED BY '123';
-- ERROR 1819 (HY000): Your password does not satisfy the current policy requirements

CREATE USER seg_strong@'%' IDENTIFIED BY 'Str0ng#Pass!2026';
-- OK  (8+ chars, maiúscula, número e símbolo)
```

### Ajustar a política

```sql
SET GLOBAL validate_password.policy = STRONG;   -- LOW | MEDIUM | STRONG
SET GLOBAL validate_password.length = 12;
SET PERSIST validate_password.policy = STRONG;  -- persiste após restart
```

Para valer sempre, instale o componente no `my.cnf` de inicialização ou deixe-o instalado (o `INSTALL COMPONENT` persiste no catálogo).

---

## Expiração e histórico de senha

```sql
-- Expirar a cada 90 dias
ALTER USER 'app'@'%' PASSWORD EXPIRE INTERVAL 90 DAY;

-- Forçar troca no próximo login
ALTER USER 'joao'@'%' PASSWORD EXPIRE;

-- Impedir reuso das últimas N senhas / por tempo
ALTER USER 'joao'@'%' PASSWORD HISTORY 5;
ALTER USER 'joao'@'%' PASSWORD REUSE INTERVAL 365 DAY;
```

Padrões globais no `my.cnf`:

```ini
[mysqld]
default_password_lifetime = 90
password_history          = 5
password_reuse_interval   = 365
```

---

## Bloqueio de conta e brute force

```sql
-- Bloquear/desbloquear uma conta
ALTER USER 'joao'@'%' ACCOUNT LOCK;
ALTER USER 'joao'@'%' ACCOUNT UNLOCK;

-- Travar após N falhas de senha, por T segundos (anti brute-force)
ALTER USER 'joao'@'%' FAILED_LOGIN_ATTEMPTS 5 PASSWORD_LOCK_TIME 2;
```

> As contas internas do cluster (`mysql.session`, `mysql.sys`, `mysql.infoschema`) aparecem com `account_locked = Y` — é o esperado; elas não fazem login de rede.

---

## Checklist

1. **Nenhuma conta sem senha** (cheque `authentication_string = ''`).
2. Todas em `caching_sha2_password` (migre as `mysql_native_password`).
3. `validate_password` instalado, política MEDIUM/STRONG.
4. Expiração e histórico de senha definidos.
5. `FAILED_LOGIN_ATTEMPTS` para conter brute force.
6. Sem `root@'%'`; root local com senha forte.

---

## Não incluído neste documento

- Autenticação externa (LDAP/PAM/Kerberos) e autenticação multifator.
- Rotação automatizada de segredos (Vault etc.).
- Detecção de brute force via logs — `security-auditoria-logging.md`.
