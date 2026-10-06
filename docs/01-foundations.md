# 1. Foundations

## 1.1 Why web frameworks?

**What it is:** A web framework is a ready-made toolbox and structure for building websites and web apps. It already solves the jobs every app needs: reading URLs, talking to a database, logging users in, sending email, and showing pages.

**Why it matters:** Without a framework, you rewrite the same plumbing in every project and make the same security mistakes. A framework lets you spend your time on the pharmacy logic, not on the plumbing.

Think of it like building a pharmacy shop. You can make your own bricks, or you can buy bricks, shelves and a till, then focus on arranging the medicines.

## 1.2 What is Laravel?

**What it is:** Laravel is a free PHP web framework. It gives you routing, a database layer (Eloquent), templates (Blade), authentication, queues, email, testing and a command-line tool (Artisan), all working together.

**The MVC pattern Laravel follows:**

| Part | Job | PharmaCare example |
| --- | --- | --- |
| **Model** | Talks to the database table | `Medicine` reads and writes the `medicines` table |
| **View** | What the user sees (HTML) | `medicines/index.blade.php` shows the medicine list |
| **Controller** | The middleman: takes the request, asks the model, returns a view | `MedicineController@index` |

A request flows like this: browser → route → controller → model → database, then back: controller → view → browser.

## 1.3 Installing Laravel

**What you need:** PHP 8.2+, Composer (PHP's package manager), and a database (MySQL, PostgreSQL or SQLite).

There are three common ways to get a working setup:

| Option | What it is | Best for |
| --- | --- | --- |
| **Composer + local PHP** | Install PHP and Composer yourself | Linux servers, full control |
| **Laravel Herd** | One-click app that installs PHP, Composer and Laravel tools (Mac and Windows) | Fastest local setup |
| **Laravel Sail** | Runs Laravel inside Docker containers (PHP, MySQL, Redis) | Same environment on every machine |

**How to do it (Composer route):**

```bash
# 1. Install the Laravel installer once
composer global require laravel/installer

# 2. Check it works
laravel --version
```

**How to do it (Sail route):**

```bash
# Create a project with Sail, choosing MySQL
curl -s "https://laravel.build/pharmacare?with=mysql" | bash
cd pharmacare
./vendor/bin/sail up -d          # start containers in background
./vendor/bin/sail artisan migrate  # run artisan inside Docker
```

## 1.4 Create a new project

```bash
# Option A: Laravel installer (asks you questions: starter kit, database, testing)
laravel new pharmacare

# Option B: Composer directly
composer create-project laravel/laravel pharmacare

cd pharmacare
php artisan serve      # starts the app at http://127.0.0.1:8000
```

`php artisan serve` starts a small development server. Open the URL in your browser and you should see the Laravel welcome page.

## 1.5 Folder structure

**What it is:** Every Laravel project has the same folders, so any Laravel developer can find things in any project.

| Folder | What lives there | PharmaCare example |
| --- | --- | --- |
| `app/` | Your application code | Models, controllers, jobs |
| `app/Http/Controllers` | Controllers | `MedicineController.php` |
| `app/Models` | Eloquent models | `Medicine.php`, `Sale.php` |
| `bootstrap/` | Starts the framework; `app.php` registers middleware, routes and exceptions | Add the `role` middleware here |
| `config/` | Settings files, one per feature | `database.php`, `mail.php` |
| `database/` | Migrations, seeders, factories | `create_medicines_table.php` |
| `public/` | The only folder the web server exposes; holds `index.php`, CSS, JS, images | Logo, compiled CSS |
| `resources/` | Blade views, raw CSS/JS, language files | `views/medicines/index.blade.php` |
| `routes/` | URL definitions | `web.php`, `api.php`, `console.php` |
| `storage/` | Logs, cache, uploaded files | `logs/laravel.log`, prescription scans |
| `tests/` | Automated tests | `Feature/SaleTest.php` |
| `vendor/` | Packages installed by Composer. Never edit | Laravel itself |

## 1.6 Configuration: `.env` and `config/`

**What it is:** `.env` holds values that change per machine or are secret (database password, API keys). The files in `config/` read those values with `env()` and give them sensible defaults.

**Why it matters:** Your laptop, the test server and the live server use different databases. You change `.env`, not the code. Never commit `.env` to Git.

```ini
# .env
APP_NAME=PharmaCare
APP_ENV=local          # local | staging | production
APP_DEBUG=true         # MUST be false in production
APP_URL=http://localhost:8000

DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=pharmacare
DB_USERNAME=root
DB_PASSWORD=secret

PHARMACY_LOW_STOCK=20  # our own custom setting
```

**How to add your own setting:** create `config/pharmacy.php`, then read it anywhere with `config()`.

```php
// config/pharmacy.php
return [
    'low_stock_threshold' => env('PHARMACY_LOW_STOCK', 20),
    'expiry_warning_days' => 90,
];

// anywhere in the app
$limit = config('pharmacy.low_stock_threshold'); // 20
```

Rule: call `env()` only inside `config/` files. In the rest of the app, use `config()`. When the config is cached in production, `env()` outside config returns `null`.

## 1.7 Artisan (the command line)

**What it is:** Artisan is Laravel's built-in command-line tool. It generates files, runs migrations, clears caches and runs your own custom commands.

```bash
php artisan list                  # every available command
php artisan help make:model       # options for one command
php artisan about                 # app version, environment, drivers
php artisan tinker                # interactive PHP shell inside your app
```

**Tinker example** — try code against the real database without writing a page:

```php
> App\Models\Medicine::count();
= 42
> App\Models\Medicine::where('name', 'like', 'Amox%')->first();
```

**How to create your own Artisan command (PharmaCare: list expiring batches):**

```bash
php artisan make:command CheckExpiringBatches
```

```php
// app/Console/Commands/CheckExpiringBatches.php
namespace App\Console\Commands;

use App\Models\Batch;
use Illuminate\Console\Command;

class CheckExpiringBatches extends Command
{
    // How you call it: php artisan pharmacy:expiring --days=60
    protected $signature = 'pharmacy:expiring {--days=90 : Days ahead to check}';
    protected $description = 'List medicine batches that expire soon';

    public function handle(): int
    {
        $days = (int) $this->option('days');

        $batches = Batch::with('medicine')
            ->where('expires_at', '<=', now()->addDays($days))
            ->where('quantity', '>', 0)
            ->get();

        $this->table(
            ['Medicine', 'Batch', 'Qty', 'Expires'],
            $batches->map(fn ($b) => [$b->medicine->name, $b->batch_no, $b->quantity, $b->expires_at->toDateString()])
        );

        $this->info("{$batches->count()} batch(es) expire within {$days} days.");
        return self::SUCCESS;
    }
}
```

```bash
php artisan pharmacy:expiring --days=60
```

**What it does:** finds every batch with stock left that expires within the given days, prints a table, then a summary line. Laravel finds the command automatically; no registration needed.

## 1.8 Package management with Composer

**What it is:** Composer installs PHP libraries (packages) into `vendor/` and records them in `composer.json`. `composer.lock` pins the exact versions so every machine installs the same thing.

```bash
composer require barryvdh/laravel-dompdf       # add a package (PDF receipts)
composer require --dev laravel/pint            # add a dev-only package
composer install       # install exactly what composer.lock says (use on servers)
composer update        # upgrade packages within allowed versions (use with care)
composer remove vendor/package
composer dump-autoload # rebuild the class map after moving classes
```

Rule: commit `composer.json` and `composer.lock`; never commit `vendor/`. On a server, always run `composer install --no-dev --optimize-autoloader`.
