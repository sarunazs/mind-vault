# Live-URL verification before declaring work shipped

"Shipped" means **a human can open the live URL and see the feature working**. Green test suite + open PR + container "Up" status are necessary but **not sufficient**. Load this reference at the end of `/work`'s verification stage when the project has a public URL, before flipping the plan to `status: shipped` or claiming the work is done.

## Why this matters

Test suites run against test-shaped state — in-memory SQLite, `Auth::actingAs`, request factories that bypass real middleware. The production stack runs against MySQL/Postgres, real PHP-FPM workers with cached env vars, real session cookies, real reverse-proxy headers, real asset manifests on disk.

Every layer the test suite *bypasses* is a layer where "green locally" can co-exist with "broken on the URL":

- **DB**: tests use in-memory SQLite; production uses MySQL. JSON-column behaviour, collation, `whereJsonContains` semantics differ. Migrations might have been run on prod but **seeders forgotten** after a `migrate:fresh`.
- **PHP-FPM workers**: long-lived; they cache `.env` in process memory. A config change isn't visible until the container restarts.
- **Filesystem permissions**: php-fpm typically runs as `www-data` (uid 33). Anything a host-side `chown` made owned by uid 1000 is unwritable to FPM until the entrypoint re-chowns on next container start.
- **Assets**: `public/build/manifest.json` must exist before nginx serves. Tests never hit `@vite` directives that resolve through the manifest.
- **Auth flows**: Filament / Livewire admin login is XHR-driven; curl can't easily replicate it. Tests use `actingAs`. The two paths exercise different middleware stacks.
- **Reverse proxies**: SSL termination, host headers, `X-Forwarded-*` trust — the test framework's `Request::create()` defaults to `localhost` without any of this.

Any one of these can be the difference between "tests pass" and "user sees 404 / 500 / blank page".

## The verification checklist

Run these against the **production URL**, not localhost / not the test runner. Treat any failure as not-shipped.

1. **Hit the public landing URL**: `curl -sI <live URL>/` — expect HTTP 200, not 301-to-itself, not 404, not 500.
2. **Hit one non-trivial route**: deep link to a known content URL (a service page, a blog post, a case study). 200 + content present in the body.
3. **Hit an authenticated path** (`/admin`, `/dashboard`, equivalent): expect a sane response — the login page if anonymous, or the rendered page if you have a session cookie.
4. **If 500**: tail the production log AND grep the HTML response for the exception title. Laravel's debug page renders the full exception in HTML when `APP_DEBUG=true`.
5. **For complex auth flows** the tests bypass (Filament Livewire login, OAuth callbacks): use **server-side request simulation** — see § "Server-side request simulation" below.

## Server-side request simulation

When the production stack has authentication you can't easily replicate via curl (Filament's Livewire-driven login, OAuth callbacks, two-factor enrolment), simulate the request inside the running container:

```php
// Laravel example — drop into `php artisan tinker` on the production container
$admin = User::firstWhere('email', 'admin@example.com');
Auth::login($admin);
$kernel = app(Illuminate\Contracts\Http\Kernel::class);

foreach ([
    '/admin',
    '/admin/<resource>',
    '/admin/<resource>/create',
    '/admin/<resource>/1/edit',
] as $url) {
    $response = $kernel->handle(Illuminate\Http\Request::create($url, 'GET'));
    echo $response->getStatusCode() . '  ' . $url . PHP_EOL;
}
```

This runs the **full middleware + controller + view stack** against the **live production DB**, exercising every code path a real authenticated request would — without needing to script Filament's Livewire XHR auth dance.

The same shape works for Django (`Client.force_login(user)` + `client.get(url)`), Rails (`session[:user_id] = user.id; get url`), and any framework with a request-simulation API.

## The "ship sequence" for app-with-URL projects

When `/work`'s last commit lands on `feat/<slug>` and CI is green, before declaring done:

1. **Re-deploy** if necessary (push to remote, rebuild assets, restart workers).
2. **Run migrations on production DB** — `migrate --force`. New schema must exist.
3. **Run seeders on production DB** — `db:seed --force` (idempotent seeders). Easy to forget after a `migrate:fresh` cleared the seed data.
4. **Build production assets** — `npm ci && npm run build`. Manifest must exist before nginx serves any `@vite`-driven response.
5. **Fix runtime permissions** — `chown -R <service-user>:<service-user> storage bootstrap/cache && chmod -R 775` (Laravel-on-docker specific). Skip if the entrypoint handles it on container start AND you've restarted the container since the last host-side `chown`.
6. **Clear caches** — `artisan optimize:clear` after config / route changes.
7. **Re-verify SSL** if the cert was just issued or the proxy was reconfigured.
8. **Run the live-URL checklist above.** Public routes + auth path + content spot-check.

## When NOT to use this

- The work doesn't ship to a URL (CLI tool, library, agent skill update). Live-URL verification is moot.
- The project is local-only (dev playground). The test suite is the ground truth.
- Mid-implementation, between commits. This is a pre-merge gate, not a per-commit gate.

## Anti-patterns surfaced from real incidents

- ❌ **"Shipped — see PR"** when the PR is open but the live URL still 404s. The PR is a delivery artefact, not a delivery state.
- ❌ **"All tests pass"** when no tests authenticated against the admin / dashboard. Test coverage that never runs through the auth middleware will miss `FilamentUser::canAccessPanel()` rejections and similar gates.
- ❌ **`chown -R <host-uid> /var/www/html`** mid-session inside a container running `php-fpm` as `www-data` — breaks storage perms until next container restart. Use the entrypoint's chown rules; don't fight them. If you need host-edit access to a generated file, use `docker compose exec` + `cat > file <<EOF` instead of recursively re-owning the app dir.
- ❌ **Skipping `db:seed --force` on production** after a `migrate:fresh`. The schema is there; the content isn't. Every page that reads from the empty table 404s.
- ❌ **Building assets after bringing the stack up.** Every request between `up -d` and `npm run build` 500s on `@vite` resolution. Build first, then bring stack up.

## How to apply during /work

After CI clears on the feature branch:

1. **Tier 1 (fast)**: hit the live URL with curl. 200 on home + one non-trivial path + auth path returns sanity.
2. **Tier 2 (catches what curl can't)**: server-side request simulation for any auth-gated paths. Covers every Resource's list/create/edit URL for an admin CMS like Filament.
3. **Tier 3 (CRUD-level)**: drive Livewire (or framework-equivalent) test client against the **production DB** via tinker — `fillForm`, `call('save')`, `callAction(Delete)`. Tests run against production data; restore from seed afterwards.
4. **Only then** can the plan flip to `status: shipped` and the PR description move past "ready for review" to "verified live".

The cost of skipping tier 1 is a user opening the URL and reporting a 404 / 500 the test suite never noticed. The cost of running tier 1 is a 10-second curl loop.

---

**Last Updated**: 2026-05-22 (compounded from a Laravel + Filament project where green Pest + open PR was declared "shipped" while the live URL returned 404 for a missing seed step, then 500 for broken storage perms, then 404 again for an unbuilt asset manifest, then served wrong-locale nav links because of `app()->setLocale()`'s mid-request config mutation. Each layer was invisible to the in-memory SQLite test suite.)
