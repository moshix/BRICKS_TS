# SQL in BRICKS

This guide covers everything needed to learn, understand, and
operate SQL in BRICKS: how it is configured, how the CEDA
DATABASE screen manages the database catalogue, the full
`EXEC SQL` surface available to COBOL and REXX programs, error
handling, the bundled sample transactions, and how to monitor SQL
activity at runtime.

It is a companion to [`PROGRAMMING.md`](PROGRAMMING.md) Chapter 26
(the language-reference view) and [`README.md`](README.md) (the
operator's `bricks.cnf` reference). Where they overlap, this guide
is the tutorial; they are the reference.

---

## Contents

1. [What SQL support is](#1-what-sql-support-is)
2. [Operator setup](#2-operator-setup)
3. [CEDA DATABASE — managing the catalogue](#3-ceda-database--managing-the-catalogue)
4. [The EXEC SQL surface](#4-the-exec-sql-surface)
5. [Host variables](#5-host-variables)
6. [Statement reference](#6-statement-reference)
7. [Null indicators](#7-null-indicators)
8. [WHENEVER — declarative error handling](#8-whenever--declarative-error-handling)
9. [The SQLCA and the SQLCODE catalogue](#9-the-sqlca-and-the-sqlcode-catalogue)
10. [Column-value coercion](#10-column-value-coercion)
11. [The sample transactions — SQLD and SQLR](#11-the-sample-transactions--sqld-and-sqlr)
12. [Monitoring SQL at runtime](#12-monitoring-sql-at-runtime)
13. [Restrictions](#13-restrictions)
14. [Quick reference](#14-quick-reference)

---

## 1. What SQL support is

BRICKS programs — COBOL and REXX alike — can read and write a
PostgreSQL database with embedded `EXEC SQL` statements, the same
way a mainframe CICS program talks to DB2. bricks is not itself a
database; it connects, as a client, to a Postgres server you run.

By having a central Postgres, multiple BRICKS regions can participate
in a BRICKS MULTI-REGION cluster for added high availability and 
thruput. 

The components:

* **The Postgres server.** One server, described by the `db_*`
  block in `bricks.cnf` (host, port, credentials).
* **The database catalogue.** `runtime/databases.conf` lists the
  databases on that server that bricks opens connection pools to.
  CEDA DATABASE manages this file.
* **The SQL executor.** A per-task component inside bricks that
  receives each `EXEC SQL` statement, binds host variables, runs
  it against Postgres, and writes the outcome back into the
  program's SQL communications area (SQLCA).
* **The program.** Your COBOL or REXX transaction, which issues
  `EXEC SQL … END-EXEC` blocks and inspects `SQLCODE` afterward.

One BRICKS  *task* (one transaction run) owns one Postgres
*transaction*. Statements accumulate until the program issues
`EXEC SQL COMMIT` / `EXEC SQL ROLLBACK`, or the task ends (an
implicit rollback of anything uncommitted).

```
  COBOL / REXX program
        │  EXEC SQL …
        ▼
  bricks SQL executor ──── connection pool ────► PostgreSQL
        │                                          (your server)
        ▼
  SQLCA written back into the program's variables
  (SQLCODE, SQLSTATE, SQLERRMC, SQLERRD…, SQLWARN…)
```

When `bricks.cnf` has no `db_host`, bricks runs perfectly well
**SQL-less**: programs still load and run, and the first
`EXEC SQL` they reach returns `SQLCODE = -1` ("SQL not
configured") so the program can degrade gracefully.

---

## 2. Operator setup

### 2.1 `bricks.cnf` — the server

The `db_*` keys describe the **Postgres server**. All are
optional; an empty `db_host` disables SQL entirely.

The Postgres database can be on the same machine as BRICKS or
on the other side of the planet. It doens't matter to BRICKS. 

```
db_host=localhost
db_port=5432
db_user=bricks
db_password=secret
db_sslmode=disable
db_max_conns=8
db_stmt_timeout=30s
db_retry_transient=yes
db_retry_max=1
databases_file=runtime/databases.conf
```

| Key | Default | Meaning |
|---|---|---|
| `db_host` | (none) | Postgres hostname. Empty → SQL not configured. |
| `db_port` | `5432` | Postgres TCP port. |
| `db_user` | (none) | Login role. |
| `db_password` | (none) | Password (URL-escaped into the DSN). |
| `db_sslmode` | `disable` | libpq sslmode (`disable`, `require`, `verify-full`, …). |
| `db_max_conns` | `8` | Per-database connection-pool cap. |
| `db_stmt_timeout` | `30s` | Per-statement **wall-clock** cap — see §2.5. |
| `db_retry_transient` | `yes` | Auto-retry deadlock/serialization failures — see §2.6. |
| `db_retry_max` | `1` | Max re-attempts on a transient error. |
| `databases_file` | `runtime/databases.conf` | The database catalogue. |

### 2.2 `databases.conf` — the catalogue

One Postgres database per line, `name[:description]`. The **first
row is the default** — transactions that don't bind to a specific
database use it.

```
# bricks databases catalogue. One PG database per line.
bricks:default application data
orders:order-management system
ledger:general ledger
```

Adding a row here only tells BRICKS  the database *exists*; it does
not create it in Postgres. Use CEDA DATABASE `C` (§3) — or `psql`
— for that.

### 2.3 Binding a transaction to a database

`transactions.conf` rows have an optional 5th colon-separated
field naming the database:

```
SQLD:cobol:sqld.cob:public,users,admin            # default database
ORDQ:cobol:ordq.cob:public:orders                 # the 'orders' database
LEDQ:cobol:ledq.cob:admin,users:ledger            # the 'ledger' database
```

Field layout: `TRANID:type:program[:groups[:database]]`. An empty
or absent 5th field binds the transaction to the default
database. A program can also switch databases mid-task with
`EXEC SQL CONNECT TO` (§6.7).

### 2.4 Creating the database and tables

BRICKS reads and writes tables; it does not invent your schema.
The bundled `sql_bricks_statements.sql` creates the demo
`customers_sql` table the sample transactions need:

```
psql -h localhost -p 5432 -U bricks -d bricks -f sql_bricks_statements.sql
```

It connects to the **`bricks` database** (the default catalogue
row) and creates the **`customers_sql` table** inside it, seeded
with three rows (`K001`/Alice, `K002`/Bob, `K003`/Carol).

> On PostgreSQL 15 and later the `public` schema no longer grants
> `CREATE` to every user. If the script fails with
> `permission denied for schema public`, run this once as a
> superuser: `GRANT ALL ON SCHEMA public TO bricks;`

### 2.5 Statement timeout — `db_stmt_timeout`

`db_stmt_timeout` caps how long a single statement may run,
measured as **wall-clock time from the bricks client side**: it
bounds the whole round trip — sending the SQL, the server's
execution, and receiving the result — not just the server's CPU
time. (Postgres's own `statement_timeout` is server-side only and
would miss a network-bound stall; bricks layers both, so either
firing produces SQLSTATE `57014` → `SQLCODE -952` /
`SQL-TIMEOUT`.)

Accepts a Go duration (`5s`, `100ms`, `1m30s`), a bare integer
(seconds), or `0` / `off` / `none` to disable. Cursors are
exempt — only single-shot statements get the per-statement
deadline; `FETCH` iteration is bounded only by the server-side
timer.

### 2.6 Transient-error retry — `db_retry_transient`

A serialization failure or deadlock (`SQLCODE -911`) is
transient: Postgres has already rolled the whole transaction
back, and re-running often succeeds. With `db_retry_transient=yes`
bricks automatically retries — but **only on the first data
statement of a task's transaction**, because retrying a later
statement after a full rollback would silently discard the
earlier committed-intent work. `db_retry_max` bounds the
re-attempts (default 1); a small backoff sits between tries. A
program can always do its own retry from `WHEN SQL-DEADLOCK`
(§8, §9).

---

## 3. CEDA DATABASE — managing the catalogue

`CEDA` is the bricks resource-definition transaction. Its
`DATABASE` screen (`CEDA` → `D`) is where an operator views and
manages the Postgres database catalogue without restarting
bricks. It is admin-gated and every mutation is audit-logged.

The screen lists every row of `databases.conf` with its live
connection state:

```
DATABASE       DESCRIPTION                       STATE
─────────────  ────────────────────────────────  ────────
bricks (def)   default application data          ONLINE
orders         order-management system           ONLINE
ledger         general ledger                    OFFLINE
```

`STATE` is `ONLINE` (pool connected), `OFFLINE` (configured but
the last connection attempt failed), or `NEVER` (never pinged).

### 3.1 Row actions

Type a one-letter action in the selector column beside a row,
then ENTER:

| Cmd | Action |
|---|---|
| `A` | **Add** a catalogue row — a form prompts for name + description, then `databases.conf` is rewritten. The row exists in bricks; the PG-side database may not yet. |
| `C` | **Create** the PG-side database for the selected row. bricks runs `CREATE DATABASE <name>` over its maintenance connection, then re-pings so the row flips to `ONLINE`. |
| `D` | **Delete** the catalogue row (refuses the default row). Removes the line from `databases.conf` and closes the pool. Does **not** drop the PG database. |
| `X` | **Drop** the PG-side database. A confirmation form makes you re-type the database name; the bricks pool is closed first so Postgres doesn't refuse with "other users connected". |
| `U` | **Alter** the row's description. |
| `R` | **Retest** the row's connection (re-ping). |
| `L` | **List** the user-schema tables in that database (one-line summary). |

### 3.2 Lifecycle flows

* **Add a brand-new database:** `A` (catalogue row) → `C`
  (create it in Postgres) → `R` (confirm `ONLINE`).
* **Decommission a database:** `X` (drop it in Postgres,
  re-typing the name to confirm) → `D` (remove the catalogue
  row).
* **Point at an existing database:** `A` only — if the Postgres
  database already exists, the pool comes up `ONLINE` immediately.

### 3.3 Audit and safety

Every CEDA DATABASE mutation flows through the bricks audit log:

```
ceda=DATABASE op=PG-CREATE target=orders status=OK term=T01 user=admin
ceda=DATABASE op=PG-DROP   target=ledger status=OK term=T01 user=admin
ceda=DATABASE op=ADD       target=orders            term=T01 user=admin
```

`databases.conf` is rewritten atomically (write-temp-then-rename),
so a crash mid-mutation cannot corrupt the catalogue. The `X`
(drop) action is the only irreversible one — hence the
re-type-the-name confirmation gate.

### 3.4 Why programs cannot CREATE / DROP databases

`EXEC SQL CREATE DATABASE` / `DROP DATABASE` (and `CREATE`/`DROP`
of `USER` / `ROLE` / `TABLESPACE`) from a program are **rejected**
with `SQLCODE -2` (`SQL-DDLREJECTED`). Database lifecycle is an
operator responsibility, performed through CEDA DATABASE where it
is gated and audited. `CREATE TABLE` / `CREATE INDEX` and other
in-database DDL are *not* rejected — programs may run those.

---

## 4. The EXEC SQL surface

### 4.1 Statement form

A SQL statement is wrapped in an `EXEC SQL … END-EXEC` block.

COBOL — terminated with a period like any COBOL statement:

```cobol
EXEC SQL
    SELECT name INTO :NM
    FROM customers_sql
    WHERE id = :CUSTID
END-EXEC.
```

REXX — no terminator; the block may span several lines:

```rexx
EXEC SQL
    SELECT name INTO :CUSTNM
    FROM customers_sql
    WHERE id = :CUSTID
END-EXEC
```

Both languages drive the *same* executor. Anything this guide
says about a statement applies identically to COBOL and REXX
unless explicitly noted.

### 4.2 What runs where

The SQL between `EXEC SQL` and `END-EXEC` is sent to PostgreSQL
almost verbatim — bricks rewrites only two things:

* `:host-variable` references become positional `$N` parameters
  (the values are bound separately — this is what makes bricks
  immune to SQL injection through host variables).
* The `INTO` clause of a `SELECT INTO`, and any null-indicator
  tokens, are stripped before the statement reaches Postgres
  (Postgres does not understand them — they are a bricks/DB2
  host-language construct).

Everything else — the `WHERE`, the joins, the functions, the
`ORDER BY` — is plain PostgreSQL SQL. Use the PostgreSQL manual
for the SQL dialect itself; this guide covers only the bricks
embedding.

---

## 5. Host variables

A **host variable** is a program variable referenced inside SQL
with a leading colon: `:CUSTID`, `:NM`, `:BALANCE`.

**COBOL** — host variables are ordinary `WORKING-STORAGE` items:

```cobol
01 CUSTID PIC X(10).
01 NM     PIC X(30).
...
EXEC SQL SELECT name INTO :NM
         FROM customers_sql WHERE id = :CUSTID END-EXEC.
```

**REXX** — host variables are ordinary REXX variables; REXX is
dynamically typed, so there is nothing to declare:

```rexx
CUSTID = 'K001'
EXEC SQL SELECT name INTO :CUSTNM
         FROM customers_sql WHERE id = :CUSTID END-EXEC
```

Two directions:

* **Input** — a host variable in a `WHERE`, `VALUES`, or `SET`
  clause supplies a value *to* the statement. Its current value
  is read from the program and bound.
* **Output** — a host variable in an `INTO` clause (of
  `SELECT INTO` or `FETCH`) receives a column value *from* the
  result row.

> **REXX naming caution.** In REXX a compound reference `STEM.NM`
> resolves the tail through the *value* of the simple symbol
> `NM`. If you use `NM` as both a SQL host variable and a stem
> tail, after `EXEC SQL … INTO :NM` sets `NM = 'Alice'` the line
> `SCR.NM = NM` resolves to `SCR.Alice`. Pick host-variable names
> that are not also stem tails — the bundled SQLR uses `CUSTNM`,
> not `NM`, for exactly this reason.

---

## 6. Statement reference

### 6.1 SELECT INTO — single-row read

```cobol
EXEC SQL SELECT name, balance INTO :NM, :BAL
         FROM accounts WHERE id = :ACCT END-EXEC.
```

Reads exactly one row. Outcomes:

| SQLCODE | Meaning |
|---|---|
| `0` | One row found; the `INTO` variables hold the column values. |
| `+100` | No row matched. The `INTO` variables are cleared. |
| `-811` | More than one row matched — `SELECT INTO` requires a unique result. Add a tighter `WHERE` or use a cursor. |

After a successful `SELECT INTO`, `SQLERRD3` is `1` (one row).

### 6.2 INSERT / UPDATE / DELETE

```cobol
EXEC SQL INSERT INTO accounts (id, name, balance)
         VALUES (:ACCT, :NM, :BAL) END-EXEC.

EXEC SQL UPDATE accounts SET balance = :BAL
         WHERE id = :ACCT END-EXEC.

EXEC SQL DELETE FROM accounts WHERE id = :ACCT END-EXEC.
```

On success `SQLCODE` is `0` and **`SQLERRD3` holds the number of
rows affected** (DB2 convention). An `INSERT`/`UPDATE`/`DELETE`
that matches zero rows is still `SQLCODE 0` — check `SQLERRD3` if
"did anything change?" matters.

### 6.3 COMMIT and ROLLBACK

```cobol
EXEC SQL COMMIT   END-EXEC.
EXEC SQL ROLLBACK END-EXEC.
```

`COMMIT` makes every change since the last commit durable;
`ROLLBACK` discards them. A task that ends without an explicit
`COMMIT` rolls back — uncommitted work never persists by
accident. Both verbs are idempotent: issuing one with no
transaction in flight is a harmless no-op.

### 6.4 Cursors — multi-row reads

A `SELECT` that returns many rows is read with the four-verb
cursor lifecycle:

```cobol
EXEC SQL DECLARE C1 CURSOR FOR
    SELECT id, name FROM customers_sql ORDER BY id
END-EXEC.

EXEC SQL OPEN C1 END-EXEC.

PERFORM UNTIL SQLCODE = 100
    EXEC SQL FETCH C1 INTO :ID, :NM END-EXEC
    IF SQLCODE = 0 THEN
        DISPLAY 'row: ' ID ' ' NM
    END-IF
END-PERFORM.

EXEC SQL CLOSE C1 END-EXEC.
```

REXX, identical four verbs:

```rexx
EXEC SQL DECLARE C1 CURSOR FOR SELECT id, name FROM customers_sql END-EXEC
EXEC SQL OPEN C1 END-EXEC
DO FOREVER
    EXEC SQL FETCH C1 INTO :ID, :CUSTNM END-EXEC
    IF SQLCODE = 100 THEN LEAVE
    SAY 'row:' ID CUSTNM
END
EXEC SQL CLOSE C1 END-EXEC
```

* `DECLARE` records the cursor's name and `SELECT` body; nothing
  contacts Postgres yet.
* `OPEN` runs the query, binding any input host variables at that
  instant.
* `FETCH … INTO` pulls the next row into the host variables.
  End-of-data is `SQLCODE +100`.
* `CLOSE` releases the cursor.

Cursors are closed automatically on `COMMIT`, `ROLLBACK`, or task
end, so a program that forgets `CLOSE` cannot leak a Postgres
result set. A closed cursor stays declared — `OPEN` it again to
rewind.

### 6.5 SELECT without INTO

A `SELECT` with no `INTO` clause runs as a side-effecting
statement (e.g. `SELECT pg_advisory_lock(…)`). It returns no rows
to the program.

### 6.6 In-database DDL

`CREATE TABLE`, `CREATE INDEX`, `ALTER TABLE`, `DROP TABLE`, and
the like are permitted from `EXEC SQL`. Only `DATABASE` / `USER` /
`ROLE` / `TABLESPACE` DDL is rejected (§3.4).

### 6.7 CONNECT TO — switching databases mid-task

```cobol
EXEC SQL CONNECT TO 'orders' END-EXEC.
```

Switches the executor to a different database from the catalogue.
The named database must exist in `databases.conf` — an unknown
name returns `SQLCODE -1`. Any in-flight transaction on the
previous database is committed first (DB2 convention; issue
`ROLLBACK` beforehand if you want to discard it).

Most programs never need `CONNECT TO`: the transaction's 5th
`transactions.conf` field already binds the starting database.
`CONNECT TO` is for the rare program that must touch two
databases in one task.

---

## 7. Null indicators

SQL `NULL` is distinct from an empty string or zero. A **null
indicator** is a second host variable, written immediately after
the value variable, separated by whitespace (no comma):

```cobol
01 NM    PIC X(30).
01 NIND  PIC S9(8).
...
EXEC SQL SELECT name INTO :NM :NIND
         FROM customers_sql WHERE id = :CUSTID END-EXEC.

EVALUATE NIND
    WHEN 0   DISPLAY 'name is ' NM
    WHEN -1  DISPLAY 'name is NULL'
END-EVALUATE.
```

**Reading (output direction):**

* column was non-NULL → indicator `0`, value variable holds the
  data.
* column was NULL → indicator `-1`, value variable is cleared to
  empty (so a program that ignores its indicator can't read a
  stale value).

**Writing (input direction):**

```cobol
MOVE -1 TO NIND.
EXEC SQL INSERT INTO customers_sql (id, name)
         VALUES (:K, :NM :NIND) END-EXEC.
```

* indicator `-1` at bind time → bricks sends SQL `NULL`,
  regardless of what the value variable holds.
* indicator `0` (or unset) → the value variable binds normally.

A value/indicator pair is two `:name` tokens separated only by
whitespace. `:NM, :NIND` (with a comma) is two *independent* host
variables, not a pair. The `:hv INDICATOR :ind` keyword form is
not supported — use the juxtaposed form.

---

## 8. WHENEVER — declarative error handling

Rather than test `SQLCODE` after every statement, a program can
declare a standing rule:

```cobol
EXEC SQL WHENEVER SQLERROR  GO TO SQL-ERROR-EXIT END-EXEC.
EXEC SQL WHENEVER NOT FOUND GO TO NO-MORE-ROWS  END-EXEC.
```

After every subsequent `EXEC SQL`, the matching condition is
checked:

| Condition | Fires when |
|---|---|
| `SQLERROR` | `SQLCODE` is negative |
| `NOT FOUND` | `SQLCODE` = `+100` |
| `SQLWARNING` | `SQLWARN0` = `'W'` (rare — bricks errors rather than warns) |

Each condition takes one action:

* `CONTINUE` — ignore it, fall through. This is the default for
  any condition never declared.
* `GO TO label` / `GOTO label` — branch to the paragraph (COBOL)
  or label (REXX). The branch unwinds any enclosing `PERFORM`
  just as an explicit `GO TO` would.

A later `WHENEVER` for the same condition replaces the earlier
one. REXX uses identical syntax; the branch is a `SIGNAL`:

```rexx
EXEC SQL WHENEVER SQLERROR GOTO SQLERR END-EXEC
EXEC SQL SELECT name INTO :CUSTNM FROM customers_sql
         WHERE id = :CUSTID END-EXEC
SAY 'customer:' CUSTNM
EXIT
SQLERR:
SAY 'SQL failed, SQLCODE' SQLCODE
EXIT
```

**Scoping caveat.** bricks's `WHENEVER` is *execution-order*
scoped — the most recently *executed* `WHENEVER` governs. Real
DB2 scopes *lexically* (by source position). For the standard
pattern — one `WHENEVER` near the top of the program — the two
are identical. Keep `WHENEVER` declarations at the top of a
paragraph to avoid surprises.

---

## 9. The SQLCA and the SQLCODE catalogue

### 9.1 The SQLCA fields

After every `EXEC SQL`, bricks writes the outcome into the SQL
communications area. For COBOL these fields are **auto-injected**
— no declaration needed:

| Field | PIC | Meaning |
|---|---|---|
| `SQLCODE` | `S9(8)` | Primary result code — see the catalogue below. |
| `SQLSTATE` | `X(5)` | 5-char ANSI/PG state (`00000` ok, `02000` no-data, …). |
| `SQLERRMC` | `X(70)` | Short human-readable message. |
| `SQLERRP` | `X(8)` | Product tag — bricks writes `BRICKS`. |
| `SQLERRD1`–`SQLERRD6` | `S9(9)` | Diagnostic array. `SQLERRD3` = rows affected. |
| `SQLWARN0`–`SQLWARN9`, `SQLWARNA` | `X` | 11 single-char warning flags. |

In REXX these arrive as ordinary variables of the same names.
bricks resets them before every statement, so a value never
leaks from the previous one.

### 9.2 The SQLCA copybook

`runtime/cobolcopy/SQLCA.cpy` provides **named constants** to
compare `SQLCODE` and `SQLSTATE` against — pull it in with
`COPY SQLCA.` in `WORKING-STORAGE`:

```cobol
WORKING-STORAGE SECTION.
COPY SQLCA.
...
EVALUATE SQLCODE
    WHEN SQL-OK          PERFORM SHOW-ROW
    WHEN SQL-NODATA      PERFORM SHOW-NOT-FOUND
    WHEN SQL-DUPKEY      PERFORM SHOW-DUPLICATE
    WHEN OTHER           PERFORM SHOW-ERROR
END-EVALUATE.
```

Every constant is also delivered under its IBM-DB2 alias
(`DB2-SUCCESS`, `DB2-NOTFOUND`, `DB2-DUP-KEY`, `DB2-DEADLOCK`, …)
so code ported from real DB2 keeps compiling.

### 9.3 The SQLCODE catalogue

| SQLCODE | Constant | Condition |
|---|---|---|
| `0` | `SQL-OK` | Success. |
| `+100` | `SQL-NODATA` | No row matched (`SELECT INTO` / `FETCH`). |
| `-1` | `SQL-NOCONFIG` | SQL not configured in `bricks.cnf`. |
| `-2` | `SQL-DDLREJECTED` | `CREATE`/`DROP DATABASE`/`USER`/… from a program — use CEDA. |
| `-104` | `SQL-SYNTAX` | Malformed SQL. |
| `-203` | `SQL-AMBIG-COL` | Ambiguous column reference. |
| `-204` | `SQL-UNDEF-TBL` | Table or view not found. |
| `-206` | `SQL-UNDEF-COL` | Column not found. |
| `-407` | `SQL-NOTNULL` | NULL into a `NOT NULL` column. |
| `-433` | `SQL-STRTRUNC` | Value too long for the destination. |
| `-530` | `SQL-FKVIOL` | Foreign-key violation. |
| `-545` | `SQL-CHKVIOL` | Check-constraint violation. |
| `-551` | `SQL-INSUF-PRIV` | Operator not authorised. |
| `-802` | `SQL-NUMOVERFLOW` | Numeric value out of range. |
| `-803` | `SQL-DUPKEY` | Duplicate key — `UNIQUE`/`PRIMARY KEY` violation. |
| `-811` | `SQL-MULTIPLEROWS` | `SELECT INTO` returned more than one row. |
| `-911` | `SQL-DEADLOCK` | Deadlock / serialization rollback — transient, retryable. |
| `-924` | `SQL-CONNLOST` | Postgres connection failed. |
| `-952` | `SQL-TIMEOUT` | Statement cancelled (`db_stmt_timeout` tripped). |
| `-100` | `SQL-GENERIC` | Any other PG error — `SQLSTATE` carries the precise code. |

### 9.4 Where the full error goes

`SQLERRMC` is a *short* message. The **full** Postgres diagnostic
(wrapper detail, host:port, internal context) is written to the
bricks console and the per-run log file on every error path — it
is deliberately kept off the program's `SQLERRMC` so a transaction
screen never exposes server internals. Programs should branch on
`SQLCODE`; operators read the bricks log for the detail.

---

## 10. Column-value coercion

Postgres's wire format does not always match what COBOL or REXX
expects. BRICKS  normalises three cases before a value reaches a
host variable:

* **`boolean` columns** — Postgres returns `t` / `f`; bricks
  converts to `1` / `0` so COBOL `PIC X(1)` flags and REXX
  `IF VAR = 1` idioms work.
* **`NUMERIC(p,s)` columns** — Postgres drops trailing fractional
  zeros for computed values; bricks pads to the column's declared
  scale so a COBOL implicit-decimal target (`PIC 9(p)V99`)
  receives `1.50`, not `1.5`, and lines up correctly.
* **`CHAR(n)` columns** — Postgres returns these space-padded;
  bricks trims trailing spaces so a REXX literal compare doesn't
  fail on `'X'` vs `'X    '`. `VARCHAR` is left alone.

Integers, `TEXT`, `DATE`, `TIMESTAMP`, JSON, and the rest flow
through verbatim. COBOL's `MOVE` handles integer sign and
zero-padding; REXX is dynamically typed so the string form serves
for both display and arithmetic.

---

## 11. The sample transactions — SQLD and SQLR

BRICKS ships two equivalent sample transactions — the same demo
in each language — that read the `customers_sql` table:

| TRANSID | Language | Program | Map |
|---|---|---|---|
| `SQLD` | COBOL | `runtime/cobol/sqld.cob` | `SQLD1` |
| `SQLR` | REXX | `runtime/rexx/sqlr.rexx` | `SQLD1` |

Both present the `SQLD1` screen: the operator types a customer id
(`K001`, `K002`, `K003` are seeded), presses ENTER, and the
program runs

```
EXEC SQL SELECT name INTO :NM :NMIND
         FROM customers_sql WHERE id = :CUSTID END-EXEC
```

then renders the fetched name plus `SQLCODE` / `SQLSTATE` and a
short status message. `PF3` exits. Each program is intentionally
written to show the Phase 4 surface — a `WHENEVER` directive, a
null indicator on the `SELECT INTO`, and an `EVALUATE` /
`SELECT/WHEN` over the expanded SQLCODE catalogue.

### 11.1 Walkthrough — SQLD (COBOL)

`sqld.cob` is a compact, complete example worth reading in full.
It deliberately exercises the Phase 4 SQL surface. Its shape:

1. `COPY DFHAID. COPY DFHRESP. COPY SQLCA.` — bring in the AID
   keys, response codes, and the SQLCODE constants (§9.2).
2. `01 NMIND PIC S9(8).` — a null indicator for the name column.
3. `EXEC CICS CONVERSE MAP('SQLD1')` — show the input screen and
   read the operator's customer id.
4. `EXEC SQL WHENEVER SQLERROR CONTINUE END-EXEC` — declares,
   explicitly, that the program inspects `SQLCODE` itself rather
   than branching to a handler paragraph (§8). The comment beside
   it shows the `GO TO` alternative.
5. `EXEC SQL SELECT name INTO :NM :NMIND FROM customers_sql
   WHERE id = :CUSTID END-EXEC` — the query, with a **null
   indicator** (`:NM :NMIND`, §7).
6. `EVALUATE SQLCODE` against the `SQLCA.cpy` constants —
   `SQL-OK` (and within it `IF NMIND = -1` for a NULL name),
   `SQL-NODATA`, `SQL-NOCONFIG`, `SQL-MULTIPLEROWS`,
   `SQL-UNDEF-TBL`, `SQL-TIMEOUT`, `SQL-DEADLOCK`,
   `SQL-CONNLOST`, and `OTHER` — to set a short, friendly status
   line. It uses the named constants, *not* the raw `SQLERRMC`,
   which would expose PG internals on the terminal.
7. `EXEC CICS CONVERSE` again to show the result.

### 11.2 Walkthrough — SQLR (REXX)

`sqlr.rexx` is the same demo in REXX, exercising the same Phase 4
surface: a `WHENEVER SQLERROR CONTINUE` directive, a null
indicator on the `SELECT INTO`, and a `SELECT/WHEN` over the
expanded SQLCODE catalogue.

Its host variables are named `CUSTNM` and `CUSTNMIND`, not `NM` /
`NMIND` — see the REXX naming caution in §5: `NM` is a tail of
the `SCR.` map stem, so reusing it as a simple host variable
would mis-resolve `SCR.NM = CUSTNM`. Picking host-variable names
that are not stem tails is the fix.

### 11.3 Running them

With Postgres configured and `customers_sql` seeded (§2.4), sign
on to BRICKS  and enter `SQLD` (or `SQLR`). Type `K001`, ENTER —
the screen shows `Alice`, `SQLCODE 0`, `SQLSTATE 00000`. Type an
unknown id — `SQLCODE +100` and "No customer with that id."

---

## 12. Monitoring SQL at runtime

`CEMT MONITOR` (the performance screen) counts `EXEC SQL`
statements separately from `EXEC CICS` verbs:

```
EXEC CICS (total)      12345
  EXEC CICS/s (intvl)    8.0
EXEC SQL (total)        2207
  EXEC SQL/s (intvl)     3.0
  ...
  SQL SELECT             1900
  SQL INSERT              210
  SQL FETCH                97
```

The per-verb breakdown (`SQL SELECT`, `SQL INSERT`, …) appears
once SQL traffic exists. This is the at-a-glance view of database
load distinct from terminal / control-block activity.

Operationally, watch the bricks console and per-run log file for
`SQL error SQLCODE=… SQLSTATE=…` lines — that is where the full
Postgres diagnostic lands (§9.4).

---

## 13. Restrictions

| Not supported | Notes |
|---|---|
| Stored-procedure `EXEC SQL CALL` | Use a plain `SELECT` from a function instead. |
| Multi-row `INSERT … VALUES (…), (…)` | Issue one statement per row. |
| `PREPARE` / `EXECUTE` / `EXECUTE IMMEDIATE` (dynamic SQL) | Statements are static at the `EXEC SQL` site. |
| `:hv INDICATOR :ind` keyword form | Use the juxtaposed `:hv :ind` form (§7). |
| `CREATE`/`DROP` of `DATABASE`/`USER`/`ROLE`/`TABLESPACE` from a program | CEDA DATABASE only (§3.4). |
| Cursors held across `EXEC CICS RETURN` chains | Each task gets a fresh connection; cursors close at task end. |
| Lexically-scoped `WHENEVER` | bricks is execution-order scoped (§8). |

---

## 14. Quick reference

**Statements**

```
EXEC SQL SELECT cols INTO :v[, :v…] FROM … WHERE …      END-EXEC
EXEC SQL INSERT INTO t (cols) VALUES (:v[, :v…])        END-EXEC
EXEC SQL UPDATE t SET col = :v WHERE …                  END-EXEC
EXEC SQL DELETE FROM t WHERE …                          END-EXEC
EXEC SQL COMMIT                                         END-EXEC
EXEC SQL ROLLBACK                                       END-EXEC
EXEC SQL CONNECT TO 'dbname'                            END-EXEC
EXEC SQL DECLARE c CURSOR FOR SELECT …                  END-EXEC
EXEC SQL OPEN c                                         END-EXEC
EXEC SQL FETCH c INTO :v[, :v…]                         END-EXEC
EXEC SQL CLOSE c                                        END-EXEC
EXEC SQL WHENEVER {SQLERROR|NOT FOUND|SQLWARNING}
         {CONTINUE | GO TO label}                       END-EXEC
```

**Outcomes to test**

```
SQLCODE = 0      → success
SQLCODE = +100   → no row / end of cursor
SQLCODE < 0      → error (see §9.3 catalogue; SQLSTATE has detail)
SQLERRD3         → rows affected (INSERT/UPDATE/DELETE)
indicator = -1   → column was NULL
```

**Operator checklist for a new SQL deployment**

1. Set the `db_*` block in `bricks.cnf` (§2.1).
2. List the database(s) in `databases.conf` (§2.2).
3. `CEDA` → `DATABASE` → `C` to create the PG-side database, or
   create it in `psql` (§3.2).
4. Create your tables (`sql_bricks_statements.sql` for the demo,
   §2.4).
5. Bind transactions to a database with the 5th
   `transactions.conf` field if not the default (§2.3).
6. Restart bricks; confirm the startup log shows the database
   `ONLINE`.
7. Run `SQLD` / `SQLR` to smoke-test (§11.3).
