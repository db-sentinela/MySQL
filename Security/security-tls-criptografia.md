# MySQL Security — TLS e criptografia

Proteger os dados **em trânsito** (TLS) e **em repouso** (encryption at rest). Com a simulação real mostrando a diferença entre conexão em texto puro e conexão forçada a TLS.

> Pré-leitura: `security-fundamentos-hardening.md`.

---

## Em trânsito: por que TLS

Sem TLS, **senhas e dados trafegam em texto puro** na rede — qualquer sniffer entre a app e o banco lê tudo. O MySQL 9.7 já vem com TLS **disponível** (certificados auto-gerados no `datadir`), mas no lab ele **não é obrigatório** (`require_secure_transport = OFF`).

### Simulação real: com e sem TLS

```sql
-- (1) HOJE: conexao SEM TLS e aceita
```
```bash
mysql --ssl-mode=DISABLED -u clusteradmin -p -e "SHOW SESSION STATUS LIKE 'Ssl_cipher'"
```
```
Ssl_cipher    (vazio)     <- texto puro, sem criptografia
```

Agora criamos uma conta que **exige** TLS e tentamos sem:

```sql
CREATE USER seg_tls@'%' IDENTIFIED BY 'Tls#Forte2026' REQUIRE SSL;
```
```bash
# sem TLS -> REJEITADO
mysql --ssl-mode=DISABLED -u seg_tls -p... -e "SELECT 1"
# ERROR 2061 (HY000): Authentication requires secure connection.

# com TLS -> OK
mysql --ssl-mode=REQUIRED -u seg_tls -p... -e "SHOW SESSION STATUS LIKE 'Ssl_cipher'"
# Ssl_cipher = TLS_AES_128_GCM_SHA256   (TLSv1.3)
```

A diferença é clara: a mesma conta, o mesmo comando — sem TLS é barrada, com TLS conecta cifrada.

---

## Tornando TLS obrigatório

Duas formas (use as duas):

### 1. Global — todas as conexões

```sql
SET PERSIST require_secure_transport = ON;
```
```ini
[mysqld]
require_secure_transport = ON
```
Depois disso, **qualquer** conexão sem TLS é recusada.

> ⚠️ Em cluster, teste com cuidado: garanta que app, Router e ferramentas conectem com `--ssl-mode=REQUIRED` antes de forçar, para não se trancar para fora. O Group Replication tem a sua própria configuração de SSL entre os nós.

### 2. Por conta — exige e opcionalmente valida o certificado

```sql
ALTER USER 'app'@'%' REQUIRE SSL;         -- exige canal TLS
ALTER USER 'app'@'%' REQUIRE X509;        -- exige certificado de cliente valido
ALTER USER 'app'@'%' REQUIRE SUBJECT '...' ISSUER '...';  -- mTLS estrito
```

### O lado do cliente: `--ssl-mode`

| Modo | Comportamento |
|---|---|
| `DISABLED` | Nunca usa TLS (texto puro) |
| `PREFERRED` | Usa TLS se der (padrão) — **não protege contra MITM** |
| `REQUIRED` | Exige TLS, mas não valida o certificado do servidor |
| `VERIFY_CA` | Exige TLS **e** valida a CA |
| `VERIFY_IDENTITY` | Exige TLS, valida CA **e** o hostname (recomendado) |

Para proteção real contra MITM, a app deve usar **`VERIFY_CA`/`VERIFY_IDENTITY`** com a CA correta — não basta `REQUIRED`.

---

## Certificados próprios (produção)

Os certificados auto-gerados servem para ligar TLS, mas em produção use uma **CA própria**:

```ini
[mysqld]
ssl_ca   = /etc/mysql/ssl/ca.pem
ssl_cert = /etc/mysql/ssl/server-cert.pem
ssl_key  = /etc/mysql/ssl/server-key.pem
tls_version = TLSv1.2,TLSv1.3
```

Gerar um conjunto rapidamente: `mysql_ssl_rsa_setup` cria CA + certs no datadir. Para uma CA real/corporativa, emita os certs e aponte as variáveis acima. Mantenha `tls_version` só em **TLSv1.2 e TLSv1.3** (desative versões antigas).

---

## Em repouso: encryption at rest

Protege os **arquivos** do banco caso o disco/backup seja roubado. Baseado em **keyring** + criptografia de tablespace:

```sql
-- 1) carregar um keyring (ex.: component_keyring_file; em producao, um KMS/HSM)
--    configurado via manifest do servidor
-- 2) criptografar uma tabela/tablespace
CREATE TABLE segredos (...) ENCRYPTION='Y';
ALTER TABLE clientes ENCRYPTION='Y';

-- 3) exigir criptografia por padrao
SET GLOBAL default_table_encryption = ON;
```

Também dá para criptografar **binlog** e **redo/undo log**:

```ini
[mysqld]
binlog_encryption      = ON
innodb_redo_log_encrypt = ON
innodb_undo_log_encrypt = ON
```

> A proteção em repouso só é efetiva se a **chave do keyring** ficar fora do servidor de dados (KMS/HSM). Chave no mesmo disco que os dados ≈ sem criptografia.

---

## Checklist

1. TLS **obrigatório** (`require_secure_transport = ON` e/ou `REQUIRE SSL` por conta).
2. Apps conectando com `--ssl-mode=VERIFY_CA`/`VERIFY_IDENTITY`.
3. `tls_version` só TLSv1.2/1.3.
4. Certificados de uma CA própria em produção (não os auto-gerados).
5. Avaliar encryption at rest (tablespace + binlog/redo) com keyring externo.

---

## Não incluído neste documento

- Configuração detalhada de SSL do Group Replication entre os nós.
- mTLS fim-a-fim com CA corporativa e rotação de certificados.
- Integração de keyring com KMS/HSM específicos.
