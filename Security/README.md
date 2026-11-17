# DB Sentinel | MySQL — Security

Parte do **DB Sentinel Database Procedures Framework**: biblioteca técnica de procedimentos operacionais (SOPs) para administração de bancos de dados.

Esta subpasta cobre **segurança em MySQL**: hardening, privilégios, autenticação, criptografia e as vulnerabilidades clássicas — **simuladas de verdade em laboratório** para fins defensivos. Os procedimentos são independentes de clientes, não contêm dados de ambientes reais, e cada vulnerabilidade é mostrada **junto da mitigação**.

> ⚠️ Conteúdo com técnicas ofensivas usadas **apenas** em laboratório controlado, para ensinar a defender. Não aplique contra sistemas sem autorização.

## Por onde começar

1. **Fundamentos e hardening** — modelo de defesa em camadas e o checklist.
2. Depois, os controles específicos (privilégios, autenticação, TLS).
3. As **simulações** (privilege escalation, SQL injection) para entender o risco na prática.

## Procedimentos

| Procedimento | Tema | Status |
|---|---|---|
| [Fundamentos e hardening](security-fundamentos-hardening.md) | Defesa em camadas, checklist, audit de config | Validado |
| [Usuários e privilégios](security-usuarios-privilegios.md) | Least privilege, roles, auditoria de grants | Validado |
| [Autenticação e senhas](security-autenticacao-senhas.md) | Plugins, `validate_password`, expiração, bloqueio | Validado |
| [Privilege escalation](security-privilege-escalation.md) | Abuso de `FILE`, `secure_file_priv`, `local_infile` | Validado |
| [TLS e criptografia](security-tls-criptografia.md) | TLS em trânsito, encryption at rest | Validado |
| [SQL Injection](security-sql-injection.md) | Mecânica, prepared statements, least privilege | Validado |
| [Auditoria e logging](security-auditoria-logging.md) | Audit plugin, error log, detecção | Referência |

> **Status:** *Validado* = cenário/vulnerabilidade reproduzido e saída capturada em laboratório.

## O que foi simulado no lab

- **Audit de config:** TLS não obrigatório, sem `validate_password`, `root@localhost` **sem senha**, `clusteradmin@%` superprivilegiado — todos achados reais.
- **FILE privilege:** `INTO OUTFILE`/`LOAD_FILE` funcionando no diretório permitido e **bloqueados** fora dele (`/etc/passwd` → NULL) pelo `secure_file_priv`.
- **SQL injection:** `' OR '1'='1` vazou todas as contas na query concatenada; **0 linhas** com prepared statement.
- **TLS:** conexão em texto puro aceita; conta `REQUIRE SSL` **barrada** sem TLS (ERROR 2061) e OK com TLSv1.3.
- **Política de senha:** senha `123` aceita sem política; **rejeitada** (ERROR 1819) após instalar `validate_password`.

## Ferramentas e plugins que agregam

O **hardening do servidor** está nos docs acima. Para o **código das aplicações** que acessam o banco (onde vive o SQL injection):

| Ferramenta | Para quê |
|---|---|
| **`/security-review`** (nativo do Claude Code) | Audita o diff atual em busca de vulnerabilidades |
| **OWASP Security Audit** (plugin) | OWASP Top 10:2025, relatório com evidências e severidade |
| **Spec Security Review** (plugin) | SAST/SCA, autorização, supply-chain, *source-to-sink*, read-only |
| **Security Audit Loop by ATeam** (plugin) | Loop auditar→corrigir→verificar com scanners de segredos/deps/infra |

São a **camada de aplicação** do defense-in-depth — complementam, não substituem, o hardening do banco.

## Ambiente de homologação

| Item | Valor |
|---|---|
| MySQL Server | 9.7.4 |
| Autenticação | `caching_sha2_password` |
| TLS | disponível (TLSv1.2/1.3), não obrigatório por padrão |
| `secure_file_priv` | `/var/lib/mysql-files/` |
| `local_infile` | OFF |

Simulações executadas no primário do InnoDB Cluster do lab.

## Estrutura

```
Security/
├── README.md                              (este arquivo)
├── security-fundamentos-hardening.md
├── security-usuarios-privilegios.md
├── security-autenticacao-senhas.md
├── security-privilege-escalation.md
├── security-tls-criptografia.md
├── security-sql-injection.md
└── security-auditoria-logging.md
```

## Aviso

Procedimentos e simulações validados em **laboratório controlado**. As técnicas ofensivas servem para entender e **fechar** as brechas. Teste mudanças de segurança em homologação antes de produção — forçar TLS ou política de senha sem preparo pode trancar aplicações para fora.

## Mantenedor

DB Sentinel Consultoria em Tecnologia da Informacao Ltda.
