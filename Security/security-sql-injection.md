# MySQL Security — SQL Injection (visão do banco)

A vulnerabilidade de aplicação mais clássica, vista do lado do banco: como ela funciona, como neutralizar (prepared statements) e como o **least privilege** reduz o estrago. Com simulação real.

> Pré-leitura: `security-usuarios-privilegios.md`. A correção definitiva é no **código da aplicação**; o banco é a última linha de defesa.

---

## O que é

SQL injection acontece quando a aplicação **monta a query concatenando a entrada do usuário** como se fosse código. O atacante digita algo que "fecha" a string e injeta SQL próprio.

---

## Simulação real (no lab)

Tabela de exemplo `segdemo.usuarios` com duas contas. A app autentica por `WHERE login = '<entrada>'`. O atacante digita no campo de login:

```
' OR '1'='1
```

### (1) Código VULNERÁVEL — concatenação

A app monta: `SELECT login,senha FROM usuarios WHERE login = '' OR '1'='1'`. Saída **real**:

```
+-------+------------+
| login | senha      |
+-------+------------+
| admin | S3nh@Forte |
| joao  | 123        |
+-------+------------+
```

💥 **Vazou todas as contas e senhas.** O `' OR '1'='1'` fez o `WHERE` ser sempre verdadeiro. Uma consulta legítima (`login='admin'`) devolveria 1 linha; a injeção devolveu a tabela inteira.

### (2) Código SEGURO — prepared statement

A entrada vira **parâmetro** (`?`), tratada como **dado**, nunca como código:

```sql
PREPARE s FROM 'SELECT login,senha FROM usuarios WHERE login = ?';
SET @x = "' OR '1'='1";
EXECUTE s USING @x;
```

Saída real: **0 linhas**. O texto `' OR '1'='1` foi procurado **literalmente** como um login (que não existe). Injeção neutralizada.

---

## A defesa nº 1: prepared statements / parâmetros

Em **toda** linguagem, use consultas parametrizadas — nunca concatene entrada:

```python
# Python (errado) — NUNCA faca isso
cur.execute("SELECT * FROM usuarios WHERE login = '" + entrada + "'")

# Python (certo) — placeholder, a lib escapa
cur.execute("SELECT * FROM usuarios WHERE login = %s", (entrada,))
```
```java
// Java — PreparedStatement
PreparedStatement ps = con.prepareStatement("SELECT * FROM usuarios WHERE login = ?");
ps.setString(1, entrada);
```

ORMs (Hibernate, Django ORM, SQLAlchemy, Sequelize) parametrizam por padrão — o risco volta quando alguém usa *raw SQL* concatenado.

---

## Defesa em profundidade (o banco como última linha)

Mesmo com a app corrigida, o banco deve limitar o estrago de um eventual bug:

1. **Least privilege:** se a conta da app só tem `SELECT/INSERT/UPDATE` em um schema, uma injeção **não** apaga tabelas (`DROP`), não lê `mysql.user`, não usa `FILE`. Veja a diferença:
   - App com `ALL ON *.*` + `FILE` → injeção vira leitura de `/etc/passwd` e dump do servidor.
   - App com `SELECT,INSERT,UPDATE ON appdb.*` → injeção fica presa no schema da app.
2. **Sem `FILE`/`GRANT`/`SUPER`** na conta da app (ver privilege escalation).
3. **Views e stored procedures** com `SQL SECURITY DEFINER` podem expor só o necessário.
4. **WAF/validação de entrada** na borda ajuda, mas **não substitui** parâmetros — é camada extra, não a principal.

---

## Como caçar injeção no código

- Procure concatenação de string com entrada em queries: `"... " + var`, `f"...{var}..."`, `String.format` em SQL, `${var}` em templates de SQL.
- Use ferramentas de SAST (camada de aplicação):
  - **`/security-review`** (nativo do Claude Code) no diff da app.
  - Plugins **OWASP Security Audit** / **Spec Security Review** — rastreiam *source-to-sink* (da entrada do usuário até a query) e apontam arquivo/linha.

---

## Tipos de injeção (para reconhecer)

| Tipo | Como se manifesta |
|---|---|
| **In-band** (`UNION`/erro) | O resultado volta direto na resposta |
| **Blind booleana** | Sem resultado visível; o atacante infere por true/false |
| **Blind temporal** | Usa `SLEEP()` para inferir pela demora |
| **Out-of-band** | Exfiltra via DNS/HTTP (precisa de `FILE`/UDF — mais um motivo para least privilege) |

---

## Checklist

1. **Toda** query com entrada usa parâmetros/prepared statements.
2. Conta da app com **privilégio mínimo** (sem `FILE`, sem `*.*`, sem `DROP` se não precisar).
3. Revisão de código (SAST) no pipeline.
4. Erros de SQL **não** vazam para o usuário (mensagens genéricas).
5. Logs de erro/consulta monitorados para padrões de injeção.

---

## Não incluído neste documento

- Exploração ofensiva detalhada (UNION-based, extração via `information_schema`).
- Configuração de WAF.
- Revisão automatizada contínua no CI — ver os plugins de auditoria.
