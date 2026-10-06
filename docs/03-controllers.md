# 3. Controllers and the request/response flow

## 3.1 The request lifecycle

**What it is:** The path every request travels from the browser to your code and back. Knowing it tells you where to put things and where to look when something breaks.

1. The browser asks for `/medicines`. The web server sends every request to `public/index.php`.
2. `index.php` loads Composer's autoloader and creates the Laravel application from `bootstrap/app.php`.
3. Service providers start up and register the app's services (database, cache, mail, your own).
4. The HTTP kernel passes the request through global middleware (trim strings, maintenance mode check).
5. The router finds the matching route and runs that route's middleware (`auth`, `role:pharmacist`).
6. The controller method runs, uses models to read or write data, and returns a response.
7. The response travels back out through the middleware and is sent to the browser.

## 3.2 Basic controllers

**What it is:** A class that groups the code for related routes, so `routes/web.php` stays short and readable.

```bash
php artisan make:controller ReportController
```

```php
// app/Http/Controllers/ReportController.php
namespace App\Http\Controllers;

use App\Models\Sale;

class ReportController extends Controller
{
    public function daily()
    {
        $sales = Sale::whereDate('created_at', today())->get();
        $total = $sales->sum('total');

        return view('reports.daily', compact('sales', 'total'));
    }
}
```

```php
// routes/web.php
use App\Http\Controllers\ReportController;

Route::get('/reports/daily', [ReportController::class, 'daily'])->name('reports.daily');
```

## 3.3 Resource controllers

**What it is:** A controller pre-filled with the seven CRUD methods that match `Route::resource()` (see section 2.7).

```bash
php artisan make:controller MedicineController --resource --model=Medicine
php artisan make:controller Api/MedicineController --api --model=Medicine   # no create/edit
```

```php
class MedicineController extends Controller
{
    public function index()
    {
        $medicines = Medicine::with('category')->orderBy('name')->paginate(20);
        return view('medicines.index', compact('medicines'));
    }

    public function create()
    {
        return view('medicines.create', ['categories' => Category::all()]);
    }

    public function store(Request $request)
    {
        $data = $request->validate([
            'name'        => 'required|string|max:150',
            'category_id' => 'required|exists:categories,id',
            'price'       => 'required|integer|min:0',
        ]);

        Medicine::create($data);
        return redirect()->route('medicines.index')->with('success', 'Medicine added.');
    }

    public function show(Medicine $medicine)
    {
        return view('medicines.show', compact('medicine'));
    }

    public function edit(Medicine $medicine)
    {
        return view('medicines.edit', ['medicine' => $medicine, 'categories' => Category::all()]);
    }

    public function update(Request $request, Medicine $medicine)
    {
        $medicine->update($request->validate(['price' => 'required|integer|min:0']));
        return back()->with('success', 'Price updated.');
    }

    public function destroy(Medicine $medicine)
    {
        $medicine->delete();
        return redirect()->route('medicines.index');
    }
}
```

## 3.4 Single-action controllers

**What it is:** A controller that does exactly one job, using the `__invoke` method. Good for actions that are not CRUD, like "print a receipt".

```bash
php artisan make:controller PrintReceiptController --invokable
```

```php
class PrintReceiptController extends Controller
{
    public function __invoke(Sale $sale)
    {
        return view('sales.receipt', ['sale' => $sale->load('items.medicine')]);
    }
}

// routes/web.php — no method name needed
Route::get('/sales/{sale}/receipt', PrintReceiptController::class)->name('sales.receipt');
```

## 3.5 The Request object

**What it is:** Everything the browser sent (form fields, query string, files, headers, the logged-in user), wrapped in one object.

```php
public function search(Request $request)
{
    $request->input('q');                  // form field or query string
    $request->query('page', 1);            // only the query string, default 1
    $request->only(['name', 'price']);     // just these fields
    $request->except(['_token']);          // everything except these
    $request->has('category_id');          // field present?
    $request->filled('q');                 // present AND not empty?
    $request->boolean('in_stock');         // "1", "true", "on" → true
    $request->file('prescription');        // an uploaded file
    $request->user();                      // logged-in staff member
    $request->ip();
    $request->isMethod('post');
    $request->expectsJson();               // API client?
}
```

## 3.6 Creating responses

**What it is:** Whatever a controller returns becomes the response. Laravel converts it for you.

```php
// HTML page
return view('medicines.index', ['medicines' => $medicines]);

// Plain text with a status code
return response('Out of stock', 409);

// JSON (arrays and models are converted to JSON automatically)
return response()->json(['stock' => 120, 'medicine' => 'Paracetamol'], 200);
return $medicine;  // also returns JSON

// Add headers
return response($csv)->header('Content-Type', 'text/csv');
```

## 3.7 Redirects

```php
return redirect('/dashboard');
return redirect()->route('sales.show', $sale);
return back();                                          // previous page
return back()->withInput()->withErrors(['qty' => 'Not enough stock']);
return redirect()->route('medicines.index')->with('success', 'Saved!');  // flash message
return redirect()->away('https://tmda.go.tz');          // external site
```

A flash message lives for one request only. Show it in Blade with `session('success')`.

## 3.8 Files: download, display, stream

```php
// Force a download (e.g. a prescription scan)
return response()->download(storage_path('app/prescriptions/rx-15.pdf'), 'prescription-15.pdf');

// Show in the browser instead of downloading
return response()->file(storage_path('app/prescriptions/rx-15.pdf'));

// Streamed download: build a big CSV row by row without loading everything into memory
return response()->streamDownload(function () {
    $out = fopen('php://output', 'w');
    fputcsv($out, ['Date', 'Sale #', 'Total (TZS)']);
    Sale::orderBy('id')->lazy()->each(function ($sale) use ($out) {
        fputcsv($out, [$sale->created_at, $sale->id, $sale->total]);
    });
    fclose($out);
}, 'sales-report.csv');
```

**Streamed responses** send data in chunks as it is produced. Use them for large exports, so a year of sales does not run the server out of memory.
