# Sari-Sari Store POS + Inventory System — Design Spec

- **Date:** 2026-09-25
- **Author:** Brandon Dylan Narito
- **Status:** Approved design, ready for implementation planning

## 1. Purpose and scope

A point-of-sale and inventory web app for a single Filipino sari-sari store. It is built as a
portfolio showcase that is also deployment-ready for real clients.

**Users:** one owner account. No cashier/helper roles in v1.

**Primary device:** laptop / desktop PC at the counter, optionally with a USB barcode scanner
(which behaves as keyboard input). Non-checkout pages should remain usable on a phone.

**Deployment model:** one install per store (not multi-tenant). The same codebase runs:

- **Hosted** (portfolio demo and clients with usable internet) on MySQL.
- **Local install** on the store PC (clients without reliable internet) on SQLite, fully offline.

### In scope for v1

- Checkout (cash and utang), with barcode scanning and keyboard shortcuts
- Products and categories (one unit per product; no pack-to-piece conversion)
- Stock tracking with a full stock movement log
- Restocking with restock records
- Manual stock adjustments
- Low-stock alerts (in-app only)
- Utang / credit tracking with customer ledgers, payments, and per-customer credit limits
- Voiding sales
- Dashboard and reports
- Automatic local backups for local installs, with optional external drive and manual download

### Out of scope for v1

Printable receipts, cashier roles, multi-store or branches, offline sync for hosted installs,
unit conversion (tingi packs to pieces), CSV export, cloud backup, email/SMS notifications,
supplier management beyond a free-text name, NativePHP desktop packaging.

## 2. Tech stack

| Layer           | Choice                                                                |
| --------------- | --------------------------------------------------------------------- |
| Backend         | Laravel 13, PHP 8.4+                                                  |
| Frontend        | React 19 + TypeScript via the official Laravel React starter kit      |
| Bridge          | Inertia.js (no separate REST API); Laravel Wayfinder for typed routes |
| UI              | Tailwind CSS, shadcn/ui (Radix)                                       |
| Charts          | Recharts (bundled, works offline)                                     |
| Database        | MySQL (hosted), SQLite (local installs)                               |
| Auth            | Starter kit session auth; public registration disabled                |
| Tests           | Pest                                                                  |
| Build           | Vite                                                                  |
| Dev environment | Laravel Herd + MySQL on Windows                                       |

### Cross-cutting rules

1. **Money is stored as integers in centavos.** ₱12.50 is stored as `1250`. Formatting to pesos
   happens only in the UI via one shared helper.
2. **Database-agnostic code.** Use Eloquent and the query builder only. No MySQL-specific SQL
   or functions, so SQLite works for local installs. Date grouping in reports must work on both drivers.
3. **App timezone is `Asia/Manila`.**
4. **No external services in core features.** Checkout, utang, inventory, and reports must work
   with zero internet.
5. **Business operations live in single-purpose Action classes** (see §8), not in controllers.

## 3. Data model

All tables have `id` and timestamps unless noted. Money columns are integers (centavos).
Quantities are integers.

**users** — the owner (starter kit default).

**settings** — key/value store: `store_name`, `external_backup_path` (nullable).

**categories** — `name` (unique).

**products**

- `category_id` (nullable FK)
- `name`
- `barcode` (nullable, unique)
- `price`, `cost`
- `stock` (integer, may go negative)
- `low_stock_threshold` (integer, default 5)
- `is_active` (boolean, default true)

Products are deactivated, never deleted. Inactive products are hidden from checkout and restock
search but remain in history and reports.

**customers**

- `name`, `phone` (nullable), `notes` (nullable)
- `credit_limit` (nullable; null means no limit)
- `archived_at` (nullable)

**sales**

- `customer_id` (nullable FK; required when `payment_type` = `utang`)
- `payment_type`: `cash` | `utang`
- `total`
- `amount_tendered`, `change` (nullable; cash only)
- `voided_at` (nullable), `void_reason` (nullable)

**sale_items**

- `sale_id`, `product_id`
- `product_name`, `unit_price`, `unit_cost` (snapshots at time of sale)
- `quantity`, `subtotal`

**customer_payments** — `customer_id`, `amount`, `note` (nullable).

**restocks** — `supplier_name` (nullable), `total_cost`, `note` (nullable).

**restock_items** — `restock_id`, `product_id`, `quantity`, `unit_cost`.

**stock_movements**

- `product_id`
- `type`: `sale` | `restock` | `adjustment` | `void`
- `quantity_change` (signed integer)
- `stock_after` (integer; the product's stock after this movement)
- `reference_type`, `reference_id` (polymorphic link to the sale, restock, or null for adjustments)
- `reason` (nullable; for adjustments: `recount` | `expired` | `damaged` | `personal_use` | `other`)
- `note` (nullable)

### Derived values (computed, never stored)

- **Customer utang balance** = sum of non-voided utang sale totals − sum of payments.
- **Gross profit** = sum of `subtotal` − sum of (`unit_cost` × `quantity`) over non-voided sale items.

### Invariant

`products.stock` always equals the sum of that product's `stock_movements.quantity_change`.
Every code path that changes stock goes through an Action that writes both in one transaction.

## 4. Checkout

### Screen (laptop-first)

- **Left:** a search/scan input that is always focused, plus quick-tap buttons for frequent items.
- **Right:** the cart (lines with quantity controls) and a large total.

### Behavior

- An exact barcode match adds the product immediately. Scanning an item already in the cart
  increments its quantity.
- Typing a name shows matches; **Enter** adds the top result.
- Quantities are edited with + / − or by typing; **Delete** removes the selected line.
- **F2** opens the payment dialog. **Esc** closes dialogs.
    - **Cash:** enter the amount received; change is displayed prominently.
    - **Utang:** select an existing customer or quick-add a new one by name. If the sale would push
      the customer over their `credit_limit`, show a warning. The owner can still proceed.
- **Enter** confirms the sale. On success, the cart clears and focus returns to the scan input.
- The cart lives entirely in React state. The server is only called on confirm.
- The Pay button is disabled while the request is in flight, to prevent double submission.
- On a server error the cart is preserved and a plain-language error message is shown.

### Server: `CompleteSale` action

Input: a list of `{product_id, quantity}`, `payment_type`, `amount_tendered` (cash), and
`customer_id` (utang).

1. **Validate** (Form Request):
    - Cart is not empty; every quantity is at least 1.
    - Products exist and are active.
    - Cash: `amount_tendered` ≥ the server-computed total.
    - Utang: customer is required and not archived.
2. **Recompute prices from the database.** Never trust client prices.
3. **In one DB transaction,** with product rows locked for update:
    - Create the sale and its sale items, with name, price, and cost snapshots.
    - Decrement stock.
    - Write a `sale` stock movement per item.
4. **Stock at or below zero does not block the sale.** Stock may go negative, and the product is
   then flagged "Needs recount". The UI shows a non-blocking warning when adding an item whose
   stock is ≤ 0.

### Voiding: `VoidSale` action

- Requires confirmation and an optional reason.
- In one transaction: set `voided_at` / `void_reason`, restore stock, and write `void` movements.
- A voided sale is excluded from reports and from utang balances. It cannot be voided twice.

## 5. Utang (credit tracking)

### Customer list page

- Columns: name, balance, last payment date, last utang date.
- Default sort: balance, descending. Searchable by name.
- The header shows the total outstanding utang across all customers.
- **Overdue highlight:** balance > 0 and no payment in the last 30 days. If the customer has
  never paid, the 30 days are measured from their earliest non-voided utang sale.

### Customer detail page (ledger)

- One timeline of utang sales and payments, newest first, with a running balance.
- Utang sales expand to show their items.
- Voided utang sales appear struck through and do not count toward the balance.
- **Record Payment:** amount plus optional note. Partial payments are allowed.

### Rules

- Utang is only created through checkout. There is no manual "add utang" entry.
- A payment amount must be > 0 and ≤ the current balance. No store credit.
- A customer with a balance > 0 cannot be archived. Archived customers are hidden from
  checkout selection but keep their history.
- Credit limit is optional per customer and is warn-only at checkout.

## 6. Inventory

### Products page

- Table with search, category filter, and sorting by name or stock.
- Stock status badge per product:
    - **OK:** stock > threshold
    - **Low:** 0 < stock ≤ threshold
    - **Out:** stock = 0
    - **Needs recount:** stock < 0
- Create/edit form: name, category, barcode (scannable into the field), price, cost,
  low-stock threshold. Show a warning (not an error) when price < cost.
- Duplicate barcodes are rejected by validation.
- Products are deactivated or reactivated, never deleted.

### Restocking: `RecordRestock` action

- A New Restock screen with the same scan/search pattern as checkout. Each line takes a
  quantity (≥ 1) and a unit cost. Optional supplier name and note.
- In one transaction: create the restock and its items, increment stock, write `restock`
  movements, and set each product's `cost` to the latest `unit_cost`.
- A restock history list shows date, supplier, item count, and total cost; each restock can be opened.

### Stock adjustments: `AdjustStock` action

- **Set counted quantity** (reason `recount`): the system computes the difference.
- **Subtract** with reason `expired`, `damaged`, `personal_use`, or `other`, plus an optional note.
- Writes an `adjustment` movement.

### Low-stock alerts (in-app only)

- A dashboard panel, "Kailangan i-restock", lists products that are Low, Out, or Needs recount.
- A badge count on the Products navigation item.
- **"Create restock from low-stock list"** pre-fills a new restock with those products.

### Stock history

Each product has a movement log with type, quantity change, stock after, reference link,
reason/note, and date.

## 7. Dashboard and reports

### Dashboard (home page)

- **Today:** sales total, transaction count, gross profit, cash vs. utang split.
- Kailangan i-restock panel.
- Top 5 customers by utang balance.
- Last 7 days sales chart.
- A prominent **New Sale** button.
- **Local installs only:** a dismissible backup reminder showing "Last external backup: never"
  or "N days ago" when N > 7 or no external backup path is set.

### Reports page

A date range selector (Today, This week, This month, Custom) and four tabs. Voided sales are
always excluded.

1. **Sales summary:** revenue, cost, gross profit, transaction count, average sale, and a daily chart.
2. **Top products:** ranked by quantity sold and by gross profit.
3. **Utang:** total outstanding (as of now), new utang in the period, payments collected in the
   period, and a list of overdue customers.
4. **Transactions:** a searchable list of sales with detail view and void action.

All aggregates are computed in the database with sums and group-bys, not by loading rows into PHP.

## 8. Application structure

Each business operation is one Action class with one public method. Every action that moves
money or stock runs in a DB transaction.

| Action                  | Responsibility                        |
| ----------------------- | ------------------------------------- |
| `CompleteSale`          | Checkout (§4)                         |
| `VoidSale`              | Void a sale, restore stock (§4)       |
| `RecordRestock`         | Restock, update costs (§6)            |
| `AdjustStock`           | Manual adjustments and recounts (§6)  |
| `RecordCustomerPayment` | Utang payment with balance check (§5) |
| `BackupDatabase`        | Local SQLite backup (§10)             |

Controllers only validate (via Form Requests), call an Action, and return an Inertia response.
Balance and profit calculations live in model query scopes or small query classes so reports
and pages share one definition.

### Pages (Inertia/React)

Dashboard, Checkout, Products (list, form, stock history), Restocks (list, new, detail),
Customers (list, detail/ledger), Reports, Settings.

## 9. Auth

- Starter kit login. Public registration is disabled.
- The owner account is created with an artisan command: `php artisan pos:create-owner`.
- All app routes require authentication.

## 10. Deployment and backups

### Hosted

A VPS or Laravel-friendly host with MySQL. Database backups are handled by the host's
tooling; the in-app backup feature is disabled when the driver is not SQLite.

### Local install (store PC, offline)

- SQLite database. The app starts automatically when the PC boots.
- A desktop shortcut opens the app in a Chrome app window (no address bar or tabs).
- A documented update script: pull files, install dependencies, run migrations, rebuild assets.

### Backups (local installs only): `BackupDatabase` action + `pos:backup` command

- Uses SQLite `VACUUM INTO` for a consistent copy, never a raw file copy.
- **Always:** writes to a local backups folder, keeping the latest 14 daily copies.
- **Optional:** if `external_backup_path` is set and reachable, also copies there. If it is
  unreachable (e.g., USB not plugged in), that destination is skipped silently and logged.
- Runs daily through the Laravel scheduler (triggered by Windows Task Scheduler) and also once
  at startup. It skips if a backup for today already exists, so a missed day is caught on the next boot.
- **Settings:** set or clear the external backup path, see the last backup times, and use a
  **Download backup** button.

## 11. Error handling

- Form Request validation on every write. Errors appear inline via Inertia.
- Toast notifications for success ("Sale saved", "Payment recorded").
- Confirmation dialogs for void, deactivate product, and archive customer.
- Server errors on checkout keep the cart intact.

## 12. Testing (Pest)

### Feature tests (priority)

- A sale decrements stock, writes movements, and snapshots name, price, and cost.
- Client-sent prices are ignored in favor of database prices.
- A cash payment below the total is rejected.
- An utang sale without a customer is rejected; a credit-limit breach warns but succeeds.
- Selling at zero stock succeeds and results in negative stock.
- Voiding restores stock, excludes the sale from reports, reduces the utang balance, and
  cannot be done twice.
- A payment greater than the balance is rejected; partial payments reduce the balance.
- A customer with a balance cannot be archived.
- A restock increments stock and updates product cost.
- Adjustments (recount and subtract) produce the correct stock and movements.
- The stock invariant holds after a mixed sequence of operations.
- Report totals and profit are correct and exclude voided sales.
- All routes require authentication.

### Unit tests

- Peso formatting helper.
- Balance and profit calculations.

The React UI is tested manually for v1.

## 13. Demo data

A `DemoSeeder` creates realistic data: Filipino sari-sari products across categories (some low,
out of stock, and negative), about 15 customers with varied utang balances and payment histories,
and about 60 days of sales, restocks, and a few voids. It must be re-runnable to reset the portfolio demo.
