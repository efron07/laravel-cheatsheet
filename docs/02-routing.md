# 2. Routing

**What it is:** A route connects a URL and an HTTP method (GET, POST, PUT, DELETE) to the code that should run. It is the reception desk of your app: it reads the request and sends it to the right person.

**Where routes live:**

| File | Used for | Gets automatically |
| --- | --- | --- |
| `routes/web.php` | Pages for browsers | Sessions, cookies, CSRF protection |
| `routes/api.php` | JSON API (mobile app, M-Pesa callbacks) | `/api` prefix, stateless. Create it with `php artisan install:api` |
| `routes/console.php` | Closure commands and the scheduler | — |

**HTTP methods in plain words:** GET = read, POST = create, PUT/PATCH = update, DELETE = remove.

## 2.1 Basic routes

```php
// routes/web.php
use Illuminate\Support\Facades\Route;

Route::get('/', function () {
    return 'Welcome to PharmaCare';
});

Route::post('/sales', function () { /* save a sale */ });
Route::put('/medicines/{id}', function ($id) { /* update */ });
Route::delete('/medicines/{id}', function ($id) { /* delete */ });
Route::match(['get', 'post'], '/search', fn () => 'search');
Route::any('/ping', fn () => 'pong');
```

## 2.2 View and redirect routes

Shortcuts for routes that only show a page or only redirect.

```php
Route::view('/about', 'pages.about');                  // show resources/views/pages/about.blade.php
Route::view('/contact', 'pages.contact', ['phone' => '0754 000 000']);

Route::redirect('/home', '/dashboard');                 // 302 temporary
Route::permanentRedirect('/old-stock', '/batches');     // 301 permanent
```

## 2.3 Route parameters

**What it is:** Parts of the URL that change, written in curly braces. Laravel passes them to your function.

```php
// Required parameter: /medicines/15
Route::get('/medicines/{id}', fn (string $id) => "Medicine #{$id}");

// Optional parameter (note the ? and the default value)
Route::get('/reports/{year?}', fn (?string $year = null) => $year ?? now()->year);

// Restrict what a parameter may contain
Route::get('/sales/{id}', fn ($id) => $id)->whereNumber('id');
Route::get('/batches/{code}', fn ($code) => $code)->where('code', '[A-Z]{3}-[0-9]{4}-[0-9]{2}');
```

## 2.4 Named routes

**What it is:** A nickname for a route. You link to the name, not the URL, so if the URL changes later, no links break.

```php
Route::get('/medicines/{medicine}', [MedicineController::class, 'show'])
    ->name('medicines.show');

// Generate the URL anywhere
$url = route('medicines.show', ['medicine' => 15]);  // http://.../medicines/15
return redirect()->route('medicines.show', 15);
```

```html+php
<a href="{{ route('medicines.show', $medicine) }}">View</a>
```

## 2.5 Route groups

**What it is:** Apply the same settings (middleware, URL prefix, name prefix, controller) to many routes at once.

```php
// Everything under /admin, named admin.*, only for logged-in admins
Route::middleware(['auth', 'role:admin'])
    ->prefix('admin')
    ->name('admin.')
    ->group(function () {
        Route::get('/users', [UserController::class, 'index'])->name('users.index');       // admin.users.index
        Route::get('/suppliers', [SupplierController::class, 'index'])->name('suppliers.index');
    });

// Share one controller across routes
Route::controller(ReportController::class)->prefix('reports')->group(function () {
    Route::get('/daily', 'daily');
    Route::get('/monthly', 'monthly');
});
```

## 2.6 Route model binding

**What it is:** Instead of receiving an ID and looking up the record yourself, type-hint the model and Laravel fetches it. If the record does not exist, Laravel returns a 404 page automatically.

```php
// Without binding (manual)
Route::get('/medicines/{id}', function ($id) {
    $medicine = Medicine::findOrFail($id);
    return view('medicines.show', compact('medicine'));
});

// With binding (the parameter name must match the variable name)
Route::get('/medicines/{medicine}', function (Medicine $medicine) {
    return view('medicines.show', compact('medicine'));
});

// Look up by a different column, e.g. a URL slug or batch code
Route::get('/batches/{batch:batch_no}', fn (Batch $batch) => $batch);
```

## 2.7 Resource routes

**What it is:** One line that creates all seven standard CRUD routes for a resource.

```php
Route::resource('medicines', MedicineController::class);
Route::apiResource('api/medicines', Api\MedicineController::class); // no create/edit pages
Route::resource('medicines', MedicineController::class)->only(['index', 'show']);
```

| Method | URL | Controller method | Route name | Purpose |
| --- | --- | --- | --- | --- |
| GET | `/medicines` | `index` | `medicines.index` | List all |
| GET | `/medicines/create` | `create` | `medicines.create` | Show add form |
| POST | `/medicines` | `store` | `medicines.store` | Save new |
| GET | `/medicines/{medicine}` | `show` | `medicines.show` | Show one |
| GET | `/medicines/{medicine}/edit` | `edit` | `medicines.edit` | Show edit form |
| PUT/PATCH | `/medicines/{medicine}` | `update` | `medicines.update` | Save changes |
| DELETE | `/medicines/{medicine}` | `destroy` | `medicines.destroy` | Delete |

## 2.8 Route commands

```bash
php artisan route:list                     # every route, method, name, controller
php artisan route:list --name=medicines    # filter by name
php artisan route:list --path=api          # filter by URL
php artisan route:cache                    # speed up routes in production
php artisan route:clear                    # undo the cache
```
