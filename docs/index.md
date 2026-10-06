# Laravel Cheat Sheet — PharmaCare Edition

## How to use this sheet

This sheet follows the roadmap.sh Laravel roadmap from the very first step to deployment. Every topic has the same four parts, so you always know where to look:

1. **What it is** — a plain-language definition a beginner can follow.
2. **Why it matters** — the problem it solves.
3. **How to do it** — the Artisan command (if any) and the steps.
4. **Example** — working code from one shared project: PharmaCare.

The code targets Laravel 11 and later (the slim skeleton with `bootstrap/app.php`). Run `php artisan --version` to check yours.

## The example project: PharmaCare

PharmaCare is a small pharmacy management system. Every example in this sheet adds one piece to it, so by the end you have seen a full app built step by step.

**What the pharmacy does:** keeps a list of medicines, tracks stock by batch and expiry date, sells to customers at the counter, records prescriptions, and alerts staff when stock is low or a batch is about to expire.

**The main data (models):**

| Model | What it stores | Example record |
| --- | --- | --- |
| `Category` | Groups of medicines | Antibiotics, Painkillers |
| `Supplier` | Who we buy stock from | Shelys Pharmaceuticals |
| `Medicine` | One product on the shelf | Amoxicillin 500mg, price 500 TZS |
| `Batch` | One delivery of a medicine, with its own expiry | Batch AMX-2026-01, 200 units, expires 2027-03-31 |
| `Customer` | A buyer | Name, phone |
| `Sale` | One checkout at the counter | Total 12,000 TZS, paid by M-Pesa |
| `SaleItem` | One line inside a sale | 2 × Amoxicillin |
| `Prescription` | A doctor's prescription linked to a sale | Scanned image, doctor name |
| `User` | Staff who log in | Role: admin, pharmacist or cashier |

**The staff roles:**

- **Admin** — manages users, suppliers and prices; sees all reports.
- **Pharmacist** — adds medicines, receives stock, approves prescription sales.
- **Cashier** — makes sales only.

Money is stored as whole numbers of TZS (integers), never as floats. That rule is repeated in the database section because it matters in any system that handles money.

## Sections

| # | Section | Covers |
| --- | --- | --- |
| 1 | [Foundations](01-foundations.md) | Installing, project structure, `.env`, Artisan, Composer |
| 2 | [Routing](02-routing.md) | Routes, parameters, named routes, groups, model binding |
| 3 | [Controllers](03-controllers.md) | Request lifecycle, controllers, requests, responses |
| 4 | [Middleware](04-middleware.md) | Middleware, CORS, rate limiting |
| 5 | [Views & Blade](05-views-blade.md) | Blade syntax, layouts, components, forms, CSRF |
| 6 | [Database](06-database.md) | Migrations, factories, seeders, query builder, transactions |
| 7 | [Eloquent ORM](07-eloquent.md) | Models, relationships, eager loading, scopes, API resources |
| 8 | [Validation](08-validation.md) | Rules, form requests, custom rules, errors |
| 9 | [Authentication](09-authentication.md) | Starter kits, manual login, Sanctum, Passport |
| 10 | [Authorization](10-authorization.md) | Gates and policies |
| 11 | [Service container](11-service-container.md) | Dependency injection, providers, facades |
| 12 | [Events & queues](12-events-queues.md) | Events, jobs, scheduler, notifications |
| 13 | [Caching & storage](13-caching-storage.md) | Cache, files, localization, hashing, encryption |
| 14 | [Errors & logging](14-errors-logging.md) | Exceptions, logs, Debugbar, Telescope |
| 15 | [Testing](15-testing.md) | PHPUnit, Pest, feature and unit tests, fakes |
| 16 | [Frontend](16-frontend.md) | Vite, Livewire, Inertia |
| 17 | [Performance & deployment](17-performance-deployment.md) | Optimize, Octane, Pulse, Pint, deploying |
| 18 | [Commands](18-commands.md) | Every command in one place |

Tip: press `/` or `S` on any page to search the whole sheet.
