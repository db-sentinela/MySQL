# MySQL InnoDB Cluster — Backup e Restore

Backup lógico e restore em um InnoDB Cluster, com a pegadinha de GTID que derruba quem não presta atenção. Inclui um backup e um restore **executados no lab**.

> Pré-leitura: `innodb-cluster-fundamentos-alta-disponibilidade.md` (GTID).

---

## HA **não** é backup

O cluster replica tudo — inclusive os seus erros. Um `DROP TABLE` errado, um `DELETE` sem `WHERE` ou uma corrupção lógica são copiados para os 3 nós **na hora**. A replicação protege contra **falha de hardware**, não contra **engano humano** nem contra "quero o estado de ontem". Para isso: **backup**.

| Protege contra... | HA (replicação) | Backup |
|---|---|---|
| Nó/host que morre | ✅ | ✅ |
| `DELETE`/`DROP` errado | ❌ (replica o erro) | ✅ |
| Voltar no tempo (point-in-time) | ❌ | ✅ |
| Corrupção lógica | ❌ | ✅ |

---

## Tipos de backup

| Tipo | Ferramenta | Quando |
|---|---|---|
| **Lógico** | `mysqldump`, MySQL Shell `util.dumpInstance()` | Portável, seletivo (uma tabela), bom para bases pequenas/médias |
| **Físico** | Clone, Percona XtraBackup, MySQL Enterprise Backup | Rápido para bases grandes; restore de instância inteira |

**Regra de ouro no cluster:** rode o backup **em um SECUNDÁRIO**, não no primário — assim você não rouba recurso de quem atende a escrita da aplicação.

---

## Backup lógico com `mysqldump` (executado no lab)

Rodando **no secundário `mysql-db-02`**, backup da tabela `halab.probe`:

```bash
mysqldump --single-transaction --no-tablespaces --set-gtid-purged=OFF \
  halab probe > /tmp/halab_probe.sql
```

Saída real (conteúdo do arquivo):

```
arquivo: 2208 bytes -> /tmp/halab_probe.sql
CREATE TABLE `probe` ( ... )
INSERT INTO `probe` VALUES (1,'mysql-db-01','baseline',...),(2,...),(3,...),(4,...);
```

O que cada opção faz, e **por que importam no cluster**:

- `--single-transaction`: backup consistente **sem travar** as tabelas (usa uma transação com snapshot). Essencial para não impactar a replicação. (Vale para tabelas InnoDB, que é o nosso caso.)
- `--set-gtid-purged=OFF`: **a pegadinha**. Por padrão o `mysqldump` tenta embutir o estado de GTID (`SET @@GLOBAL.gtid_purged=...`) no dump. Se você restaurar esse dump **dentro do mesmo cluster vivo**, isso **corrompe o GTID** do destino e quebra a replicação. Para restore no mesmo cluster, use `OFF`. (Use `ON`/`AUTO` só quando o dump serve para **semear um nó/cluster novo e vazio**.)
- `--no-tablespaces`: evita um warning/privilégio desnecessário em dump lógico simples.

> Base inteira em vez de uma tabela: troque `halab probe` por `--all-databases` (ou `--databases db1 db2`). Lembre do `--set-gtid-purged` conforme o destino.

---

## Restore (executado no lab)

Teste de restore **em um schema separado** (para validar sem tocar no original), via porta de escrita do Router:

```bash
mysql -h mysql-db-01 -P 6446 -u <user> -p -e "CREATE DATABASE halab_restore;"
mysql -h mysql-db-01 -P 6446 -u <user> -p halab_restore < /tmp/halab_probe.sql
```

Validação real (original × restaurado):

```
+------------+--------+
| fonte      | linhas |
+------------+--------+
| original   |      4 |
| restaurado |      4 |
+------------+--------+
```

A contagem bateu: restore íntegro. Repare que o restore foi feito **pelo Router (6446)**, ou seja, escreveu no PRIMARY e **replicou** para os secundários — não se restaura direto em um secundário (`super_read_only` bloqueia).

> Restaurar por cima de dados existentes é destrutivo. Em produção, **restaure primeiro em um schema/instância de teste** e só promova depois de conferir.

---

## Cenários de restore

| Preciso... | Caminho |
|---|---|
| Recuperar **uma tabela** apagada | Restore lógico da tabela em schema de teste → copia as linhas de volta (foi o que fizemos) |
| Reconstruir **um nó** | Mais simples usar **clone** (ver `innodb-cluster-substituicao-no-clone.md`), não dump |
| Recriar o **cluster do zero** | Restaurar dump `--all-databases` (com `--set-gtid-purged=ON`) em um nó vazio e recriar o cluster |
| Voltar a um **ponto no tempo** | Backup full + aplicar binlogs até o instante desejado (point-in-time recovery) |

---

## MySQL Shell: backup paralelo (alternativa moderna)

Para bases maiores, o utilitário do Shell é bem mais rápido (paralelo, comprimido):

```bash
# Backup da instancia inteira
mysqlsh --uri clusteradmin@mysql-db-02:3306 --js \
  --execute="util.dumpInstance('/backup/full-$(date +%F)', {threads:4})"

# Restore (em um destino adequado; cuidado com GTID/ambiente)
mysqlsh --uri clusteradmin@mysql-db-01:3306 --js \
  --execute="util.loadDump('/backup/full-2026-11-10')"
```

> `util.loadDump` tem opções específicas de GTID (`updateGtidSet`, `skipBinlog`). Leia a doc antes de usar em cluster vivo — a mesma pegadinha de GTID se aplica.

---

## Agendamento (exemplo)

```bash
# /etc/cron.d/mysql-backup  — roda no secundario, 02:30 todo dia
30 2 * * * dba  mysqldump --single-transaction --no-tablespaces --set-gtid-purged=OFF \
  --all-databases | gzip > /backup/full-$(date +\%F).sql.gz
```

Guarde os backups **fora dos hosts do cluster** (outro servidor/armazenamento). Backup no mesmo disco que pode falhar junto não é backup.

---

## Checklist

1. Backup rodou **em um secundário**, com `--single-transaction`.
2. `--set-gtid-purged` correto para o destino (`OFF` = mesmo cluster; `ON` = nó/cluster novo).
3. Restore testado **em schema/instância separada** e **conferido** (contagem/consistência).
4. Restore feito **pelo PRIMARY** (via Router 6446), nunca direto num secundário.
5. Cópias guardadas **fora** dos hosts do cluster.

---

## Não incluído neste documento

- Point-in-time recovery detalhado (aplicação de binlogs).
- Percona XtraBackup / MySQL Enterprise Backup (backup físico a quente).
- Política de retenção, verificação automática de restore e criptografia dos backups.
