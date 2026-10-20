---
name: lc:generate-spatie-permissions
description: Scaffold spatie/laravel-permission setup - add the HasRoles trait to a model and generate a roles & permissions seeder.
argument-hint: "[ModelName]"
user-invocable: true
allowed-tools: Read Grep Bash Edit Write Glob
---

# Generate Spatie Permissions Setup

Scaffold the roles & permissions setup for **[spatie/laravel-permission](https://github.com/spatie/laravel-permission)**: add the `HasRoles` trait to the target model (default `User`) and generate a roles & permissions seeder following the project's conventions.

This is a generator skill — it always creates files, no analyze or fix mode. It is **specific to spatie/laravel-permission**; other permission packages (e.g. `larallow`) get their own generator.

## Subcommands

| Subcommand | Description |
|---|---|
| *(no argument)* | Default to the `User` model. Ask which roles/permissions to scaffold. |
| `[ModelName]` | Add `HasRoles` to the given model and generate the seeder. |

## Step 0: Verify spatie/laravel-permission Is Installed

1. Use `Grep` to check whether `spatie/laravel-permission` is in `composer.json`.
2. If NOT found, stop and tell the user:
   ```
   spatie/laravel-permission is not installed. Install it with:
   composer require spatie/laravel-permission
   php artisan vendor:publish --provider="Spatie\Permission\PermissionServiceProvider"
   php artisan migrate
   ```
3. If found, confirm the migration has been published/run (check `config/permission.php` exists and the `permissions`/`roles` tables are referenced). If `config/permission.php` is missing, tell the user to run the `vendor:publish` + `migrate` commands above before proceeding.

## Step 1: Read the Installed API

Read from vendor so the generated code matches the **installed version**, not assumptions:

1. `Read` `vendor/spatie/laravel-permission/src/Traits/HasRoles.php` to confirm the trait namespace (`Spatie\Permission\Traits\HasRoles`).
2. `Read` `config/permission.php` for the configured `models` (`Permission`, `Role` classes), `guard_name` defaults, and `column_names`.
3. Note the correct model classes to import (`Spatie\Permission\Models\Role`, `Spatie\Permission\Models\Permission`, unless the project overrides them in config).

## Step 2: Parse Arguments & Gather Intent

- **Model**: the argument, or `User` by default. Resolve its path (`app/Models/{Model}.php`).
- **Guard**: default `web`. Ask if the project uses a non-default guard (e.g. `api`).
- **Roles & permissions**: ask the user which roles and permissions to scaffold, unless they already specified them. If they want a starting point, propose a minimal, generic set (e.g. roles `admin`, `user`; permissions per resource as `view-X`, `create-X`, `update-X`, `delete-X`) and confirm before writing — do NOT invent a large matrix unprompted.

## Step 3: Add the HasRoles Trait to the Model

1. `Read` `app/Models/{Model}.php`.
2. If the model does not already use the trait, add:
   - `use Spatie\Permission\Traits\HasRoles;` at the top.
   - `use HasRoles;` inside the class (alongside existing traits).
3. Do not change anything else in the model.

## Step 4: Generate the Seeder

Create `database/seeders/RolesAndPermissionsSeeder.php` (skip/append if it already exists — never overwrite without confirmation):

```php
<?php

namespace Database\Seeders;

use Illuminate\Database\Seeder;
use Spatie\Permission\Models\Permission;
use Spatie\Permission\Models\Role;
use Spatie\Permission\PermissionRegistrar;

class RolesAndPermissionsSeeder extends Seeder
{
    public function run(): void
    {
        app(PermissionRegistrar::class)->forgetCachedPermissions();

        $permissions = [
            // 'view-posts', 'create-posts', 'update-posts', 'delete-posts',
        ];

        foreach ($permissions as $permission) {
            Permission::findOrCreate($permission, 'web');
        }

        $admin = Role::findOrCreate('admin', 'web');
        $admin->givePermissionTo(Permission::all());

        Role::findOrCreate('user', 'web');
    }
}
```

Requirements:
- Always call `forgetCachedPermissions()` first — new permissions won't apply otherwise.
- Use `findOrCreate` (idempotent) so the seeder is safe to re-run.
- Use the guard chosen in Step 2 (not hardcoded `web` if the project differs).
- Fill `$permissions` and roles with what the user actually asked for. Keep it minimal — no extra logging, comments, or unrequested roles.
- Match the project's existing seeder style if other seeders exist (inspect them first with `Glob`/`Read`).

## Step 5: Wire the Seeder

1. `Read` `database/seeders/DatabaseSeeder.php`.
2. Add `$this->call(RolesAndPermissionsSeeder::class);` to its `run()` if not already present.

## Step 6: Verify & Show Usage

1. Run `php -l` on the changed files (Docker-aware: detect container from `docker-compose.yml`).
2. Tell the user to run the seeder: `php artisan db:seed --class=RolesAndPermissionsSeeder`.
3. Show usage:

```php
// Assign
$user->assignRole('admin');
$user->givePermissionTo('update-posts');

// Check
$user->hasRole('admin');
$user->can('update-posts');

// Blade
@can('update-posts') ... @endcan
@role('admin') ... @endrole

// Route middleware (see note below — the alias must be registered in v6)
Route::put('/posts/{post}', ...)->middleware('permission:update-posts');
```

If the user wants to use the `permission:`/`role:` route middleware, remind them that **spatie v6 does not auto-register the aliases**. Offer to add them:

- **Laravel 11+** in `bootstrap/app.php`:
  ```php
  ->withMiddleware(function (Middleware $middleware) {
      $middleware->alias([
          'role' => \Spatie\Permission\Middleware\RoleMiddleware::class,
          'permission' => \Spatie\Permission\Middleware\PermissionMiddleware::class,
          'role_or_permission' => \Spatie\Permission\Middleware\RoleOrPermissionMiddleware::class,
      ]);
  })
  ```
- **Laravel 9/10** in `app/Http/Kernel.php` `$middlewareAliases`:
  ```php
  'role' => \Spatie\Permission\Middleware\RoleMiddleware::class,
  'permission' => \Spatie\Permission\Middleware\PermissionMiddleware::class,
  'role_or_permission' => \Spatie\Permission\Middleware\RoleOrPermissionMiddleware::class,
  ```

## Notes

- Generates a clean, minimal setup. Do not add extra roles, permissions, comments, or a super-admin `Gate::before` unless the user asks.
- If the model already has `HasRoles` or the seeder already exists, report it and ask before modifying.
- After seeding, the audit counterpart is `/lc:spatie-permissions-audit` to verify everything lines up.
