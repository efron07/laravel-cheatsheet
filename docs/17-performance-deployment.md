# 17. Performance, code quality and deployment

## 17.1 Optimization commands

**What it is:** In production, Laravel can pre-build its config, routes, views and events into cached files, so it does not re-read hundreds of files on every request.

```bash
php artisan optimize          # caches config, routes, views and events in one go
php artisan optimize:clear    # clears all of them

# Individually
php artisan config:cache   / config:clear
php artisan route:cache    / route:clear
php artisan view:cache     / view:clear
php artisan event:cache    / event:clear
```

Rules: run `optimize` on every deploy, after updating code. Do not run it on your laptop while developing, or your changes to config and routes will seem to be ignored. If something "does not change", run `php artisan optimize:clear` first.

**Quick performance checklist for PharmaCare:**

- Eager-load relationships (section 7.4) and turn on `preventLazyLoading` in development.
- Add database indexes on columns you filter or sort by (`name`, `barcode`, `expires_at`, `created_at`).
- Cache expensive dashboard numbers (section 13.1).
- Move slow work (PDFs, SMS, emails) to queues (section 12.3).
- Use Redis for cache, sessions and queues in production.
- Run `composer install --optimize-autoloader --no-dev` on the server.

## 17.2 Octane

**What it is:** Normally PHP starts Laravel from scratch for every request, then throws it away. Octane keeps the app loaded in memory using a high-performance server (FrankenPHP, Swoole or RoadRunner), so each request skips the start-up cost. Apps can serve several times more requests per second.

```bash
composer require laravel/octane
php artisan octane:install          # choose FrankenPHP, Swoole or RoadRunner
php artisan octane:start --watch    # development
php artisan octane:reload           # after deploying new code
```

**The catch:** because the app stays in memory, data can leak between requests. Avoid storing request-specific data in static properties or singletons. Only adopt Octane after the basics in 17.1 are done and you have measured a real need.

## 17.3 Pulse

**What it is:** A live health dashboard for production at `/pulse`: slow requests, slow queries, slow jobs, exceptions, cache hit rates, busiest users, and server CPU, memory and disk.

```bash
composer require laravel/pulse
php artisan vendor:publish --provider="Laravel\Pulse\PulseServiceProvider"
php artisan migrate
```

```php
// Who may view /pulse — in AppServiceProvider::boot()
Gate::define('viewPulse', fn (User $user) => $user->role === 'admin');
```

**Telescope vs Pulse:** Telescope records every detail for debugging (development). Pulse shows summary trends cheaply enough to run in production.

## 17.4 Health route

**What it is:** A URL that answers "is the app alive?" so uptime monitors and load balancers can check it. New Laravel apps already have one at `/up`, set in `bootstrap/app.php`:

```php
->withRouting(
    web: __DIR__.'/../routes/web.php',
    commands: __DIR__.'/../routes/console.php',
    health: '/up',
)
```

`/up` returns 200 if the app boots, 500 if it does not. To also check the database or queue, listen for its event:

```php
// AppServiceProvider::boot()
use Illuminate\Foundation\Events\DiagnosingHealth;

Event::listen(function (DiagnosingHealth $event) {
    DB::select('select 1');   // throws → /up returns 500 if the database is down
});
```

Point a free uptime monitor at `https://pharmacare.co.tz/up` to get alerted when the site goes down.

## 17.5 Pint (code style)

**What it is:** Pint automatically reformats your PHP code to one consistent style (spacing, imports, quotes). The whole team's code looks the same and reviews focus on logic. It is installed in every new Laravel project.

```bash
./vendor/bin/pint              # fix every file
./vendor/bin/pint --test       # only report problems (use in CI)
./vendor/bin/pint --dirty      # only files changed in Git
./vendor/bin/pint app/Models   # one folder
```

Optional `pint.json` in the project root to pick a preset: `{ "preset": "laravel" }`.

## 17.6 Deployment

**What it is:** Putting your app on a server so real users can reach it.

| Option | What it does | You manage |
| --- | --- | --- |
| **Laravel Forge** | Sets up and manages servers you rent (DigitalOcean, AWS, Hetzner, others): Nginx, PHP, MySQL, SSL, queue workers, scheduler, push-to-deploy | The server bill; Forge does the setup |
| **Laravel Cloud** | Fully managed platform: push code and it runs, with autoscaling, databases and queues built in | Almost nothing |
| **Your own VPS** | You install and configure everything by hand | Everything |

**What every deployment must do (the order matters):**

```bash
# 1. Get the new code
git pull origin main

# 2. Install PHP packages (production only, optimized)
composer install --no-dev --optimize-autoloader --no-interaction

# 3. Build CSS/JS
npm ci && npm run build

# 4. Update the database
php artisan migrate --force

# 5. Rebuild caches
php artisan optimize

# 6. Restart queue workers so they load the new code
php artisan queue:restart
```

**Production `.env` checklist:**

- `APP_ENV=production` and `APP_DEBUG=false`
- `APP_KEY` set (and backed up)
- `APP_URL=https://...` with HTTPS
- Real database credentials, `QUEUE_CONNECTION=redis` (or `database`), `CACHE_STORE=redis`
- `LOG_CHANNEL=daily` (or a stack with Slack for critical errors)
- Mail settings for real email delivery

**Server checklist:** web server points to `/public` only; `storage/` and `bootstrap/cache/` are writable by the web user; queue workers run under Supervisor; the scheduler cron line is installed (section 12.4); daily database backups exist and have been test-restored.

**Maintenance mode** while doing risky upgrades:

```bash
php artisan down --secret="pharmacare-admin"   # site shows 503; you can still enter via /pharmacare-admin
php artisan up
```
