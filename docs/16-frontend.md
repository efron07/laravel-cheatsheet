# 16. Frontend choices: Blade, Livewire, Inertia

**The question:** Laravel handles the server side. How do you build the screens users click on? There are three main roads, plus a fourth for separate apps.

| Approach | How it works | You write | Best for |
| --- | --- | --- | --- |
| **Blade only** | Server builds full HTML pages; every click loads a new page | PHP + Blade (+ a little JS) | Simple admin pages, reports, content sites |
| **Livewire** | Blade components that update parts of the page without reloading, via small AJAX calls handled for you | PHP + Blade only | Interactive back-office apps (POS screens, live search) without writing JavaScript |
| **Inertia** | Laravel routes and controllers send data to React or Vue pages; feels like a SPA | PHP controllers + React/Vue | Teams strong in React/Vue who want one codebase |
| **Separate frontend** | Laravel is only a JSON API (Sanctum); a Next.js / mobile app calls it | PHP API + any JS framework | Mobile apps, public web apps, multiple clients |

## 16.1 Assets with Vite

**What it is:** Vite compiles and bundles your CSS and JavaScript (including Tailwind). It is set up in every new Laravel project.

```bash
npm install
npm run dev      # development: watches files and reloads the browser
npm run build    # production: creates optimised files in public/build
```

```html+php
<head>
    @vite(['resources/css/app.css', 'resources/js/app.js'])
</head>
```

If the page looks unstyled with a "Vite manifest not found" error, you forgot `npm run dev` (local) or `npm run build` (server).

## 16.2 Livewire

**What it is:** A Livewire component is a PHP class plus a Blade view. Public properties in the class are shown in the view; when the user types or clicks, Livewire sends the change to the server, re-runs the PHP and updates only that part of the page.

```bash
composer require livewire/livewire
php artisan make:livewire MedicineSearch
```

**PharmaCare example: live medicine search at the counter**

```php
// app/Livewire/MedicineSearch.php
namespace App\Livewire;

use App\Models\Medicine;
use Livewire\Component;

class MedicineSearch extends Component
{
    public string $search = '';
    public array $cart = [];      // [medicine_id => qty]

    public function addToCart(int $id): void
    {
        $this->cart[$id] = ($this->cart[$id] ?? 0) + 1;
    }

    public function render()
    {
        return view('livewire.medicine-search', [
            'results' => strlen($this->search) < 2
                ? collect()
                : Medicine::where('name', 'like', "%{$this->search}%")->withSum('batches', 'quantity')->limit(10)->get(),
        ]);
    }
}
```

```html+php
{{-- resources/views/livewire/medicine-search.blade.php (ONE root element) --}}
<div>
    <input type="text" wire:model.live.debounce.300ms="search" placeholder="Search medicine...">

    @foreach ($results as $medicine)
        <div wire:key="med-{{ $medicine->id }}">
            {{ $medicine->name }} — {{ number_format($medicine->price) }} TZS
            ({{ $medicine->batches_sum_quantity ?? 0 }} in stock)
            <button wire:click="addToCart({{ $medicine->id }})">Add</button>
        </div>
    @endforeach

    <p>Items in cart: {{ array_sum($cart) }}</p>
</div>
```

```html+php
{{-- Use it in any page --}}
<livewire:medicine-search />
```

**Key directives:** `wire:model` (bind an input to a property; `.live` updates as you type), `wire:click` (call a method), `wire:submit` (form submit), `wire:loading` (show while waiting), `wire:key` (identify list items), `wire:poll.10s` (refresh every 10 seconds, e.g. a live stock board).

Validation works inside Livewire with `$this->validate([...])` and the same rules as section 8.

## 16.3 Inertia

**What it is:** Inertia is the glue between Laravel and React or Vue. Your controller returns `Inertia::render()` with data instead of a Blade view; the data arrives as props in a React/Vue page component. No separate API needed.

```php
// Controller
use Inertia\Inertia;

public function index()
{
    return Inertia::render('Medicines/Index', [
        'medicines' => MedicineResource::collection(Medicine::with('category')->paginate(20)),
        'filters'   => request()->only('search'),
    ]);
}
```

```jsx
// resources/js/Pages/Medicines/Index.jsx
import { Link, router } from '@inertiajs/react';

export default function Index({ medicines }) {
    return (
        <div>
            <h1>Medicines</h1>
            {medicines.data.map((m) => (
                <div key={m.id}>
                    {m.name} — {m.price} TZS
                    <Link href={route('medicines.edit', m.id)}>Edit</Link>
                    <button onClick={() => router.delete(route('medicines.destroy', m.id))}>Delete</button>
                </div>
            ))}
        </div>
    );
}
```

The easiest way to start with Inertia is choosing the React or Vue starter kit when creating the project (section 9.1).

## 16.4 How to choose

- Back-office tool used by staff on a desktop (most of PharmaCare): **Livewire**.
- Your team already thinks in React components: **Inertia + React**.
- A customer mobile app or several frontends must share the same backend: **API + Sanctum**, frontend separate.
- Mostly static pages and forms: **Blade only**.
