# 6. Database: migrations, seeders and the query builder

## 6.1 Database configuration

**What it is:** Laravel reads the database connection from `.env` (section 1.6) through `config/database.php`. It supports MySQL, MariaDB, PostgreSQL, SQLite and SQL Server.

```bash
php artisan db:show                 # connection details, tables, sizes
php artisan db:table medicines      # columns and indexes of one table
php artisan db                      # open the database CLI
```

## 6.2 Migrations

**What it is:** A migration is a PHP file that creates or changes a database table. It is version control for your database: every developer and every server runs the same files and ends up with the same tables.

**Why it matters:** No more "send me your SQL dump". You run one command and the database is built.

```bash
php artisan make:migration create_medicines_table
php artisan make:migration add_strength_to_medicines_table   # change an existing table
php artisan make:model Medicine -m                           # model + migration together
```

**Create a table (PharmaCare `medicines` and `batches`):**

```php
// database/migrations/2026_10_06_000001_create_medicines_table.php
public function up(): void
{
    Schema::create('medicines', function (Blueprint $table) {
        $table->id();                                              // BIGINT auto-increment primary key
        $table->foreignId('category_id')->constrained()->restrictOnDelete();
        $table->foreignId('supplier_id')->nullable()->constrained()->nullOnDelete();
        $table->string('name', 150);
        $table->string('generic_name')->nullable();
        $table->string('barcode')->unique()->nullable();
        $table->unsignedInteger('price');                          // TZS as whole numbers, never float
        $table->boolean('requires_prescription')->default(false);
        $table->unsignedInteger('reorder_level')->default(20);
        $table->timestamps();                                      // created_at, updated_at
        $table->softDeletes();                                     // deleted_at ("trash" instead of delete)

        $table->index('name');
    });
}

public function down(): void
{
    Schema::dropIfExists('medicines');     // how to undo this migration
}
```

```php
// create_batches_table
Schema::create('batches', function (Blueprint $table) {
    $table->id();
    $table->foreignId('medicine_id')->constrained()->cascadeOnDelete();
    $table->string('batch_no')->unique();
    $table->unsignedInteger('quantity');
    $table->unsignedInteger('cost_price');
    $table->date('expires_at');
    $table->timestamps();
});
```

**Change an existing table:**

```php
// add_strength_to_medicines_table
public function up(): void
{
    Schema::table('medicines', function (Blueprint $table) {
        $table->string('strength')->nullable()->after('name');   // e.g. "500mg"
        $table->renameColumn('generic_name', 'active_ingredient');
    });
}

public function down(): void
{
    Schema::table('medicines', function (Blueprint $table) {
        $table->dropColumn('strength');
        $table->renameColumn('active_ingredient', 'generic_name');
    });
}
```

**Common column types:** `string`, `text`, `integer`, `unsignedInteger`, `bigInteger`, `decimal('x', 12, 2)`, `boolean`, `date`, `dateTime`, `timestamp`, `json`, `enum('role', ['admin','pharmacist','cashier'])`, `uuid`, `foreignId`.

**Common modifiers:** `->nullable()`, `->default(0)`, `->unique()`, `->index()`, `->after('name')`, `->comment('...')`.

**Migration commands:**

```bash
php artisan migrate                  # run all new migrations
php artisan migrate:status           # which have run, which have not
php artisan migrate:rollback         # undo the last batch
php artisan migrate:rollback --step=1
php artisan migrate:fresh --seed     # DROP ALL tables, re-run, seed (local only!)
php artisan migrate --force          # required to run in production
```

Rule: never edit a migration that has already run on a shared or live database. Create a new migration instead.

## 6.3 Factories

**What it is:** A factory is a recipe for making fake records. You use it for test data and for filling a development database.

```bash
php artisan make:factory MedicineFactory --model=Medicine
```

```php
// database/factories/MedicineFactory.php
public function definition(): array
{
    return [
        'category_id'           => Category::factory(),
        'name'                  => fake()->unique()->randomElement(['Paracetamol', 'Amoxicillin', 'Ibuprofen', 'Metformin', 'Omeprazole']) . ' ' . fake()->randomElement(['250mg', '500mg']),
        'price'                 => fake()->numberBetween(200, 25000),
        'requires_prescription' => fake()->boolean(30),
        'reorder_level'         => 20,
    ];
}

// A "state": a named variation
public function prescriptionOnly(): static
{
    return $this->state(fn () => ['requires_prescription' => true]);
}
```

```php
Medicine::factory()->count(50)->create();
Medicine::factory()->prescriptionOnly()->create();
Medicine::factory()->has(Batch::factory()->count(3))->create();   // medicine with 3 batches
```

## 6.4 Seeders

**What it is:** A seeder fills the database with starting data: real lookup data (categories, the first admin user) or fake data from factories.

```bash
php artisan make:seeder CategorySeeder
```

```php
// database/seeders/CategorySeeder.php
public function run(): void
{
    foreach (['Antibiotics', 'Painkillers', 'Antimalarials', 'Vitamins', 'Antidiabetics'] as $name) {
        Category::firstOrCreate(['name' => $name]);
    }
}

// database/seeders/DatabaseSeeder.php — the entry point
public function run(): void
{
    $this->call([CategorySeeder::class]);

    User::factory()->create(['name' => 'Admin', 'email' => 'admin@pharmacare.test', 'role' => 'admin']);
    Medicine::factory()->count(100)->has(Batch::factory()->count(2))->create();
}
```

```bash
php artisan db:seed                          # runs DatabaseSeeder
php artisan db:seed --class=CategorySeeder   # one seeder only
```

## 6.5 Query builder

**What it is:** A fluent way to write SQL in PHP using the `DB` facade. It protects you from SQL injection and works on every supported database. Use it for reports and heavy queries; use Eloquent (section 7) for normal app work.

```php
use Illuminate\Support\Facades\DB;

// SELECT
$medicines = DB::table('medicines')->get();                       // all rows
$medicine  = DB::table('medicines')->where('id', 5)->first();      // one row
$names     = DB::table('medicines')->pluck('name');                // one column
$count     = DB::table('medicines')->count();

// WHERE variations
DB::table('medicines')
    ->where('price', '>', 1000)
    ->where('requires_prescription', true)
    ->orWhere('name', 'like', '%amox%')
    ->whereIn('category_id', [1, 2])
    ->whereNull('deleted_at')
    ->whereBetween('price', [500, 5000])
    ->orderBy('name')
    ->limit(10)
    ->get();

// JOIN + GROUP BY: total stock per medicine
$stock = DB::table('medicines')
    ->join('batches', 'batches.medicine_id', '=', 'medicines.id')
    ->select('medicines.name', DB::raw('SUM(batches.quantity) as total_stock'))
    ->groupBy('medicines.id', 'medicines.name')
    ->having('total_stock', '<', 20)
    ->get();

// INSERT / UPDATE / DELETE
DB::table('categories')->insert(['name' => 'Antiseptics', 'created_at' => now(), 'updated_at' => now()]);
DB::table('batches')->where('id', 7)->decrement('quantity', 2);
DB::table('batches')->where('expires_at', '<', now())->update(['quantity' => 0]);
DB::table('medicines')->where('id', 9)->delete();
```

## 6.6 Transactions (critical for money and stock)

**What it is:** A transaction groups several database changes so they all succeed or all fail together. If a sale is saved but the stock deduction fails, the whole thing is undone.

```php
DB::transaction(function () use ($cart, $user) {
    $sale = Sale::create(['user_id' => $user->id, 'total' => $cart->total()]);

    foreach ($cart->items as $item) {
        // lockForUpdate stops two cashiers selling the last box at the same moment
        $batch = Batch::where('medicine_id', $item->medicine_id)
            ->where('quantity', '>=', $item->qty)
            ->orderBy('expires_at')            // FEFO: first expiry, first out
            ->lockForUpdate()
            ->firstOrFail();

        $batch->decrement('quantity', $item->qty);
        $sale->items()->create(['medicine_id' => $item->medicine_id, 'batch_id' => $batch->id, 'qty' => $item->qty, 'price' => $item->price]);
    }
});
```

Any exception inside the closure rolls everything back automatically.
