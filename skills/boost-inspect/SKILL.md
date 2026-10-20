---
name: lc:boost-inspect
description: Runtime-verified Laravel inspection using the Laravel Boost MCP server (Tinker, Database Query/Schema, Last Error) instead of static grep. Falls back to Docker/Tinker if Boost is not installed.
argument-hint: "[report | models | schema | errors | target]"
user-invocable: true
allowed-tools: Read Grep Bash Edit Write Glob mcp__laravel-boost__*
---

# Boost Inspect

Inspect a running Laravel application using the [Laravel Boost](https://laravel.com/docs/13.x/boost) MCP server. Most LaraClaude analyzers reason about the code statically (grep + read). This skill reasons about the **live application**: the real database schema, real row counts, loaded Eloquent models, registered routes/config, and the last runtime errors. That makes findings precise instead of inferred (e.g. an orphaned FK confirmed against actual data, not guessed from a migration).

If the Boost MCP server is not present, the skill degrades gracefully to running the same checks through `php artisan tinker` in the project's Docker container.

## Subcommands

| Subcommand | Description |
|---|---|
| *(no argument)* / `report` | Full runtime inspection: app info, models, schema drift, orphaned FKs, recent errors. Read-only. |
| `models` | List Eloquent models with their real relationships and verify each relationship resolves at runtime. |
| `schema` | Compare the live database schema against `database/migrations/` and report drift. |
| `errors` | Read and analyze the last application errors / log entries from the running app. |
| `[target]` | Focus the inspection on a single model or table (e.g. `Property`, `properties`). |

This is a read-only analyzer. It never modifies files or data. When it finds an issue it points to the matching LaraClaude skill that can fix it (`/lc:orphaned-records`, `/lc:check-foreign-keys`, `/lc:analyze-model`, ...).

## Process

### Step 0: Detect Laravel Boost (dynamically)

Do NOT hardcode exact MCP tool names — Boost may add, rename, or reorganize them between versions. Discover what is actually available in this session and map tools to capabilities by intent.

1. Use `Grep` to check whether `laravel/boost` is in `composer.json`.
2. **Learn the tool names from the installed version (primary source).** Use `Glob` on `vendor/laravel/boost/src/Mcp/Tools/*.php` and convert each class basename to kebab-case (`DatabaseSchema` → `database-schema`) — this is exactly how `laravel/mcp` derives tool names (`Str::kebab(class_basename())`), so it gives you the real tool set for the version actually installed, regardless of docs or release. If the `vendor/` directory is unavailable (e.g. Boost runs in a different container), skip this and rely on the reference table below.
3. **Discover the Boost MCP tools available in this session.** Boost registers its MCP server under the key `laravel-boost`, so its tools surface as `mcp__laravel-boost__<tool>` (where `<tool>` is the kebab-case name from step 2). Enumerate every available tool whose name starts with `mcp__laravel-boost__` (also accept the underscore variant `mcp__laravel_boost__` in case the host sanitizes the dash).
4. **Map discovered tools to capabilities by keyword, not by a fixed string.** For each capability below, pick the available `laravel-boost` tool whose name (or, if needed, its description) best matches the listed keywords. If none matches, treat that single capability as unavailable and use the fallback for it:

   | Capability | Match tool name containing | Reference name (Boost as of v2.x) |
   |---|---|---|
   | App info | `application`, `info` | `application-info` |
   | Run read query | `query` | `database-query` |
   | Read schema | `schema` | `database-schema` |
   | List connections | `connection` | `database-connections` |
   | Execute PHP | `tinker` | `tinker` |
   | Last error | `error` | `last-error` |
   | Read logs | `log` | `read-log-entries` |
   | Search docs | `doc` | `search-docs` |

   The reference column is a hint for the current release, not a dependency. Always prefer whatever the discovery step actually returned.
5. Decide the execution mode:
   - **Boost mode** (preferred): at least the schema/query/tinker capabilities resolved to real `laravel-boost` tools. Use the discovered tools directly instead of shelling out.
   - **Fallback mode**: Boost is not installed, its MCP server is not running, or discovery returned no usable tools. Tell the user once:
     ```
     Laravel Boost not detected. Running in fallback mode via tinker (Docker / Windows / local — auto-detected).
     For richer, runtime-verified inspection install it:
       composer require laravel/boost --dev
       php artisan boost:install
     ```
     Then run the equivalent checks through tinker, choosing the right invocation for the environment (see Step 0b).
6. State which mode and execution environment you are using at the top of the report.

### Step 0b: Pick the Right Environment (Docker, Windows, or local)

Fallback mode shells out to `php artisan`, so detect how this project actually runs before assuming `docker exec`:

1. **Docker:** if a `docker-compose.yml`/`compose.yml` exists, detect the app service container (as `/lc:orphaned-records` does) and run `docker exec {container} php artisan tinker --execute="..."`.
2. **Native Windows (no Docker):** the shell is `cmd`/PowerShell, not bash. Run `php artisan tinker --execute="..."` directly (PHP from Laragon/Herd/XAMPP must be on `PATH`). Mind the quoting — on Windows wrap the `--execute` body in double quotes and avoid the bash-style `\$` escaping used in the Docker examples; use plain `$`. Use Windows path separators only for filesystem paths, never inside SQL.
3. **WSL:** treat it like local Linux (bash) — run `php artisan tinker` directly inside the WSL project path; do not prefix with `docker exec` unless the app itself runs in a container.
4. **Plain local (Herd/Valet/Sail):** run `php artisan tinker --execute="..."` from the project root. For Sail, prefer `./vendor/bin/sail artisan tinker` if `docker-compose.yml` defines the `sail` service.

Detect, do not assume: check for `docker-compose.yml`, then the OS/shell, then fall back to a direct `php artisan` call. State the chosen environment in the report header. **Boost mode is unaffected by all of this** — it goes through the MCP server, so it works the same on Windows, WSL, Docker, or native.

### Step 1: Application Info

In Boost mode, call **Application Info**. In fallback mode, gather the same with tinker (`app()->version()`, `PHP_VERSION`, `DB::getDriverName()`) and `composer.json`.

Report: Laravel version, PHP version, environment, database engine/connection, and the count of ecosystem packages and Eloquent models detected. Flag anything notable (e.g. `APP_ENV=production` while inspecting — warn the user the checks run real queries).

### Step 2: Models & Relationships (`models` or full report)

1. Get the model list from **Application Info** (Boost) or by `Glob` on `app/Models/*.php` (fallback).
2. For each model (or just `[target]` if provided), verify the relationships actually resolve at runtime using **Tinker** / `Database Query`. For example, for a `hasMany`/`belongsTo`:
   ```php
   // Confirm the relation method exists and returns a Relation, and the FK column exists
   $m = new \App\Models\Property;
   collect((new ReflectionClass($m))->getMethods())
     ->filter(fn($x) => $x->class === \App\Models\Property::class);
   ```
   Use `Database Schema` to confirm the FK column the relationship depends on really exists in the table.
3. Report relationships whose underlying column is missing, or models with no table, as `CRITICAL`. Suggest `/lc:analyze-model {Model}` for a deeper view.

### Step 3: Schema Drift (`schema` or full report)

1. Read the **live** schema via Boost `Database Schema` (or `DB::select` on `information_schema` in fallback).
2. Use `Glob` + `Read` on `database/migrations/*.php` to build the expected schema (tables, columns, FKs).
3. Diff them and report:
   - Columns present in the DB but not in any migration → `WARNING` (manual/un-tracked changes).
   - Columns/tables in migrations but missing from the DB → `CRITICAL` (migrations not run, or `migrate:fresh` needed). Suggest `/lc:migration-fresh-test`.
   - FK constraints in the DB with no matching relationship in models → `INFO`. Suggest `/lc:check-foreign-keys`.

### Step 4: Orphaned Foreign Keys (full report)

For each FK found in the live schema, run a real count via `Database Query` (Boost) or tinker (fallback):
```sql
SELECT COUNT(*) FROM {table} t
LEFT JOIN {ref_table} r ON t.{column} = r.{ref_column}
WHERE t.{column} IS NOT NULL AND r.{ref_column} IS NULL
```
Report any non-zero result as `WARNING` with the real count, and point the user to `/lc:orphaned-records fix {table}` to clean it up. Skip polymorphic relationships (note them).

### Step 5: Recent Errors (`errors` or full report)

1. In Boost mode, call **Last Error** and **Read Log Entries** (last ~20). In fallback mode, read `storage/logs/laravel.log` tail.
2. For each distinct error, summarize the exception class, message, and the originating file:line.
3. Read the referenced source file to give a root-cause one-liner (like `/lc:analyze-error`), and suggest `/lc:analyze-error` for a full diagnosis.

### Step 6: Report

Use the standard LaraClaude report format. Always state the mode and group by area:

```
=== Boost Inspect Report ===
Mode: Boost MCP   (laravel-boost detected)
App:  Laravel 12.3 · PHP 8.4 · mysql · env=local · 41 models · 63 packages

Models & Relationships
  Property::owner (belongsTo User)
    [CRITICAL] FK column properties.user_id does not exist in the live schema
    → /lc:analyze-model Property

Schema Drift
  table: invoices
    Line —: [WARNING] column `legacy_ref` exists in DB but not in any migration
    Line —: [CRITICAL] migration adds `invoices.paid_at` but column missing in DB
    → /lc:migration-fresh-test

Orphaned Foreign Keys
  operations.client_seller_id -> clients.id
    [WARNING] 9 orphaned rows (verified against live data)
    → /lc:orphaned-records fix operations

Recent Errors
  [WARNING] QueryException — Unknown column 'status' (app/Http/.../Foo.php:42)
    → /lc:analyze-error

Summary: 4 issues found (2 critical, 2 warnings) · mode=Boost MCP
```

## Important Notes

- **Read-only.** This skill inspects; it does not mutate code or data. All queries are `SELECT`/count only.
- It runs **real queries against the configured database**. Use it in development/staging. If `APP_ENV=production`, warn the user before running Step 4.
- Prefer the Boost MCP tools when available — they are faster and already scoped to the app. Only shell out to Docker/tinker in fallback mode.
- This skill is a **router as much as an analyzer**: its job is to confirm issues against the running app, then hand off to the specialized LaraClaude fix skills.
- It pairs naturally with Laravel Boost rather than replacing it: Boost supplies the runtime context and docs; LaraClaude supplies the task-specific analysis and fixes.
