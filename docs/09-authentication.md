# 9. Authentication

**What it is:** Authentication answers "**who are you?**" — login, logout, registration, password reset. Authorization (section 10) answers "**what may you do?**".

**Key terms:**

- **Guard** — how a user is identified on each request. `web` uses a session cookie (browsers); `sanctum` uses a token (APIs, mobile apps).
- **Provider** — where users are loaded from (usually the `users` table through the `User` model).
- Settings live in `config/auth.php`.

## 9.1 Starter kits (fastest way)

**What it is:** A ready-made package that adds login, registration, password reset, email verification and a profile page, with views styled in Tailwind.

| Starter kit | What you get | Note |
| --- | --- | --- |
| **Laravel 12+ starter kits** | Choose React, Vue or Livewire when running `laravel new` | Current default for new projects |
| **Breeze** | Simple, minimal auth with Blade, Livewire, React or Vue | Older kit; very common in existing projects |
| **Jetstream** | Breeze plus two-factor auth, API tokens, teams, session management | Older and heavier; for apps that need teams |

**How to do it:**

```bash
# New project with a starter kit (the installer asks which one)
laravel new pharmacare

# Existing project with Breeze (older projects)
composer require laravel/breeze --dev
php artisan breeze:install blade
php artisan migrate
npm install && npm run dev
```

After installing, you have `/login`, `/register`, `/forgot-password` and `/dashboard` working. For PharmaCare, remove public registration: only an admin should create staff accounts.

## 9.2 Using the logged-in user

```php
use Illuminate\Support\Facades\Auth;

Auth::user();            // the User model, or null
Auth::id();              // the user's id
Auth::check();           // logged in?
auth()->user()->role;    // helper version
$request->user();        // inside controllers

// Record which cashier made the sale
Sale::create([... , 'user_id' => auth()->id()]);
```

```html+php
@auth  Welcome, {{ auth()->user()->name }} @endauth
```

## 9.3 Manual authentication (build it yourself)

**Why learn it:** Starter kits hide this. Knowing it lets you customise login, for example blocking deactivated staff.

```php
// app/Http/Controllers/Auth/LoginController.php
use Illuminate\Support\Facades\Auth;

public function store(Request $request)
{
    $credentials = $request->validate([
        'email'    => ['required', 'email'],
        'password' => ['required'],
    ]);

    // Extra condition: only active staff may log in
    if (Auth::attempt([...$credentials, 'is_active' => true], $request->boolean('remember'))) {
        $request->session()->regenerate();             // prevents session fixation attacks
        return redirect()->intended(route('dashboard')); // go where they were heading
    }

    return back()
        ->withErrors(['email' => 'These credentials do not match our records.'])
        ->onlyInput('email');
}

public function destroy(Request $request)
{
    Auth::logout();
    $request->session()->invalidate();
    $request->session()->regenerateToken();
    return redirect('/login');
}
```

```php
// routes/web.php
Route::middleware('guest')->group(function () {
    Route::get('/login', [LoginController::class, 'create'])->name('login');
    Route::post('/login', [LoginController::class, 'store'])->middleware('throttle:5,1');
});
Route::post('/logout', [LoginController::class, 'destroy'])->middleware('auth')->name('logout');
```

**Passwords** are never stored as plain text. Laravel hashes them (see section 13.4):

```php
User::create(['name' => 'Asha', 'email' => 'asha@pharmacare.test', 'password' => Hash::make('secret123'), 'role' => 'cashier']);
// With the 'password' => 'hashed' cast on User, you can pass the plain password and it is hashed for you.
```

## 9.4 Sanctum (API tokens and SPA auth)

**What it is:** A lightweight package for authenticating APIs. Use it for a mobile app, a POS tablet, or a separate React/Next.js frontend.

**Two modes:**

- **API tokens** — the app logs in once, gets a token, and sends it with every request in the header `Authorization: Bearer <token>`.
- **SPA cookies** — a frontend on the same top-level domain uses normal session cookies. No tokens to store.

**How to do it:**

```bash
php artisan install:api        # installs Sanctum, creates routes/api.php and the tokens table
```

```php
// app/Models/User.php
use Laravel\Sanctum\HasApiTokens;

class User extends Authenticatable
{
    use HasApiTokens, HasFactory, Notifiable;
}
```

```php
// routes/api.php — issue a token for the POS tablet
Route::post('/token', function (Request $request) {
    $request->validate(['email' => 'required|email', 'password' => 'required', 'device' => 'required']);

    $user = User::where('email', $request->email)->first();

    if (! $user || ! Hash::check($request->password, $user->password)) {
        return response()->json(['message' => 'Invalid credentials'], 401);
    }

    // Abilities limit what this token may do
    $token = $user->createToken($request->device, ['sales:create', 'medicines:read']);
    return ['token' => $token->plainTextToken];
});

// Protected API routes
Route::middleware('auth:sanctum')->group(function () {
    Route::get('/me', fn (Request $r) => $r->user());
    Route::apiResource('medicines', Api\MedicineController::class)->only(['index', 'show']);
    Route::post('/sales', [Api\SaleController::class, 'store']);
});
```

```php
// Check a token's ability inside a controller
if (! $request->user()->tokenCan('sales:create')) abort(403);

// Log out this device / all devices
$request->user()->currentAccessToken()->delete();
$request->user()->tokens()->delete();
```

```bash
# Calling the API
curl -H "Authorization: Bearer 1|abc123..." -H "Accept: application/json" https://pharmacare.test/api/medicines
```

## 9.5 Passport (full OAuth2 server)

**What it is:** Passport turns your app into a full OAuth2 server, the "Login with Google" style flow where other companies' apps get access to your users' data.

**When to use it:** only if third-party apps (another clinic's system, an insurer) must access PharmaCare on behalf of your users. For your own mobile app or frontend, Sanctum is simpler and enough.

```bash
composer require laravel/passport
php artisan passport:install
```
