# Bricks Transaction Serer

A Go implementation of an IBM CICS-compatible 3270 transaction server. Users
dial in with a 3270 terminal emulator, sign on via a built-in CSSN scrxeen, and
run REXX or COBOL programs whose `EXEC CICS` commands are interpreted by a
built-in REXX or COBOL VMs with `EXEC CICS` handlers backed by an on-disk record store.

Both dialects and interpreter implementations of REXX and COBOL are mine, and not related
to BREXX, or Regina. 

The EXEC CICS syntax is compatible with CICS, and there are enough calls to enable 
pseudo-conversational and conversational programs to run as usual. 

Bricks also features a built-in VSAM access method which is then stored inside a BoltDB 
database for easy management, backup etc. 

Cobol and REXX programs are parsed once and then cached so repeat dispatches skip
the lex+parse cost. Since every instantiated program has its own heap and stack,there are
no re-entrancy issues to deal with. Bricks expressely disallows the use of calculated GOTOs 
in COBOL programs, also for the same reason. 

---

## Quick start

```sh
# Add a user (admin / admin already exists in runtime/users.conf).
./add_brick_user.bash alice "alice's-password" admin,users

# Run the server. Edit bricks.cnf first (see "Configuration").
./bricks --conf=bricks.cnf

# Connect with any 3270 emulator (c3270, x3270, tn3270 …):
c3270 -port 2300 localhost
```

On connect you'll see the `bricks.logo` splash in blue. Press ENTER to reach
the TRANSID prompt; type `CSSN` to sign on, then any defined TRANSID
(e.g. `HELO`).

---


## Configuration — `bricks.cnf`

Key=value, one per line, `#` for comments. Keys are case-insensitive.

| Key                          | Default                       | Notes |
|------------------------------|-------------------------------|-------|
| `port`                       | `2300`                        | Plain-TCP listener. |
| `tlsport`                    | `2023`                        | Used only when `start_TLS=yes`. |
| `start_TLS`                  | `no`                          | `yes` requires `tlscert` + `tlskey`. |
| `enforce_secure_login`       | `no`                          | `yes` blocks every TRANSID except the logon TRANSID until the session is authenticated. |
| `tlscert`                    | (none)                        | Path to PEM cert. |
| `tlskey`                     | (none)                        | Path to PEM key. |
| `secure_login_transacton`    | `CSSN`                        | The 4-character logon TRANSID. Built-in `CSSN` is implemented in Go; any other value must exist in `transactions.conf`. Misspelling preserved for back-compat; `secure_login_transaction` and `logon_transid` are also accepted. |
| `users_file`                 | `runtime/users.conf`          | Auth source. |
| `transactions_file`          | `runtime/transactions.conf`   | TRANSID dispatch table. |
| `maps_dir`                   | `runtime/map`                 | Directory of `*.map` files. |
| `rexx_dir`                   | `runtime/rexx`                | Directory of REXX programs. |
| `data_dir`                   | `data`                        | Holds `files.boltdb` (FILE store + TS queues). |
| `idle_timeout_secs`          | `900`                         | Read deadline applied to the CSSN sign-on flow and to every blank/logon prompt. A peer that holds the connection open without sending input is dropped at this many seconds. |
| `max_conns_per_ip`           | `8`                           | Per-client cap. |
| `banner`                     | `BRICKS Transaction Server`   | Shown at top of system screens. |
| `dns_name`                   | (none)                        | Informational; printed at startup. |
| `start_web3270`              | `no`                          | `yes` enables the in-process browser-based 3270 emulator. |
| `web3270_port`               | `9000`                        | HTTP port for the web3270 frontend (only used when `start_web3270=yes`). |
| `start_metrics`              | `yes`                         | `yes` exposes a JSON `/metrics` endpoint with runtime + counter snapshots. Independent of `start_web3270`. |
| `metrics_port`               | `9100`                        | HTTP port for the dedicated `/metrics` listener. The same route is also mounted on the web3270 listener when both are on. |

Command-line flags (in addition to `--conf`):

| Flag             | Notes |
|------------------|-------|
| `--no-console`   | Disable the framed operator console; emit raw `log.Printf` lines on stderr. Use under `nohup` / `systemd` / when piping through `tee`. |

---

## Authentication procedure

The connection lifecycle is owned by `main.go::handle()`:

1. **Accept** — telnet/3270 negotiation runs (`tn3270.Negotiate`); device size
   and codepage are captured.
2. **TCB** — a fresh `session.TCB` is created with a unique `TermID` (T0001,
   T0002, …) and registered in the global `session.Registry`.
   `Authenticated` is `false`.
3. **Splash** — `tn3270.ShowLogoSplash` paints `bricks.logo` in blue,
   centered. No input fields. Returns when the user presses any AID.
4. **Logon prompt** — `tn3270.LogonPrompt` shows the logo plus a TRANSID
   input field. When `enforce_secure_login=yes` and the session is not yet
   authenticated, a blue notice on row 0 reads `Sign on with <transid> to
   continue.`. PF3 / CLEAR / PA1-3 disconnect.
5. **Auth gate** —
   * If the typed TRANSID equals `secure_login_transacton`, the configured
     logon flow runs. The default `CSSN` is built into `auth/cssn.go`: it
     loads `runtime/map/cssn.map`, prompts for userid+password, looks the user up in
     `runtime/users.conf`, verifies the bcrypt hash with `golang.org/x/crypto/bcrypt`,
     and on success sets `tcb.UserID`, `tcb.Groups`, `tcb.Authenticated=true`,
     and attaches the TCB to a UCB via `Registry.AttachUserToTerminal`.
     Failures bump `Registry.AuthFailure` and re-prompt.
   * Otherwise, if `enforce_secure_login=yes` and the session is not
     authenticated, the dispatcher is bypassed and the user is shown
     `Not signed on. Run <logon> first.` then sent back to the prompt.
   * Else the dispatcher runs the TRANSID.
6. **Dispatch** — `txn.Dispatcher.Run` chains through `tcb.NextTransid` after
   each `EXEC CICS RETURN TRANSID(...)`. When the next TRANSID is empty,
   control returns to step 4. Before each dispatch the per-transaction
   ACL gate fires (see [Per-transaction ACL](#per-transaction-acl)
   below); a denied dispatch shows `TRANSID "X": access denied -- ...`
   on the operator screen and logs the user / groups / required list
   to the console for grep.
7. **Sign off** — typing `CSSF LOGOFF` (any case; argument required) at the
   blank prompt detaches the UCB via
   `Registry.DetachUserFromTerminal(tcb)`, clears `tcb.UserID` and
   `tcb.Authenticated`, and sends the terminal back to the unauthenticated
   logon prompt. **The TCP connection stays open**: only the user's TCP
   close cuts it. Bare ENTER, PF3, CLEAR, and PA1-3 at the blank prompt
   redisplay the same screen — they no longer disconnect.
8. **Disconnect** — `defer registry.RemoveTerminal(tcb)` drops the TCB; if the
   user was signed on and this was their last terminal the UCB is also
   dropped.

Each top-level prompt sets a `conn.SetReadDeadline(now + idle_timeout_secs)`
before reading and clears the deadline on success, so a peer that completes
telnet negotiation but never sends a screen response is bumped after
`idle_timeout_secs` rather than tying up a `max_conns_per_ip` slot
indefinitely.

`runtime/users.conf` format (the comment header in the file is the source of
truth):

```
# user:bcrypt_hash:groups(comma-separated)
admin:$2a$10$.....:admin,users
alice:$2a$10$.....:users
```

To add or rotate passwords:

```sh
./add_brick_user.bash alice newpassword users          # add
./add_brick_user.bash --update alice newpassword admin # update / change groups
go run ./cmd/brickspw "raw password"                   # just emit a hash
```

The script refuses to overwrite an existing user without `--update`.

---

## Per-transaction ACL

`runtime/transactions.conf` accepts an **optional last field** —
comma-separated, case-insensitive group names — that gates dispatch
per transaction. Format:

```
transid:type:program[:groups]
```

Three-field entries keep the legacy behaviour: `enforce_secure_login`
in `bricks.cnf` is the only check (so a signed-on user can run
anything, an unsigned-on user nothing if the gate is on). Add the 4th
field and the listed groups become an enforced ACL — checked in
`txn/dispatcher.go::Run` after the table lookup and before the REXX
program loads, and again in `LinkProgram` so a low-privilege task
can't `EXEC CICS LINK PROGRAM('ADMN')` to escalate.

Two reserved tokens:

| Token | Effect |
|---|---|
| `public` | Allow unauthenticated callers. Without it, a 4-field tx requires sign-on. |
| `*` | Allow any signed-on user, regardless of group. |

Decision precedence: `public` > unauthenticated denial > `*` > any
shared group. Real groups come from the `groups` column of
`runtime/users.conf` (attached to the TCB by `auth.RunCSSN`); both
sides are uppercased for comparison.

Examples:

```
HELO:rexx:hello.rexx                  # legacy — open to any signed-on caller (or anyone if enforce_secure_login=no)
HELP:rexx:help.rexx:public            # always reachable, even pre-CSSN
QAGE:rexx:qage.rexx:public,users,admin
PROD:rexx:prod.rexx:users,admin       # signed on AND in users or admin
ADMN:rexx:admn.rexx:admin             # admin only
ANYI:rexx:anyi.rexx:*                 # any signed-on user
```

A denied dispatch surfaces:

* On the 3270 screen: `TRANSID "QAGE": access denied -- requires
  group ADMIN, USERS` (or `-- sign on first` for an unsigned-on
  caller hitting a non-`public` ACL).
* In the console log: `term=T0001 transid=QAGE access denied;
  user="ALICE" groups=[USERS] required=[ADMIN]`.

`CEMT INQUIRE TRANSACTION` adds a `GROUPS` column showing each
transaction's ACL (`-` for legacy 3-field entries):

```
TRANSID  LANG  PROGRAM      INVOKED  CACHED  CACHE%  GROUPS
QAGE     REXX  qage.rexx    124      123     99%     PUBLIC,USERS,ADMIN
HELP     REXX  help.rexx    8        7       88%     PUBLIC
HELO     REXX  hello.rexx   3        2       67%     -
ADMN     REXX  admn.rexx    0        0       -       ADMIN
```

The table is hot-reloaded on `transactions.conf` mtime change, so
adding or tightening an ACL takes effect on the next dispatch with no
bricks restart.

---

## In-memory control blocks

Bricks tracks four kinds of in-memory control blocks plus a process-wide
registry that owns them. All four live in package `session/`.

| Kind  | Scope                          | Lifetime |
|-------|--------------------------------|----------|
| TCB   | one per terminal connection    | accept → disconnect |
| UCB   | one per signed-on user         | first sign-on → last terminal disconnects |
| FCB   | one per CICS FILE name         | first access → process exit |
| TxCB  | one per running transaction    | dispatcher BeginTxn → EndTxn |


### `TCB` — termid_control_block (`session/session.go`)

One per accepted terminal connection. Holds everything bricks needs while
that connection is alive. Created at telnet-negotiation time, dropped on
disconnect. Field summary:

| Field            | Purpose |
|------------------|---------|
| `Conn`           | Underlying `net.Conn` (plain or `*tls.Conn`). |
| `Dev`            | `go3270.DevInfo` — terminal capabilities. |
| `IsTLS`          | True when the connection is on the TLS listener. |
| `TermID`         | Unique 4-digit terminal id (T0001 …). Mirrors `EIBTRMID`. |
| `UserID`         | Empty until the user signs on. |
| `Groups`         | Group membership snapshot from `users.conf`. |
| `RemoteIP`       | Remote host for per-IP accounting. |
| `Cols`, `Rows`   | Effective terminal size from `Dev.AltDimensions()`. |
| `Connected`      | Wall-clock time of accept. |
| `Authenticated`  | True after a successful CSSN sign-on. |
| `EIBAID`, `EIBCPOSN`, `EIBTRMID`, `EIBRESP`, `EIBRESP2` | EIB shadow used by EXEC CICS handlers. |
| `NextTransid`    | Set by `EXEC CICS RETURN TRANSID(...)`; consumed by the dispatcher. |
| `Commarea`       | COMMAREA bytes flowed between pseudo-conversational invocations. |
| `LastResponse`, `LastMapName` | Captured by `SEND MAP`; consumed by `RECEIVE MAP`. |
| `LockedRec`      | File→key map for `READ FILE UPDATE` / `REWRITE` protocol. |
| Counters (atomic): `TxnRun`, `TxnFailed`, `ScreensSent`, `ScreensRcvd`, `CommandsExec`, `BytesIn`, `BytesOut`. |

Once authenticated, `tcb.UCB()` returns the linked `*UCB`; nil before sign-on.

### `UCB` — userid_control_block (`session/ucb.go`)

One per signed-on user, regardless of how many terminals that user is using.
Created on the first successful sign-on for that userid; dropped when the
last terminal for that user disconnects.

| Field         | Purpose |
|---------------|---------|
| `UserID`      | The authenticated username. |
| `Groups`      | Groups from the most recent sign-on. |
| `FirstLogin`  | When the UCB was first created. |
| `LastLogin`   | When the most recent attached terminal signed on. |
| `LoginCount`  | Atomic counter, incremented on each `attach`. |
| `TxnRun`      | Atomic counter; the dispatcher bumps after every successful TRANSID. |
| `Terminals()` | Snapshot of every TCB this user is currently on. |
| `AnyTerminal()` | Convenience helper when only `Cols`/`Rows`/`IsTLS` is needed. |

`UCB.Terminals()` is the answer to "where is this user signed on right now,
and what are the terminal characteristics there?" — caller takes any TCB and
reads `Cols`, `Rows`, `IsTLS`, `Connected`, etc.

### `FCB` — file_control_block (`session/fcb.go`)

One per CICS FILE name. Created lazily the first time any session touches a
file (via READ/WRITE/REWRITE/DELETE FILE) and kept for the life of the
process.

| Field         | Purpose |
|---------------|---------|
| `Name`        | Uppercased FILE name. |
| `FirstAccess` | When the FCB was first registered. |
| `LastAccess`  | Atomic unix-nano of the most recent access. |
| Counters (atomic): `Reads`, `Writes`, `Rewrites`, `Deletes`, `NotFound`, `IOErrors`. |
| `Lock(termID)` / `Unlock(termID)` / `LockedBy()` | Track the holder of the current `READ FILE UPDATE` lock; the actual record-level mutex lives on `cics.Store`. |

### `TxCB` — transaction_control_block (`session/txcb.go`)

One per running transaction. Created by `Registry.BeginTxn` immediately
before the dispatcher runs the REXX program and removed by `EndTxn` after
the program returns or aborts.

| Field         | Purpose |
|---------------|---------|
| `ID`          | Sequential id (`X0000001 …`). |
| `TransID`     | The 4-character transid. |
| `Program`     | Program file as written in `transactions.conf`. |
| `StartedAt` / `EndedAt` | Wall-clock bookends. `Duration()` returns elapsed (running or final). |
| `TCB`         | Pointer to the terminal this transaction runs on. |
| `UCB`         | Pointer to the signed-on user (nil during the logon transaction itself). |
| `AddFCB(f)` / `FCBs()` | Set of FCBs touched by this transaction; populated automatically by the EXEC CICS file handlers. |

### `Registry` (`session/registry.go`)

Process-wide owner of every TCB, UCB, FCB, and TxCB. One instance is created
at startup and shared across the dispatcher, the auth flow, and the cics
store.

```go
reg := session.NewRegistry()
reg.AddTerminal(tcb)                              // on telnet-negotiation success
reg.AttachUserToTerminal(tcb, userID, groups)     // on CSSN success
reg.DetachUserFromTerminal(tcb)                   // on CSSF LOGOFF
reg.RemoveTerminal(tcb)                           // on disconnect

fcb  := reg.GetOrCreateFCB("USERS")               // called by cics.Store
txcb := reg.BeginTxn(tcb, "MENU", "menu.rexx")    // called by txn.Dispatcher
defer reg.EndTxn(txcb)

terms, users := reg.Snapshot()                    // for admin tools / CEMT
fcbs := reg.AllFCBs()
txns := reg.AllTxns()
nTCB, nUCB, nFCB, nTxCB := reg.Counts()
```

`Registry.Snapshot`, `AllFCBs`, and `AllTxns` are the entry points the CEMT
transaction uses to populate its Control Blocks screens.

**Lock layout.** The registry no longer uses a single global mutex over all
four collections. `tcbs`, `ucbs`, and `fcbs` each have their own
`sync.RWMutex`; `txcbs` is a `sync.Map` paired with an `atomic.Int64`
count, so `BeginTxn` / `EndTxn` (the per-transaction hot path) are
lock-free. Lock-ordering rule when more than one is needed:
`tMu` → `uMu` → per-block locks (`u.mu`, `t.mu`). `RemoveTerminal` and
`DetachUserFromTerminal` are the only paths that take `uMu` after `tMu`.

---

## Performance counters

All counters are `atomic.Uint64` so they can be sampled at any time without
locking.

### Process-level (`*session.Registry`)

| Counter             | Bumped when |
|---------------------|-------------|
| `Accepts`           | A TCB is added to the registry. |
| `Rejects`           | (Reserved for the per-IP cap path.) |
| `AuthSuccess`       | A UCB is attached to a TCB by the auth flow. |
| `AuthFailure`       | The auth store returns `ErrUnknownUser` or `ErrBadPassword`. |
| `TotalTxnRun`       | Every successful TRANSID dispatch. |
| `TotalTxnFailed`    | A REXX parse / runtime / IO error in `txn.Dispatcher.runRexx`. |
| `StartedAt`         | Server start (timestamp, not a counter). |

### Terminal-level (`*session.TCB`)

| Counter            | Bumped when |
|--------------------|-------------|
| `TxnRun`           | After a successful TRANSID on this terminal. |
| `TxnFailed`        | When the dispatcher records a failure for this terminal. |
| `ScreensSent`      | Reserved — `tn3270.SendMap` will increment once instrumentation lands. |
| `ScreensRcvd`      | Reserved — bumped by RECEIVE MAP. |
| `CommandsExec`     | Reserved — bumped by every `EXEC CICS` dispatch. |
| `BytesIn`/`BytesOut` | Reserved for an instrumented `net.Conn` wrapper. |

### User-level (`*session.UCB`)

| Counter      | Bumped when |
|--------------|-------------|
| `LoginCount` | Each time a TCB attaches to this UCB (re-sign-on, additional terminal). |
| `TxnRun`     | Every successful TRANSID by any of this user's terminals. |

### File-level (`*session.FCB`)

| Counter    | Bumped when |
|------------|-------------|
| `Reads`    | A successful `READ FILE`. |
| `Writes`   | A successful `WRITE FILE`. |
| `Rewrites` | A successful `REWRITE FILE`. |
| `Deletes`  | A successful `DELETE FILE`. |
| `NotFound` | The record did not exist on read/rewrite/delete. |
| `IOErrors` | The underlying filesystem returned a non-`NotExist` error. |

### EXEC CICS verb (`cics/metrics.go`)

| Counter                  | Bumped when |
|--------------------------|-------------|
| `cics.ExecTotal()`       | Every parsed EXEC CICS verb dispatch (`atomic.Int64`). |
| `cics.ExecPerVerb()`     | Snapshot of `(verb, count)` pairs sorted by count desc. Backed by a `sync.Map` of `*atomic.Int64` so the hot path is lock-free for any verb already seen. |

Live counters can be inspected from a 3270 terminal via the CEMT
transaction's INQUIRE CONTROLBLOCKS sub-tree and the MONITOR screen
(see below).

---

## Operator console

Pass `--no-console` to disable the frame and emit raw log output
(suitable for `nohup` / `systemd` / piping through `tee`).


---

## CEMT — master-operator transaction

`CEMT` is a built-in TRANSID (no entry needed in `transactions.conf`,
implemented in package `cemt/`). The MONITOR and PERFORM branches plus
the CONTROLBLOCKS sub-tree under INQUIRE are gated on the `admin` group;
non-admin operators only see the read-only INQUIRE views.

```
+-----------------------------------------------------------+
| BRICKS Transaction Server  •  CEMT — master terminal      |
|                                                           |
|   Pick an option and press ENTER. PF3 to back out.        |
|                                                           |
|     I  INQUIRE   resources, CICS-style                    |
|     M  MONITOR   process metrics (admin)                  |
|     P  PERFORM   scans + TS purge (admin)                 |
|     Q  QUIT      (or press PF3)                           |
|                                                           |
|   Choice: _                                               |
+-----------------------------------------------------------+
```

Every node accepts any unambiguous abbreviation of its name (so `MON`,
`MONIT`, `MONITOR` all reach MONITOR; `INQ`, `PERF`, etc. follow the
same rule), and tokens chain — `CEMT P T` jumps straight to PERFORM →
TRANS without any intermediate menu.

### CEMT INQUIRE

The CICS-style read-only resource views, plus a CONTROLBLOCKS sub-tree
that exposes the live runtime control blocks for admins:

```
S  TS              TS queue stats
F  FILE            file resources
U  USER            signed-on users
T  TERMINAL        connected terminals
R  TRANSACTION     entries from transactions.conf
C  CONTROLBLOCKS   TCBs / UCBs / TXCBs / FCBs (admin)
   ├── T  TCBS    (3 active terminals)
   ├── U  UCBS    (1 signed-on users)
   ├── X  TXCBS   (0 running transactions)
   └── F  FCBS    (2 known files)
```

Each detail screen renders a fixed-width table. Columns are auto-sized
to the widest value, with a fallback to ellipsis when the row would
overflow. PF3 exits the screen, ENTER refreshes counters in place.

### CEMT MONITOR

Process metrics (`cemt/perf.go`). Single screen — a renamed home for
what used to be `CEMT P PERFORMANCE`:

```
+----------------------------------------------------------------------+
| BRICKS Transaction Server • CEMT — Performance • TERM=T0001          |
|                                                                      |
|  Process                              Activity                       |
|  ───────────────────────              ───────────────────────        |
|  Memory (heap)        12.4 MB         EXEC CICS total       1,234    |
|  Memory (sys)         45.1 MB           SEND                  312    |
|  Heap objects        12,345             RECEIVE               312    |
|  GC runs                  3             READ                  140    |
|  GC last pause         1.20 ms          ASSIGN                 80    |
|  CPU user             2.50 s            LINK                   33    |
|  CPU sys              0.30 s            …                            |
|  CPU% (avg)           1.8%                                           |
|  Goroutines               7                                          |
|  Uptime               3m 12s                                         |
|                                                                      |
|  Sessions                                                            |
|  ─────────────────────────────────────────────────────────────────── |
|  Active terminals    2          Active transactions    1             |
|  Signed-on users     1          Known files            1             |
|                                                                      |
|  ENTER=Refresh  PF3=Back                                             |
+----------------------------------------------------------------------+
```

### CEMT PERFORM

The TS-queue purge screen (which used to live under the old `CEMT C S`)
plus the diagnostic rescans grouped under their own sub-branch:

```
U  PURGE     (N queues -- type P to purge) -- TS queue purge selector
R  RESCAN    trans / maps / programs       -- on-disk diagnostic scans
   ├── T  TRANS     (N transactions, M missing)   -- rescan transactions.conf
   ├── M  MAP       (N maps, M syntax errors)     -- parse every *.map in MapsDir
   └── P  PROGRAMS  (N programs, M orphans)       -- walk rexx_dir + cobol_dir
```

* **PERFORM PURGE** (`CEMT P U`) is the TS-queue list with a per-row
  P-selector and a confirmation overlay (was `CEMT C S`).
* **RESCAN TRANS** (`CEMT P R T`) re-stats every program path declared
  in `runtime/transactions.conf` and renders TRANSID / LANG / PROGRAM /
  STATUS / PATH. STATUS is `OK` when the file is present, `MISSING` when
  it is not, or `ERROR: <message>` for any other stat error (e.g. a
  permission problem). ENTER re-runs the scan, so an operator can fix
  the conf or drop a file in place and watch a row flip without leaving
  the screen.
* **RESCAN MAP** (`CEMT P R M`) walks `MapsDir`, parses every `*.map`
  with `mapdsl.Parse`, and renders FILE / NAME / STATUS / SYNTAX.
  STATUS is `OK` when the file is readable, `MISSING` (or the stat
  error) otherwise. SYNTAX is `Pass` when the parser accepts the file,
  the parser error verbatim when it doesn't, and `-` when the file
  isn't readable. The catalog reload path silently keeps the prior
  catalog when a parse fails — this screen is how an operator finds
  out which file is broken.
* **RESCAN PROGRAMS** (`CEMT P R P`) walks `rexx_dir` and `cobol_dir`,
  lists every regular file, and shows the TRANSIDs in
  `transactions.conf` that reference it. Files with no matching
  TRANSID render with TRANSID=`-` so stale leftovers stand out.


---

## The map DSL

Custom, line-oriented, BMS-flavored. Comments start with `*`. Layout:

```
* CICS sign-on screen
MAP CSSN SIZE 24x80
  FIELD AT 1,28 LEN 24 PROT BRIGHT COLOR=TURQUOISE  "BRICKS SIGN-ON"
  FIELD AT 6,15 LEN 9  PROT                         "Userid:"
  INPUT USERID   AT 6,25 LEN 8 UNDERSCORE COLOR=GREEN
  STOP           AT 6,34
  FIELD AT 8,15 LEN 9  PROT                         "Password:"
  INPUT PASSWORD AT 8,25 LEN 8 HIDDEN UNDERSCORE COLOR=RED
  STOP           AT 8,34
  CURSOR         AT USERID
ENDMAP
```

Statements:
* `MAP <NAME> SIZE rowsxcols` — the header.
* `FIELD AT r,c LEN n <attrs> "literal"` — display-only field.
* `INPUT NAME AT r,c LEN n <attrs> [DEFAULT "value"]` — input field. `PROT`
  makes it write-protected but still named (useful for `SEND MAP FROM(STEM.)`
  output).
* `STOP AT r,c` — autoskip stop (ends an input field).
* `CURSOR AT <field-name>` *or* `CURSOR AT r,c` — explicit cursor home. The
  named form looks up any defined `FIELD` or `INPUT` in the map and resolves
  to `(field.Row, field.Col + 1)` automatically — i.e. one byte to the
  right of the field's leading 3270 attribute byte, which is the writable
  cell. It works on display fields too, so maps without any `INPUT` (e.g.
  splash screens) can still anchor the cursor on a named display `FIELD`.
  The numeric form is kept for cases where the cursor target is not on
  any defined field. Forward references work — `CURSOR AT BAR` may appear
  before `INPUT BAR …` because the name is resolved after the whole map is
  parsed. When `CURSOR` is omitted entirely, the renderer falls back to
  `firstInput.Row, firstInput.Col + 1`.
* `ENDMAP` — ends the map.

Attributes: `PROT`, `UNPROT`, `BRIGHT`, `DIM`, `UNDERSCORE`, `HIDDEN`,
`NUMERIC`, `MDT`, `BLINK`, `REVERSE`, `COLOR=BLUE|RED|PINK|GREEN|TURQUOISE|YELLOW|WHITE`.

`mapdsl.NewCatalog(dir)` loads every `*.map` file at startup and self-refreshes
on subsequent edits (each `Lookup` stats the directory + the source file
backing the requested map name; on mtime change the directory is reparsed
and swapped atomically; parse errors keep the prior catalog). Map names must
be unique across the directory (case-insensitive).

---

## REXX Syntax

Hand-rolled lexer + parser + tree-walking interpreter (`rexx/`). Supported:

* Variables (case-insensitive, NOVALUE returns the uppercased name).
* Stems with default value: `STEM. = 'unset'; STEM.42 = 'forty-two'`.
* **Compound-variable tail substitution**. Non-numeric tail symbols are
  resolved at every reference: with `J = 3`, `A.J` reads/writes `A.3`. Pure
  numeric tails (`A.0`, `A.42`) and unset tail symbols (REXX NOVALUE)
  remain literal. Multi-segment tails work too: `A.I.J` with `I=1, J=2`
  references `A.1.2`. Implemented in `rexx/parser.go` (`makeVarExpr`) and
  `rexx/interp.go` (`resolveCompoundName`).
* Procedures with their own scope: `NAME: PROCEDURE [EXPOSE list]` …
  `RETURN`. EXPOSE re-routes named/stem variables to the caller's frame
  recursively.
* Control: `IF expr THEN [ELSE]`, `SELECT … WHEN … OTHERWISE … END`,
  `DO`, `DO N`, `DO var=a TO b BY s`, `DO WHILE`, `DO UNTIL`, `DO FOREVER`,
  `DO var OVER stem.` (iterate over each tail of a stem; numeric tails
  sort first, then lexicographic).
* Loop control: `LEAVE [ctrlvar]` exits the innermost (or named outer) DO;
  `ITERATE [ctrlvar]` skips to the next iteration of the innermost (or
  named outer) DO.
* `DROP name [name…]` removes one or more variables. A trailing `.` drops
  the entire stem (default + every tail) — common idiom: `DROP RECS.` to
  reset an accumulator between paginated reads.
* I/O: `SAY`, `PARSE [UPPER] {VAR var | VALUE … WITH | ARG | PULL} template`
  with full template support — string anchors, absolute (`n`) and relative
  (`+n`/`-n`) column markers, `.` placeholder, bare-variable runs.
* `CALL`, `RETURN`, `EXIT`, `SIGNAL <label>`, `NOP` (a real no-op statement
  so `OTHERWISE NOP` works inside `SELECT`).
* `SIGNAL ON {ERROR | NOVALUE | SYNTAX | HALT} [NAME label]` — armed
  conditions are honoured at runtime: ERROR fires when an `EXEC CICS`
  command returns non-zero RC; NOVALUE fires on reference to an unset
  simple or compound variable; SYNTAX catches any other interpreter error
  (bad numeric, divide by zero, unknown function, etc.). Without an armed
  trap, the legacy "test EIBRESP after every verb" pattern still works.
  When a trap fires, `SIGL` is set to the source line of the failing
  statement before jumping to the labelled handler. `SIGNAL OFF cond`
  disarms.
* `INTERPRET expr` — evaluate the string value of `expr` as REXX source
  and execute it in the current frame.
* `NUMERIC DIGITS n` / `NUMERIC FUZZ n` / `NUMERIC FORM SCIENTIFIC|ENGINEERING`
  — the syntax and basic settings are honoured; arithmetic is float64
  internally so DIGITS controls output rounding (via `FORMAT(x, before, after)`)
  rather than driving a true decimal engine. `DIGITS()`, `FUZZ()`, `FORM()`
  return the current settings.
* `ADDRESS <env>` — switches the active `AddressHandler`. Bare strings inside
  an `ADDRESS` scope are commands; bricks ships a `CICS` handler.
* Built-in functions, organised by family:

  | Family | Functions |
  |---|---|
  | Length / index | `LENGTH`, `POS`, `LASTPOS`, `WORDS`, `WORDPOS`, `WORDINDEX`, `WORDLENGTH`, `COUNTSTR` |
  | Substring | `SUBSTR`, `LEFT`, `RIGHT`, `SUBWORD`, `WORD`, `DELSTR`, `DELWORD`, `INSERT`, `OVERLAY`, `CHANGESTR` |
  | Whitespace / case | `STRIP`, `SPACE`, `CENTER` (alias `CENTRE`), `JUSTIFY`, `UPPER`, `LOWER`, `REVERSE`, `COPIES`, `ABBREV` |
  | Translation | `TRANSLATE`, `VERIFY`, `COMPARE`, `XRANGE`, `BITAND`, `BITOR`, `BITXOR` |
  | Conversion | `C2X`, `X2C`, `D2X`, `X2D`, `D2C`, `C2D`, `B2X`, `X2B` |
  | Numeric | `ABS`, `MAX`, `MIN`, `INT`, `TRUNC`, `MOD` (= `//` operator), `SIGN`, `FORMAT(num, before, after)`, `DIGITS`, `FUZZ`, `FORM` |
  | Type / data | `DATATYPE` (with `N`, `W`, `A` options), `LENGTH` |
  | Date / time | `DATE` (`N`/`S`/`E`/`U`/`O`/`B`; `B` does basedate arithmetic), `TIME` |
  | Variables / args | `VALUE`, `ARG`, `RANDOM`, `ERRORTEXT` |

  `VALUE` accepts the canonical 1-arg read form (`VALUE('SCR.ROW' || J)`)
  and the 2-arg assignment form (`CALL VALUE 'SCR.ROW' || J, LINE` — sets
  the variable named at runtime and returns the prior value). `C2X` is the
  standard way to compare `EIBAID` to a PF-key code:
  `IF C2X(EIBAID) = 'F7' THEN …` for PF7. `DATE('B')` and
  `DATE('B', 'YYYYMMDD', 'S')` give days since 0001-01-01 — subtract two
  basedates for an exact day delta.
* Operators: `+ - * / % // **`, comparisons (numeric when both sides parse
  as numbers, trimmed-string otherwise), `||` and juxtaposition concat,
  `& |`, unary `\`.

### Compound-symbol pitfall

`STEM.tail` with `tail` an *unset* symbol resolves to `STEM.<TAIL>` (a
literal tail). With `tail` a *set* symbol it resolves to `STEM.<value-of-tail>`
— so reusing a map field name (`OUT.BIRTH = …` when `BIRTH` is also a local
variable) silently writes the wrong tail. The convention used by
`runtime/rexx/cust.rexx` is to give locals distinct names from map fields
(`AKT` vs `ACTION`, `CKEY` vs `CUSTNO`, `BSTR`/`NDAYS` vs `BIRTH`/`DAYS`).

---

## COBOL Syntax

A free-form COBOL interpreter (`cobol/`) sits beside REXX as a second
front end on the same `EXEC CICS` surface. Same `cics.Handler`, same
`cics.Frame` interface, same response-code semantics — the only
language-specific layer is `cobol/frame.go`, which adapts COBOL's
group-item world to the REXX-style `STEM.TAIL` lookup the CICS handlers
use. A program is a COBOL transaction iff its `transactions.conf`
entry's type column is `cobol`:

```
HELC:cobol:hello.cob:public
GUST:cobol:gust.cob:public
QAGC:cobol:qagc.cob:public,users,admin
```

`CEMT INQUIRE TRANSACTION` shows `COBOL` in the LANG column for these
entries; `COBOL` and `REXX` programs can `EXEC CICS LINK` to each
other through the same dispatcher with `DFHCOMMAREA` marshalled as
opaque bytes.

### Source shape

Free-form: no column 1-72 ruling, no Area A / Area B, hyphens allowed
in identifiers (`CUST-RECORD`). Comments use modern `*>` anywhere on a
line, or legacy `*` in column 1. Strings use `'...'` or `"..."` with
quote-doubling for embedded quotes. Hex literals like `X'F3'` and
`X"7C"` decode to their byte value — used for `IF EIBAID = X'F3'` to
check PF3 / PF12 / etc. without `C2X(EIBAID) = 'F3'` round-trips.

The four divisions parse in their canonical order:

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. HELC.

       ENVIRONMENT DIVISION.        *> optional, must be empty

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01 SCR.
          05 INFOLINE PIC X(78).
          05 GREETING PIC X(20).

       PROCEDURE DIVISION.
       MAIN.
           MOVE 'Hello' TO GREETING.
           EXEC CICS SEND MAP('HELO1') FROM(SCR) ERASE END-EXEC.
           EXEC CICS RETURN END-EXEC.
           STOP RUN.
```

`LINKAGE SECTION` is not yet parsed; `DFHCOMMAREA` is auto-injected as
`PIC X(2000)` if the program doesn't declare it (see
`cobol.ensureSystemItems`), so a sub-program can `MOVE DFHCOMMAREA TO
key` immediately. The dispatcher rstrip's trailing space when reading
the COBOL frame's `DFHCOMMAREA` back out, so a fixed-width buffer
round-trips cleanly across an inter-language `EXEC CICS LINK`.

### DATA DIVISION

* PIC clauses: `X(n)` alphanumeric, `9(n)` integer, `S9(n)` signed,
  `9(n)V99` decimal (the `V` is positional, no real binary scaling
  yet — arithmetic is float64 internally).
* `VALUE 'literal'`, `VALUE 42`, `VALUE SPACES / ZEROS / HIGH-VALUES /
  LOW-VALUES / QUOTES`.
* Group items: `01 PARENT. 05 CHILD PIC X(8). 05 OTHER PIC X(4).`
  Children are stored as offsets into a single parent buffer, so
  `MOVE` to the parent fans out and `EXEC CICS SEND MAP FROM(PARENT)`
  walks the children for field values.
* **All data names are globally unique.** `DataByName` is one map per
  program — you can't have `CUSTNO` as a child of both `SCR` and
  `DET`. The convention used by `runtime/cobol/gust.cob` is to prefix
  the second occurrence (`DCUSTNO`, `DNAME`, `DMSG`) since real COBOL
  would normally use `MOVE x OF DET` qualification, which the bricks
  parser doesn't support yet.

### PROCEDURE DIVISION

Statements supported: `MOVE`, `DISPLAY`, `STOP RUN`, `GOBACK`, `EXIT`,
`EXIT PROGRAM`, `CONTINUE` (no-op), `IF ... [ELSE] ... END-IF`,
`EVALUATE subject WHEN value [WHEN value] ... [WHEN OTHER] ...
END-EVALUATE`, `PERFORM para`, `PERFORM para UNTIL cond`, `GO TO para`,
`COMPUTE target = expr`, `ADD a TO b [GIVING c]`, `SUBTRACT a FROM b
[GIVING c]`, `STRING ... DELIMITED BY (SIZE | 'lit') INTO target
END-STRING`, `UNSTRING source DELIMITED BY 'lit' INTO t1 t2 ...
END-UNSTRING`, `EXEC CICS ... END-EXEC`.

Periods are scope-terminators, not per-statement: a statement inside
an `IF ... END-IF` body has no period; only the unscoped statement at
the paragraph level does. IBM-style postfix NOT (`X NOT = Y`,
`X NOT > Y`) is supported in `IF` / `EVALUATE` / `PERFORM UNTIL`
conditions.

`FUNCTION UPPER-CASE(item)` and `FUNCTION LOWER-CASE(item)` are the
two intrinsic functions shipped today — enough for the
`MOVE FUNCTION UPPER-CASE(ACTION) TO ACTION` idiom that lets a menu
tolerate lowercase typing without an `INSPECT REPLACING` rewrite.
More IBM intrinsics (NUMVAL, LENGTH, etc.) land in Phase 7.

`EVALUATE` only supports the simple value form. `EVALUATE TRUE` /
`EVALUATE FALSE` is rejected at parse time with a hint to rewrite as
`IF/ELSE` — those forms (and `WHEN ... THRU ...` ranges, multi-
subject `ALSO` clauses, condition-name arms) are Phase 7.

### EIB block

`EIBRESP`, `EIBRESP2`, `EIBAID`, `EIBCPOSN`, `EIBCALEN`, `EIBTRMID`,
`RC`, and `DFHCOMMAREA` are auto-injected if not declared, so most
programs need no boilerplate. After every `EXEC CICS`, `EIBRESP` /
`EIBRESP2` are populated by the handler the same way they are for
REXX. The legacy "test EIBRESP after every verb" pattern is the
recommended idiom (no `RAISING` / `SIGNAL ON ERROR` equivalent yet).

### EXEC CICS

The COBOL parser collects every token between `EXEC CICS` and
`END-EXEC` and reconstructs the body verbatim for the same
`cics.ParseCommand` REXX uses. Two consequences:

* **Map-field names must match group-child names exactly.** When
  `EXEC CICS SEND MAP('CUST1') FROM(SCR)` fires, the CICS handler
  asks the frame for `SCR.INFOLINE`, `SCR.MSG`, etc. — so SCR must
  have children spelled exactly as the map declares fields. Real
  COBOL solves this with BMS-generated copybooks; bricks doesn't ship
  those, so the operator declares fields by hand.
* **`DFHCOMMAREA` is the COMMAREA marshalling slot.** Caller passes
  bytes in via the frame; sub-program reads with `MOVE DFHCOMMAREA TO
  ...`, mutates a working copy, and `MOVE ... TO DFHCOMMAREA`. Trailing
  space is stripped by the dispatcher on the way back out.

A few `ASSIGN` options are bricks-specific so the COBOL subset can do
date math without REXX-style built-ins:

| Option | Returns |
|---|---|
| `DATE(target)` | Today as YYYYMMDD |
| `TIME(target)` | Now as HHMMSS |
| `TODAYYR(target)` / `TODAYMO(target)` / `TODAYDY(target)` | Today's year / month / day individually |
| `DAYCOUNT(target)` | Days since 1970-01-01 |

`runtime/cobol/qagc.cob` uses `TODAYYR/TODAYMO/TODAYDY` to compute age
without reference modification.

### Pre-installed Transactions

| Transid | File | Notes |
|---|---|---|
| `HELC` | `runtime/cobol/hello.cob` | Hello-world; SEND MAP('HELO1') FROM(SCR), RETURN. Smallest end-to-end demo. |
| `QAGC` | `runtime/cobol/qagc.cob` | COBOL twin of `QAGR` (REXX). Validates the QAGE1 birthdate, computes age in years and approximate days, sends QAGR1. Pseudo-conversational redisplay of QAGE1 on validation errors. |
| `GUST` | `runtime/cobol/gust.cob` | Cut-down COBOL `CUST`. A=Add, Q=Query, D=Delete; U/L/S print a "Phase 7" stub message because they need STRING/UNSTRING/INSPECT. |
| `GUSV` | `runtime/cobol/gusv.cob` | COBOL twin of `CUSV`. Validates the customer-number COMMAREA (LINK target). |
| `GUSL` | `runtime/cobol/gusl.cob` | COBOL twin of `CUSL`, page-1-only. Demonstrates STARTBR/READNEXT/ENDBR; paging + filtering need PERFORM VARYING + dynamic field assignment (Phase 7). |

### What's deferred (Phase 7)

* `LINKAGE SECTION` end-to-end (right now `DFHCOMMAREA` is auto-injected
  in WORKING-STORAGE; a real LINKAGE SECTION would let an operator
  declare per-program COMMAREA shapes).
* `INSPECT TALLYING / REPLACING`, `POS` (substring search), and the
  rest of the FUNCTION library (NUMVAL, LENGTH, TRIM, etc.) beyond
  UPPER-CASE / LOWER-CASE. Substring search is what blocks GUSL's
  filter-by-search-term path; today GUST's `S` action prints a
  REXX-only message rather than degrading silently.
* Reference modification (`DATE-FIELD(1:4)` for substring access).
* `PERFORM N TIMES` / `PERFORM VARYING ... FROM ... BY ... UNTIL ...`
  and `OCCURS` arrays — needed for clean paginated browses and
  dynamic field assignment. GUSL's row 1..15 fan-out is unrolled
  by hand today.
* `MULTIPLY / DIVIDE`, `ROUNDED`, `ON SIZE ERROR` (the parser
  accepts and ignores `ROUNDED` / `ON SIZE ERROR` today).
* Group-item qualification (`MOVE x OF DET TO y OF SCR`) — until
  this lands, group children must have unique names across the
  program.
* SCREENHT-based map family suffix (e.g. `CUST1L` on a mod-4 screen).
  REXX programs do this with a runtime `IF SCRH >= 43 THEN ...`
  fallback after a MAPFAIL; the COBOL twins always render the
  unsuffixed mod-2 maps for now.

---

## EXEC CICS commands

Bricks accepts two equivalent surface forms inside an `ADDRESS CICS` scope:

**1. IBM-canonical `EXEC CICS … END-EXEC`** (the form experienced CICS
programmers expect, matching the COBOL/PL/I/REXX-on-CICS reference):

```rexx
ADDRESS CICS

EXEC CICS ASSIGN USERID(USR) TERMID(TRM) CONNECTED(CT) END-EXEC

EXEC CICS SEND MAP('HELO1')
              FROM(SCR.)
              ERASE
END-EXEC

EXEC CICS RETURN END-EXEC
```

A small preprocessor (`rexx/preprocess.go`, called from `rexx.Parse`) rewrites
each `EXEC CICS … END-EXEC` block into the equivalent quoted-string command
before lexing. Multi-line bodies are collapsed to a single command string;
trailing newlines are inserted to keep error line numbers honest. Comments
and string literals are left alone.

**2. Bare-string-under-`ADDRESS CICS`** (terser, identical semantics):

```rexx
ADDRESS CICS
"ASSIGN USERID(USR) TERMID(TRM)"
"SEND MAP('HELO1') FROM(SCR.) ERASE"
"RETURN"
```

Both forms dispatch through the same path: `cics.ParseCommand` parses the
verb plus options/flags, the matching handler runs, and `EIBRESP` /
`EIBRESP2` / `RC` are set after every call.

The supported verbs and options are listed below — implemented in
`cics/handler.go` (dispatch) and `cics/files.go` / `cics/ts.go`.

| Verb                                     | Notes |
|------------------------------------------|-------|
| `SEND MAP(name) [FROM(stem.)] [ERASE] [CURSOR(pos)]` | Loads the named map, fills named fields from the REXX stem, paints the screen. Captures the response on the TCB for `RECEIVE MAP`. |
| `RECEIVE MAP(name) [INTO(stem.)]`        | Pulls the response stored by `SEND MAP` into a stem (`MAP.<field>` by default). `MAPFAIL` if no prior matching SEND. |
| `RETURN [TRANSID(id)] [COMMAREA(data)]`  | Sets `tcb.NextTransid` / `tcb.Commarea` and ends the task. |
| `ASSIGN <FIELD>(var) …`                  | Reads EIB / session fields into REXX vars: `USERID`, `TERMID`, `EIBAID`, `EIBCPOSN`, `EIBCALEN`, `ALTSCRNHT`/`ALTSCRNWD`/`SCREENHT`/`SCREENWD`. |
| `ABEND [ABCODE(code)] [NODUMP]`          | Ends the task with the supplied code. |
| `XCTL PROGRAM(name) [COMMAREA(d)]`       | Transfer of control (no return). |
| `LINK PROGRAM(name) [COMMAREA(var\|'lit')]` | Synchronous sub-program call. The target program (resolved through `transactions.conf`) runs in a fresh REXX frame with `DFHCOMMAREA` pre-loaded; on return, the sub-program's final `DFHCOMMAREA` is written back to the caller's `COMMAREA(var)`. Caller state (`NextTransid`, `Commarea`, `LastResponse`/`LastMapName`) is saved and restored so the LINK is transparent. |
| `READ FILE(f) {INTO\|SET}(var) RIDFLD(k) [UPDATE] [LENGTH(var)]` | Reads a record from the KSDS by key (B+tree lookup, O(log n)). `UPDATE` records a per-session lock that gates a subsequent `REWRITE`. When `LENGTH(var)` is a bare REXX variable, the actual record length is written back. |
| `WRITE FILE(f) FROM(var) RIDFLD(k)`      | Creates a record. `DUPREC` (RESP=14) if a record with the same key already exists. The bucket is created implicitly on first WRITE — no DEFINE FILE needed. |
| `REWRITE FILE(f) FROM(var)`              | Replaces the record locked by the most recent `READ … UPDATE` on the same FILE. INVREQ if there's no prior READ UPDATE. Releases the per-FCB update lock. |
| `DELETE FILE(f) [RIDFLD(k)]`             | Removes a record. Uses RIDFLD if supplied, otherwise the key from the most recent READ UPDATE. |
| `STARTBR FILE(f) [RIDFLD(start)] [GTEQ\|EQUAL] [GENERIC] [KEYLENGTH(n)]` | Opens a B+tree browse cursor on the file (bbolt MVCC read tx — sees a stable snapshot). With no RIDFLD, positions on the first key. With RIDFLD + GTEQ (default) on the first key ≥ start. With EQUAL, requires an exact match (NOTFND if absent). With GENERIC + KEYLENGTH(n), positions on and walks only keys whose first n bytes match the first n bytes of RIDFLD. |
| `READNEXT FILE(f) INTO(var) RIDFLD(var) [LENGTH(var)]` | Forward step on the open browse. Writes the record into INTO(var), the matching key back to RIDFLD(var), and the actual length to LENGTH(var) when each is a bare REXX variable. Returns `ENDFILE` (RESP=20) past the last key (or past the GENERIC prefix). |
| `READPREV FILE(f) INTO(var) RIDFLD(var) [LENGTH(var)]` | Backward step on the open browse. Same writeback rules; ENDFILE before the first key. |
| `RESETBR FILE(f) RIDFLD(start) [GTEQ\|EQUAL] [GENERIC] [KEYLENGTH(n)]` | Repositions an open cursor without closing the read tx. Cheaper than ENDBR + STARTBR when a program wants to jump within the same browse session. |
| `ENDBR FILE(f)`                          | Releases the browse cursor and the underlying bbolt read transaction. The dispatcher also releases any cursor the program forgot to ENDBR via a `defer handler.CloseBrowses()`. |
| `READQ TS QUEUE(q) INTO(var) [ITEM(n)] [NEXT] [NUMITEMS(var)] [LENGTH(var)]` | Reads a TS queue item. With no `ITEM` (or with `NEXT`), advances the **per-task implicit cursor**: the first cursor-less READQ on a queue returns item 1, the second returns item 2, etc. — IBM TRL semantics. The cursor is keyed on the running TxCB and released when the task ends, so a fresh `CONS` invocation starts at item 1 again. `QNAME` is accepted as a synonym for `QUEUE`. |
| `WRITEQ TS QUEUE(q) FROM(var) [ITEM(n) REWRITE]` | Appends an item, or rewrites item `n`. Items are stored in a bbolt sub-bucket per queue (8-byte big-endian item number → payload), so a single queue scales to millions of items without filesystem-directory pressure and reads/writes are O(log N). Append uses an in-memory high-water-mark counter (no scan); writes commit in one bbolt transaction so a crash mid-write leaves either the prior state or the new one. When `ITEM(var)` is a bare variable on append, the assigned item number is stored back. `QNAME` is accepted as a synonym for `QUEUE`. |
| `DELETEQ TS QUEUE(q)`                    | Drops the queue's sub-bucket and resets in-memory counters / cursors. |

`READQ TD` / `WRITEQ TD` / `DELETEQ TD` are parsed (the verb has a `TD`
sub-form) but explicitly rejected with an `INVREQ` and a clear error
message — bricks only implements TS today, and silently routing TD to
TS would mask a real semantic difference (transient data has one-shot
read-and-destroy semantics that TS doesn't).

**TS queue counters and CEMT visibility.** Every queue carries
in-memory counters for reads, writes, rewrites, and a `LastAccess`
timestamp; `CEMT INQUIRE TS` (or `CEMT I S`) renders all of them
plus the live item count. The screen refreshes on every ENTER:

```
QUEUE   ITEMS  READS  WRITES  REWRT  LASTACC      STATUS
BENCHQ  1234   980    1234    2      18:42:11.084 ENABLED
```

**Worked examples:** `runtime/rexx/prod.rexx` is a producer that does
`WRITEQ TS QUEUE(QNAME) FROM(PAYLOAD)` from an interactive PROD1 map;
`runtime/rexx/cons.rexx` is a consumer that loops over the cursor-
advancing READQ, with PF4 to `DELETEQ` and PF5 to rewind by chaining
back to itself (so the dispatcher's task-end hook clears the cursor).
Both are conversational: the screen stays up across operator inputs.

```rexx
/* PROD: write one item per ENTER */
ADDRESS CICS
DO FOREVER
  EXEC CICS SEND MAP('PROD1') FROM(SCR.) ERASE END-EXEC
  EXEC CICS RECEIVE MAP('PROD1')           END-EXEC
  IF C2X(EIBAID) = 'F3' THEN EXEC CICS RETURN END-EXEC
  EXEC CICS WRITEQ TS QUEUE(MAP.QNAME) FROM(MAP.PAYLOAD) END-EXEC
END
```

```rexx
/* CONS: cursor-advancing read */
ADDRESS CICS
DO FOREVER
  EXEC CICS SEND MAP('CONS1') FROM(SCR.) ERASE END-EXEC
  EXEC CICS RECEIVE MAP('CONS1')           END-EXEC
  IF C2X(EIBAID) = 'F3' THEN EXEC CICS RETURN END-EXEC
  EXEC CICS READQ TS QUEUE(QNM) INTO(REC) ITEM(GOTI) END-EXEC
  IF EIBRESP = 26 THEN /* ITEMERR — end of queue */ NOP
END
```

`FILE(...)` and `QUEUE(...)` names are validated against
`^[A-Za-z0-9_-]{1,64}$` before any bucket / path is composed; an invalid name
yields `INVREQ` with a clean error message. See
[Performance and security hardening](#performance-and-security-hardening).

Response codes are the IBM-standard subset in `cics/resp.go`
(`NORMAL=0`, `NOTFND=13`, `DUPREC=14`, `INVREQ=16`, `IOERR=17`, `ENDFILE=20`,
`ITEMERR=26`, `PGMIDERR=27`, `MAPFAIL=36`, `QIDERR=44`, …).

---

## How file storage works

CICS FILEs in bricks are **KSDS** (key-sequenced data sets), backed by a
single embedded B+tree database (`go.etcd.io/bbolt`) at
`data/files.boltdb`. Each CICS FILE is one bbolt **bucket** inside the
shared database; user-supplied keys map directly to the raw record bytes.

```
data/
    files.boltdb          ← single B+tree file
        bucket "CUSTOMERS"
            "00100" → "Alice Adams|123 Main St|Springfield, NY|212-555-0100"
            "00101" → "Bob Brooks|45 Elm St|Riverton, CA|415-555-0101"
            …
        bucket "ACCOUNTS"
            "A0001" → … (each app picks its own record format)
        bucket "_catalog"
            "CUSTOMERS" → JSON{records:250, key_max:6, rec_max:80, …}
            "ACCOUNTS"  → JSON{…}
```

Properties of the KSDS:

* **Record bodies are opaque.** Bricks does not impose any internal
  structure on a record — applications choose their own layout (separator-
  delimited, fixed-width, packed, JSON, raw EBCDIC). The example above
  uses `name|addr|city|phone` because *that's the application's choice*;
  bricks stores those bytes verbatim.
* **B+tree index.** READ by exact key is O(log n). STARTBR positions on
  any key in O(log n) and READNEXT walks in B+tree order in O(1) per step.
* **MVCC snapshot reads.** STARTBR opens a bbolt read transaction; the
  cursor walks a stable point-in-time view, so concurrent WRITE/REWRITE/
  DELETE on the same FILE don't disturb an in-progress browse.
* **Atomic writes.** WRITE/REWRITE/DELETE run inside a bbolt write
  transaction with `fsync` on commit; partial updates are never visible
  on disk after a crash.
* **Implicit DEFINE.** First WRITE to a FILE creates its bucket. There
  is no `EXEC CICS DEFINE FILE` step.
* **Per-FILE metadata.** A `_catalog` bucket tracks record count,
  last-modified, max key length, max record length, and creation time,
  so `CEMT INQUIRE FILE` shows accurate numbers without scanning the
  data bucket. The catalog is bricks-internal — REXX programs never see
  it.
* **Initial mmap.** bbolt is opened with a 4 MiB initial mmap so a
  long-running browse cursor doesn't deadlock against a write that needs
  to grow the file. Demo workloads (a few thousand records) never hit
  the grow path.

What this means for `EXEC CICS READ` / `WRITE` / `REWRITE` / `DELETE`:

| Verb | What bricks does on disk |
|------|-------------------------|
| `READ FILE('CUSTOMERS') INTO(REC) RIDFLD(K)` | One bbolt `View` tx, one B+tree lookup. The record bytes (whatever the app stored) come back into REC unchanged. |
| `WRITE FILE('CUSTOMERS') FROM(REC) RIDFLD(K)` | One bbolt `Update` tx: B+tree insert, `_catalog` bookkeeping, fsync. DUPREC if the key is already present. |
| `REWRITE FILE('CUSTOMERS') FROM(REC)` | Update tx that overwrites the value at the key locked by the prior READ UPDATE. Releases the per-FCB update lock at end of tx. |
| `DELETE FILE('CUSTOMERS') RIDFLD(K)` | Update tx that deletes the bucket entry; `_catalog` record count drops by one. |

What this means for `STARTBR / READNEXT / READPREV / RESETBR / ENDBR`:

```
EXEC CICS STARTBR FILE('CUSTOMERS') RIDFLD('NY-')
                  GENERIC KEYLENGTH(3) END-EXEC
DO FOREVER
  EXEC CICS READNEXT FILE('CUSTOMERS') INTO(REC) RIDFLD(K) RESP(R) END-EXEC
  IF R = DFHRESP(ENDFILE) THEN LEAVE
  SAY K ':' REC
END
EXEC CICS ENDBR FILE('CUSTOMERS') END-EXEC
```

* STARTBR opens a bbolt read tx + cursor; positions on the first key
  whose first 3 bytes are `NY-`.
* READNEXT advances the cursor; with GENERIC active, returns `ENDFILE`
  the moment the prefix breaks (no full-file scan).
* READPREV walks backward from current position with the same prefix
  rule. Useful for paginating backwards through a key range.
* RESETBR repositions the cursor without closing the tx — cheaper than
  ENDBR + STARTBR when a program jumps inside the same browse session.
* ENDBR commits-rollback the read tx and releases its MVCC snapshot.

Pre-load 250 sample customers for the CUST transaction:

```sh
go run ./cmd/seed-customers
```

The seeder is idempotent; re-running adds only the missing rows.

---

## Adapting to terminal size (mod 2 vs mod 4)

A 3270 connection negotiates one of several screen models — typically
mod 2 (24 × 80) or mod 4 (43 × 80). Bricks captures the size from the
telnet/3270 handshake into `session.TCB.Rows/Cols`, and exposes it to
REXX programs via `EXEC CICS ASSIGN`:

```rexx
EXEC CICS ASSIGN SCREENHT(SCRH) SCREENWD(SCRW)
                 ALTSCRNHT(AH)  ALTSCRNWD(AW)  END-EXEC
```

`SEND MAP` always passes the negotiated `DevInfo` through to
`go3270.ScreenOpts.AltScreen`, so the underlying datastream uses
Erase/Write Alternate (`0x7e`) and the terminal clears its full
buffer — but a 24-row map painted on a 43-row screen leaves rows
24-42 blank. To use the extra real estate the program has to dispatch
to a sized map variant.

**Convention.** Author one map per model. The mod-2 map keeps its bare
name; bigger models add a single-letter suffix:

| Model | Suffix | Example map names                |
|-------|--------|----------------------------------|
| mod 2 | (none) | `HELO1`, `CUST1`, `CUST2`, `CUSTL` |
| mod 3 | `M`    | `HELO1M`, `CUST1M`, …            |
| mod 4 | `L`    | `HELO1L`, `CUST1L`, `CUST2L`, `CUSTLL` |
| mod 5 | `W`    | `HELO1W`, …                      |

The REXX program builds the suffixed name once and dispatches:

```rexx
EXEC CICS ASSIGN SCREENHT(SCRH) END-EXEC
SUFFIX = ''
IF SCRH >= 43 THEN SUFFIX = 'L'
ELSE IF SCRH >= 32 THEN SUFFIX = 'M'

EXEC CICS SEND MAP('HELO1' || SUFFIX) FROM(SCR.) ERASE END-EXEC
IF EIBRESP = 36 THEN DO            /* MAPFAIL — sized variant missing */
  EXEC CICS SEND MAP('HELO1') FROM(SCR.) ERASE END-EXEC
END
```

Three properties make this work without any DSL changes:

1. **Same field names across the family.** `helo1.map` and `helo1l.map`
   both declare `INFOLINE`, `GREETING`, `FOOTER`. The same `SCR.` stem
   feeds either one. Bonus tails (e.g. `INFO1` / `INFO2` / `ACT1` …) are
   silently ignored on the smaller map (the renderer only writes values
   for fields that map declares).
2. **MAPFAIL fallback.** If the suffixed map isn't on disk, the SEND
   returns `EIBRESP = 36` and the program retries with the bare name —
   so an operator who deletes `helo1l.map` doesn't break mod-4
   connections; they just see the 24×80 layout.
3. **Paging arithmetic adapts at runtime.** `cusl.rexx` reads `SCREENHT`
   and uses `ROWS_PER_PAGE = 35` on mod 4 vs `15` on mod 2, then picks
   `CUSTLL` vs `CUSTL` accordingly.

Bricks ships sized variants for every map the demo transactions use:

| Mod-2 map (24×80) | Mod-4 sibling (43×80) |
|-------------------|------------------------|
| `runtime/map/helo1.map` | `runtime/map/helo1l.map` — adds system-information + recent-activity panes |
| `runtime/map/cust1.map` (menu) | `runtime/map/cust1l.map` — adds recent-activity history |
| `runtime/map/cust2.map` (detail) | `runtime/map/cust2l.map` — adds an audit-log pane |
| `runtime/map/custl.map` (15-row list) | `runtime/map/custll.map` (35-row list) |

To see the difference: connect with `c3270 -model 2 localhost 2300`
vs. `c3270 -model 4 localhost 2300`, sign on, and run `HELO` or
`CUST`. The mod-4 view fills the bottom three-quarters of the screen
with extra panels.

---

## API for embedders

```go
// 1. Configure.
cfg, _ := config.Load("bricks.cnf")

// 2. Wire dependencies.
users, _   := auth.LoadFile(cfg.UsersFile)
maps, _    := mapdsl.ParseDir(cfg.MapsDir)
table, _   := txn.LoadTable(cfg.TransactionsFile, cfg.RexxDir)
registry   := session.NewRegistry()
store, err := cics.NewStore(cfg.DataDir, registry)  // opens data/files.boltdb
if err != nil { log.Fatal(err) }
defer store.Close()
dispatch   := txn.NewDispatcher(table, maps, store, registry, cfg.Banner)

// 3. For each accepted connection, build a TCB and drive it manually.
tcb := &session.TCB{Conn: conn, Dev: dev, TermID: session.NextTermID(), …}
registry.AddTerminal(tcb)
defer registry.RemoveTerminal(tcb)

tn3270.ShowLogoSplash(conn, dev, "bricks.logo")
auth.RunCSSN(tcb, registry, users, cfg.MapsDir, cfg.EnforceSecureLogin, idle)

tcb.NextTransid = "MENU"
dispatch.Run(tcb)
```

For an `EXEC CICS` handler that talks to your own backend rather than the
disk-backed store, replace `cics.New(tcb, maps, store)` with an
implementation of `rexx.AddressHandler` and pass it via
`rexx.Options.Addresses`.

---

## CLI utilities

| Command                           | Purpose |
|-----------------------------------|---------|
| `./bricks --conf=bricks.cnf`      | Run the server. |
| `./bricksload --help`             | Stress-test bricks; live dashboard + final report. See [Stress testing](#stress-testing--bricksload). |
| `go run ./cmd/seed-customers`     | Idempotently load 250 sample customer records into the CUSTOMERS KSDS. |
| `go run ./cmd/brickspw <pw>`      | Print a bcrypt hash for a password. |
| `./add_brick_user.bash <u> <p> [groups]` | Add a user (refuses duplicates without `--update`). |
| `./add_brick_user.bash --update <u> <p> [groups]` | Replace an existing user's hash/groups. |
| `./build.bash`                    | Cross-compile `bricks`, `bricksload`, `brickspw` for linux/amd64, linux/386, linux/armv7, freebsd/amd64 into `bin/` (CGO_ENABLED=0). |

---

## Testing

```sh
go test ./...
```

Per-package coverage is small but focused:

* `mapdsl` — parser table tests.
* `tn3270` — render maps the on-disk `.map` files to `go3270.Field` slices.
* `auth` — bcrypt round-trip, malformed/duplicate rejection, unknown-user
  timing.
* `rexx` — lexer, IF/SELECT/DO, stems, PROCEDURE/EXPOSE, PARSE templates,
  ADDRESS commands, the canonical HELO sample end-to-end.
* `cics` — command parser, file store round-trip (DUPREC, NOTFND), TS queue
  append/rewrite/delete.

---

## Stress testing — `bricksload`

`cmd/bricksload` is a TN3270 stress tester that opens many concurrent
sessions, runs a scripted flow on each, and reports throughput,
latency percentiles, and bricks-side process metrics in real time.
It speaks the actual TN3270 protocol (telnet negotiation + 3270 data
streams) by reusing `web3270/client.go` — there's no JSON shortcut,
so what bricks sees from `bricksload` is indistinguishable from a
real `c3270` / `x3270` client.

### Building

```sh
./build.bash                               # produces bin/bricksload-<ver>-<os>-<arch>
go build -o bricksload ./cmd/bricksload    # quick local binary
```

### Quick start

```sh
# Full live dashboard (default), CUST query flow, 8 × 2,000 = 16,000 iterations.
./bricksload -clients=8 -iterations=2000 -flow=cust-q -refresh-ms=200

# Pipe-friendly: no dashboard, plain text report goes to a logfile.
./bricksload -no-dashboard -clients=50 -iterations=10000 -flow=splash > run.log

# JSON output, ready for jq / a downstream consumer.
./bricksload -out=json -clients=8 -iterations=500 -flow=cust-q | jq
```

### Flags

| Flag             | Default                            | Purpose |
|------------------|------------------------------------|---------|
| `-host`          | `localhost`                        | TN3270 host |
| `-port`          | `2300`                             | TN3270 port |
| `-clients`       | `10`                               | concurrent sessions |
| `-iterations`    | `100`                              | per-client iteration count |
| `-flow`          | `splash`                           | `splash` \| `cust-q` \| `qage` \| `prod` \| `cons` |
| `-userid`        | `""`                               | optional CSSN userid (sign-on once before iterations) |
| `-password`      | `""`                               | paired with `-userid` |
| `-warmup`        | `0`                                | iterations to discard from latency stats |
| `-timeout`       | `30s`                              | per-iteration wallclock cap (Go duration: `5s`, `250ms`…) |
| `-metrics-url`   | `http://localhost:9000/metrics`    | bricks `/metrics` endpoint URL (empty → skip server-side block) |
| `-poll-ms`       | `500`                              | `/metrics` poll cadence in milliseconds (≥50) |
| `-refresh-ms`    | `500`                              | dashboard redraw cadence in milliseconds (≥50) |
| `-no-dashboard`  | `false`                            | suppress live UI; print only the final report |
| `-out`           | `text`                             | `text` (live + final) \| `json` (skip dashboard, emit one JSON object) |

### Flows

* **`splash`** — one iteration is a complete connection lifecycle:
  dial → telnet negotiate → wait for splash → AID Enter → wait for
  prompt → close. Pure connection-handling benchmark; drives 0
  EXEC CICS verbs because the splash + blank-prompt screens are
  rendered Go-side (`tn3270.ShowLogoSplash` / `BlankPrompt`) and never
  enter REXX dispatch.

* **`cust-q`** — one iteration is one full `CUST → Q → 100` round-trip
  on an already-open session: send `CUST` ENTER → wait CUST1; send
  action `Q` + key `100` ENTER → wait CUST2 with the customer record;
  ENTER → wait CUST1 with `Query of 100 complete.` MSG; F12 → wait
  blank prompt. About 11 EXEC CICS dispatches per iteration (ASSIGN +
  SEND/RECEIVE pairs across CUST/CUST2 menus + LINK to CUSV + READ
  FILE + RETURN). Pass `-userid`/`-password` if `enforce_secure_login=yes`.

* **`qage`** — one iteration is `QAGE → birthdate → QAGR result` over
  the pseudo-conversational chain (`RETURN TRANSID('QAGR')` +
  `RECEIVE MAP` from prior SEND). 6 EXEC CICS dispatches per iteration:
  ASSIGN + SEND + RETURN in qage.rexx, ASSIGN + RECEIVE + SEND +
  RETURN in qagr.rexx. CEMT INQ TR shows the `QAGE` and `QAGR` invocation
  counts climbing in lockstep — handy for verifying the chain works.

* **`prod`** — TS queue producer. Drives the conversational `PROD`
  transaction: type the queue name + a per-iteration payload, ENTER,
  wait for the redisplayed map showing `Wrote item N`. 3 EXEC CICS
  dispatches per iteration (SEND + RECEIVE + WRITEQ TS). Hardcoded to
  queue `BENCHQ`; payloads are `c<client>-i<iter>` so items are
  uniquely traceable to the iteration that wrote them. Pair with
  `cons` (see below) running on a different bricksload invocation, or
  watch `CEMT I S` while it runs to see writes accumulate.

* **`cons`** — TS queue consumer. Drives the conversational `CONS`
  transaction: ENTER advances the implicit per-task cursor and reads
  the next item. 3 EXEC CICS dispatches per iteration (SEND + RECEIVE
  + READQ TS). Useful pattern: run `prod` to fill `BENCHQ`, then run
  `cons` to drain it. Iterations after the queue is exhausted still
  count as successes (the round-trip happened, the screen just shows
  `End of queue (ITEMERR)`); use `-iterations` close to the producer
  count to avoid a tail of empty reads.

  Example end-to-end TS workout:

  ```
  ./bricksload -flow=prod -clients=4 -iterations=5000   # ~20K items into BENCHQ
  ./bricksload -flow=cons -clients=4 -iterations=5000   # drain
  ```

  Watch `CEMT I S` between runs — `READS`, `WRITES`, and `LASTACC`
  for `BENCHQ` reflect what just happened.

### Live dashboard

Pure ANSI redraw — same approach as the `console.go` operator console.
Hidden cursor, in-place redraw every `-refresh-ms`, restored on exit.

```
bricksload — running                                    elapsed 19.7s   eta 22.3s
═══════════════════════════════════════════════════════════════════════════════
Target:      localhost:2300                Flow:    cust-q
Clients:     50    iter done 23,481   failed 2
Progress:    23,483 / 50,000 iter (47%)   total tx ≈ 258,313

Throughput:    last 5s   1,194 iter/s    run-to-date  1,191 iter/s
Latency 5s:    p50  18ms   p95  64ms   p99  121ms        max 287ms
Latency all:   p50  18ms   p95  63ms   p99  118ms

Errors:        timeout 2     screen_mismatch 0     connect_fail 0     proto 0

bricks process (sampled every 500ms via /metrics)
─────────────────────────────────────────────────────────────────────────────
  Heap   12.4 → 39.1 MB   peak 41.2 MB     Sys      45.1 → 78.4 MB
  Goroutines  7 → 105  peak 108            GC runs  Δ 18   last pause 7.4 ms
  CPU Δ        user 3.8 s   sys 0.4 s
  EXEC CICS    Δ 258,313   (harness 258,313 ✓)
  CICS txn/s   last 5s  1,194    avg  1,191
  EXEC CICS/s  last 5s 13,134    avg 13,101
  Per-verb Δ   SEND 70,449  RECEIVE 70,449  RETURN 46,966  ASSIGN 23,483  LINK 23,483

[Ctrl+C: graceful abort — clients drain, report prints]
```

The two rate lines (`CICS txn/s`, `EXEC CICS/s`) are computed from a
rolling history of `/metrics` snapshots:

* **`avg`** — `(latest.total − start.total) / Δuptime`. Stable
  run-to-date number.
* **`last 5s`** — pulls the oldest sample at least 5 s ago and
  computes `(latest − old) / Δt`. Reflects current load, not the
  warmup period.

The cross-check on the `EXEC CICS Δ` row (`harness X ✓ / ≈`) compares
`exec_cics.total` from `/metrics` against the harness's own iteration
count × `txPerIter()` for the active flow. ✓ means perfect match;
`≈` means within tolerance (counts shift slightly due to timed-out
iterations and the moment of the final snapshot).

### `/metrics` endpoint (`bricks/metrics`)

Bricks exposes a JSON snapshot at `http://<host>:<metrics_port>/metrics`
— the same numbers `CEMT → M` (MONITOR) shows on the 3270, but
machine-readable. Default port `9100`; gated on `start_metrics=yes`
(which is the default). Independent of `start_web3270` — turning the
browser frontend off does **not** turn metrics off. When both
`start_web3270=yes` and `start_metrics=yes` are set, the `/metrics`
route is mounted on both ports; either works.

```json
{
  "uptime_seconds": 3812.4,
  "memory":   { "heap_alloc_bytes": 12998144, "sys_bytes": 47185920, "heap_objects": 12345 },
  "gc":       { "num": 12, "last_pause_ns": 1234567, "total_pause_ns": 18234567 },
  "cpu":      { "user_seconds": 2.50, "sys_seconds": 0.30 },
  "runtime":  { "goroutines": 7, "num_cpu": 8, "go_version": "go1.25.0" },
  "registry": {
    "active_terminals": 2, "signed_on_users": 1,
    "active_transactions": 1, "known_files": 1,
    "accepts": 100, "rejects": 0, "auth_success": 50, "auth_failure": 3,
    "total_txn_run": 200, "total_txn_failed": 0
  },
  "exec_cics": {
    "total": 1234,
    "by_verb": { "SEND": 312, "RECEIVE": 312, "READ": 140, "ASSIGN": 80, "LINK": 33 }
  },
  "wallclock_unix": 1747250000
}
```

Counters (`accepts`, `total_txn_run`, `exec_cics.total`, …) are
**absolute since process start**. Memory and GC fields are
**snapshot values**. To compute rates, take two snapshots and divide
the delta by the time delta — that's exactly what `bricksload`'s
dashboard does.

`exec_cics.by_verb` maps verb name → count, sourced from
`cics.ExecPerVerb()`. Only verbs that have actually been dispatched
appear.

### Final report

When the run ends (clients drain or Ctrl+C), the final report prints
below the dashboard area. Same fields, plus `min/p50/p95/p99/max`
latency, the per-class error breakdown, and the bricks-process
deltas (heap start→end→peak, CPU Δ, GC Δ, CICS-txn rate,
EXEC-CICS-verb rate). `-out=json` emits all of this as a single JSON
object — see `cmd/bricksload/report.go::Report` for the schema.

### Operational notes

* **`max_conns_per_ip`** defaults to `8`. Running `-clients > 8` from
  one IP produces `connect_fail` rejections for the surplus
  connections. Raise it in `bricks.cnf` (e.g. `max_conns_per_ip=200`)
  and **restart bricks** — this key is read once at startup and is
  not on the live-reload list.
* **`enforce_secure_login=yes`** blocks every TRANSID until CSSN sign-on
  succeeds. The `cust-q` flow will fail at the auth gate unless you
  pass `-userid` / `-password` so the harness signs on once at session
  start before the iteration loop.
* The dashboard uses ANSI escape codes; `-no-dashboard` is required
  when redirecting stdout to a file or pipe (`-out=json` implies it).

---

## Performance and security hardening

### Parsed-program cache

Every TRANSID dispatch and every `EXEC CICS LINK PROGRAM(...)` resolves
through `Table.LoadProgram(tx)` (`txn/transactions.go`), which caches the
parsed `*rexx.Program` keyed by file path and `mtime`. The first reference
runs preprocess + lex + parse and stores the AST; subsequent references
short-circuit with an `RWMutex` RLock + mtime equality check. Edits to a
program file are picked up on the next dispatch automatically.

### Resource-name validation

FILE and QUEUE names that flow from a REXX program into the on-disk store
are validated by `validResourceName` against `^[A-Za-z0-9_-]{1,64}$`
before any path is composed (`cics/store.go`). Invalid names yield
`INVREQ` with a clean error string. Without this, `WRITE FILE('../tmp/X')`
would have escaped `data_dir` because `filepath.Join` collapses `..`
segments.

### Idle read deadlines on prompt screens

Each top-level prompt (`BlankPrompt`, `LogonPrompt`) sets
`conn.SetReadDeadline(now + idle_timeout_secs)` before reading and clears
the deadline after a successful read (`main.go::handle`). A peer that
completes telnet negotiation but never sends a screen response is
dropped, freeing its `max_conns_per_ip` slot.

### CSSF LOGOFF cleanup

`CSSF LOGOFF` calls `Registry.DetachUserFromTerminal(tcb)` which severs
the UCB↔TCB link and drops the UCB if the terminal set is empty. The
prior session no longer leaves an orphan UCB visible in `CEMT → I → C → U`,
and a subsequent `CSSN` for a different userid does not create a
duplicate UCB.

### Registry locking

The session registry no longer uses a single global `RWMutex`. `tcbs`,
`ucbs`, and `fcbs` each have their own per-collection mutex; `txcbs`
moved to `sync.Map` with an `atomic.Int64` count, so `BeginTxn` /
`EndTxn` are lock-free in the steady state. CEMT snapshots no longer
block transaction starts. Lock order when more than one is needed:
`tMu` → `uMu` → per-block locks (`u.mu`, `t.mu`).

### Lock-free EXEC CICS metrics

Per-verb counters (`cics.ExecPerVerb`) are stored in a `sync.Map` of
`*atomic.Int64`. The hot path is lock-free for any verb already seen;
only the first sighting of a new verb pays a `LoadOrStore`. The
EXEC CICS command parser pre-allocates its token slice
(`make([]ctok, 0, 16)`) so the dispatch loop avoids slow-start growth.

### Bounded browse reconciliation

`READNEXT FILE` skips records that were deleted by a concurrent
transaction between the `STARTBR` snapshot and the read. The skip is
implemented as a bounded forward loop over the snapshot
(`cics/files.go::readNextFile`), replacing an unbounded recursion, so no
amount of in-flight deletes can blow the goroutine stack.

---

## Roadmap

* SQL — bricks does not embed a database; data records are simple files.
* RACF / Top Secret / LDAP — `auth.Authenticator` is the seam to plug those
  in later.
* Distributed CICS features (DPL, ISC/MRO, queued transactions across
  regions).
  could be added if real BMS sources need to be imported.
