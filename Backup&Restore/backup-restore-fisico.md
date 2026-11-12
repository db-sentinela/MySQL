# MySQL Backup & Restore — Backup físico

Backup físico = cópia dos **arquivos** do InnoDB (datadir). É o caminho para bases grandes: backup e, principalmente, **restore muito mais rápidos** que o lógico. Cobre três técnicas, duas delas **executadas e validadas no lab**.

> Pré-leitura: `backup-restore-fundamentos.md` (lógico × físico).

---

## As três técnicas

| Técnica | A quente? | Nativo? | Quando usar |
|---|---|---|---|
| **CLONE LOCAL** | Sim | Sim (plugin do MySQL) | Snapshot físico rápido, sem instalar nada |
| **Percona XtraBackup** | Sim | Não (ferramenta externa) | Padrão de produção; incremental, streaming |
| **Cópia a frio** | **Não** (exige parar o MySQL) | Sim | Simples, quando downtime é aceitável |

> ⚠️ **Nunca** copie o datadir de um MySQL **ligado** com um `cp`/`tar` simples — os arquivos estão sendo escritos e o backup sai inconsistente. Para backup a quente, use CLONE ou XtraBackup. Cópia simples só com o MySQL **parado** (a frio).

---

## 1. CLONE LOCAL (nativo, a quente) — validado

O plugin **clone** (ACTIVE no lab) tira um snapshot físico consistente do datadir, com o MySQL no ar:

```sql
CLONE LOCAL DATA DIRECTORY = '/caminho/backup';
```

Acompanhe o resultado:

```sql
SELECT STATE, ERROR_NO, ERROR_MESSAGE FROM performance_schema.clone_status;
```

Execução real no lab:

```
STATE = Completed   ERROR_NO = 0
```

O diretório gerado é um **datadir completo** (136 MB), pronto para ser iniciado:

```
#clone  #innodb_redo  ibdata1  mysql.ibd  <schemas...>  (appdb, bkp, tshoot, ...)
```

### Restore validado: subir o backup como instância

Provamos que o backup funciona iniciando um `mysqld` **descartável** apontando para ele:

```bash
mysqld --no-defaults --user=mysql --datadir=/caminho/backup --port=3308 \
  --socket=/caminho/backup/m.sock --mysqlx=0 --daemonize \
  --loose-group_replication_start_on_boot=OFF
```

Conectando na porta 3308, os dados estavam **todos lá** (saída real):

```
porta = 3308   datadir = /tmp/clone1/
tshoot.pedidos = 500000
bkp.clientes   = 5
```

Ou seja: do backup físico a uma instância viva servindo os dados. Para **restaurar de verdade** um nó, você substitui o `/var/lib/mysql` dele por este datadir (com o MySQL parado) e sobe.

### Pegadinha real: permissão do destino

O primeiro CLONE para `/backups` **falhou**:

```
Can't create database '/backups/clone1/' (errno: 13 - Permission denied)
```

Causa: no Ubuntu, o **AppArmor** confina o `mysqld` a `/var/lib/mysql` (e a `/tmp`). O processo não conseguiu escrever em `/backups`. Soluções: clonar para um caminho permitido (ex.: `/tmp`, como fizemos), ou ajustar o perfil do AppArmor (`/etc/apparmor.d/usr.sbin.mysqld`) para liberar o diretório de backup. **O destino também precisa pertencer ao usuário `mysql`.**

> CLONE também existe na forma **remota** (`CLONE INSTANCE FROM ...`), que é como o InnoDB Cluster provisiona um nó novo — ver `InnoDB-Cluster/innodb-cluster-substituicao-no-clone.md`.

---

## 2. Percona XtraBackup (padrão de produção, a quente)

Ferramenta externa, a referência para backup físico de MySQL. No lab está a versão **9.7.1**, compatível com o server 9.7.

### Backup

```bash
xtrabackup --backup --target-dir=/backup/xb \
  --host=127.0.0.1 --port=3306 --user=clusteradmin --password='***'
```

Execução real (final do log):

```
[Xtrabackup] Backup size: 133.49 MiB (139970445 bytes)
[Xtrabackup] Transaction log of lsn (133792761) to (133801274) was copied.
[Xtrabackup] completed OK!
```

A mensagem **`completed OK!`** é o que você confere sempre.

### Prepare (obrigatório antes de restaurar)

O backup recém-tirado está "cru" (com transações em andamento no momento da cópia). O `--prepare` aplica o redo e deixa o datadir **consistente e restaurável**:

```bash
xtrabackup --prepare --target-dir=/backup/xb
```

Execução real (final do log): também termina com **`completed OK!`** — a partir daí o backup está pronto para restaurar.

### Restore (copy-back)

Com o MySQL **parado** e o datadir vazio:

```bash
sudo systemctl stop mysql
sudo rm -rf /var/lib/mysql/*                       # cuidado!
sudo xtrabackup --copy-back --target-dir=/backup/xb --datadir=/var/lib/mysql
sudo chown -R mysql:mysql /var/lib/mysql
sudo systemctl start mysql
```

### Por que XtraBackup em produção

- **Incremental:** `--incremental-basedir` copia só o que mudou desde o último backup (janela e espaço menores).
- **Streaming/compressão:** `--stream=xbstream | gzip`, backup direto para outro host.
- **Coordenadas prontas:** gera `xtrabackup_binlog_info` (arquivo, posição e **GTID**) — essencial para PITR e para montar réplica.

---

## 3. Cópia a frio (cold backup)

A mais simples, mas exige **parar o MySQL** (downtime). Faça em um secundário:

```bash
sudo systemctl stop mysql
sudo tar czf /backup/datadir-$(date +%F).tar.gz -C /var/lib/mysql .
sudo systemctl start mysql
```

Restore: pare o MySQL, limpe o datadir, extraia o tar, ajuste dono (`chown -R mysql:mysql`), suba. Consistente porque o MySQL estava parado — e por isso não serve para ambientes 24x7 sem janela.

---

## Comparando os três

| | CLONE LOCAL | XtraBackup | A frio |
|---|---|---|---|
| Downtime | Não | Não | **Sim** |
| Instala algo | Não | Sim | Não |
| Incremental | Não | **Sim** | Não |
| Coordenada p/ PITR | binlog via clone_status | `xtrabackup_binlog_info` | manual |
| Esforço | Baixo | Médio | Baixo |

---

## Boas práticas

- Rode em um **secundário**.
- **Prepare/valide** o backup antes de confiar (XtraBackup: `--prepare`; CLONE: subir uma instância de teste, como fizemos).
- Guarde a **coordenada de binlog/GTID** junto do backup — é a ponte para o **PITR**.
- Backup físico casa com PITR: full físico + binlogs = restore rápido até um instante exato.

---

## Comandos úteis

```sql
-- CLONE nativo
CLONE LOCAL DATA DIRECTORY = '/tmp/clone1';
SELECT STATE, ERROR_NO, ERROR_MESSAGE FROM performance_schema.clone_status;
```
```bash
# XtraBackup
xtrabackup --backup  --target-dir=/backup/xb --user=... --password=...
xtrabackup --prepare --target-dir=/backup/xb
xtrabackup --copy-back --target-dir=/backup/xb --datadir=/var/lib/mysql
```

---

## Não incluído neste documento

- Point-in-time recovery (full físico + binlogs) — `backup-restore-pitr.md`.
- Backup incremental detalhado do XtraBackup (cadeia de incrementais).
- Ajuste do perfil AppArmor para diretórios de backup.
