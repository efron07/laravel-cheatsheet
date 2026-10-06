# 10. Authorization: gates and policies

**What it is:** Authorization decides what a logged-in user is allowed to do. A cashier may sell, but may not change prices or delete medicines.

| Tool | Use it for | PharmaCare example |
| --- | --- | --- |
| **Gate** | A general permission not tied to one model | "Can view the profit report?" |
| **Policy** | All permissions for one model, in one class | "Can update this medicine? Can void this sale?" |

## 10.1 Gates

**How to do it:** define gates in `AppServiceProvider::boot()`.

```php
use Illuminate\Support\Facades\Gate;

public function boot(): void
{
    Gate::define('view-reports', fn (User $user) => in_array($user->role, ['admin', 'pharmacist']));
    Gate::define('manage-users', fn (User $user) => $user->role === 'admin');

    // Runs before every check: admins can do everything
    Gate::before(fn (User $user) => $user->role === 'admin' ? true : null);
}
```

**Use a gate:**

```php
// In a controller
Gate::authorize('view-reports');          // throws 403 if not allowed

if (Gate::allows('manage-users')) { ... }
if (Gate::denies('view-reports')) { ... }
$request->user()->can('view-reports');

// On a route
Route::get('/reports/profit', ProfitReportController::class)->middleware('can:view-reports');
```

```html+php
@can('view-reports')
    <a href="{{ route('reports.daily') }}">Reports</a>
@endcan
```

## 10.2 Policies

**What it is:** A class that holds every rule for one model. Each method name matches an action.

```bash
php artisan make:policy MedicinePolicy --model=Medicine
php artisan make:policy SalePolicy --model=Sale
```

Laravel finds policies automatically when you follow the naming: `App\Models\Medicine` → `App\Policies\MedicinePolicy`.

```php
// app/Policies/MedicinePolicy.php
class MedicinePolicy
{
    public function viewAny(User $user): bool { return true; }                 // all staff see the list
    public function view(User $user, Medicine $medicine): bool { return true; }

    public function create(User $user): bool
    {
        return in_array($user->role, ['admin', 'pharmacist']);
    }

    public function update(User $user, Medicine $medicine): bool
    {
        return in_array($user->role, ['admin', 'pharmacist']);
    }

    public function delete(User $user, Medicine $medicine): bool
    {
        // Only admins, and never a medicine that has sales history
        return $user->role === 'admin' && ! $medicine->saleItems()->exists();
    }
}
```

```php
// app/Policies/SalePolicy.php
use Illuminate\Auth\Access\Response;

public function void(User $user, Sale $sale): Response
{
    if ($user->role === 'cashier') {
        return Response::deny('Only a pharmacist can void a sale.');
    }

    return $sale->created_at->isToday()
        ? Response::allow()
        : Response::deny('Sales can only be voided on the same day.');
}
```

**Use a policy:**

```php
// Controller
public function update(Request $request, Medicine $medicine)
{
    Gate::authorize('update', $medicine);      // 403 if not allowed
    ...
}

public function store(Request $request)
{
    Gate::authorize('create', Medicine::class);  // no instance yet → pass the class name
    ...
}

// Route middleware
Route::put('/medicines/{medicine}', [MedicineController::class, 'update'])->middleware('can:update,medicine');
Route::post('/sales/{sale}/void', VoidSaleController::class)->middleware('can:void,sale');

// Anywhere
$user->can('delete', $medicine);
$user->cannot('void', $sale);
```

```html+php
@can('update', $medicine)
    <a href="{{ route('medicines.edit', $medicine) }}">Edit</a>
@endcan

@cannot('delete', $medicine)
    <small>This medicine has sales and cannot be deleted.</small>
@endcannot
```

Rule: hiding a button in Blade is not security. Always check in the controller, route or form request too. A user can send the request without clicking your button.

**Growing beyond simple roles:** when you need many permissions per role (for example, separate permissions for receiving stock, approving discounts and voiding sales), use the `spatie/laravel-permission` package. It stores roles and permissions in the database and still works with `@can` and `can:` middleware.
