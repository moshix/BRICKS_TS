# Bricks ISPF Editor — User Manual

The `ISPF` transaction is a built-in source editor that lets operators
browse and edit the REXX, COBOL, and BMS-map source trees from a 3270
session. It models the classic IBM ISPF editor: a command line at the
top, a 6-byte prefix area on every content row for line commands, and
PF-key shortcuts.

Most mainframers are very familiar with this editor or with the very
simialr REVEDIT editor. 

## Contents

- [Access](#access)
- [Menu](#menu)
- [File browser](#file-browser)
- [Editor screen layout](#editor-screen-layout)
- [PF keys](#pf-keys)
- [Command-line commands](#command-line-commands)
- [Line-prefix commands](#line-prefix-commands)
- [Block commands](#block-commands)
- [Save flow and syntax validation](#save-flow-and-syntax-validation)
- [Multi-file editing](#multi-file-editing)
- [Edit locks](#edit-locks)
- [Syntax highlighting](#syntax-highlighting)
- [Limitations](#limitations)

---

## Access

`ISPF` is a built-in TRANSID. It does **not** appear in
`runtime/transactions.conf`; it's dispatched by name from the bricks
core. To use it you must:

1. Sign on with `CSSN` (the bricks logon transaction).
2. Belong to the `dev` group in `runtime/users.conf`.

Non-authenticated users, and users who now in the dev group will not
be able to call this transaction.

Type `ISPF` at the blank prompt and press ENTER. Non-`dev` users see
`ISPF requires DEV group membership.` and are returned to the prompt.

---

## Menu

The menu screen offers three editing areas:

```
------------------------ ISPF EDIT UTILITY ----------------------------
OPTION  ===>  _

   1  -  Edit REXX programs        (runtime/rexx)
   2  -  Edit COBOL programs       (runtime/cobol)
   3  -  Edit BMS map files        (runtime/map)
   4  -  Edit PL/I files           (runtime/pli)
   
Press ENTER after typing the option number, or F3 to exit ISPF.
...
                                                          F3=Exit
```

Type `1`, `2`, '3', or `4` in the `OPTION ===>` field and press ENTER. PF3
exits ISPF and returns you to the blank prompt.

The directories are taken from your `bricks.cnf`:

| Area | Config key | Default |
|---|---|---|
| REXX | `rexx_dir` | `runtime/rexx` |
| COBOL | `cobol_dir` | `runtime/cobol` |
| MAPS | `maps_dir` | `runtime/map` |
| PL/I | `pli_dir` | `runtime/pli` |
---

## File browser

After picking an area, the browser shows the directory contents in a
two-column paged list:

```
------------------------ FILE BROWSER -- REXX -- Sort: NAME -- Page 1/1
Command ===>                          Sort: N=Name D=Date
S Filename           Size Date  Syntax       S Filename           ...
   cons.rexx         3787 01May  OK
   cusl.rexx         4458 01May  Err:127
   cust.rexx         7167 01May  OK
...
F3=Menu  F6=New  F7=Up  F8=Down  any char=Open  D=Delete
```

Each row has a single-byte **selector field** (the `S` column) to its
left. Type a character into the selector and press ENTER:

| Selector | Action |
|---|---|
| Any letter except `D` | Open the file in the editor |
| `D` | Delete the file (a confirmation overlay appears; press F9 to confirm, anything else cancels) |

The right-most **Syntax** column shows the live parse status of every
file in the directory:

| Value | Meaning |
|---|---|
| `OK` (green) | The file parses cleanly with the area's parser. |
| `Err:N` (red) | The parser reported a syntax error at line N. |
| `Err` (red, no line) | I/O error or no line number available. |

Notice that the file browser does a syntax check on maps, REXX and COBOL and
shows the status of the syntax of each file. 

### Browser command line

Type into the `Command ===>` field:

| Command | Effect |
|---|---|
| `N` or `NAME` | Sort by filename (case-insensitive). |
| `D` or `DATE` | Sort by modification time, newest first. |

### Browser PF keys

| Key | Action |
|---|---|
| `F3` | Return to the menu. |
| `F6` | Create a new file. Prompts for a filename, auto-appends the area extension (`.rexx`, `.cob`, `.map`) if you omit it, validates the name, creates an empty file, acquires the edit lock, and drops into the editor. |
| `F7` | Page up. |
| `F8` | Page down. |
| `F9` | (Only when the delete-confirmation overlay is showing.) Confirm the delete. |

### Filename rules (PF6 "new file")

The browser's new-file prompt enforces a sandbox:

- Non-empty.
- No `/`, `\`, NUL byte, or `..` substring.
- ASCII printable bytes only (0x20–0x7E).
- Must not already exist on disk.
- The area extension is auto-appended if the name has no `.`.

A path that resolves outside the area directory is rejected.

---

## Editor screen layout

```
EDIT  runtime/rexx/cusl.rexx                        Columns 00001 00071   1
Command ===> _                                          Scroll ===> PAGE_
****** **************************** Top of Data ******************************
''''''     /* CUSL -- list / search the customers file, ...           */
''''''     ADDRESS CICS
...
****** *************************** Bottom of Data ****************************
```

- **Row 0** — `EDIT <title>` plus `Columns NNNNN NNNNN` showing the
  visible-column range. The digit at column 78 is the active-file
  number (`1` to `9`) when multiple files are open. Error and status
  messages also paint here, after column 23.
- **Row 1** — `Command ===>` writable field, then `Scroll ===>`
  writable field showing the current scroll mode.
- **Row 2 (only at top of file)** — `Top of Data` marker. The `******`
  segment at the start of this row is writable: type a line command
  (e.g. `I3`) here to insert lines before line 1.
- **Content rows** — each has a 6-byte prefix (`''''''` by default,
  or `000123` when `NUMBER ON` is active, or `CC` etc. when a block
  command is pending), then the line content.
- **Bottom-of-Data row** — `Bottom of Data` marker. Its `******`
  segment is writable too: type `I3` here to append blank lines past
  the last content line.

Prefix colors:

| Color | Meaning |
|---|---|
| Red | Default — unchanged line. |
| Yellow | Cursor's current row. |
| Turquoise | Line modified in the most recent interaction. |

---

## PF keys

| Key | Action |
|---|---|
| `F1` | Show the help overlay (any key dismisses). |
| `F2` | Open the file browser without closing the editor. The current file stays open in the multi-file slot; pick another file to open it as buffer 2. |
| `F3` | Close the **current** buffer. If it's modified, an abandon-confirmation overlay appears; F9 confirms the discard. **The editor stays alive as long as any other buffer is open** — it switches to the next remaining buffer. Only when the last buffer is closed does the editor return to the browser. See [Multi-file editing](#multi-file-editing). |
| `F4` | Open this manual's help text as an additional editable buffer. F9 cycles back to your file. |
| `F5` | Repeat the last `FIND` (continues from the previous match). |
| `F7` | Scroll up by the amount in the `Scroll ===>` field. |
| `F8` | Scroll down by the amount in the `Scroll ===>` field. |
| `F9` | Cycle to the next open file in the multi-file slot. |
| `F10` | Scroll left by 8 columns. |
| `F11` | Scroll right by 8 columns. |
| `F12` | Save the **current** buffer and close it (same close-and-cycle behaviour as F3 on a clean buffer). On a parse error, the message paints on row 0 and the buffer stays open; press F12 again to save anyway. |

The `Scroll ===>` field accepts:

| Mode | Lines per F7 / F8 |
|---|---|
| `CSR` (default) | 1 |
| `HALF` | half the visible-line count |
| `PAGE` | full visible-line count |
| `DATA` | visible-line count minus 1 (always leaves one row of context) |

Partial input is resolved by prefix and Levenshtein distance: `P` →
`PAGE`, `PGE` → `PAGE`, `CURSOR` → `CSR`.

---

## Command-line commands

Type into the `Command ===>` field and press ENTER. Verb matching is
case-insensitive; quoted arguments preserve case.

### Find / change

| Command | Effect |
|---|---|
| `FIND text` or `F text` | Locate teh next occurence of `text` (case-insensitive). Wraps from end of file back to the start. The cursor lands on the match, every match across the buffer is highlighted in red + underscore, and the status line shows `Found N matches`. |
| `F` (no argument) or `RFIND` | Repeat the last FIND from the current cursor position. Same as PF5. |
| `CHANGE old new` or `C old new` | Replace the first occurrence of `old` with `new` and report `Changed N`. The status line shows `not found.` if the search string isn't present. |
| `CHANGE old new ALL` or `C old new ALL` | Replace every occurrence in every line. |

Quotes can be used to embed spaces: `FIND 'no records'` /
`CHANGE 'a b' 'c d'`.

### Navigation

| Command | Effect |
|---|---|
| `LOCATE n` / `LOC n` / `L n` | Set the top-line cursor to line N (1-indexed). Clamped to the valid range. |
| `TOP` / `T` | Scroll to the start of the buffer. |
| `BOTTOM` / `BOT` | Scroll to the end. |

### Save and exit

| Command | Effect |
|---|---|
| `SAVE` | Persist the current buffer to disk and stay in the editor (does **not** close the buffer — unlike F12). On a parse error the message is shown on row 0; a second `SAVE` (or F12) saves anyway. |
| `CANCEL` / `CAN` | Close teh current buffer without saving, no abandon-confirmation prompt. Same close-and-cycle behaviour as F3-then-F9: if other buffers are open, the editor switches to the next one; only the last buffer triggers an exit. |
| `UNDO` | Restore the buffer from the most recent undo snapshot. Same as the `U` line command from the prefix area. |

### Display toggles

| Command | Effect |
|---|---|
| `NUMBER ON` / `NUM ON` | Show 6-digit line numbers in the prefix area instead of `''''''`. |
| `NUMBER OFF` / `NUM OFF` | Restore the `''''''` placeholder. |
| `NUMBER` / `NUM` | Toggle the current setting. |
| `COLS` / `COL` | Toggle the column ruler at row 2 (replaces the Top-of-Data marker while on). |
| `HI ON` | Turn syntax highlighting on. |
| `HI OFF` | Turn syntax highlighting off. |
| `HI` | Toggle the current setting. |

### Other

| Command | Effect |
|---|---|
| `RESET` / `RES` | Clear FIND highlights, the column ruler, excluded line ranges, and the last-command echo. Does **not** clear the turquoise "recently modified" prefix markers. |
| `HELP` | Show the help overlay. Same as F1. |
| `EDIT` | Open the file browser without leaving the editor. Same as F2. |

Anything else lands on row 0 as `Invalid command`.

---

## Line-prefix commands

Type into the 6-byte prefix area on the line you want to act on and
press ENTER. Whitespace and leftover `'` characters are stripped before
parsing. Most verbs accept an optional numeric count (`D3` = delete 3
lines, `R5` = repeat 5 times).

### Editing the buffer

| Command | Effect |
|---|---|
| `D` / `Dn` | Delete this line / N lines starting here. |
| `I` / `In` | Insert N blank lines (default 1) **after** this line. |
| `R` / `Rn` | Repeat this line N times immediately after itself. |

### Copy / move / paste

| Command | Effect |
|---|---|
| `C` / `Cn` | Copy this line / N lines into the copy buffer. The lines stay where they are. |
| `M` / `Mn` | Move this line / N lines into the move buffer (the source lines are removed). |
| `A` | Paste the most recent copy/move buffer **after** this line. |
| `B` | Paste **before** this line. |
| `O` | Overlay — paste the copy/move buffer onto this line and the lines below it, replacing their content row-for-row. |

The copy/move buffer is cleared after `A`, `B`, or `O` consumes it.
The TOD and BOD markers also accept `A` (after) and `B` (before) to
paste at the buffer boundaries.

### Case and indent

| Command | Effect |
|---|---|
| `U` | Uppercase this line. |
| `L` | Lowercase this line. |
| `)` / `)n` | Shift this line right N spaces (default 1). |
| `(` / `(n` | Shift this line left N spaces (up to the first non-space character). |

### Exclusion

| Command | Effect |
|---|---|
| `X` / `Xn` | Exclude this line / N lines from view. Excluded lines collapse into a turquoise summary row: `- - - 5 Line(s) excluded`. Type `RESET` to bring them back. |

### Inserting at file boundaries

The `******` segment of the Top-of-Data and Bottom-of-Data marker rows
is writable. Type an `I` or `In` there to insert blank lines before
line 1 (TOD) or after the last line (BOD).

---

## Block commands

Doubled-verb forms mark a range. Type the verb on the first line of
the range, press ENTER (the prefix shows the pending verb, e.g.
`CC    `), then on a later screen type the same doubled verb on the
last line of the range and press ENTER. The block executes:

| Form | Single equivalent | Effect on the range |
|---|---|---|
| `DD..DD` | `D` | Delete every line. |
| `CC..CC` | `C` | Copy every line into the copy buffer. |
| `MM..MM` | `M` | Move every line into the move buffer (deletes the originals). |
| `UU..UU` | `U` | Uppercase every line. |
| `LL..LL` | `L` | Lowercase every line. |
| `))..))` | `)` | Shift the block right (count from either marker). |
| `((..((` | `(` | Shift the block left. |
| `XX..XX` | `X` | Exclude the block from view. |

If you change your mind, type `RESET` on the command line to discard
the pending half-block marker.

---

## Save flow and syntax validation

Bricks runs the area's parser (`rexx.Parse`, `cobol.Parse`,
`mapdsl.Parse`) on the buffer before writing. The flow is **warn-then-
save**:

1. First PF12 or `SAVE` with a parser error: the error message is
   painted on row 0 in red and the buffer is **not** written. A bypass
   flag is armed internally.
2. Second PF12 or `SAVE`: the bypass is honored, the buffer is written
   regardless of the parse error.

This applies to every area (REXX, COBOL, MAP). A clean parse saves
immediately on the first press.

The status messages match the canonical ISPF wording:

| Message | When |
|---|---|
| `Saved` | Successful save (via PF12 or `SAVE` command). |
| `Save error` | The filesystem write failed. |
| `<raw parser error>` | First save with a parse error — bypass armed. |

---

## Multi-file editing

The editor holds **up to 9 buffers open simultaneously**. Each
buffer keeps its own content, cursor position, scroll offset, undo
snapshot, modified-line markers, and dirty bit. They never share
state — editing one buffer can't affect anothr.

### Opening a second (third, …) file

Buffers are added to the slot when:

- The browser opens a file for the first time (always becomes
  buffer 1 if none are open).
- You press **F2** from inside the editor: the current buffer stays
  resident, the browser appears, you pick another file → it opens
  as the next available buffer.
- You press **F4** to open this manual's help text as an additional
  editable buffer.

The current buffer's **file number** (1–9) is painted in yellow
intense at column 78 of row 0, so you always know which slot you're
on.

### Cycling between buffers

**F9** cycles forward through `OpenFiles`. Modified buffers keep
their dirty bit on the round trip — turquoise prefix markers, the
`Modified` flag, and any pending block-command markers all survive
when you leave a buffer and come back.

### Closing a buffer — the hard rule

The editor **never exits while other buffers are still open**. F3,
F12, the `CANCEL` command, and the abandon-confirm overlay (F9 over
"Changes have not been saved!") all act on the **active buffer
only**:

| Action | What happens to the active buffer | What happens to OTHER buffers |
|---|---|---|
| F3 (clean buffer) | Dropped from the slot | Untouched — editor cycles to the next |
| F3 (dirty buffer) → F9 confirm | Dropped, edits discarded | Untouched — editor cycles to the next |
| F12 (successful save) | Saved to disk, then dropped | Untouched — editor cycles to the next |
| F12 (validator refused) | Stays open with error on row 0 | Untouched |
| `CANCEL` command | Dropped, edits discarded | Untouched — editor cycles to the next |
| `SAVE` command | Saved to disk, **stays open** | Untouched — operator keeps editing |

The editor returns to the browser **only when the last open buffer
is closed**. To exit the whole ISPF session with multiple buffers
open you have to F3 / F12 / CANCEL each one in turn — every dirty
buffer gets its own abandon-confirm so unsaved edits in buffer N
can't be silently lost when you F3 buffer 1.

### Edit locks across buffers

Each open buffer holds its own exclusive edit lock for the duration
of the editing session. When you F9-cycle from buffer A to buffer B,
the editor releases the lock on A and acquires the lock on B before
the first render of B. A concurrent `dev` operator can grab A's
lock the moment you leave it; when you F9 back, if A is now held by
someone else you'll see the `Locked by USER123` message and return
to the browser — A's buffer is dropped from your slot since you no
longer own the file.

### The 9-buffer limit

The 10th open attempt errors with `Maximum files open (9)` on the
file browser's error line. Close one of the existing buffers (F3 or
F12) and try again.

### What survives a F2 round-trip

When you F2 back to the browser and then ENTER on the same file
you're already editing, the editor reuses the existing buffer
instead of re-reading from disk. If another tool has modified the
file on disk in the meantime, the row-0 banner shows `External
changes - buffer kept` so you know your in-memory copy is stale.

---

## Edit locks

While a file is open in the editor it is locked process-wide,
**per-buffer**. A second `dev` operator who tries to open the same
file in their browser sees a red `Locked by USER123 since HH:MM`
message and the file's syntax-status column shows `edit` in
turquoise. Locks are also consulted by the file browser before any
`D` (delete) on the file — a delete against a held file is refused
with the same "Locked by" message.

A buffer's lock is held from the moment it becomes the active
buffer (open, F2-pick-again, or F9-cycle-to) until the moment it
stops being active (F9 to another buffer, F3/F12/CANCEL closing it,
or the TCP connection dropping). Closing a buffer with F3 releases
**only that buffer's** lock; other open buffers keep their locks
until you close them too.

Locks release automatically when:

- You close the buffer with F3, F12, or `CANCEL`.
- You F9-cycle away from the buffer (and acquire the lock for the
  new active buffer).
- Your TCP connection drops or your sign-on times out.
- The bricks process panics during your session (deferred cleanup
  catches this).

The browser paints the **filename** of a currently-locked file in
red intense so you can see contention without trying to open.

---

## Syntax highlighting

Highlighting is **off by default** because per-line tokenizing has
edge cases (notably REXX `/* ... */` block comments that span lines).
Turn it on with the `HI` command:

```
Command ===> HI ON
```

When on, the editor paints token-color overlays on each visible line:

| Token class | Color |
|---|---|
| Keywords | Blue intense |
| Strings | Turquoise |
| Numbers | Yellow |
| Comments | Green |
| Identifiers / default | Green |

The highlighter is per-language — picked from the area at editor
open. The browser's per-file syntax-status indicator (`OK` / `Err:N`)
uses the same parser independently of whether highlighting is on.

---

## Limitations

- No regex search; `FIND` and `CHANGE` are plain-string,
  case-insensitive.
- Undo is single-level: each modifying line-command pass overwrites
  the previous snapshot. CHANGE does not capture undo (matches tsu).
- REXX `/* ... */` block comments that span lines won't fully
  highlight past the opening line when `HI ON` is set.
- The 6-byte prefix area is the only place to type line commands;
  there is no menu-driven equivalent.
- File path validation in the browser rejects subdirectories under
  the area root — files only.
