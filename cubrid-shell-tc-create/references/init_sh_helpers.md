# CTP Helper Functions Reference

Functions exposed by `cubrid-testtools/CTP/shell/init_path/init.sh`.

Conventions:
- Only **parent (user-facing)** functions are listed. Internal wrappers/helpers
  (e.g. `change_parameter`, `delete_ini`, `restore_*_conf`, `db_status`,
  `xkill_java_windows`, `parse_build_version`, …) are intentionally omitted —
  use the parent function listed here instead.
- A function appears in exactly one section; no duplicates.

---

## Lifecycle

| Function | Purpose |
|---|---|
| `init test \| answer` | Initialize test env (must be called right after sourcing init.sh). `test` runs/compares; `answer` regenerates `.answer` files. |
| `finish` | Final cleanup — stops services, restores all conf files, releases broker shared memory, deletes lob/err logs. Must be the last call of every test. |

`finish` internally invokes `count_time`, `release_broker_sharedmemory`,
`delete_ini`, and `restore_all_conf` — never call those directly.

---

## Database

| Function | Purpose |
|---|---|
| `cubrid_createdb [options] <dbname>` | Version-aware wrapper around `cubrid createdb` (handles charset on 9.1+/10.x). Always use this instead of raw `cubrid createdb`. |
| `create_ccidb` | Build the canned `ccidb` test DB from the bundled `ccidb.sql` / `ccidbbak` snapshot, used by CCI/JDBC scenarios. |

---

## Configuration

All `change_*` helpers stash the original conf into `*.conf.org` on first call;
`finish` restores them automatically.

### Single-key parameter change (writes to default section)

| Function | Target file |
|---|---|
| `change_db_parameter "key=value"` | `$CUBRID/conf/cubrid.conf` |
| `change_broker_parameter "key=value"` | `$CUBRID/conf/cubrid_broker.conf` (broker1) |
| `change_ha_parameter "key=value"` | `$CUBRID/conf/cubrid_ha.conf` |

### Section-aware parameter change

For configs with multiple `[section]` blocks (`[common]`, `[%BROKER1]`, etc.):

| Function | Target file |
|---|---|
| `change_db_section_parameter <section> "key=value"` | `cubrid.conf` |
| `change_broker_section_parameter <section> "key=value"` | `cubrid_broker.conf` |
| `change_gateway_section_parameter <section> "key=value"` | `cubrid_gateway.conf` |
| `change_ha_section_parameter <section> "key=value"` | `cubrid_ha.conf` |

Example:
```bash
change_db_parameter "java_stored_procedure=yes"
change_broker_parameter "SQL_LOG=ON"
change_db_section_parameter common "log_max_archives=1"
change_broker_section_parameter "%BROKER1" "MAX_NUM_APPL_SERVER=5"
```

---

## Result Reporting

| Function | Purpose |
|---|---|
| `write_ok [description]` | Record a passing check (`case_no` auto-increments). |
| `write_nok [file or description]` | Record a failing check; if arg is a log file, its contents are appended as evidence. |
| `compare_result_between_files <answer> <result>` | Diff two files (line-number-tolerant) and auto-emit `write_ok`/`write_nok`. |
| `make_answer_or_compare_result` | When `init answer` was used, write `$temp_result` to a `.answer` file; otherwise diff against the existing `.answer`. Pairs with `$temp_result` capture. |

---

## SQL & Command Execution

| Function | Purpose |
|---|---|
| `exec_sql <dbname> "<sql>" [csql_opts]` | Run SQL via `csql`, auto-picking `-C` (client) or `-S` (standalone) by current server state. Echoes the result. |
| `test_exec_sql <dbname> "<sql>" [csql_opts] [error]` | Run SQL and assert success — passing `error` as the last arg inverts the expectation. Calls `write_ok`/`write_nok`. |
| `test_exec_command "<cmd>" [error]` | Run an arbitrary shell command and assert its exit code (with optional `error` inversion). |

---

## Output Normalization

Apply these before any `diff` / `compare_result_between_files` so volatile
content doesn't flake the test.

| Function | What it removes / rewrites |
|---|---|
| `format_csql_output <file>` | csql header/footer, execution time (`0.00xxx sec`), CAS info |
| `format_query_plan <file>` | Query-plan volatile fields (costs, OIDs, etc.) |
| `format_path_output <file>` | Absolute path strings |
| `format_number_output <file>` | Patterns like `0\|123\|456` → `?` |
| `format_instance_oid_output <file>` | Object instance OIDs `a\|b\|c` |
| `format_tran_index_lockdb_output <file>` | `Tran_index = N, Granted_mode` blocks in `lockdb` output |
| `format_cubrid_version <file>` | CUBRID version/build strings |
| `cas_info_replace <file>` | Replace dynamic CAS info lines |
| `get_csql_execution_time <file>` | Extract csql elapsed time (utility for selective stripping) |

---

## Process Management

| Function | Purpose |
|---|---|
| `xkill [-f] "<pattern>"` | Kill user-owned processes matching pattern. `-f` matches the full command line. Cross-platform (handles Windows via `wmic`). |
| `xkill_pid <pid> [pid ...]` | Force-kill specific PIDs (cross-platform). |

Never use raw `kill -9` / `pkill` — `xkill` honours the Windows process
whitelist (`csrss.exe`, `services.exe`, regression-service procs, etc.) so the
test won't take down the host.

---

## Environment Variables

| Function | Purpose |
|---|---|
| `set_CUBRID_CHARSET <charset>` | Set `CUBRID_CHARSET` correctly per server version (export only on 9.1+/10.x; registry write on Windows). |
| `setENVParam <NAME> <VALUE>` | Export an env var; on Windows also persists to registry. |
| `recoverENVParam <NAME> <VALUE>` | Restore an env var to a prior value; on Windows drops or rewrites the registry key as needed. |
| `unsetENVParam <NAME>` | Unexport and (on Windows) drop the registry key. |

---

## Locale & Timezone

`finish` does **not** auto-revert locale/timezone builds — pair each `do_*`
with its corresponding revert before `finish`.

| Function | Purpose |
|---|---|
| `do_make_locale [force] [debug\|release] [locale]` | Run `make_locale` against `$CUBRID` (cross-platform). |
| `delete_make_locale` | Restore locale binaries to the pre-`do_make_locale` state. |
| `do_make_tz [new\|extend] [release] [nocheck]` | Run `make_tz` (full rebuild or extend mode). |
| `revert_tz` | Restore timezone binaries to the pre-`do_make_tz` state. |

---

## Compilation Utility

| Function | Purpose |
|---|---|
| `xgcc [gcc options] <source>` | Cross-platform C/C++ compile wrapper. Auto-resolves include / lib paths against `$CUBRID` and the bundled MinGW toolchain on Windows; picks 32/64-bit toolchain based on the CUBRID build. |

---

## OS / Port / Time

| Function | Returns |
|---|---|
| `get_os` | `Linux`, `AIX`, or `Windows_NT`. Also exported as `$OS` after sourcing init.sh. |
| `get_broker_port_from_shell_config` | Broker port read from `$init_path/shell_config.xml`. Preferred for tests that need the runtime port. |
| `get_cubrid_port_id` | `cubrid_port_id` from `cubrid.conf` `[common]` (defaults to `1523`). |
| `generate_port <start>` | Find the next free TCP port above `<start>` via `netstat`. Discouraged unless absolutely necessary — prefer `get_broker_port_from_shell_config`. |
| `get_curr_second` | Current Unix timestamp (used by the internal `count_time` accounting). |

---

## Platform Exclusion Macros

Place the macro **before** sourcing `init.sh` to skip the case on that OS:

```bash
#!/bin/bash
WINDOWS_NOT_SUPPORTED      # or LINUX_NOT_SUPPORTED / AIX_NOT_SUPPORTED
. $init_path/init.sh
init test
```

Available:

| Macro | Skips on |
|---|---|
| `LINUX_NOT_SUPPORTED` | Linux |
| `WINDOWS_NOT_SUPPORTED` | Windows |
| `AIX_NOT_SUPPORTED` | AIX |

---

## Source of Truth

The authoritative implementation lives at
`cubrid-testtools/CTP/shell/init_path/init.sh`. When in doubt about exact
argument parsing or edge-case behaviour, read the function body there.
