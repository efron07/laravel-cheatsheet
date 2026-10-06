# 11. Service container, dependency injection, providers and facades

These four ideas are the engine under Laravel. Once you understand them, the rest of the framework stops feeling like magic.

## 11.1 Dependency injection (DI)

**What it is:** Instead of a class creating the objects it needs, it **asks for them** in its constructor or method, and something else hands them over.

**Simple picture:** A cashier needs a card machine. Bad: the cashier builds a card machine every morning. Good: the shop gives the cashier a card machine. If the shop switches from card machine A to B, the cashier does not change.

```php
// Without DI: the controller is welded to one payment provider
class SaleController
{
    public function pay()
    {
        $mpesa = new MpesaGateway('key', 'secret');   // hard-coded
        $mpesa->charge(...);
    }
}

// With DI: the controller just asks for "a payment gateway"
class SaleController
{
    public function __construct(private PaymentGateway $gateway) {}

    public function pay(Sale $sale)
    {
        $this->gateway->charge($sale->customer->phone, $sale->total);
    }
}
```

**Why it matters:** you can swap M-Pesa for Airtel Money in one place, and in tests you can hand in a fake gateway that never charges real money.

## 11.2 The service container

**What it is:** The container is Laravel's object factory. When a class asks for something in its constructor, the container builds it, and builds everything *that* needs too. Most of the time this happens automatically with no setup ("auto-wiring").

```php
// A plain class with no interface: the container builds it automatically, nothing to register
class StockService
{
    public function totalFor(Medicine $m): int { return $m->batches()->sum('quantity'); }
}

class MedicineController extends Controller
{
    public function show(Medicine $medicine, StockService $stock)   // injected automatically
    {
        return view('medicines.show', ['medicine' => $medicine, 'stock' => $stock->totalFor($medicine)]);
    }
}

// Get something from the container yourself
$stock = app(StockService::class);
$stock = app()->make(StockService::class);
```

**When you must register (binding):** when a class asks for an **interface**, the container cannot guess which implementation you want. You tell it.

```php
// app/Contracts/PaymentGateway.php — the "contract"
interface PaymentGateway
{
    public function charge(string $phone, int $amount): string;   // returns a transaction id
}

// app/Services/MpesaGateway.php — one implementation
class MpesaGateway implements PaymentGateway
{
    public function __construct(private string $key, private string $secret) {}

    public function charge(string $phone, int $amount): string
    {
        // call the M-Pesa API here
        return 'MP' . now()->timestamp;
    }
}
```

```php
// app/Providers/AppServiceProvider.php
public function register(): void
{
    // bind: a NEW object every time it is requested
    $this->app->bind(PaymentGateway::class, fn ($app) => new MpesaGateway(
        config('services.mpesa.key'),
        config('services.mpesa.secret'),
    ));

    // singleton: ONE shared object for the whole request
    $this->app->singleton(StockService::class);
}
```

To switch to Airtel Money, change that one binding. No controller changes.

## 11.3 Service providers

**What it is:** Service providers are the start-up scripts of your app. Every feature of Laravel (database, mail, queues) is switched on by a provider. Your own setup goes in `app/Providers/AppServiceProvider.php`.

| Method | Runs | Put here |
| --- | --- | --- |
| `register()` | First, for all providers | Container bindings only. Do not use other services yet |
| `boot()` | After every provider has registered | Gates, rate limiters, event listeners, model settings, view composers |

```bash
php artisan make:provider PaymentServiceProvider   # auto-added to bootstrap/providers.php
```

```php
class PaymentServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        $this->app->singleton(PaymentGateway::class, function () {
            return match (config('pharmacy.payment_driver')) {
                'airtel' => new AirtelMoneyGateway(config('services.airtel.key')),
                default  => new MpesaGateway(config('services.mpesa.key'), config('services.mpesa.secret')),
            };
        });
    }

    public function boot(): void
    {
        // Share data with every view
        View::share('pharmacyName', config('app.name'));
    }
}
```

## 11.4 Facades

**What it is:** A facade is a short static-looking name for a service inside the container. `Cache::get('x')` looks like a static call, but behind the scenes Laravel fetches the real cache object from the container and calls `get()` on it.

```php
use Illuminate\Support\Facades\Cache;
use Illuminate\Support\Facades\Log;
use Illuminate\Support\Facades\DB;

Cache::put('daily_sales_total', 450000, now()->addMinutes(10));
Log::info('Stock received', ['batch' => 'AMX-2026-01']);
DB::table('medicines')->count();
```

**The three ways to reach the same service:**

```php
Cache::get('key');                       // facade
cache()->get('key');                     // helper function
public function __construct(private \Illuminate\Contracts\Cache\Repository $cache) {}  // injection
```

All three hit the same object. Facades are quick to write; injection makes a class's needs visible in its constructor. Both are testable: `Cache::shouldReceive('get')->andReturn(5)` fakes a facade in tests.

**Common facades:** `Route`, `DB`, `Cache`, `Log`, `Auth`, `Gate`, `Mail`, `Notification`, `Queue`, `Storage`, `Hash`, `Http`, `Validator`, `Event`, `Schedule`.

**Real-time facade:** put `Facades\` in front of any class namespace to use it like a facade:

```php
use Facades\App\Services\StockService;

StockService::totalFor($medicine);   // resolved from the container on the fly
```
