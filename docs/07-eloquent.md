# 7. Eloquent ORM

**What it is:** Eloquent is Laravel's ORM (Object-Relational Mapper). Each database table gets a PHP class called a model. One row becomes one object, so you write `$medicine->price` instead of SQL.

**Naming rules (follow them and Eloquent configures itself):** model `Medicine` (singular, PascalCase) → table `medicines` (plural, snake\_case) → primary key `id` → foreign key in other tables `medicine_id`.

## 7.1 Creating a model

```bash
php artisan make:model Medicine                 # model only
php artisan make:model Medicine -mfsc           # + migration, factory, seeder, controller
php artisan make:model Medicine --all           # everything incl. policy, form requests
php artisan model:show Medicine                 # inspect columns, relations, casts
```

```php
// app/Models/Medicine.php
namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\SoftDeletes;

class Medicine extends Model
{
    use HasFactory, SoftDeletes;

    // Columns that may be filled with create()/update() — protects against "mass assignment"
    protected $fillable = ['category_id', 'supplier_id', 'name', 'barcode', 'price', 'requires_prescription', 'reorder_level'];

    // Alternative: protect only these, allow the rest
    // protected $guarded = ['id'];
}
```

**Mass assignment** means filling many columns at once from user input. `$fillable` stops a user from sneaking extra fields (like `role=admin`) into a form.

## 7.2 CRUD operations

```php
// CREATE
$medicine = Medicine::create(['name' => 'Paracetamol 500mg', 'category_id' => 2, 'price' => 300]);

$medicine = new Medicine();
$medicine->name = 'Ibuprofen 400mg';
$medicine->category_id = 2;
$medicine->price = 800;
$medicine->save();

// READ
Medicine::all();
Medicine::find(5);                        // null if missing
Medicine::findOrFail(5);                  // 404 if missing
Medicine::where('price', '<', 1000)->orderBy('name')->get();
Medicine::where('barcode', $code)->first();
Medicine::firstWhere('name', 'Paracetamol 500mg');
Medicine::count();
Medicine::where('requires_prescription', true)->exists();

// UPDATE
$medicine->update(['price' => 350]);
Medicine::where('category_id', 2)->update(['reorder_level' => 30]);   // many rows
$medicine->increment('reorder_level', 5);

// UPDATE OR CREATE
Medicine::updateOrCreate(['barcode' => '6001234'], ['name' => 'Cetirizine 10mg', 'price' => 200, 'category_id' => 3]);
Category::firstOrCreate(['name' => 'Antifungals']);

// DELETE
$medicine->delete();                      // soft delete (sets deleted_at)
Medicine::destroy([1, 2, 3]);
Medicine::withTrashed()->find(5);         // include soft-deleted
$medicine->restore();                     // un-delete
$medicine->forceDelete();                 // permanently delete
```

**Large tables:** `get()` loads every row into memory. For thousands of rows, process in pieces:

```php
Medicine::chunk(500, function ($medicines) { /* 500 at a time */ });
foreach (Medicine::lazy() as $medicine) { /* one at a time, low memory */ }
```

## 7.3 Relationships

**What it is:** How tables connect. You define the link once in the model, then reach related data like a property.

| Relationship | Meaning | PharmaCare example |
| --- | --- | --- |
| `belongsTo` | This row points to one parent | A medicine belongs to a category |
| `hasMany` | One parent has many children | A category has many medicines |
| `hasOne` | One parent has exactly one child | A sale has one prescription |
| `belongsToMany` | Many-to-many through a pivot table | Medicines ↔ suppliers (several suppliers per medicine) |
| `hasManyThrough` | Reach grandchildren through children | A medicine has many sale items through its batches |
| `morphMany` | One table serves many parent types | Notes attached to sales, customers or batches |

```php
// app/Models/Medicine.php
public function category()   { return $this->belongsTo(Category::class); }
public function batches()    { return $this->hasMany(Batch::class); }
public function saleItems()  { return $this->hasMany(SaleItem::class); }
public function suppliers()
{
    return $this->belongsToMany(Supplier::class)          // pivot table: medicine_supplier
        ->withPivot('supplier_price')
        ->withTimestamps();
}

// app/Models/Category.php
public function medicines()  { return $this->hasMany(Medicine::class); }

// app/Models/Sale.php
public function items()        { return $this->hasMany(SaleItem::class); }
public function cashier()      { return $this->belongsTo(User::class, 'user_id'); }   // custom FK
public function customer()     { return $this->belongsTo(Customer::class); }
public function prescription() { return $this->hasOne(Prescription::class); }
```

**Using relationships:**

```php
$medicine->category->name;                       // property = the related record
$medicine->batches;                               // collection of batches
$medicine->batches()->where('quantity', '>', 0)->get();   // method = a query you can extend
$medicine->batches()->create(['batch_no' => 'AMX-2026-02', 'quantity' => 100, 'cost_price' => 350, 'expires_at' => '2027-06-30']);

$medicine->suppliers()->attach($supplierId, ['supplier_price' => 320]);
$medicine->suppliers()->sync([1, 4]);           // keep exactly these
$medicine->suppliers()->detach(4);
```

## 7.4 Eager loading and the N+1 problem

**What it is:** If you list 50 medicines and print each category, Laravel runs 1 query for the medicines plus 50 for the categories: 51 queries. This is the **N+1 problem**, and it is the most common cause of slow Laravel pages. `with()` loads all categories in one extra query: 2 queries total.

```php
// Bad: 51 queries
$medicines = Medicine::all();

// Good: 2 queries
$medicines = Medicine::with('category')->get();
$medicines = Medicine::with(['category', 'batches' => fn ($q) => $q->where('quantity', '>', 0)])->get();

// Counts and sums without loading the rows
$medicines = Medicine::withCount('batches')->withSum('batches', 'quantity')->get();
// $medicine->batches_count, $medicine->batches_sum_quantity

// Load later on an existing model
$sale->load('items.medicine');
```

Catch it during development: add `Model::preventLazyLoading(! app()->isProduction());` to `AppServiceProvider::boot()`. Laravel then throws an error whenever you forget `with()`.

## 7.5 Casts and accessors

**Casts** convert a column automatically when you read or write it.

```php
protected function casts(): array
{
    return [
        'requires_prescription' => 'boolean',      // 1/0 → true/false
        'expires_at'            => 'date',          // string → Carbon date object
        'meta'                  => 'array',         // JSON column → PHP array
        'status'                => SaleStatus::class, // PHP enum
        'password'              => 'hashed',
    ];
}

$batch->expires_at->diffForHumans();   // "in 5 months"
$batch->expires_at->isPast();          // true/false
```

**Accessors and mutators** run your own code when reading (get) or writing (set) an attribute.

```php
use Illuminate\Database\Eloquent\Casts\Attribute;

// Mutator: always store names in Title Case. Accessor: read as normal.
protected function name(): Attribute
{
    return Attribute::make(
        get: fn (string $value) => $value,
        set: fn (string $value) => ucwords(strtolower(trim($value))),
    );
}

// A computed attribute that is not a real column
protected function priceFormatted(): Attribute
{
    return Attribute::make(get: fn () => number_format($this->price) . ' TZS');
}

$medicine->price_formatted;   // "1,500 TZS"  (camelCase method → snake_case property)
```

## 7.6 Query scopes

**What it is:** A reusable, named filter you define once on the model and chain anywhere.

```php
// app/Models/Batch.php — local scopes (Laravel 12+ can also use the #[Scope] attribute)
public function scopeInStock($query)
{
    return $query->where('quantity', '>', 0);
}

public function scopeExpiringWithin($query, int $days)
{
    return $query->whereBetween('expires_at', [now(), now()->addDays($days)]);
}

public function scopeExpired($query)
{
    return $query->where('expires_at', '<', now());
}

// Use them (drop the word "scope")
Batch::inStock()->expiringWithin(90)->orderBy('expires_at')->get();
Batch::expired()->inStock()->sum('quantity');   // expired stock still on the shelf
```

**Global scope:** a filter applied to every query on the model automatically. Soft deletes work this way. Example: a multi-branch pharmacy where staff only see their own branch:

```php
protected static function booted(): void
{
    static::addGlobalScope('branch', function ($query) {
        if (auth()->check()) {
            $query->where('branch_id', auth()->user()->branch_id);
        }
    });
}

Sale::withoutGlobalScope('branch')->get();   // admin report across branches
```

## 7.7 Pagination

```php
$medicines = Medicine::with('category')->orderBy('name')->paginate(20);   // page numbers
$medicines = Medicine::simplePaginate(20);    // only Next/Previous (faster)
$medicines = Sale::latest()->cursorPaginate(50);   // fastest for huge tables / infinite scroll
$medicines->withQueryString();                 // keep ?search=... when changing page
```

```html+php
@foreach ($medicines as $medicine) ... @endforeach
{{ $medicines->links() }}     {{-- page links, Tailwind-styled by default --}}
```

Returned from an API route, a paginator becomes JSON with `data`, `links` and `meta` keys automatically.

## 7.8 API resources

**What it is:** A resource is a translator between your model and the JSON your API returns. It controls exactly which fields go out, so you never leak internal columns like `cost_price` to a mobile app.

```bash
php artisan make:resource MedicineResource
php artisan make:resource MedicineCollection   # optional, for extra collection metadata
```

```php
// app/Http/Resources/MedicineResource.php
public function toArray(Request $request): array
{
    return [
        'id'                    => $this->id,
        'name'                  => $this->name,
        'price'                 => $this->price,
        'price_formatted'       => $this->price_formatted,
        'requires_prescription' => $this->requires_prescription,
        'category'              => new CategoryResource($this->whenLoaded('category')),  // only if eager-loaded
        'stock'                 => $this->whenNotNull($this->batches_sum_quantity),
        'supplier_id'           => $this->when($request->user()?->role === 'admin', $this->supplier_id),
    ];
}
```

```php
// Api/MedicineController
public function index()
{
    return MedicineResource::collection(
        Medicine::with('category')->withSum('batches', 'quantity')->paginate(20)
    );
}

public function show(Medicine $medicine)
{
    return new MedicineResource($medicine->load('category'));
}
```
