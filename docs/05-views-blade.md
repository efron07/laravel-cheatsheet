# 5. Views and Blade

**What it is:** A view is the HTML the user sees. Blade is Laravel's template language: normal HTML plus short tags like `{{ }}` and `@if` that insert data and logic. Blade files end in `.blade.php` and live in `resources/views/`.

```bash
php artisan make:view medicines.index     # creates resources/views/medicines/index.blade.php
php artisan view:clear                    # delete compiled views if a change is not showing
```

## 5.1 Displaying data

```html+php
{{-- This is a Blade comment: it never reaches the browser --}}

<h1>{{ $medicine->name }}</h1>                        {{-- escaped: safe from XSS --}}
<p>Price: {{ number_format($medicine->price) }} TZS</p>
<p>Supplier: {{ $medicine->supplier?->name ?? 'Unknown' }}</p>

{!! $medicine->description_html !!}                     {{-- NOT escaped: only for HTML you trust --}}

<script>
    const medicine = @json($medicine);                  // pass PHP data to JavaScript
</script>
```

Rule: always use `{{ }}`. It escapes HTML, so a customer typing `<script>` into a name field cannot attack your staff. Use `{!! !!}` only for HTML you created yourself.

## 5.2 Directives: conditions

```html+php
@if ($medicine->stock <= 0)
    <span class="badge red">Out of stock</span>
@elseif ($medicine->stock < config('pharmacy.low_stock_threshold'))
    <span class="badge orange">Low stock</span>
@else
    <span class="badge green">In stock</span>
@endif

@unless ($medicine->requires_prescription)
    <button>Add to cart</button>
@endunless

@isset($customer) Selling to {{ $customer->name }} @endisset
@empty($sales) No sales yet today. @endempty

@auth  Logged in as {{ auth()->user()->name }} @endauth
@guest <a href="{{ route('login') }}">Login</a> @endguest

@can('update', $medicine) <a href="{{ route('medicines.edit', $medicine) }}">Edit</a> @endcan

@switch($sale->payment_method)
    @case('cash')  Cash @break
    @case('mpesa') M-Pesa @break
    @default       Other
@endswitch
```

## 5.3 Directives: loops

```html+php
<table>
@forelse ($medicines as $medicine)
    <tr class="{{ $loop->even ? 'bg-gray' : '' }}">
        <td>{{ $loop->iteration }}</td>              {{-- 1, 2, 3 ... --}}
        <td>{{ $medicine->name }}</td>
        <td>{{ $medicine->stock }}</td>
    </tr>
@empty
    <tr><td colspan="3">No medicines found.</td></tr>
@endforelse
</table>

@foreach ($sale->items as $item) ... @endforeach
@for ($i = 1; $i <= 5; $i++) ... @endfor
@while ($condition) ... @endwhile
```

**The `$loop` variable** (available in every loop): `$loop->index` (starts at 0), `$loop->iteration` (starts at 1), `$loop->first`, `$loop->last`, `$loop->count`, `$loop->even`, `$loop->odd`.

## 5.4 Layouts

**What it is:** One master page with the header, menu and footer. Every other page fills in only its own content. There are two styles; components are the modern choice.

**Style A: component layout (recommended)**

```html+php
{{-- resources/views/components/layouts/app.blade.php --}}
<!DOCTYPE html>
<html>
<head>
    <title>{{ $title ?? 'PharmaCare' }}</title>
    @vite(['resources/css/app.css', 'resources/js/app.js'])
</head>
<body>
    <nav>PharmaCare | {{ auth()->user()?->name }}</nav>

    @if (session('success'))
        <div class="alert">{{ session('success') }}</div>
    @endif

    <main>{{ $slot }}</main>
</body>
</html>
```

```html+php
{{-- resources/views/medicines/index.blade.php --}}
<x-layouts.app title="Medicines">
    <h1>All medicines</h1>
    ...
</x-layouts.app>
```

**Style B: template inheritance (older, still common)**

```html+php
{{-- layouts/master.blade.php --}}
<body>
    @include('partials.nav')
    @yield('content')
    @stack('scripts')
</body>

{{-- medicines/index.blade.php --}}
@extends('layouts.master')

@section('content')
    <h1>All medicines</h1>
@endsection

@push('scripts')
    <script src="/js/medicines.js"></script>
@endpush
```

## 5.5 Components

**What it is:** A reusable piece of UI, like a stock badge or an alert box, written once and used everywhere with an `<x-...>` tag.

```bash
php artisan make:component StockBadge          # class + view
php artisan make:component Alert --view        # anonymous: view only, no class
```

```php
// app/View/Components/StockBadge.php
class StockBadge extends Component
{
    public function __construct(public int $stock) {}

    public function colour(): string
    {
        return match (true) {
            $this->stock <= 0  => 'red',
            $this->stock < 20  => 'orange',
            default            => 'green',
        };
    }

    public function render()
    {
        return view('components.stock-badge');
    }
}
```

```html+php
{{-- resources/views/components/stock-badge.blade.php --}}
<span {{ $attributes->merge(['class' => 'badge ' . $colour()]) }}>
    {{ $stock }} in stock
</span>

{{-- Use it --}}
<x-stock-badge :stock="$medicine->stock" class="ml-2" />
```

Note the colon: `:stock="$medicine->stock"` passes a PHP value; `stock="20"` passes the plain text "20".

## 5.6 Forms and CSRF

**What it is:** CSRF (Cross-Site Request Forgery) is an attack where another website tricks a logged-in user's browser into submitting a form to your app. Laravel blocks it by requiring a secret token in every form. Without `@csrf`, the form fails with error **419 Page Expired**.

```html+php
<form method="POST" action="{{ route('medicines.update', $medicine) }}">
    @csrf                    {{-- hidden security token: REQUIRED --}}
    @method('PUT')           {{-- HTML forms only support GET/POST; this fakes PUT --}}

    <label>Name</label>
    <input name="name" value="{{ old('name', $medicine->name) }}">
    @error('name')
        <p class="error">{{ $message }}</p>
    @enderror

    <label>Category</label>
    <select name="category_id">
        @foreach ($categories as $category)
            <option value="{{ $category->id }}" @selected(old('category_id', $medicine->category_id) == $category->id)>
                {{ $category->name }}
            </option>
        @endforeach
    </select>

    <input type="checkbox" name="requires_prescription" value="1" @checked(old('requires_prescription', $medicine->requires_prescription))>

    <button type="submit">Save</button>
</form>

{{-- Delete button --}}
<form method="POST" action="{{ route('medicines.destroy', $medicine) }}">
    @csrf @method('DELETE')
    <button onclick="return confirm('Delete this medicine?')">Delete</button>
</form>
```

**`old('name')`** refills the field with what the user typed after a validation error, so they do not retype the whole form.
