# MySQL Security — Auditoria e logging

Sem registro, um ataque não deixa rastro — e você não sabe o que foi acessado. Como registrar quem conecta, o que faz, e detectar abuso. Config real do lab.

> Pré-leitura: `security-fundamentos-hardening.md`.

---

## O que registrar (e onde)

| Fonte | Registra | Estado no lab |
|---|---|---|
| **Error log** | Erros, avisos, falhas de conexão, mudanças de GR | `/var/log/mysql/error.log` ✅ |
| **General query log** | **Toda** query de todos (muito verboso) | `OFF` (ligar só pontual) |
| **Slow query log** | Queries lentas | ver subpasta Troubleshooting |
| **Binary log** | Toda alteração (para replicação/PITR) | `ON` ✅ |
| **Audit plugin** | Conexões, queries, por usuário/objeto, filtrável | não instalado |

A peça que falta para **auditoria de segurança de verdade** é um **audit plugin**.

---

## Audit plugin (o que você quer em produção)

O MySQL Community **não** traz um audit plugin pronto; as opções:

- **Percona Audit Log** (grátis, open source) — recomendado para MySQL Community.
- **MariaDB Audit Plugin** (também funciona em MySQL).
- **MySQL Enterprise Audit** (apenas Enterprise).

Exemplo com o Percona Audit (ideia geral):

```ini
[mysqld]
plugin-load-add = audit_log.so
audit_log_format = JSON
audit_log_file   = /var/log/mysql/audit.log
audit_log_policy = ALL            # ou LOGINS, QUERIES
```

Ele registra, por evento: **quem** (usuário/host), **quando**, **o quê** (query/conexão), **resultado**. É isso que responde "quem acessou a tabela de clientes ontem às 3h?".

### Filtrar para não afogar em log

Audite o que importa: logins (sucesso e **falha**), comandos administrativos (`CREATE USER`, `GRANT`, `DROP`), acesso a tabelas sensíveis. Logar *toda* query gera volume e custa performance.

---

## Sem audit plugin: o que dá para fazer nativo

### Falhas de conexão e hosts bloqueados

```sql
-- erros de conexao/autenticacao por host (inclui brute force)
SELECT IP, HOST, COUNT_AUTHENTICATION_ERRORS, COUNT_HANDSHAKE_ERRORS, COUNT_HOST_BLOCKED_ERRORS
FROM performance_schema.host_cache
WHERE COUNT_AUTHENTICATION_ERRORS > 0;
```

Muitos `COUNT_AUTHENTICATION_ERRORS` de um IP = tentativa de brute force. O MySQL bloqueia um host após `max_connect_errors` falhas — ajuste e monitore.

### Error log com mais detalhe

```sql
SET GLOBAL log_error_verbosity = 3;   -- 1=erros, 2=+avisos (lab), 3=+notas
```
No error log aparecem `Access denied`, bloqueios de host e mudanças de replicação.

### General log pontual (investigação)

```sql
SET GLOBAL general_log = ON;     -- só durante a investigação
-- ... reproduzir/observar ...
SET GLOBAL general_log = OFF;    -- desligar (é muito verboso e pesa)
```

### Quem está conectado agora

```sql
SELECT user, host, db, command, time, state FROM information_schema.processlist;
SELECT * FROM performance_schema.accounts;     -- conexoes por conta
```

---

## O que monitorar / alertar

| Sinal | Significado |
|---|---|
| Pico de `COUNT_AUTHENTICATION_ERRORS` | Brute force |
| `CREATE USER`/`GRANT`/`DROP USER` fora de janela | Possível escalonamento |
| Acesso a `mysql.user`/`information_schema` por conta de app | App comprometida (recon) |
| `LOAD_FILE`/`INTO OUTFILE` | Tentativa de ler/escrever arquivos |
| Conexões de IP inesperado | Burlar firewall |
| `SET GLOBAL` / mudança de variável | Desligar auditoria, baixar proteção |

Envie os logs para **fora do servidor** (syslog/SIEM) — um atacante com acesso ao host apaga o log local. Logs centralizados são à prova de adulteração local.

---

## Integridade e retenção

- **Logs fora do host** (SIEM, syslog remoto, bucket) — resistem a quem comprometeu o servidor.
- **Rotação** (`logrotate`) e **retenção** conforme a política/compliance.
- **Proteja os próprios logs**: só `mysql`/root com escrita; ninguém da app lê o audit.

---

## Checklist

1. Audit plugin (Percona/MariaDB) instalado, logando logins e DDL/DCL.
2. Error log em `verbosity >= 2`, monitorado.
3. `host_cache` vigiado para brute force.
4. Logs **centralizados** fora do host, com rotação e retenção.
5. General log **desligado** (ligar só para investigação).
6. Alertas para os sinais da tabela acima.

---

## Não incluído neste documento

- Instalação passo a passo do Percona Audit Log.
- Integração com SIEM específico (Splunk, ELK, Wazuh).
- Correlação de eventos e regras de detecção avançadas.
