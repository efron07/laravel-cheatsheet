# 4. Middleware, CORS and rate limiting

## 4.1 What is middleware?

**What it is:** Middleware is a checkpoint that every request passes through before reaching the controller (and on the way back out). Each checkpoint can let the request through, change it, or stop it.

Think of the pharmacy entrance: a guard checks you are staff (`auth`), then checks your role before you enter the dispensary (`role:pharmacist`).

**Built-in middleware you will use often:**

| Alias | What it does |
| --- | --- |
| `auth` | Only logged-in users; others go to the login page |
| `guest` | Only users who are NOT logged in (login, register pages) |
| `verified` | Only users with a verified email |
| `throttle:60,1` | Max 60 requests per minute |
| `can:update,medicine` | Only users allowed by a policy (section 10) |
| `signed` | Only valid signed URLs |

## 4.2 Global vs route middleware

- **Global middleware** runs on every request (e.g. trimming whitespace from inputs, maintenance mode).
- **Route middleware** runs only on the routes you attach it to.

```php
// Attach to one route
Route::get('/dashboard', DashboardController::class)->middleware('auth');

// Attach to a group
Route::middleware(['auth', 'verified'])->group(function () {
    Route::resource('medicines', MedicineController::class);
});

// Skip one middleware for one route
Route::get('/stock/public', StockController::class)->withoutMiddleware('auth');
```

## 4.3 How to create your own middleware

**Pharma****c****are example:** only allow staff with a certain role.

```bash
php artisan make:middleware EnsureUserHasRole
```

```php
// app/Http/Middleware/EnsureUserHasRole.php
namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

class EnsureUserHasRole
{
    // Usage: ->middleware('role:pharmacist,admin')
    public function handle(Request $request, Closure $next, string ...$roles): Response
    {
        if (! $request->user() || ! in_array($request->user()->role, $roles)) {
            abort(403, 'You are not allowed to access this area.');
        }

        return $next($request);   // let the request continue to the controller
    }
}
```

**Register an alias** in `bootstrap/app.php` (Laravel 11+; there is no `Kernel.php` any more):

```php
// bootstrap/app.php
->withMiddleware(function (Middleware $middleware) {
    $middleware->alias([
        'role' => \App\Http\Middleware\EnsureUserHasRole::class,
    ]);

    // To run on every web request instead:
    // $middleware->web(append: [\App\Http\Middleware\LogPharmacyActivity::class]);
})
```

```php
// Use it
Route::middleware(['auth', 'role:pharmacist,admin'])->group(function () {
    Route::post('/batches', [BatchController::class, 'store']);   // receive stock
});
```

**Code before `$next($request)`** runs before the controller. **Code after it** runs after the controller, on the response:

```php
public function handle(Request $request, Closure $next): Response
{
    $start = microtime(true);          // before
    $response = $next($request);
    $response->headers->set('X-Time-Ms', round((microtime(true) - $start) * 1000));  // after
    return $response;
}
```

## 4.4 CORS

**What it is:** Cross-Origin Resource Sharing. Browsers block a web page on one domain from calling an API on another domain unless the API allows it. Example: a React dashboard at `app.pharmacare.co.tz` calling the API at `api.pharmacare.co.tz`.

**How to do it:** Laravel handles CORS for you. Publish the config only if you need to change it:

```bash
php artisan config:publish cors
```

```php
// config/cors.php
'paths' => ['api/*', 'sanctum/csrf-cookie'],
'allowed_origins' => ['https://app.pharmacare.co.tz'],
'allowed_methods' => ['*'],
'supports_credentials' => true,
```

CORS only affects browsers. Mobile apps and server-to-server calls (like an M-Pesa callback) ignore it.

## 4.5 Rate limiting

**What it is:** A limit on how many requests a user or IP can make in a time window. It protects login pages from password guessing and your API from abuse.

```php
// Quick: 5 attempts per minute on login
Route::post('/login', [LoginController::class, 'store'])->middleware('throttle:5,1');
```

**Named limiter** — define it once in `AppServiceProvider::boot()`:

```php
use Illuminate\Cache\RateLimiting\Limit;
use Illuminate\Support\Facades\RateLimiter;

public function boot(): void
{
    RateLimiter::for('api', function (Request $request) {
        return Limit::perMinute(60)->by($request->user()?->id ?: $request->ip());
    });

    RateLimiter::for('sales', function (Request $request) {
        return Limit::perMinute(30)->by($request->user()->id)
            ->response(fn () => response('Too many sales too fast. Slow down.', 429));
    });
}
```

```php
Route::middleware('throttle:sales')->post('/sales', [SaleController::class, 'store']);
```

When a limit is hit, Laravel returns HTTP **429 Too Many Requests**.
