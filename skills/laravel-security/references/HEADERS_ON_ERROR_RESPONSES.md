# Security headers on error responses (the route-middleware gap)

**TL;DR** — In Laravel 11/12, a middleware that mutates the response *after* `$next()`
(the canonical security-headers / CORS pattern) silently **fails to cover thrown
error responses** — 429 throttle, `abort(404)`, `abort(403)`, any `HttpException`.
Re-applying the headers in the `withExceptions(... ->respond(...))` hook closes the
gap. Registering the middleware "first" in the group does **not** help.

## Why the obvious version is wrong

The example most security guides ship (including this skill's `## Security Headers`
section) looks like:

```php
public function handle(Request $request, Closure $next): Response
{
    $response = $next($request);          // <-- if $next throws, we never get here
    $response->headers->set('X-Frame-Options', 'DENY');
    // ...
    return $response;
}
```

When a downstream middleware or controller **throws** an `HttpException`
(`ThrottleRequestsException` for a 429, `abort(404)`, `abort(403)`, a failed
`->validate()`, model-binding 404, etc.), the exception unwinds the pipeline
straight to the kernel's exception handler. The handler renders the error response
and returns it **without** sending it back down through the route/group middleware.
So every line after `$next()` is skipped for exactly the responses an attacker is
most likely to probe: throttle blocks, 403s, 404s, 500s.

**Ordering does not fix it.** "Register `security-headers` first in the group so it
wraps `throttle`" is a natural assumption — and false. First-in-the-group means its
`$next()` is the *outermost* call; the throw still unwinds past its post-`$next()`
code. The headers land on 200s and on responses the app *returns* (e.g. a 200
maintenance page, a `->response()`-callback 429), but never on a *thrown* one.

This is easy to miss by reading code and only shows up under test — a test asserting
the header on a 429 fails even though every "normal" page has it.

## The fix — apply in BOTH places

Expose the always-on headers as a static method and call it from the middleware
(normal/returned responses) **and** from the exception `respond()` hook
(thrown/rendered responses):

```php
// app/Http/Middleware/SecurityHeaders.php
final class SecurityHeaders
{
    public function handle(Request $request, Closure $next): Response
    {
        $response = $next($request);
        self::applyBaseHeaders($response);
        return $response;
    }

    public static function applyBaseHeaders(Response $response): void
    {
        $response->headers->set('X-Frame-Options', 'DENY');
        $response->headers->set('X-Content-Type-Options', 'nosniff');
        $response->headers->set('Referrer-Policy', 'no-referrer');
    }
}
```

```php
// bootstrap/app.php
->withExceptions(function (Exceptions $exceptions) {
    // Error responses (429/404/403/500) are thrown PAST the route middleware,
    // so re-apply the always-on headers on the rendered error response here.
    $exceptions->respond(function (Response $response): Response {
        SecurityHeaders::applyBaseHeaders($response);
        return $response;
    });
})
```

Now every response — returned or thrown — carries the triad.

## Scope notes

- **Triad vs CSP.** Apply the cheap always-on headers (`X-Frame-Options`,
  `X-Content-Type-Options`, `Referrer-Policy`) in both places. A full
  `Content-Security-Policy` is usually surface-specific (different policy for the
  public site vs an admin panel that runs inline JS / `eval`), so keep CSP in the
  middleware only and leave error pages with just the triad — a bare error page has
  no scripts to constrain.
- **Same gap hits CORS.** `HandleCors` and any after-`$next()` header mutation share
  this blind spot on thrown responses. The `respond()` hook is the general fix.
- **HSTS belongs at the TLS terminator** (reverse proxy / load balancer), not in the
  app, when something upstream terminates SSL.
- **`abort()` vs returned response.** A 429 built from a named limiter's
  `->response(...)` callback is *returned* (gets headers from the middleware); a 429
  from the default throttle path is *thrown* (needs the hook). You generally can't
  predict which, so always wire the hook.

## Verify it

A render-and-assert test is the only reliable check — reasoning misses it:

```php
it('keeps the triad on a throttle-rejected (429) response', function () {
    config(['some.throttle.per_minute' => 1]);
    $this->get('/page')->assertOk();
    $this->get('/page')->assertStatus(429)->assertHeader('X-Frame-Options', 'DENY');
});
```

---

**Provenance**: surfaced 2026-06 building a surface-aware `SecurityHeaders`
middleware on a Laravel 12 + Filament project — a test asserting the triad on a 429
failed despite the middleware being registered first in the group; the
`withExceptions(...->respond(...))` hook was the fix.
