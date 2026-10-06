# 14. Errors, logging and debugging

## 14.1 Handling exceptions

**What it is:** An exception is PHP's way of saying "something went wrong, stop here". Laravel catches every uncaught exception, logs it, and shows an error page (detailed when `APP_DEBUG=true`, a plain page when `false`).

**Throw your own meaningful exception (PharmaCare: not enough stock):**

```bash
php artisan make:exception InsufficientStockException
```

```php
// app/Exceptions/InsufficientStockException.php
class InsufficientStockException extends Exception
{
    public function __construct(public Medicine $medicine, public int $requested, public int $available)
    {
        parent::__construct("Only {$available} of {$medicine->name} in stock, {$requested} requested.");
    }

    // How this exception turns into a response
    public function render(Request $request)
    {
        if ($request->expectsJson()) {
            return response()->json(['message' => $this->getMessage()], 422);
        }
        return back()->withInput()->withErrors(['stock' => $this->getMessage()]);
    }

    // Extra data added to the log entry
    public function context(): array
    {
        return ['medicine_id' => $this->medicine->id, 'requested' => $this->requested];
    }
}
```

```php
// Use it inside the sale logic
if ($available < $qty) {
    throw new InsufficientStockException($medicine, $qty, $available);
}
```

**Try/catch** when you can recover:

```php
try {
    $txId = $gateway->charge($phone, $sale->total);
} catch (ConnectionException $e) {
    report($e);                          // log it but keep going
    return back()->withErrors(['payment' => 'M-Pesa is not responding. Take cash or retry.']);
}
```

**Global settings** in `bootstrap/app.php`:

```php
->withExceptions(function (Exceptions $exceptions) {
    // Do not log these at all
    $exceptions->dontReport(InsufficientStockException::class);

    // Custom JSON for all API 404s
    $exceptions->render(function (NotFoundHttpException $e, Request $request) {
        if ($request->is('api/*')) {
            return response()->json(['message' => 'Record not found.'], 404);
        }
    });
})
```

## 14.2 HTTP exceptions

**What it is:** Stop and return an HTTP error status straight away.

```php
abort(404);                                   // Not Found
abort(403, 'Cashiers cannot change prices.');  // Forbidden
abort_if($batch->expires_at->isPast(), 422, 'This batch has expired.');
abort_unless($user->is_active, 403);
```

| Code | Meaning | Typical cause |
| --- | --- | --- |
| 401 | Unauthenticated | Not logged in / bad API token |
| 403 | Forbidden | Logged in but not allowed (policy) |
| 404 | Not found | Wrong URL, missing record |
| 419 | Page expired | Missing `@csrf` or session expired |
| 422 | Unprocessable | Validation failed |
| 429 | Too many requests | Rate limit hit |
| 500 | Server error | A bug: check the log |
| 503 | Service unavailable | Maintenance mode (`php artisan down`) |

**Custom error pages:** create `resources/views/errors/404.blade.php` (or `403`, `500`, `503`). Laravel uses it automatically. To start from Laravel's own pages: `php artisan vendor:publish --tag=laravel-errors`.

## 14.3 Logging

**What it is:** Writing messages to a log file (or Slack, or a log service) so you can see what happened after the fact. Default file: `storage/logs/laravel.log`.

**Log levels, least to most serious:** `debug`, `info`, `notice`, `warning`, `error`, `critical`, `alert`, `emergency`.

```php
use Illuminate\Support\Facades\Log;

Log::info('Stock received', ['batch' => $batch->batch_no, 'qty' => $batch->quantity, 'by' => auth()->id()]);
Log::warning('Sale voided', ['sale_id' => $sale->id, 'reason' => $reason]);
Log::error('M-Pesa callback failed', ['payload' => $request->all()]);

logger('quick debug message');       // shortcut for Log::debug
```

**Channels and stacks** (`config/logging.php`, chosen by `LOG_CHANNEL` in `.env`):

| Channel | What it does |
| --- | --- |
| `single` | One file that grows forever |
| `daily` | A new file each day, keeps N days (`LOG_DAILY_DAYS=14`) |
| `slack` | Sends to a Slack channel (use for `critical` only) |
| `stack` | Sends each message to several channels at once |

```ini
LOG_CHANNEL=stack
LOG_STACK=daily,slack
LOG_LEVEL=debug
```

**A separate channel for payments** (keep money logs apart):

```php
// config/logging.php → 'channels'
'payments' => [
    'driver' => 'daily',
    'path'   => storage_path('logs/payments.log'),
    'level'  => 'info',
    'days'   => 90,
],
```

```php
Log::channel('payments')->info('M-Pesa charge', ['sale' => $sale->id, 'tx' => $txId]);

// Add context to every log line for the rest of this request
Log::withContext(['request_id' => (string) Str::uuid(), 'user_id' => auth()->id()]);
```

```bash
tail -f storage/logs/laravel.log     # watch the log live
php artisan pail                     # Laravel's live log viewer in the terminal
```

## 14.4 Debugging basics

```php
dd($medicine);                 // dump and die: print it, stop everything
dump($medicine);               // print it, keep running
$medicine->dd();               // on collections, queries and models
Medicine::where('price', '>', 1000)->dd();         // shows the SQL and bindings
Medicine::where('price', '>', 1000)->toRawSql();   // SQL with values filled in

DB::enableQueryLog();
// ... run code ...
dd(DB::getQueryLog());          // every query that ran
```

**Laravel Debugbar** — a toolbar at the bottom of every page during development showing queries (and duplicates: your N+1 detector), timing, views, session and memory.

```bash
composer require barryvdh/laravel-debugbar --dev
```

It shows only when `APP_DEBUG=true`. Never enable it in production.

**Laravel Telescope** — a full debugging dashboard at `/telescope` that records every request, query, job, exception, log, email, notification and scheduled task.

```bash
composer require laravel/telescope --dev
php artisan telescope:install
php artisan migrate
```

| Tool | Best for |
| --- | --- |
| `dd()` / `dump()` | Quick look at one value |
| Debugbar | "Why is this page slow? How many queries?" |
| Telescope | "What happened to that job / email / API request 10 minutes ago?" |
| Logs | Production: what happened while nobody was watching |

Rule: in production, `APP_DEBUG=false`. A debug error page shows your `.env` values, including database passwords, to anyone.
