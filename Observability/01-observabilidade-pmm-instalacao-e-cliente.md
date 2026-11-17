# 01 — Instalação e Arquitetura do PMM Client no MySQL

O **Percona Monitoring and Management (PMM 3)** é a plataforma open-source de observabilidade de bancos de dados da Percona. Ele unifica métricas de sistema operacional, telemetria do InnoDB/Group Replication e análise granular de consultas (**Query Analytics — QAN**) em um único painel Grafana. Este documento detalha a arquitetura do PMM Client, a instalação do agente no nó MySQL e o registro seguro no servidor central.

---

## O que é

O ecossistema do PMM 3 tem duas camadas:

### 1. PMM Server (Servidor Central)
Hospedado no nó de monitoramento (`192.168.1.16:8443`), consolida:
- **VictoriaMetrics:** TSDB de alta performance que armazena métricas de CPU, memória, I/O, status do MySQL e variáveis de replicação (Group Replication).
- **ClickHouse:** banco colunar que armazena os *digests* de consultas capturados pelo QAN.
- **Grafana (v11+):** interface visual com os dashboards oficiais da Percona.

### 2. PMM Client (Nós Gerenciados)
Instalado em cada nó MySQL (`mysql-db-01`, `db-02`, `db-03`), orquestrado pelo daemon `pmm-agent`. Gerencia os subagentes:
- **`vmagent`:** envia as métricas locais ao VictoriaMetrics central em modo *push* (HTTPS 8443).
- **`node_exporter`:** métricas de hardware/SO (CPU, memória, disco, rede) em porta local da faixa 42000+.
- **`mysqld_exporter`:** conecta ao MySQL com a conta de menor privilégio (`pmm`) e extrai `SHOW GLOBAL STATUS`, variáveis InnoDB e os contadores de **Group Replication** (`performance_schema.replication_group_*`).
- **`mysql_perfschema_agent`:** lê o Performance Schema e envia as consultas ao Query Analytics.

---

## Por que importa

1. **Centralização sem fragmentação:** uma única plataforma para telemetria de SO, engine e queries, em vez de ferramentas isoladas.
2. **Baixíssimo overhead:** o modelo *push* via `vmagent` mantém o consumo de CPU no nó de banco abaixo de 1%.
3. **Correlação de incidentes:** permite ligar um pico de latência da aplicação à causa — conexões no SO, saturação de disco ou uma query sem índice no MySQL.

---

## Como reproduzir

Execute no nó MySQL (ex.: `mysql-db-01`, 192.168.1.20):

### 1. Instalar o repositório Percona e o pacote PMM Client
Ubuntu 26.04 (`resolute`):

```bash
wget https://repo.percona.com/apt/percona-release_latest.generic_all.deb
sudo dpkg -i percona-release_latest.generic_all.deb
sudo percona-release setup pmm3-client

sudo apt update
sudo apt install -y pmm-client
```

### 2. Registrar o nó no PMM Server
Conecte o agente ao PMM Server (`192.168.1.16:8443`), aceitando o certificado TLS do laboratório:

```bash
sudo pmm-admin config \
  --server-insecure-tls \
  --server-url="https://admin:Admin%232026@192.168.1.16:8443" \
  192.168.1.20 generic mysql-db-01
```

> ⚠️ Repare no **`%23`** no lugar do `#` da senha (`Admin#2026`). Veja *Pegadinhas* — o `#` literal quebra a URL.

### 3. Verificar status e saúde dos agentes
```bash
sudo pmm-admin status
```

---

## O que você vê (saída real)

`pmm-admin status` no nó `mysql-db-01` (lab):

```text
Agent ID : ef417d54-b96c-4ca2-a61d-7807efa38d0a
Node ID  : 5e768003-a3a8-4c5f-bf35-1b5e849fe4b4
Node name: mysql-db-01

PMM Server:
	URL    : https://192.168.1.16:8443/
	Version: 3.10.0

PMM Client:
	Connected        : true
	Time drift       : -1.26467ms
	Latency          : 2.096689ms
	Connection uptime: 100
	pmm-admin version: 3.10.0
	pmm-agent version: 3.10.0
Agents:
	06811ef1-... node_exporter            Running   42000
	20ce6b8a-... mysql_perfschema_agent    Running   0
	7477c64e-... mysqld_exporter           Running   42004
	d0410446-... vmagent                   Running   42003
	75cc537b-... nomad_agent               Running   42002
```

**Pontos de atenção:**
- `Connected: true` — comunicação gRPC/HTTPS estabelecida.
- `Time drift: -1.26ms` — relógios alinhados (essencial para as séries temporais).
- Todos os agentes (`vmagent`, `node_exporter`, `mysqld_exporter`, `mysql_perfschema_agent`) em **Running**.

---

## Lições

- **O binário fica em `/usr/sbin`.** O `pmm-admin` é instalado em `/usr/sbin/pmm-admin`, que **não** está no `PATH` de shells não-interativos/SSH. Se `command -v pmm-admin` retornar vazio apesar do pacote instalado, chame pelo caminho completo (`sudo /usr/sbin/pmm-admin status`) ou confirme com `apt-cache policy pmm-client` (deve mostrar `Installed: 3.10.0-1.resolute`).
- **Sincronização de relógio (NTP):** o PMM depende de marcas de tempo alinhadas. *Time drift* alto gera anomalias gráficas e descarte de métricas. Garanta `systemd-timesyncd`/`chrony` ativo nos nós.
- **Diagnóstico rápido:** `sudo pmm-admin summary` empacota logs e configs mascaradas em um `.tar.gz`.

---

## Pegadinhas

- **O `#` na senha quebra a `--server-url`.** A senha `Admin#2026` passada literalmente faz o parser da URL tratar o `#` como *fragment*, abortando com `--server-url: parse "https://admin:Admin": invalid port ":Admin"`. **Solução:** URL-encode o `#` como **`%23`** → `https://admin:Admin%232026@...`. (O mesmo vale para `@`, `/`, `:` na senha.)
- **Certificado TLS autoassinado:** o container do PMM Server gera certificado autoassinado. Omitir `--server-insecure-tls` aborta com `x509: certificate signed by unknown authority`.
- **Nó já registrado:** se o nó já tem config de um `pmm-agent` anterior, o `config` responde `Node is already registered ... Use --force to register it again`. Isso é esperado ao reinstalar o cliente sobre uma config remanescente — a telemetria segue funcionando; use `--force` apenas se quiser recriar o registro do zero.
- **Porta 8443 no firewall:** se houver `ufw`/`iptables` entre os nós de banco e o PMM Server, o agente entra em loop de reconexão (`connection refused`/`context deadline exceeded`). Libere a 8443 (ou 443) na origem e no destino.
