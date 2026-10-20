---
name: lc:laracrate-collection
description: Add or edit a Laracrate file collection in config/laracrate.php with the correct anatomy (disk, access, types, variants, previews, extract/embed, flags, per-model scoping).
argument-hint: "[collection-name]"
user-invocable: true
allowed-tools: Read Grep Bash Edit Write Glob
---

# Laracrate Collection

Add a new Laracrate collection (or edit an existing one) in the app's `config/laracrate.php`, generating a correct config block and wiring its dependencies (disk, embeddings, usage tracking, policies).

A "collection" is the per-context grouping a file belongs to (`avatar`, `documents`, `lawsuit-document`...). It decides the disk, the access mode, the accepted types, the variants/previews to derive, whether text extraction and embeddings run, and per-model scoping. The anatomy has many keys with non-obvious rules, so this skill encodes them.

## Subcommands

| Subcommand | Description |
|---|---|
| *(no argument)* | Prompt for the collection name and its options, then add it. |
| `[collection-name]` | Add (or edit, if it already exists) the named collection. |

This is a generator skill: it edits `config/laracrate.php`. Always confirm the final block with the user before writing if anything was assumed.

## Process

### Step 0: Verify Laracrate Is Installed

1. Use `Grep` to check `edulazaro/laracrate` in `composer.json`.
2. If NOT found, stop and tell the user:
   ```
   Laracrate is not installed. Install it with:
   composer require edulazaro/laracrate
   php artisan vendor:publish --tag=laracrate-config
   ```
3. If found, continue.

### Step 1: Locate the Published Config

1. Use `Read` on `config/laracrate.php`.
2. If it does not exist, the config has not been published. Run (Docker-aware):
   ```
   php artisan vendor:publish --tag=laracrate-config
   ```
   then read it.
3. Never edit the package's own config under `vendor/`. Only the app's `config/laracrate.php`.

### Step 2: Read the Current Config

From `config/laracrate.php`, note:
- The existing `collections` array and its formatting (indentation, trailing commas) so the new block matches.
- The global `defaults` per type (`image`, `document`, `audio`, `video`): accepted mime types, max sizes, default variants. A collection inherits these and only overrides what it declares.
- The `embeddings`, `chunks`, `watermark`, and `placeholders` blocks (you may need to touch them for dependent wiring in Step 5).
- Whether a `morphMap` / `Relation::enforceMorphMap` is used in the app (affects the `models` block keys).

### Step 3: Gather Requirements

If not obvious from the argument or the conversation, ask only for what you need. Defaults in parentheses:

1. **Collection name** (e.g. `avatar`, `documents`).
2. **Disk** (required): the `Storage::disk()` name from `config/filesystems.php`. There is no default; a missing disk is an intentional error in Laracrate.
3. **Access mode**: `public` (CDN-direct URL), `signed` (temporary signed URL), or `stream` (controller with audit and per-request viewer binding). Default `public`.
4. **Accepted types**: any of `image`, `document`, `audio`, `video`. Default: infer from intent (an "avatar"/"gallery" is image, "documents" is document).
5. **Per-type detail**:
   - image: variants (name + `width`/`height`, optional `fit` to crop square, `format`, `watermark`).
   - document/video: `preview` (document: `page`, `width`, `engine`, nested `variants`; video: `frame_at`, nested `variants`).
6. **Flags** (only if relevant): `single` (one file per owner, replacing), `sensitive` (bind URL to viewer), `encrypt` (encrypt at rest, requires server-side upload), `ttl_hours` (auto-purge), `quota_bytes`, `track_usage` (live counters), `component` (default blade), `placeholder`.
7. **Text/AI**: `extract` (extract text) and `embed` (generate embeddings + index in the ChunkStore). Each is a bool or an array of types (`['document','image']`) or extras (`['video.visual']`).
8. **Per-model scoping**: should this collection be restricted to specific owner models with different config each? If yes, gather the `models` block (keyed by morph alias).

### Step 4: Build the Collection Block

Compose the block from the gathered options, following these rules exactly:

- **`variants` live INSIDE `types`** (`types.image.variants`), never at the top level of the collection.
- **`preview` lives inside `types.document` / `types.video`**; its own derivatives go in `preview.variants`.
- Only declare what overrides the global `defaults`. The merge is recursive; lists (`variants`, `accepted_mime_types`, `accepted_extensions`) replace wholesale when declared.
- **`access`** must be exactly `public`, `signed`, or `stream`.
- **There is NO `path` key.** The object key is always `{fileable_morph}/{id}/{collection}/{file}` (tenant-prefixed when the file has a tenant), built by `CreateFileAction`. Do not invent a `path`/template key, it is ignored.
- **`extract`/`embed`** are the canonical keys. `extract_text` is only a legacy boolean alias, prefer `extract`. Array form filters by file type.
- **`watermark`** is opt-in per variant: `'display' => ['width' => 1200, 'watermark' => true]`. The original is never watermarked. The watermark image/text come from the top-level `watermark` config block.
- **`placeholder`** can be a string or a callable `(collection, type, model)`. If callable, use a callable array `[App\Support\InitialsAvatar::class, 'placeholderFor']`, NOT a `Closure` (Closures break `php artisan config:cache`).
- **`models` block** restricts the collection to the listed morph aliases and merges a per-model override (`array_replace_recursive`). Any model not listed throws `CollectionNotAllowedForModel`. Omit it for a flat, all-models collection.

Reference anatomy:

| Key | Type | Purpose |
|---|---|---|
| `disk` | string | Storage disk (required, no default). |
| `access` | string | `public` / `signed` / `stream`. |
| `single` | bool | One file per owner; replacing force-deletes the previous. |
| `sensitive` | bool | Bind access to the authenticated viewer, re-validate each request. |
| `encrypt` | bool | Encrypt the binary at rest (needs server-side upload, not presigned). |
| `ttl_hours` | int | Auto-purge via `laracrate:purge-expired`. |
| `quota_bytes` | int | Limit the app checks via `UsageReporter` (not enforced by the package). |
| `track_usage` | bool | Maintain live counters in `laracrate_folderables`. |
| `component` | string | Default blade component for rendering. |
| `placeholder` | string\|callable | Fallback when the file is missing. |
| `types` | array | Per-type config keyed by `image`/`document`/`audio`/`video`. |
| `extract` | bool\|array | Extract text (`extract_text` is a legacy alias). |
| `embed` | bool\|array | Generate embeddings and index in the ChunkStore. |
| `actions` | array | Custom pipeline steps (`FileActionInterface` classes) for this collection. |
| `models` | array | Per-morph scoping and overrides. |

Examples to mirror:

```php
// Public single avatar with cropped square variants and an initials fallback.
'avatar' => [
    'disk'        => 'media',
    'access'      => 'public',
    'single'      => true,
    'placeholder' => [\App\Support\InitialsAvatar::class, 'placeholderFor'],
    'types' => [
        'image' => [
            'variants' => [
                'small'  => ['width' => 64,  'height' => 64,  'fit' => true],
                'medium' => ['width' => 128, 'height' => 128, 'fit' => true],
            ],
        ],
    ],
],

// Signed PDFs with first-page preview, text extraction and embeddings (RAG).
'documents' => [
    'disk'    => 'documents',
    'access'  => 'signed',
    'extract' => true,
    'embed'   => true,
    'types' => [
        'document' => [
            'preview' => ['page' => 1, 'width' => 2000, 'variants' => [
                'thumbnail' => ['width' => 300],
                'medium'    => ['width' => 800],
            ]],
        ],
    ],
],

// Sensitive, streamed, encrypted IDs with a watermarked display variant.
'identity' => [
    'disk'      => 'documents',
    'access'    => 'stream',
    'sensitive' => true,
    'encrypt'   => true,
    'types' => [
        'image' => ['variants' => ['display' => ['width' => 1200, 'watermark' => true]]],
        'document' => ['preview' => ['page' => 1, 'width' => 2000]],
    ],
],
```

### Step 5: Insert and Wire Dependencies

1. Use `Edit` to add the block inside the `collections` array in `config/laracrate.php`, matching the file's existing indentation and trailing-comma style. If the collection already exists, update it in place.
2. Then check and report the dependent wiring (do the edits the user confirms):
   - **Disk**: confirm the chosen `disk` exists in `config/filesystems.php`. If not, add an `s3` (R2/S3) or `local` disk, or tell the user to add it. Laracrate intentionally errors on a missing disk.
   - **`extract`/`embed`**: these only run if `embeddings.enabled` is `true`. Confirm a provider (`OpenAiEmbeddingProvider` needs `LARACRATE_EMBEDDINGS_API_KEY`) and the `chunks.driver` (`mysql` or `meilisearch`). Point the user to the env keys.
   - **`track_usage`**: counters live in `laracrate_folderables` and are rebuilt with `php artisan laracrate:recompute-usage`.
   - **`access: stream` or `sensitive`**: authorization comes from `PolicyRegistry`. Remind the user to register `viewable`/`editable`/`deletable` for the owner morph in a service provider.
   - **`placeholder` callable**: must be a callable array, not a Closure (see Step 4).

### Step 6: Verify

1. Run `php -l config/laracrate.php` (Docker-aware: detect the container from `docker-compose.yml`/`compose.yaml`).
2. Run `php artisan config:clear` so the new collection is picked up.
3. Re-read the edited block to confirm formatting and that `variants` are nested under `types`.

### Step 7: Show Usage

After a successful add, show how to use it from a model:

```php
// The owner model needs the HasFiles trait.
$model->addFile($request->file('upload'), '{collection}');
$model->setFile('{collection}', $upload);     // single collections
$url = $model->fileLink('{collection}', 'medium');
```

## Important Notes

- Edit only the app's `config/laracrate.php`, never the copy under `vendor/`.
- `variants` belong inside `types`, `preview` inside `types.document`/`types.video`. This is the most common mistake.
- Do not add a `path` key: the object key layout is fixed and not configurable.
- `extract`/`embed` need `embeddings.enabled => true` to actually run.
- A missing disk is an intentional hard error; always confirm the disk exists in `filesystems.php`.
- Respect the package's editorial rule: no em dashes in the generated config or comments.
