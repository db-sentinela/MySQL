# 02 — Adicionando uma Instância MySQL ao PMM (Single-Instance)

Adicionar um banco MySQL ao monitoramento exige duas coisas: uma conta de **menor privilégio** (`pmm`) para os exportadores e a correta instrumentação do **Performance Schema** para alimentar o **Query Analytics (QAN)**. Este é o procedimento operacional homologado para registrar um serviço MySQL no PMM 3 com segurança e telemetria de consultas.

---

## O que é

Para coletar métricas internas do InnoDB, variáveis de status e consultas, o PMM Client precisa de uma conta de acesso ao banco.
- **Nunca use `root`.** Rodar daemons de monitoramento como superusuário viola padrões de conformidade (PCI-DSS, ISO 27001).
- **Usuário dedicado `pmm`:** com privilégios estritamente necessários para ler metadados, `performance_schema` e status de replicação.
- **QAN via Performance Schema:** ler as consultas de `events_statements_summary_by_digest` evita o pesado *Slow Query Log* textual, consumindo apenas buffers de memória pré-alocados.

---

## Por que importa

1. **Segurança e contenção de conexões:** o `pmm` é criado com `MAX_USER_CONNECTIONS 10` — mesmo um loop no agente nunca consome o pool da aplicação.
2. **QAN de baixo impacto:** `--query-source=perfschema` captura taxa de queries, latência, percentis e planos em memória, com consumo de disco zero.
3. **Métricas por tabela:** o `mysqld_exporter` coleta tamanho, fragmentação e I/O por tabela, revelando candidatas a `OPTIMIZE TABLE` ou particionamento.

---

## Como reproduzir

### 1. Criar o usuário `pmm` de menor privilégio
Em um InnoDB Cluster, crie **no primário** (a DDL/DCL replica para os secundários via Group Replication). Pode conectar pela porta RW do Router (6446) para garantir que caiu no primário:

```sql
-- hosts: a rede privada (para --host=<ip do no>) e localhost
CREATE USER IF NOT EXISTS 'pmm'@'192.168.1.%' IDENTIFIED BY 'Pmm#2026' WITH MAX_USER_CONNECTIONS 10;
CREATE USER IF NOT EXISTS 'pmm'@'127.0.0.1'   IDENTIFIED BY 'Pmm#2026' WITH MAX_USER_CONNECTIONS 10;

-- privilegios minimos (MySQL): SELECT/PROCESS/REPLICATION CLIENT/RELOAD globais
GRANT SELECT, PROCESS, REPLICATION CLIENT, RELOAD ON *.* TO 'pmm'@'192.168.1.%';
GRANT SELECT, PROCESS, REPLICATION CLIENT, RELOAD ON *.* TO 'pmm'@'127.0.0.1';

-- leitura do Performance Schema (QAN e metricas de Group Replication)
GRANT SELECT ON performance_schema.* TO 'pmm'@'192.168.1.%';
GRANT SELECT ON performance_schema.* TO 'pmm'@'127.0.0.1';
```

> No MySQL, `REPLICATION CLIENT` já cobre as coordenadas de binlog — **não** existe `BINLOG MONITOR` (isso é do MariaDB). Não é preciso `SUPER`.

### 2. Confirmar os consumidores do Performance Schema
O QAN depende dos *digests*. Verifique que estão ativos (padrão no MySQL 9):

```sql
SELECT NAME, ENABLED FROM performance_schema.setup_consumers WHERE NAME LIKE '%statements%';
```
Esperado: `statements_digest = YES`, `events_statements_current = YES`, `events_statements_history = YES`.

### 3. Adicionar o serviço no PMM Client
No nó, registre a instância com o usuário `pmm`, o IP do nó e o QAN por perfschema:

```bash
sudo pmm-admin add mysql \
  --service-name=mysql-db-01 \
  --host=192.168.1.20 \
  --port=3306 \
  --username=pmm \
  --password='Pmm#2026' \
  --query-source=perfschema \
  --environment=lab
```

### 4. Validar
```bash
sudo pmm-admin list
```

---

## O que você vê (saída real)

### Adição do serviço (lab):
```text
MySQL Service added.
Service ID  : e56eb6ea-3657-4624-9f4d-cf270d4d6f7b
Service name: mysql-db-01

Table statistics collection enabled (the limit is 1000, the actual table count is 375).
```

### `pmm-admin list` com o serviço e os agentes:
```text
Service type   Service name    Address and port        Service ID
MySQL          mysql-db-01     192.168.1.20:3306       e56eb6ea-3657-4624-9f4d-cf270d4d6f7b

Agent type                 Status      Metrics Mode   Service ID                              Port
pmm_agent                  Connected
node_exporter              Running     push                                                   42000
mysqld_exporter            Running     push           e56eb6ea-...                            42004
mysql_perfschema_agent     Running                    e56eb6ea-...                            0
vmagent                    Running     push                                                   42003
```
*(O `mysqld_exporter` e o `mysql_perfschema_agent` do serviço `mysql-db-01` em **Running**.)*

### Métrica real já no PMM Server (VictoriaMetrics):
```json
mysql_global_status_uptime{
  cluster="innodb_cluster_lab", environment="lab",
  node_name="mysql-db-01", service_name="mysql-db-01",
  agent_type="mysqld_exporter"
}  →  value ok
```

---

## Lições

- **Limite de coleta de tabelas:** o PMM tem um teto de segurança (1.000 tabelas) para não saturar o `information_schema.TABLES`. Acima disso, desativa a coleta individual por tabela. No lab, 375 tabelas → coleta ativa.
- **`perfschema` × `slowlog`:** o `slowlog` exige o MySQL gravar log em disco e o agente ler o arquivo continuamente (concorre por I/O em picos de escrita). O `perfschema` usa buffers de memória, muito mais eficiente.
- **Em cluster, crie o `pmm` uma vez no primário.** A `CREATE USER`/`GRANT` replica para os secundários pelo Group Replication — não repita em cada nó.

---

## Pegadinhas

- **Serviço fantasma `mysql-01` em `127.0.0.1`.** Dependendo de um auto-registro anterior, pode existir um serviço duplicado (ex.: `mysql-01` apontando para `127.0.0.1:3306`) cujo `mysql_perfschema_agent` fica em **`Waiting`**. Ele polui os filtros do Grafana e gera ruído. Remova-o:
  ```bash
  sudo pmm-admin remove mysql mysql-01
  ```
  Padronize sempre `--service-name` explícito (`mysql-db-01`) para evitar que o PMM adote o hostname do SO e crie colisões.
- **Agente `mysql_perfschema_agent` em `Waiting`.** Verifique: (1) os consumidores de *digest* estão `YES`; (2) a senha do `pmm` no banco bate com a passada ao agente; (3) o host de conexão: se o serviço foi adicionado com `--host=127.0.0.1` mas o usuário `pmm` só existe para `192.168.1.%`, a conexão é recusada. Alinhe host do `add` com o host do `CREATE USER`.
- **Senha com `#`/`$`/`!` no shell.** No `--password`, use **aspas simples** (`--password='Pmm#2026'`). Sem elas, o shell trata `#` como comentário ou `$` como variável, corrompendo a senha enviada ao agente.
