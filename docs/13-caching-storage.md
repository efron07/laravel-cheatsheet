# 13. Caching, file storage, localization, encryption and hashing

## 13.1 Caching

**What it is:** A cache stores the result of slow work (a heavy query, an API call) so the next request reads it instantly instead of recalculating. Like writing today's sales total on a whiteboard instead of adding all receipts every time someone asks.

```ini
CACHE_STORE=database   # file | database | redis | memcached | array (tests)
```

```php
use Illuminate\Support\Facades\Cache;

// The pattern you will use 90% of the time: get it, or calculate and store it
$totals = Cache::remember('dashboard:today', now()->addMinutes(5), function () {
    return [
        'sales'   => Sale::whereDate('created_at', today())->sum('total'),
        'count'   => Sale::whereDate('created_at', today())->count(),
        'low'     => Medicine::lowStock()->count(),
    ];
});

Cache::rememberForever('categories:all', fn () => Category::orderBy('name')->get());

Cache::put('key', 'value', 600);      // seconds
Cache::get('key', 'default');
Cache::has('key');
Cache::increment('sms:sent:today');
Cache::forget('categories:all');      // remove one key
Cache::flush();                        // remove everything (careful)
```

**Clear stale cache when data changes:**

```php
// app/Models/Category.php
protected static function booted(): void
{
    static::saved(fn () => Cache::forget('categories:all'));
    static::deleted(fn () => Cache::forget('categories:all'));
}
```

**Atomic lock** — stop the same job running twice at once:

```php
Cache::lock('monthly-report', 120)->get(function () {
    // only one process can be here at a time
});
```

```bash
php artisan cache:clear
```

## 13.2 File storage

**What it is:** One API for saving and reading files, whether they live on the local disk, Amazon S3 or another cloud. You change the disk in config; the code stays the same.

**Disks** (in `config/filesystems.php`):

| Disk | Location | Visible on the web? |
| --- | --- | --- |
| `local` | `storage/app/private` | No — use for prescriptions and private documents |
| `public` | `storage/app/public` | Yes, after `php artisan storage:link` — use for medicine photos |
| `s3` | Amazon S3 / compatible | Configurable |

```bash
php artisan storage:link    # creates public/storage → storage/app/public
```

**Upload a prescription scan (private):**

```php
public function store(Request $request, Sale $sale)
{
    $request->validate(['scan' => ['required', 'file', 'mimes:pdf,jpg,jpeg,png', 'max:4096']]);

    // Saves with a random unique name, returns the path
    $path = $request->file('scan')->store('prescriptions', 'local');
    // e.g. "prescriptions/Xk3...9.pdf"

    $sale->prescription()->create([
        'file_path'   => $path,
        'doctor_name' => $request->doctor_name,
    ]);

    return back()->with('success', 'Prescription saved.');
}
```

**Medicine photo (public):**

```php
$path = $request->file('photo')->store('medicines', 'public');
$medicine->update(['photo' => $path]);
```

```html+php
<img src="{{ Storage::url($medicine->photo) }}" alt="{{ $medicine->name }}">
```

**Other file operations:**

```php
use Illuminate\Support\Facades\Storage;

Storage::disk('local')->put('reports/sep.csv', $csv);
Storage::get('reports/sep.csv');
Storage::exists('reports/sep.csv');
Storage::delete('reports/sep.csv');
Storage::files('prescriptions');
Storage::download('prescriptions/rx.pdf', 'prescription.pdf');   // response
Storage::temporaryUrl('prescriptions/rx.pdf', now()->addMinutes(5));   // S3: expiring link
```

## 13.3 Localization

**What it is:** Showing the app in more than one language. PharmaCare can show English and Swahili.

```bash
php artisan lang:publish      # creates the lang/ folder with Laravel's default messages
```

**Two ways to store translations:**

```php
// lang/sw/pharmacy.php — short keys
return [
    'out_of_stock' => 'Dawa imeisha',
    'low_stock'    => 'Dawa :name imebaki :count tu',
];
```

```js
// lang/sw.json — the English sentence itself is the key
{
    "Add medicine": "Ongeza dawa",
    "Total": "Jumla"
}
```

```html+php
{{ __('pharmacy.out_of_stock') }}
{{ __('pharmacy.low_stock', ['name' => $medicine->name, 'count' => 5]) }}
{{ __('Add medicine') }}
{{ trans_choice('{0} No items|{1} One item|[2,*] :count items', $count) }}
```

**Switch language:**

```php
// config/app.php reads APP_LOCALE=en and APP_FALLBACK_LOCALE=en from .env

App::setLocale('sw');                 // for this request
app()->getLocale();

// A middleware that uses the staff member's saved preference
public function handle(Request $request, Closure $next)
{
    App::setLocale($request->user()?->locale ?? 'en');
    return $next($request);
}
```

## 13.4 Hashing

**What it is:** Hashing turns a value into a fixed scrambled string that **cannot be reversed**. You can only check whether a new value produces the same hash. Use it for passwords and PINs.

```php
use Illuminate\Support\Facades\Hash;

$hash = Hash::make('cashier-pin-4821');
Hash::check('cashier-pin-4821', $hash);   // true
Hash::needsRehash($hash);                  // true if the algorithm settings changed
```

## 13.5 Encryption

**What it is:** Encryption scrambles a value so it **can be unscrambled** later, but only with your app's secret key (`APP_KEY` in `.env`). Use it for data you must read back, like a customer's national ID or an API secret stored in the database.

```php
use Illuminate\Support\Facades\Crypt;

$secret = Crypt::encryptString('19900101-12345-00001-21');
$plain  = Crypt::decryptString($secret);

// Easiest: let the model do it
protected function casts(): array
{
    return ['national_id' => 'encrypted'];   // stored encrypted, read as plain text
}
```

|  | Hashing | Encryption |
| --- | --- | --- |
| Reversible? | No | Yes, with `APP_KEY` |
| Use for | Passwords, PINs | National IDs, API keys, sensitive notes |
| Laravel tool | `Hash` | `Crypt`, `encrypted` cast |

Rule: generate the key once with `php artisan key:generate`. If you lose or change `APP_KEY`, every encrypted value becomes unreadable. Back it up.
