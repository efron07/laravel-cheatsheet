# 12. Events, queues, scheduling and notifications

## 12.1 Events and listeners

**What it is:** An **event** is an announcement that something happened ("a sale was completed"). **Listeners** are the people who react to it ("update stock alerts", "send receipt SMS", "record loyalty points"). The code that makes the sale does not need to know who is listening.

**Why it matters:** your `SaleController` stays focused on the sale. New reactions are added as new listeners without touching it.

```bash
php artisan make:event SaleCompleted
php artisan make:listener CheckLowStock --event=SaleCompleted
php artisan make:listener SendReceiptSms --event=SaleCompleted
php artisan event:list            # see which listeners handle which events
```

```php
// app/Events/SaleCompleted.php — just carries the data
class SaleCompleted
{
    use Dispatchable, SerializesModels;

    public function __construct(public Sale $sale) {}
}
```

```php
// app/Listeners/CheckLowStock.php
class CheckLowStock
{
    public function handle(SaleCompleted $event): void
    {
        foreach ($event->sale->items as $item) {
            $medicine = $item->medicine;
            if ($medicine->batches()->sum('quantity') <= $medicine->reorder_level) {
                LowStockAlert::dispatch($medicine);   // a job, see 12.3
            }
        }
    }
}

// app/Listeners/SendReceiptSms.php — implements ShouldQueue: runs in the background
class SendReceiptSms implements ShouldQueue
{
    public function handle(SaleCompleted $event): void
    {
        // send SMS to $event->sale->customer->phone
    }
}
```

**Fire the event:**

```php
SaleCompleted::dispatch($sale);    // or: event(new SaleCompleted($sale));
```

Laravel 11+ connects listeners to events automatically by reading the type-hint in `handle()`. No registration needed.

**Model events:** Eloquent fires events by itself: `creating`, `created`, `updating`, `updated`, `deleting`, `deleted`.

```php
// app/Models/Sale.php — give every sale a receipt number before it is saved
protected static function booted(): void
{
    static::creating(function (Sale $sale) {
        $sale->receipt_no = 'RCP-' . now()->format('Ymd') . '-' . str_pad(Sale::whereDate('created_at', today())->count() + 1, 4, '0', STR_PAD_LEFT);
    });
}
```

## 12.2 Queues: the idea

**What it is:** A queue is a to-do list for slow work. Instead of making the cashier wait while an SMS sends or a PDF report builds, the app puts the task on the queue and responds immediately. A separate background process, the **worker**, picks tasks off the queue and does them.

**Setup:**

```ini
# .env
QUEUE_CONNECTION=database   # or redis (faster, for production)
```

```bash
php artisan make:queue-table      # only if the jobs table does not exist yet
php artisan migrate
php artisan queue:work            # start a worker (keep it running)
```

| Driver | Use when |
| --- | --- |
| `sync` | Runs jobs immediately, no queue. Local debugging only |
| `database` | Simple, no extra software. Fine for small apps |
| `redis` | Fast and reliable. Production standard; pair with Laravel Horizon for a dashboard |
| `sqs` | AWS-managed queue |

## 12.3 Jobs: how to create one

**What it is:** A job is one task for the queue, written as a class with a `handle()` method.

**PharmaCare example:** generate the monthly sales report PDF and email it to the admin.

```bash
php artisan make:job GenerateMonthlyReport
```

```php
// app/Jobs/GenerateMonthlyReport.php
namespace App\Jobs;

use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Queue\Queueable;

class GenerateMonthlyReport implements ShouldQueue
{
    use Queueable;

    public int $tries = 3;          // retry up to 3 times if it fails
    public int $timeout = 120;      // kill it after 120 seconds
    public array $backoff = [60, 300];  // wait 1 min, then 5 min between retries

    public function __construct(public int $year, public int $month) {}

    public function handle(): void
    {
        $sales = Sale::whereYear('created_at', $this->year)
            ->whereMonth('created_at', $this->month)
            ->with('items.medicine')
            ->get();

        $pdf = Pdf::loadView('reports.monthly', compact('sales'));
        $path = "reports/{$this->year}-{$this->month}.pdf";
        Storage::put($path, $pdf->output());

        Mail::to(config('pharmacy.admin_email'))->send(new MonthlyReportMail($path));
    }

    // Runs once all retries are used up
    public function failed(\Throwable $e): void
    {
        Log::error('Monthly report failed', ['error' => $e->getMessage()]);
    }
}
```

**What it does:** collects the month's sales, builds a PDF, saves it to storage and emails the admin, in the background, retrying up to 3 times.

**Dispatch (send) the job:**

```php
GenerateMonthlyReport::dispatch(2026, 9);                         // queue it now
GenerateMonthlyReport::dispatch(2026, 9)->delay(now()->addMinutes(10));
GenerateMonthlyReport::dispatch(2026, 9)->onQueue('reports');     // a named queue
GenerateMonthlyReport::dispatchSync(2026, 9);                     // run right now, no queue
```

**Unique and chained jobs:**

```php
// Only one low-stock alert per medicine at a time
class LowStockAlert implements ShouldQueue, ShouldBeUnique
{
    use Queueable;
    public function __construct(public Medicine $medicine) {}
    public function uniqueId(): string { return (string) $this->medicine->id; }
    public function handle(): void { /* notify pharmacists */ }
}

// Run jobs in order: the next starts only if the previous succeeds
Bus::chain([
    new ImportSupplierPriceList($file),
    new RecalculatePrices(),
    new NotifyAdmin('Prices updated'),
])->dispatch();
```

**Queue commands:**

```bash
php artisan queue:work --queue=high,default,reports   # process queues in priority order
php artisan queue:work --tries=3 --timeout=90
php artisan queue:listen        # like work but reloads code each job (development)
php artisan queue:failed        # list failed jobs
php artisan queue:retry all     # retry failed jobs
php artisan queue:flush         # delete all failed jobs
php artisan queue:restart       # restart workers after deploying new code (IMPORTANT)
```

Rule: workers keep your old code in memory. After every deployment, run `php artisan queue:restart`. In production, keep workers alive with Supervisor (or let Forge / Laravel Cloud manage them).

## 12.4 Task scheduling

**What it is:** The scheduler runs tasks automatically at set times ("every night at 1 a.m."), defined in PHP instead of many cron lines. The server needs only **one** cron entry.

**How to do it:** define tasks in `routes/console.php`.

```php
// routes/console.php
use Illuminate\Support\Facades\Schedule;

// Run our custom command (section 1.7) every morning
Schedule::command('pharmacy:expiring --days=90')->dailyAt('07:00');

// Queue the monthly report on the 1st of each month for the previous month
Schedule::job(new GenerateMonthlyReport(now()->subMonth()->year, now()->subMonth()->month))
    ->monthlyOn(1, '02:00');

// A quick closure: mark expired batches every night
Schedule::call(function () {
    Batch::expired()->where('quantity', '>', 0)->update(['is_quarantined' => true]);
})->daily()->name('quarantine-expired')->withoutOverlapping();

// Clean up old failed jobs weekly
Schedule::command('queue:prune-failed --hours=168')->weekly();
```

**Common frequencies:** `everyMinute()`, `everyFiveMinutes()`, `hourly()`, `daily()`, `dailyAt('13:00')`, `twiceDaily(1, 13)`, `weekly()`, `weeklyOn(1, '8:00')` (Monday), `monthly()`, `cron('0 */2 * * *')`. Add `->weekdays()`, `->timezone('Africa/Dar_es_Salaam')`, `->onOneServer()`.

**The one cron line on the server:**

```bash
* * * * * cd /var/www/pharmacare && php artisan schedule:run >> /dev/null 2>&1
```

```bash
php artisan schedule:list      # see all tasks and when they next run
php artisan schedule:work      # run the scheduler locally (development)
php artisan schedule:test      # pick a task and run it now
```

## 12.5 Notifications

**What it is:** A notification is a short message to a user, sent through one or more **channels**: email, database (in-app bell icon), SMS, Slack and others. One class, many channels.

```bash
php artisan make:notification LowStockNotification
php artisan make:notifications-table     # for the database channel
php artisan migrate
```

```php
// app/Notifications/LowStockNotification.php
class LowStockNotification extends Notification implements ShouldQueue
{
    use Queueable;

    public function __construct(public Medicine $medicine, public int $stock) {}

    // Which channels to use
    public function via(object $notifiable): array
    {
        return ['mail', 'database'];
    }

    public function toMail(object $notifiable): MailMessage
    {
        return (new MailMessage)
            ->subject("Low stock: {$this->medicine->name}")
            ->greeting("Hello {$notifiable->name},")
            ->line("{$this->medicine->name} has only {$this->stock} units left.")
            ->action('Receive stock', route('batches.create', ['medicine' => $this->medicine->id]))
            ->line('Please reorder soon.');
    }

    // Stored as JSON in the notifications table
    public function toArray(object $notifiable): array
    {
        return ['medicine_id' => $this->medicine->id, 'name' => $this->medicine->name, 'stock' => $this->stock];
    }
}
```

**Send it:**

```php
// To one user (User model uses the Notifiable trait by default)
$pharmacist->notify(new LowStockNotification($medicine, 8));

// To many users
Notification::send(User::where('role', 'pharmacist')->get(), new LowStockNotification($medicine, 8));

// To someone who is not a user (e.g. the supplier's email)
Notification::route('mail', $medicine->supplier->email)->notify(new LowStockNotification($medicine, 8));
```

**Show in-app notifications:**

```html+php
<span class="bell">{{ auth()->user()->unreadNotifications->count() }}</span>
@foreach (auth()->user()->unreadNotifications as $n)
    <p>{{ $n->data['name'] }} — {{ $n->data['stock'] }} left</p>
@endforeach
```

```php
auth()->user()->unreadNotifications->markAsRead();
```

**Mail vs notification:** use a **Mailable** (`php artisan make:mail`) for a designed email like a receipt or report. Use a **Notification** for short alerts that may go to several channels.
