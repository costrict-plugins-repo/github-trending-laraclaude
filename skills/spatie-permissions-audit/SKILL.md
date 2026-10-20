---
name: lc:spatie-permissions-audit
description: Audit spatie/laravel-permission usage - permissions used in code but undefined, defined but unused, unprotected routes, guard mismatches, and cache pitfalls.
argument-hint: "[analyze | fix | fix --dry-run | file-path | fix file-path]"
user-invocable: true
allowed-tools: Read Grep Bash Edit Write Glob
---

# Spatie Permissions Audit

Audit roles and permissions in a Laravel app that uses **[spatie/laravel-permission](https://github.com/spatie/laravel-permission)**. It cross-references the permissions/roles *defined* (seeders, migrations, DB) against the ones *used* in code (`@can`, `->can()`, `hasPermissionTo`, `role`/`permission` middleware), and flags the gaps that silently break authorization.

This skill is **specific to spatie/laravel-permission**. Other permission packages (e.g. `larallow`) have their own skills — do not apply spatie-specific rules to them.

## Subcommands

| Subcommand | Description |
|---|---|
| *(no argument)* / `analyze` | Scan the whole project and report all permission issues. Read-only. |
| `fix` | Scan and auto-fix the fixable issues (e.g. add missing permissions to the seeder). Confirms before each change. |
| `fix --dry-run` | Show what would be fixed without changing anything. |
| `[file-path]` | Audit a specific file/directory only. Read-only. |
| `fix [file-path]` | Fix issues in a specific file, with confirmation. |

## Step 0: Verify spatie/laravel-permission Is Installed

1. Use `Grep` to check whether `spatie/laravel-permission` is in `composer.json`.
2. If NOT found, stop and inform the user:
   ```
   spatie/laravel-permission is not installed. Install it with:
   composer require spatie/laravel-permission
   php artisan vendor:publish --provider="Spatie\Permission\PermissionServiceProvider"
   php artisan migrate
   ```
   If the project uses a different permission package (e.g. larallow), say so and stop — this audit only understands spatie's API.
3. If found, read `config/permission.php` to learn the configured model classes, table names, and `column_names`. Read the package version from `composer.lock` so the rules match the installed API.

## Step 1: Build the "defined" set

Collect every role and permission the project declares:

1. Use `Glob` + `Read` on `database/seeders/*.php` to find `Permission::create([...])`, `Role::create([...])`, `->givePermissionTo(...)`, `->syncPermissions(...)`, and `Permission::findOrCreate(...)`. Record each permission/role `name` and its `guard_name` (default `web` when omitted).
2. Also scan `database/migrations/*.php` for any seeded permissions inserted via `DB::table('permissions')->insert(...)`.
3. If a database is reachable (Docker/tinker as `/lc:orphaned-records` does, optional), confirm against the live `permissions`/`roles` tables. Note when the report is based on code only vs. verified against the DB.

## Step 2: Build the "used" set

Find every place a permission or role is referenced:

1. Use `Grep` across `app/`, `routes/`, and `resources/views/` for:
   - Blade: `@can('...')`, `@cannot('...')`, `@role('...')`, `@hasrole('...')`, `@hasanyrole('...')`, `@hasallroles('...')`, `@hasexactroles('...')`, `@unlessrole('...')`, `@haspermission('...')` (there is no `@permission` directive in spatie — permission checks use Laravel's native `@can`)
   - PHP: `->can('...')`, `->cannot('...')`, `->hasPermissionTo('...')`, `->hasRole('...')`, `->hasAnyRole(...)`, `->hasAllRoles(...)`, `Gate::allows('...')`, `authorize('...')`
   - Middleware: `->middleware('permission:...')`, `->middleware('role:...')`, `->middleware('role_or_permission:...')`, and the array form
2. Extract the literal permission/role names. Skip dynamic names built from variables (flag them separately as "dynamic — cannot verify statically").

## Step 3: Detection Rules

### 1. Permission/role used in code but NOT defined
**Detect:** a name in the "used" set with no match in the "defined" set (respecting `guard_name`).
**Why it matters:** spatie throws `PermissionDoesNotExist` / `RoleDoesNotExist`, or the check silently returns `false` and the user is wrongly denied access. Usually a typo (`edit-posts` vs `edit_posts`).
**Severity:** CRITICAL
**Fix:** offer to add the missing permission/role to the seeder (with the right guard), or correct the typo if a close match exists in the defined set.

### 2. Permission/role defined but NEVER used
**Detect:** a defined name with zero references in the "used" set.
**Why it matters:** dead permissions add confusion and risk over-granting roles.
**Severity:** WARNING
**Fix:** flag for removal — do NOT auto-delete (it may be used dynamically, via API, or by an external app). Require explicit confirmation per item.

### 3. Guard mismatch
**Detect:** a permission/role checked against a guard different from the one it was created with; or a model using `HasRoles` whose auth guard does not match the permission `guard_name`.
**Why it matters:** spatie scopes permissions by guard; a guard mismatch makes valid permissions appear missing.
**Severity:** HIGH

### 4. Middleware alias used but not registered (v6)
**Detect:** routes use `permission:`, `role:`, or `role_or_permission:` middleware, but the alias is not registered. In Laravel 11+ check `bootstrap/app.php` for `$middleware->alias([... 'permission' => \Spatie\Permission\Middleware\PermissionMiddleware::class ...])`; in Laravel 9/10 check `app/Http/Kernel.php` `$middlewareAliases`/`$routeMiddleware`.
**Why it matters:** spatie v6 does NOT auto-register these aliases. An unregistered alias makes the route throw at runtime (or the protection never applies) — the authorization silently does nothing.
**Severity:** CRITICAL
**Fix:** add the missing alias registration for the project's Laravel version. The three classes are `RoleMiddleware`, `PermissionMiddleware`, `RoleOrPermissionMiddleware` under `Spatie\Permission\Middleware`.

### 5. Sensitive route/action without authorization
**Detect:** routes (POST/PUT/PATCH/DELETE) and controller methods that mutate data but have no `permission:`/`role:` middleware, no `authorize()`/`can()` check, and no policy.
**Why it matters:** unprotected write endpoints.
**Severity:** HIGH (but expect false positives — review in context)

### 6. Direct permission grants to users instead of roles
**Detect:** `->givePermissionTo(...)` called on a user model rather than a role.
**Why it matters:** scattering permissions per-user instead of per-role hurts maintainability.
**Severity:** INFO

### 7. Cache not reset after seeding
**Detect:** a seeder that creates roles/permissions without calling `app(\Spatie\Permission\PermissionRegistrar::class)->forgetCachedPermissions()` (or `php artisan permission:cache-reset`).
**Why it matters:** spatie caches permissions; new ones won't take effect until the cache is cleared.
**Severity:** WARNING
**Fix:** add the cache-forget call at the top of the seeder.

### 8. Missing super-admin handling (informational)
**Detect:** no `Gate::before(...)` granting a super-admin role everything, when a role named like `super-admin`/`admin` exists.
**Why it matters:** without it, the super-admin must be granted every permission explicitly. Only INFO — some projects do this on purpose.
**Severity:** INFO

## Report Format

```
=== Spatie Permissions Audit Report ===
Source: code + DB (verified)   Guards: web, api

CRITICAL (1)
  app/Http/Controllers/PostController.php:42
    [CRITICAL] permission 'edit_posts' used but not defined (closest defined: 'edit-posts')
    → fix: correct typo, or add 'edit_posts' to RolesAndPermissionsSeeder

HIGH (1)
  routes/web.php:88
    [HIGH] DELETE /reports/{id} has no permission/role middleware or policy
    → add ->middleware('permission:delete-reports') or a policy

WARNING (2)
  database/seeders/RolesAndPermissionsSeeder.php
    [WARNING] permission 'manage-legacy' defined but never used
    [WARNING] seeder creates permissions without forgetCachedPermissions()

Summary: 4 issues (1 critical, 1 high, 2 warnings) · 23 permissions defined, 19 used, 1 dynamic (skipped)
```

## Fix Mode

Apply with confirmation per change:
1. **Missing permission/role** → add to the project's permission seeder with the correct `guard_name` (or fix the typo when a close match exists).
2. **Cache not reset** → insert `app(\Spatie\Permission\PermissionRegistrar::class)->forgetCachedPermissions();` at the start of the seeder's `run()`.
3. **Unused permission** → flag and remove only on explicit confirmation per item.

Route-protection and guard-mismatch findings are reported but NOT auto-fixed (they need human judgment about which permission applies).

## Notes

- Static analysis. Dynamic permission names (built from variables, config, or DB) cannot be verified — they are listed separately, never silently dropped.
- Respect `guard_name`: the same permission name under different guards is two different permissions in spatie.
- This skill never touches data; DB access (if used) is read-only.
- Pairs with `/lc:security-audit` for broader coverage.
