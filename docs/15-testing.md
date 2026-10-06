# 15. Testing

**What it is:** Automated tests are code that checks your code. You run one command and, within seconds, know whether selling, stock deduction and permissions still work after your latest change.

**Why it matters:** in a pharmacy system, a bug that sells expired stock or miscalculates a total costs real money and real trust. Tests catch it before customers do.

## 15.1 Unit vs feature tests

| Type | Tests | Speed | PharmaCare example |
| --- | --- | --- | --- |
| **Unit** | One small piece in isolation (a class, a method), usually no database | Very fast | "Does `calculateTotal()` add 18% VAT correctly?" |
| **Feature** | A whole flow through HTTP, routes, database | Slower | "Can a cashier POST a sale, and does stock go down?" |

Most of your value comes from feature tests. Write unit tests for tricky calculations.

## 15.2 PHPUnit vs Pest

Both run the same tests underneath; Pest is a shorter syntax built on top of PHPUnit. New Laravel projects can choose either at install time.

```php
// PHPUnit style — tests/Feature/MedicineTest.php
class MedicineTest extends TestCase
{
    use RefreshDatabase;

    public function test_staff_can_see_medicine_list(): void
    {
        $user = User::factory()->create();
        $this->actingAs($user)->get('/medicines')->assertOk();
    }
}
```

```php
// Pest style — same test
uses(RefreshDatabase::class);

it('lets staff see the medicine list', function () {
    $user = User::factory()->create();
    $this->actingAs($user)->get('/medicines')->assertOk();
});
```

## 15.3 Setup and commands

```bash
php artisan make:test SaleTest                 # feature test (Pest or PHPUnit, per your project)
php artisan make:test VatCalculatorTest --unit # unit test
php artisan make:test SaleTest --pest          # force Pest

php artisan test                               # run everything
php artisan test --filter=SaleTest             # one file or test name
php artisan test --parallel                    # faster on many CPU cores
php artisan test --coverage                    # % of code covered (needs Xdebug or PCOV)
```

**Use a separate test database** so tests never touch real data. In `phpunit.xml`:

```xml
<env name="DB_CONNECTION" value="sqlite"/>
<env name="DB_DATABASE" value=":memory:"/>
```

`RefreshDatabase` rebuilds the tables and wraps each test in a transaction, so every test starts clean.

## 15.4 Feature test: making a sale (Pest)

```php
// tests/Feature/SaleTest.php
use App\Models\{User, Medicine, Batch, Sale};

uses(\Illuminate\Foundation\Testing\RefreshDatabase::class);

beforeEach(function () {
    $this->cashier  = User::factory()->create(['role' => 'cashier']);
    $this->medicine = Medicine::factory()->create(['price' => 500]);
    $this->batch    = Batch::factory()->for($this->medicine)->create(['quantity' => 10, 'expires_at' => now()->addYear()]);
});

it('records a sale and deducts stock', function () {
    $response = $this->actingAs($this->cashier)->post('/sales', [
        'payment_method' => 'cash',
        'items' => [['medicine_id' => $this->medicine->id, 'qty' => 3]],
    ]);

    $response->assertRedirect();
    $response->assertSessionHasNoErrors();

    $this->assertDatabaseHas('sales', ['user_id' => $this->cashier->id, 'total' => 1500]);
    expect($this->batch->fresh()->quantity)->toBe(7);
});

it('refuses to sell more than is in stock', function () {
    $this->actingAs($this->cashier)->post('/sales', [
        'payment_method' => 'cash',
        'items' => [['medicine_id' => $this->medicine->id, 'qty' => 50]],
    ])->assertSessionHasErrors('stock');

    expect(Sale::count())->toBe(0);
    expect($this->batch->fresh()->quantity)->toBe(10);   // nothing changed
});

it('blocks a cashier from deleting a medicine', function () {
    $this->actingAs($this->cashier)
        ->delete("/medicines/{$this->medicine->id}")
        ->assertForbidden();
});

it('requires login', function () {
    $this->get('/medicines')->assertRedirect('/login');
});
```

## 15.5 Unit test: a calculation (Pest)

```php
// app/Support/VatCalculator.php
class VatCalculator
{
    public static function add(int $amount, int $percent = 18): int
    {
        return (int) round($amount * (100 + $percent) / 100);
    }
}

// tests/Unit/VatCalculatorTest.php
it('adds 18% VAT', function () {
    expect(VatCalculator::add(1000))->toBe(1180);
});

it('handles zero', function () {
    expect(VatCalculator::add(0))->toBe(0);
});
```

## 15.6 Testing APIs, jobs, mail and notifications

```php
// API with Sanctum
it('lists medicines as JSON', function () {
    Sanctum::actingAs(User::factory()->create(), ['medicines:read']);
    Medicine::factory()->count(3)->create();

    $this->getJson('/api/medicines')
        ->assertOk()
        ->assertJsonCount(3, 'data')
        ->assertJsonStructure(['data' => [['id', 'name', 'price']]]);
});

// Fakes: check something was sent WITHOUT actually sending it
it('alerts pharmacists when stock runs low', function () {
    Notification::fake();
    $pharmacist = User::factory()->create(['role' => 'pharmacist']);

    // ... sell until stock is below reorder level ...

    Notification::assertSentTo($pharmacist, LowStockNotification::class);
});

it('queues the monthly report', function () {
    Queue::fake();
    // trigger the code that should queue the job
    GenerateMonthlyReport::dispatch(2026, 9);
    Queue::assertPushed(GenerateMonthlyReport::class);
});

// Other fakes: Mail::fake(), Event::fake(), Storage::fake('local'), Http::fake([...])

// Fake the M-Pesa API
Http::fake(['api.safaricom.co.ke/*' => Http::response(['ResponseCode' => '0'], 200)]);

// Test an Artisan command
$this->artisan('pharmacy:expiring --days=30')
    ->expectsOutputToContain('expire within 30 days')
    ->assertExitCode(0);
```

**Common assertions:** `assertOk()` (200), `assertCreated()` (201), `assertRedirect()`, `assertForbidden()` (403), `assertNotFound()` (404), `assertUnprocessable()` (422), `assertSee('text')`, `assertJson([...])`, `assertSessionHasErrors('field')`, `assertDatabaseHas()`, `assertDatabaseMissing()`, `assertSoftDeleted()`.
