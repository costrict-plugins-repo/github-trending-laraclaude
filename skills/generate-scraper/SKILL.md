---
name: lc:generate-scraper
description: Create a Larascraper scraper (v2 or v3) for a target website, chaining browser actions (click, type, wait, scroll), conditional flow (when/repeatUntil), captcha solving, and file/PDF downloads. Detects the installed major and generates the matching style.
argument-hint: "[ScraperName] [url]"
user-invocable: true
allowed-tools: Read Grep Bash Edit Write Glob WebFetch
---

# Generate Larascraper Scraper

Create a [Larascraper](https://github.com/edulazaro/larascraper) scraper that fetches a target URL with Puppeteer and parses it. The action chain is a little query builder for the page: chain browser **actions** (click, type, wait, waitForSelector, scroll, ...), branch and loop with **conditional flow** (`when()` / `repeatUntil()` + `Condition`), solve simple **captchas** (`solveCaptcha()`), and download **files/PDFs**.

**This skill supports both larascraper v2 and v3.** They are two different models and the generated code is NOT interchangeable:

- **v2** -- actions are chained at the **call site** (`{Scraper}::scrape($url)->click(...)->run()`); `handle()` parses `$this->crawler`; `run()` returns an HTTP-shaped `ScraperResponse` (`success`, `status`, `error`, `html`, `data`, `file`, `contentType`).
- **v3** -- a breaking redesign. `{Scraper}::run($url)`; actions live **inside** `handle()` on the `$this->scrape($url)->click(...)` chain; parsing moves out into a separate `Crawler` class (`$this->filter(...)`); `run()` returns a minimal `ScraperResponse` (`data`, `success`, `error` only), HTTP facts are internal via `$this->request`, and a request failure throws `RequestException`.

So the very first thing to decide is **which major is installed**, then everything version-specific branches on it. Do NOT assume v2 or v3 from memory; detect it (Step 1).

## Subcommands

| Subcommand | Description |
|---|---|
| *(no argument)* | Prompt for the scraper name and the target URL / what to scrape. |
| `[ScraperName]` | Generate the scraper class, then ask for the target URL. |
| `[ScraperName] [url]` | Generate the scraper for the given URL and infer the selectors/actions. |

This is a generator skill -- it always creates files. There is no analyze-only mode.

## Process

### Step 0: Verify Larascraper Is Installed (shared)

1. Use `Grep` to check if `edulazaro/larascraper` exists in `composer.json`.
2. If NOT found, stop and inform the user:
   ```
   Larascraper is not installed. Install it with:
   composer require edulazaro/larascraper
   php artisan larascraper:install   # installs the Node packages Puppeteer needs
   ```
3. If found, also confirm the Node side is ready (the browser driver shells out to Node + Puppeteer). If `node_modules/puppeteer-extra` is missing, tell the user to run `php artisan larascraper:install`. The runner will otherwise fail with a clear message, but it is better to catch it up front.

### Step 1: Detect the Installed Major Version (sets v2-mode / v3-mode)

Everything after this branches on the result, so pin it down first. Use TWO signals and let them agree.

**A. Read the version from the lockfile (authority).** The consumer project's `composer.lock` records the resolved version. Prefer `jq`, fall back to plain PHP (Docker-aware -- run inside the container if the project uses Sail):
```bash
jq -r '.packages[] | select(.name=="edulazaro/larascraper") | .version' composer.lock
# fallback if jq is absent:
php -r '$l=json_decode(file_get_contents("composer.lock"),true);foreach($l["packages"] as $p){if($p["name"]==="edulazaro/larascraper")echo $p["version"],PHP_EOL;}'
```
A major of `3` (`v3.x` / `3.x`) -> **v3-mode**; a major of `2` (the current v2 line, e.g. `2.3.x`) or the older `1.x` -> **v2-mode**. A branch alias (`dev-main`, `dev-master`) is not conclusive on its own -- confirm with signal B. (Note: the package's own working-tree `composer.json` has no `version` field, so the lockfile in the consumer project is the authority, not vendor's `composer.json`.)

**B. Structural tell in vendor (reliable fallback / tie-breaker).** v3 introduced two files that do not exist in v2:
```bash
ls vendor/edulazaro/larascraper/src/Crawler.php \
   vendor/edulazaro/larascraper/src/Support/FetchBuilder.php 2>/dev/null
```
- Both present -> **v3-mode** (also `src/Support/RequestResponse.php` and `src/Support/ScraperResponse.php` with a 3-field shape).
- Absent -> **v2-mode**: the base `Scraper` exposes a `$this->crawler` (Symfony DomCrawler) property, and file downloads go through a `FileScraper` class. (Careful: `src/FileScraper.php` still SHIPS in v3 as a deprecated shim, so its presence does NOT mean v2 -- rely on `Crawler.php` + `FetchBuilder.php`.)

Announce the detected mode to the user (e.g. "Detected larascraper v3 -- generating a Scraper + Crawler pair") before generating anything.

### Step 2: Read the Base Classes For the Detected Version

Read the installed package from vendor to match the **actual installed API**, not assumptions.

**v2-mode:** `Read` `vendor/edulazaro/larascraper/src/Scraper.php` and `Concerns/BuildsActions.php`. Note:
- Fluent config + terminal: `scrape()` (static), `proxy()`, `timeout()`, `headers()`, `retry()`, `run()`.
- Action methods (Step "Actions Reference").
- Conditional flow `when()` / `repeatUntil()` + `Support\Condition` (2.1+); captcha `solveCaptcha()` (2.1+); file downloads via `FileScraper` + `submitAndCapture()` (2.2+), bytes in `$result->file`.
- `handle()` parses `$this->crawler` (a `Symfony\Component\DomCrawler\Crawler`).

**v3-mode:** `Read` `vendor/edulazaro/larascraper/src/Scraper.php`, `src/Crawler.php`, `src/Support/FetchBuilder.php`, `src/Support/ScraperResponse.php`, `src/Support/RequestResponse.php`, `src/Support/CapturedFile.php`, `src/Exceptions/RequestException.php`, `src/Exceptions/ScrapeException.php`, and `Concerns/BuildsActions.php`. Note:
- Entry points are STATIC on the class: `run(...$params)`, `with(...$params)->run(...)`, `make(...)`. `handle()` is the default method run() invokes; its args come from run()'s params by reflection (positional / named / assoc-array-by-name).
- `$this->scrape($url)` returns a `FetchBuilder`; the same action methods are chained on it **inside** `handle()`.
- Terminals: `->crawl(FooCrawler::class)->run()` -> `ScraperResponse`; `->crawl('css')->text()` -> `string`; `->crawl('css')->texts()` -> `string[]`; `->capture()->file()` -> `CapturedFile`; `->run()` (no crawler) -> `ScraperResponse` whose `data` is the raw html.
- `ScraperResponse` is `{data, success, error}` only. `RequestResponse` (HTTP: `status`, `error`, `html`, `file`, `contentType`, `cookies`) is internal, reachable as `$this->request`.
- `$this->fail('code')` / `$this->ok($data)` inside `handle()`; a `ScrapeException` thrown in `handle()` or a `Crawler` is caught and folded into `success=false`; a request-level failure throws `RequestException` ($e->response->status).

If a method/class the plan needs is missing from the installed version, use only what is present and tell the user to `composer update edulazaro/larascraper`.

### Step 3: Parse Arguments (shared)

- **Scraper name**: e.g., `BikeScraper`. Append `Scraper` suffix if not already present. Supports subfolders: `News/MegaScraper`.
- **Target URL**: the page to scrape. If not provided, ask for it.
- **What to extract**: ask the user what fields they want (title, price, list of items, ...), unless obvious from the request.

### Step 4: Study Existing Scrapers (shared)

1. Use `Glob` to find existing scrapers in `app/Scrapers/` (and `app/Services/**Scraper*.php`).
2. If any exist, `Read` one or two to match conventions: imports, how `handle()` is structured, return shape, how they are invoked (proxy, retry). **They will already be in the project's larascraper style** -- a strong confirmation of the detected version (v2 parse `$this->crawler`; v3 chain `$this->scrape()` + a `Crawler` class).
3. Note the project's proxy/retry conventions so the generated usage snippet matches (several projects always chain `->retry(3, 20)->proxy(...)` in v2, or set `$tries` / `$proxy` properties in v3).

### Step 5: Generate the Class

Prefer the package's own generator (up-to-date stub for the installed version):

```bash
php artisan make:scraper {ScraperName}
```

Be **Docker-aware**: if a `docker-compose.yml` / Sail setup exists, run artisan inside the container (e.g., `docker exec -u sail -w /var/www/html <container> php artisan make:scraper {ScraperName}`). This creates `app/Scrapers/{ScraperName}.php`. If `make:scraper` is unavailable, `Write` the file manually per the variant below.

**v2 -- one class extending `Scraper`, `handle()` parses `$this->crawler`:**
```php
<?php

namespace App\Scrapers;

use EduLazaro\Larascraper\Scraper;

/**
 * Scrapes a page and returns the parsed data.
 */
class {ScraperName} extends Scraper
{
    /**
     * Parse the fetched page (already navigated + actioned) and return the data.
     *
     * @return array
     */
    protected function handle(): array
    {
        return [
            //
        ];
    }
}
```

**v3 -- a Scraper whose `handle()` drives the fetch chain, plus a SEPARATE Crawler:**

The generator's stub returns raw html (`$this->scrape($url)->run()`); rewrite `handle()` to crawl. There is no `make:crawler` command -- `Write` the Crawler class by hand.

```php
<?php

namespace App\Scrapers;

use EduLazaro\Larascraper\Scraper;
use EduLazaro\Larascraper\Support\ScraperResponse;

/**
 * Scrapes a page and returns the parsed data.
 */
class {ScraperName} extends Scraper
{
    protected int $tries = 3;   // retries / proxy / timeout live on the class

    /**
     * Default method (invoked by {ScraperName}::run($url)).
     *
     * @param  string  $url
     * @return \EduLazaro\Larascraper\Support\ScraperResponse
     */
    protected function handle(string $url): ScraperResponse
    {
        return $this->scrape($url)
            // ->click(...)->waitForSelector(...)   // actions go HERE, inside handle()
            ->crawl({CrawlerName}::class)
            ->run();
    }
}
```
```php
<?php

namespace App\Scrapers;

use EduLazaro\Larascraper\Crawler;

/**
 * Parses the fetched HTML. Independent of how it was fetched (reusable).
 */
class {CrawlerName} extends Crawler
{
    /**
     * @return array
     */
    protected function handle(): array
    {
        return [
            //
        ];
    }
}
```

### Step 6: Inspect the Target Page (shared)

Determine the right selectors **from the real page**, never guess:

1. Quick structural peek with `WebFetch` on the target URL, OR
2. **Better**, run the fetch once to get the exact HTML Puppeteer sees (JS already executed). A throwaway via tinker (Docker-aware):
   - **v2:** `php artisan tinker --execute="dd(\App\Scrapers\{ScraperName}::scrape('{url}')->run()->html);"`
   - **v3:** `php artisan tinker --execute="dd(\App\Scrapers\{ScraperName}::make()->scrape('{url}')->run()->data);"` -- `make()->scrape($url)->run()` uses the `FetchBuilder::run()` terminal, whose `->data` is the raw html regardless of what `handle()` does. (A request-level failure throws `RequestException`.)
   Read the returned HTML to pick stable selectors.
3. Decide whether the content is on load or **needs interaction** (same decision either version -- only WHERE the actions are placed differs, see Step 8):
   - Cookie/consent wall -> `click('#accept')`, or `when(Condition::selectorExists('#banner'), fn($b) => $b->click('#accept'))` if it only sometimes appears.
   - Content behind a search/filter form -> `type()` + `press('Enter', waitForNavigation: true)` or `click($submit, waitForNavigation: true)`.
   - Pagination / "load more" -> `clickAndWait('a.next')` + `waitForSelector()`, or `repeatUntil(Condition::selectorExists('#end'), fn($b) => $b->clickAndWait('a.next'), max: N)`.
   - Lazy / infinite scroll -> `repeatUntil(Condition::selectorExists('#footer'), fn($b) => $b->scrollToBottom()->wait(500), max: N)`.
   - Content rendered late by JS -> `waitForSelector()` on the element you need.
   - Simple image captcha -> `solveCaptcha('#captcha-img', '#captcha-input')`, usually inside a `repeatUntil` (Step 9). Needs `php artisan larascraper:install --captcha`.
   - The result is a file/PDF -> capture it instead of parsing HTML (Step 9).

### Step 7: Where Parsing Lives -- Fill It In

**v2 -- fill `handle()`, parsing `$this->crawler`:**
```php
protected function handle(): array
{
    return [
        'title' => $this->crawler->filter('h1')->text(''),
        'price' => $this->crawler->filter('.price')->text(''),
    ];
}
```
For a list, map over a node list:
```php
protected function handle(): array
{
    return $this->crawler->filter('.product-card')->each(function ($node) {
        return [
            'name'  => $node->filter('.name')->text(''),
            'price' => $node->filter('.price')->text(''),
            'link'  => $node->filter('a')->attr('href'),
        ];
    });
}
```

**v3 -- fill the `Crawler` class, parsing `$this->filter(...)`:**
```php
protected function handle(): array
{
    return [
        'title' => $this->filter('h1')->text(''),
        'price' => $this->filter('.price')->text(''),
    ];
}
```
List form (same Symfony DomCrawler `each()`):
```php
protected function handle(): array
{
    return $this->filter('.product-card')->each(function ($node) {
        return [
            'name'  => $node->filter('.name')->text(''),
            'price' => $node->filter('.price')->text(''),
            'link'  => $node->filter('a')->attr('href'),
        ];
    });
}
```
The Crawler can signal a **content-level failure** (expected selector missing = a block/captcha page) by throwing `ScrapeException`, which the `crawl()` terminal folds into `success=false` + `error`:
```php
protected function handle(): array
{
    if ($this->filter('.product-card')->count() === 0) {
        throw new \EduLazaro\Larascraper\Exceptions\ScrapeException('no_results');
    }
    // ...
}
```
**No Crawler class needed for a trivial extraction** -- crawl a selector inline from `handle()`:
```php
protected function handle(string $url): ScraperResponse
{
    $names = $this->scrape($url)->crawl('.bike-card h3')->texts();  // string[]
    return $this->ok($names);
    // or a single value: ->crawl('h1')->text()  // string
}
```

Both versions: use `->text('')` / `->attr('...')` with safe defaults so a missing node does not throw.

### Step 8: Build the Usage Snippet With Actions

The action methods are identical across versions (Actions Reference). **The only difference is placement:** v2 chains them at the CALL SITE before `run()`; v3 chains them INSIDE `handle()` on `$this->scrape($url)`.

**v2 -- actions at the call site, read the HTTP-shaped `ScraperResponse`:**

`run()` returns a `ScraperResponse` (`->success`, `->status`, `->error`, `->html`, `->data`); the parsed `handle()` output is `->data`.

```php
// Plain page (no interaction)
$result = {ScraperName}::scrape('{url}')
    ->retry(3, 20)
    ->run();

$items = $result->data;              // parsed handle() output (check $result->success first)
```
```php
// Cookie wall + lazy load
$result = {ScraperName}::scrape('{url}')
    ->click('#accept-cookies')
    ->scrollToBottom()
    ->waitForSelector('.product-card')
    ->run();
```
```php
// Search form
$result = {ScraperName}::scrape('{url}')
    ->type('#search', 'zelda')
    ->press('Enter', waitForNavigation: true)
    ->waitForSelector('.results')
    ->run();
```
```php
// Conditional flow (only-if + bounded retry loop)
use EduLazaro\Larascraper\Support\Condition;

$result = {ScraperName}::scrape('{url}')
    ->when(
        Condition::selectorExists('#cookie-banner'),
        fn ($b) => $b->click('#accept'),
    )
    ->repeatUntil(
        Condition::selectorExists('#end-of-list'),
        fn ($b) => $b->scrollToBottom()->wait(500),
        max: 10,
    )
    ->run();
```

**v3 -- actions inside `handle()`, read the minimal `ScraperResponse` at the caller:**

`handle()` builds the chain; `run()` normalizes its return into `ScraperResponse{data, success, error}`.

```php
// Inside the Scraper: cookie wall + lazy load, then crawl
protected function handle(string $url): ScraperResponse
{
    return $this->scrape($url)
        ->click('#accept-cookies')
        ->scrollToBottom()
        ->waitForSelector('.product-card')
        ->crawl({CrawlerName}::class)
        ->run();
}
```
```php
// Inside the Scraper: conditional flow, same Condition factory
use EduLazaro\Larascraper\Support\Condition;

protected function handle(string $url): ScraperResponse
{
    return $this->scrape($url)
        ->when(
            Condition::selectorExists('#cookie-banner'),
            fn ($b) => $b->click('#accept'),
        )
        ->repeatUntil(
            Condition::selectorExists('#end-of-list'),
            fn ($b) => $b->scrollToBottom()->wait(500),
            max: 10,
        )
        ->crawl({CrawlerName}::class)
        ->run();
}
```
```php
// Inside the Scraper: content check with $this->fail('code')
protected function handle(string $url): ScraperResponse
{
    $r = $this->scrape($url)->crawl({CrawlerName}::class)->run();

    if (! $r->success || empty($r->data)) {
        return $this->fail('no_results');   // success=false, error='no_results'; no throw, no new
    }

    return $r;                              // pass the ScraperResponse through
}
```
Caller side (v3) -- one exception to catch, content failure is data:
```php
use EduLazaro\Larascraper\Exceptions\RequestException;

try {
    $r = {ScraperName}::run('{url}');          // always a ScraperResponse
    // ...or configure first: {ScraperName}::with(driver: 'http')->run('{url}');
    if (! $r->success) {
        // content failure: $r->error is 'captcha' / 'no_results' / ...
    }
    $data = $r->data;
} catch (RequestException $e) {
    $status = $e->response->status;            // request layer only (retry 503, skip 404, ...)
}
```
Inside `handle()`, HTTP facts (when you genuinely need them) are on `$this->request`: `$this->request->status`, `$this->request->cookies`. Fold whatever you need into the returned `data`.

Match the project's proxy/retry conventions from Step 4 when present.

### Step 9: File / PDF Downloads

**v2 -- `FileScraper` used directly, bytes in `$result->file`:**
```php
use EduLazaro\Larascraper\FileScraper;

$result = FileScraper::scrape('{url}')
    ->submitAndCapture('form', ['expect' => 'application/pdf'])
    ->run();

if ($result->success && $result->file) {
    file_put_contents('document.pdf', $result->file);   // raw bytes; type in $result->contentType
}
```
File behind a captcha (stop once captured):
```php
use EduLazaro\Larascraper\Support\Condition;

FileScraper::scrape('{url}')
    ->repeatUntil(
        Condition::captured(),
        fn ($b) => $b->solveCaptcha('#captcha-img', 'input[name=captcha]')
                     ->submitAndCapture('form', ['expect' => 'application/pdf']),
        max: 8, delay: 500,
    )
    ->run();
```
`FileScraper` is used **directly** (no class to write); the bytes land in `$result->file`, not `$result->data`.

**v3 -- capture inside `handle()`, `->capture()->file()` returns a `CapturedFile`:**

Do not use `FileScraper` in v3 (it ships only as a deprecated 3.0 shim). Capture the binary from the fetch chain instead:
```php
use EduLazaro\Larascraper\Support\ScraperResponse;

protected function handle(string $url): ScraperResponse
{
    $file = $this->scrape($url)
        ->click('a.datasheet')
        ->capture('application/pdf')   // expect a content-type substring
        ->file();                      // runs the chain, returns a CapturedFile

    $text = $file->text() ?: $file->vision('ai');   // text layer, or OCR if scanned

    if (trim($text) === '') {
        return $this->fail('no_text');
    }

    return $this->ok(['text' => $text]);
}
```
`CapturedFile` API: `->text($engine = 'gs'|'poppler'|'smalot')` (text layer, fast/free, '' for scanned PDFs), `->vision($engine = 'ai'|'tesseract')` (OCR; `ai` costs money), `->bytes()`, `->save($path)`, `->contentType()`, `->size()`.

File behind a captcha (v3) -- `repeatUntil` + `Condition::captured()` inside `handle()`, then `->file()`:
```php
use EduLazaro\Larascraper\Support\Condition;

$file = $this->scrape($url)
    ->repeatUntil(
        Condition::captured(),
        fn ($b) => $b->solveCaptcha('#captcha-img', 'input[name=captcha]')
                     ->submit('form')
                     ->capture('application/pdf'),
        max: 8, delay: 500,
    )
    ->file();
```

### Step 10: Verify

1. `Read` the generated file(s) to confirm they are correct (v3: both the Scraper AND the Crawler).
2. Run `php -l` on each file (Docker-aware).
3. If it is safe to do so, run the scraper once and confirm the result:
   - **v2:** run at the call site and check `$result->success === true` and that `$result->data` is populated.
   - **v3:** `{ScraperName}::run('{url}')` and check `$r->success === true` and `$r->data`. A content miss comes back as `$r->success === false` + `$r->error`; a request failure throws `RequestException` ($e->response->status). Refine selectors/actions if data comes back empty.
4. If an action fails (a selector never appears): in v2 `$result->success` is false and `$result->error` holds the message; in v3 add a `waitForSelector()` before it or, for a genuine content block, let the Crawler throw `ScrapeException` so `$r->error` names it.

### Step 11: Show the Final Usage

Show the user the tailored call plus how to read the result, matching the detected version:
- **v2:** the call-site chain and `$result->data` / `$result->success`; remind them `handle()` parses the HTML **after** all actions run.
- **v3:** `{ScraperName}::run($url)`, the `$r->data` / `$r->success` / `$r->error` branch, and the `RequestException` catch; remind them actions live **inside** `handle()` and parsing lives in the `Crawler`.

## Actions Reference (shared -- placement differs by version)

These build an ordered list Puppeteer runs in a single browser session, after navigating and before the HTML is parsed. The waits happen inside Node (where the page is alive), not in PHP. **v2 chains them at the call site before `run()`; v3 chains them inside `handle()` on `$this->scrape($url)`.** The method names and signatures are the same.

| Method | Use it for |
|---|---|
| `->click($selector)` | Click an element (waits for it first). |
| `->click($selector, waitForNavigation: true)` / `->clickAndWait($selector)` | A click that loads a new page. |
| `->type($selector, $text)` | Fill an input. |
| `->select($selector, $value)` | Pick a `<select>` option by value. |
| `->hover($selector)` | Reveal hover menus. |
| `->press($key)` | Press a key; pass `waitForNavigation: true` when it submits. |
| `->waitForSelector($selector)` | Wait for lazy/JS content to appear. |
| `->waitForNavigation()` | Wait for a navigation to finish. |
| `->wait($ms)` | Fixed pause in milliseconds. |
| `->scroll('bottom'\|'top')` / `->scrollToBottom()` | Trigger lazy / infinite scroll. |
| `->when($cond, $then, $else?)` | Run a branch only if a condition holds (closures receive a sub-builder). |
| `->repeatUntil($cond, $body, max:, delay:)` | Repeat a branch until a condition holds. Always bounded by `max`. |
| `->solveCaptcha($img, $input, $opts?)` | OCR a simple image captcha and type it. Needs `--captcha` install. |

**File capture (Step 9) differs by version:** v2 uses `->submitAndCapture($form, ['expect'=>...])` and reads `$result->file`; v3 uses `->submit($form)->capture('application/pdf')` (or just `->capture(...)` after a click/navigation) and the `->file()` terminal returning a `CapturedFile`. v3 (and 2.3) also expose `->setValue()`, `->gotoAttr()`, `->reload()`, `->visit()` -- read `BuildsActions.php` for the installed set.

`$selector` is any CSS selector, including attribute selectors like `[name=email]` or `input[name=captcha]`.

**Conditions** (`EduLazaro\Larascraper\Support\Condition`, for `when()`/`repeatUntil()` -- identical in both versions):

| Condition | True when... |
|---|---|
| `Condition::selectorExists($selector)` | an element matching the selector exists |
| `Condition::selectorMissing($selector)` | no element matching the selector exists |
| `Condition::textContains($text, $selector?)` | the text is found (in `$selector`, or the whole page) |
| `Condition::urlContains($text)` | the current URL contains the substring |
| `Condition::captured()` | a file has been captured (by `capture()` / `submitAndCapture()`) |

**Navigation tip:** for a click or key press that loads a new page, use `waitForNavigation: true` on that action (or `clickAndWait()`) instead of a separate `->waitForNavigation()` -- it arms the wait before the action, avoiding a race.

## Important Notes

- **Detect the version first (Step 1).** The generated code is not interchangeable: v2 parses `$this->crawler` in `handle()`; v3 splits fetch (`handle()` + `$this->scrape()`) from parse (a `Crawler` with `$this->filter()`).
- Interactions never live in the parse layer. v2: `handle()` is parse-only, actions go at the call site. v3: the `Crawler` is parse-only, actions go inside `handle()` on the `$this->scrape()` chain.
- **v3 response is content-only:** `ScraperResponse` is `{data, success, error}`. HTTP facts (`status`, `cookies`) are on `$this->request` inside `handle()`, or on `RequestException->response` for a request failure. A `200` can still be `success=false, error='captcha'` -- success is content-based, decided by the scraper/crawler (via `$this->fail()` or a `ScrapeException`), not by the HTTP status.
- **v3 exceptions:** only `RequestException` reaches the caller (request/transport failure). A `ScrapeException` thrown in `handle()` or a `Crawler` is caught and folded into `success=false` -- it does not propagate.
- For pure **file/PDF downloads**: v2 uses `FileScraper` directly (bytes in `$result->file`); v3 captures inline with `->capture()->file()` -> a `CapturedFile` (`->text()` / `->vision()` / `->bytes()` / `->save()`).
- `repeatUntil()` must always be bounded: pass a sensible `max`, and a `delay` when each iteration hits a remote server, so the loop never hammers it.
- Always derive selectors from the real page (Step 6), never guess from the URL. Use `->text('')` / `->attr('...')` defaults so missing nodes do not throw.
- Respect the target site: keep timeouts and retry counts reasonable, and honor the project's existing proxy usage.
- `solveCaptcha()` only handles simple image (text) captchas, not reCAPTCHA/hCaptcha. It needs the optional OCR packages (`php artisan larascraper:install --captcha`).
- If the installed package lacks a method/class, it is an older point release -- use what is present and suggest `composer update edulazaro/larascraper`.
