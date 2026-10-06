# 18. Quick reference: commands

Every command used in this sheet, grouped by job. The last column points back to the section with the full explanation.

## Generators (`make:`)

| Command | Creates | Section |
| --- | --- | --- |
| `php artisan make:model Medicine -mfsc` | Model + migration, factory, seeder, controller | 7.1 |
| `php artisan make:controller MedicineController --resource --model=Medicine` | CRUD controller | 3.3 |
| `php artisan make:controller PrintReceiptController --invokable` | Single-action controller | 3.4 |
| `php artisan make:migration create_medicines_table` | Migration | 6.2 |
| `php artisan make:factory MedicineFactory` | Factory | 6.3 |
| `php artisan make:seeder CategorySeeder` | Seeder | 6.4 |
| `php artisan make:request StoreBatchRequest` | Form request (validation) | 8.3 |
| `php artisan make:rule TanzanianPhone` | Custom validation rule | 8.5 |
| `php artisan make:middleware EnsureUserHasRole` | Middleware | 4.3 |
| `php artisan make:policy MedicinePolicy --model=Medicine` | Policy | 10.2 |
| `php artisan make:provider PaymentServiceProvider` | Service provider | 11.3 |
| `php artisan make:event SaleCompleted` | Event | 12.1 |
| `php artisan make:listener CheckLowStock --event=SaleCompleted` | Listener | 12.1 |
| `php artisan make:job GenerateMonthlyReport` | Queued job | 12.3 |
| `php artisan make:notification LowStockNotification` | Notification | 12.5 |
| `php artisan make:mail MonthlyReportMail` | Mailable | 12.5 |
| `php artisan make:command CheckExpiringBatches` | Custom Artisan command | 1.7 |
| `php artisan make:component StockBadge` | Blade component | 5.5 |
| `php artisan make:view medicines.index` | Blade view | 5 |
| `php artisan make:resource MedicineResource` | API resource | 7.8 |
| `php artisan make:exception InsufficientStockException` | Exception | 14.1 |
| `php artisan make:test SaleTest` | Feature test (`--unit`, `--pest`) | 15.3 |
| `php artisan make:livewire MedicineSearch` | Livewire component | 16.2 |

## Database

| Command | Does | Section |
| --- | --- | --- |
| `php artisan migrate` | Run new migrations (`--force` in production) | 6.2 |
| `php artisan migrate:status` | Show which migrations ran | 6.2 |
| `php artisan migrate:rollback` | Undo the last batch | 6.2 |
| `php artisan migrate:fresh --seed` | Drop everything, rebuild, seed (local only) | 6.2 |
| `php artisan db:seed` | Run seeders (`--class=...`) | 6.4 |
| `php artisan db:show` / `db:table medicines` | Inspect the database / a table | 6.1 |
| `php artisan model:show Medicine` | Inspect a model | 7.1 |
| `php artisan tinker` | Interactive shell inside the app | 1.7 |

## Routes, queues, scheduler

| Command | Does | Section |
| --- | --- | --- |
| `php artisan route:list` | List routes (`--name`, `--path`) | 2.8 |
| `php artisan queue:work` | Start a queue worker | 12.2 |
| `php artisan queue:restart` | Restart workers after deploy | 12.3 |
| `php artisan queue:failed` / `queue:retry all` | See / retry failed jobs | 12.3 |
| `php artisan schedule:list` | Show scheduled tasks | 12.4 |
| `php artisan schedule:work` | Run the scheduler locally | 12.4 |
| `php artisan event:list` | Show events and their listeners | 12.1 |

## Setup, caches, production

| Command | Does | Section |
| --- | --- | --- |
| `laravel new pharmacare` | New project | 1.4 |
| `php artisan serve` | Local development server | 1.4 |
| `php artisan key:generate` | Create `APP_KEY` | 13.5 |
| `php artisan install:api` | Add `routes/api.php` + Sanctum | 9.4 |
| `php artisan storage:link` | Make the public disk reachable | 13.2 |
| `php artisan lang:publish` | Create the `lang/` folder | 13.3 |
| `php artisan config:publish cors` | Publish a config file | 4.4 |
| `php artisan optimize` / `optimize:clear` | Cache / clear config, routes, views, events | 17.1 |
| `php artisan cache:clear` | Clear the application cache | 13.1 |
| `php artisan down --secret=...` / `up` | Maintenance mode on / off | 17.6 |
| `php artisan about` | App info and drivers | 1.7 |
| `php artisan pail` | Live log viewer | 14.3 |
| `php artisan test` | Run tests (`--filter`, `--parallel`) | 15.3 |
| `./vendor/bin/pint` | Fix code style | 17.5 |

## Composer and npm

| Command | Does | Section |
| --- | --- | --- |
| `composer require vendor/package` | Add a package (`--dev` for dev-only) | 1.8 |
| `composer install --no-dev --optimize-autoloader` | Install for production | 1.8 |
| `npm run dev` / `npm run build` | Vite: develop / build assets | 16.1 |
