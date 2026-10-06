# 8. Validation

**What it is:** Validation checks user input before you use it. If anything is wrong, Laravel stops, sends the user back to the form with error messages and their old input (or returns a 422 JSON response for APIs).

**Why it matters:** Never trust input. A cashier can type "-5" as a quantity, or a price with letters. Validation is your first line of defence.

## 8.1 Quick validation in the controller

```php
public function store(Request $request)
{
    $validated = $request->validate([
        'name'                  => ['required', 'string', 'max:150'],
        'category_id'           => ['required', 'exists:categories,id'],
        'barcode'               => ['nullable', 'string', 'unique:medicines,barcode'],
        'price'                 => ['required', 'integer', 'min:1'],
        'requires_prescription' => ['boolean'],
    ]);

    Medicine::create($validated);   // only validated fields — safe
    return redirect()->route('medicines.index')->with('success', 'Medicine added.');
}
```

If validation fails, the code after `validate()` never runs.

## 8.2 Most-used rules

| Rule | Meaning | Example |
| --- | --- | --- |
| `required` / `nullable` | Must be present / may be empty | `'name' => 'required'` |
| `string`, `integer`, `numeric`, `boolean`, `array` | Data type | `'qty' => 'integer'` |
| `min:x` / `max:x` / `between:x,y` | Size (number value, string length, file KB) | \`'qty' => 'integer |
| `email`, `url`, `date`, `uuid` | Format | \`'email' => 'required |
| `in:a,b` | One of these values | `'payment' => 'in:cash,mpesa,card'` |
| `exists:table,column` | Must exist in the database | `'medicine_id' => 'exists:medicines,id'` |
| `unique:table,column` | Must NOT already exist | `'barcode' => 'unique:medicines'` |
| `confirmed` | Needs a matching `_confirmation` field | Password + `password_confirmation` |
| `after:date` / `before:date` | Date order | `'expires_at' => 'after:today'` |
| `regex:/.../` | Custom pattern | Tanzanian phone: `'regex:/^0[67]\d{8}$/'` |
| `file`, `image`, `mimes:pdf,jpg`, `max:2048` | Uploads (max in KB) | Prescription scan |
| `sometimes` | Validate only if the field is sent | Partial API updates |
| `required_if:field,value` | Required only in some cases | Prescription required if medicine needs one |

**Arrays (a sale with many lines):**

```php
$request->validate([
    'items'               => ['required', 'array', 'min:1'],
    'items.*.medicine_id' => ['required', 'exists:medicines,id'],
    'items.*.qty'         => ['required', 'integer', 'min:1'],
    'payment_method'      => ['required', Rule::in(['cash', 'mpesa', 'airtel', 'card'])],
]);
```

**Unique when editing** (ignore the record being edited):

```php
use Illuminate\Validation\Rule;

'barcode' => ['nullable', Rule::unique('medicines')->ignore($medicine->id)],
```

## 8.3 Form requests (the clean way)

**What it is:** A separate class that holds the validation rules, the permission check and custom messages for one form. The controller stays short.

```bash
php artisan make:request StoreBatchRequest
```

```php
// app/Http/Requests/StoreBatchRequest.php
class StoreBatchRequest extends FormRequest
{
    // Who may submit this form? false → 403 Forbidden
    public function authorize(): bool
    {
        return in_array($this->user()->role, ['pharmacist', 'admin']);
    }

    public function rules(): array
    {
        return [
            'medicine_id' => ['required', 'exists:medicines,id'],
            'batch_no'    => ['required', 'string', 'max:50', 'unique:batches,batch_no'],
            'quantity'    => ['required', 'integer', 'min:1'],
            'cost_price'  => ['required', 'integer', 'min:0'],
            'expires_at'  => ['required', 'date', 'after:today'],
        ];
    }

    // Custom error messages
    public function messages(): array
    {
        return [
            'expires_at.after'  => 'You cannot receive stock that has already expired.',
            'batch_no.unique'   => 'This batch number is already in the system.',
        ];
    }

    // Friendlier field names in default messages
    public function attributes(): array
    {
        return ['expires_at' => 'expiry date', 'batch_no' => 'batch number'];
    }

    // Clean the input before validation runs
    protected function prepareForValidation(): void
    {
        $this->merge(['batch_no' => strtoupper(trim($this->batch_no))]);
    }
}
```

```php
// The controller: type-hint it and validation runs automatically
public function store(StoreBatchRequest $request)
{
    Batch::create($request->validated());
    return back()->with('success', 'Stock received.');
}
```

## 8.4 Manual validation

**What it is:** Use the `Validator` facade when you need control over what happens on failure, for example inside a job, an import or a webhook.

```php
use Illuminate\Support\Facades\Validator;

$validator = Validator::make($row, [
    'name'  => 'required|string',
    'price' => 'required|integer|min:1',
]);

if ($validator->fails()) {
    Log::warning('Bad import row', $validator->errors()->toArray());
    return;   // skip this row, keep importing the others
}

$data = $validator->validated();

// Extra checks after the basic rules pass
$validator->after(function ($validator) use ($row) {
    if (Medicine::find($row['medicine_id'])?->totalStock() < $row['qty']) {
        $validator->errors()->add('qty', 'Not enough stock for this sale.');
    }
});
```

## 8.5 Custom rule class

```bash
php artisan make:rule TanzanianPhone
```

```php
class TanzanianPhone implements ValidationRule
{
    public function validate(string $attribute, mixed $value, Closure $fail): void
    {
        if (! preg_match('/^(?:\+255|0)[67]\d{8}$/', $value)) {
            $fail('The :attribute must be a valid Tanzanian mobile number.');
        }
    }
}

// Use it
'phone' => ['required', new TanzanianPhone],
```

## 8.6 Showing errors

```html+php
{{-- All errors at the top of the form --}}
@if ($errors->any())
    <ul class="errors">
        @foreach ($errors->all() as $error)
            <li>{{ $error }}</li>
        @endforeach
    </ul>
@endif

{{-- One field --}}
<input name="quantity" value="{{ old('quantity') }}" class="@error('quantity') is-invalid @enderror">
@error('quantity') <span class="error">{{ $message }}</span> @enderror
```

To change the default messages for the whole app, publish the language files with `php artisan lang:publish` and edit `lang/en/validation.php`.
