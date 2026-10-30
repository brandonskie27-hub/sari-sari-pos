# Sari-Sari Store POS + Inventory

A point-of-sale and inventory web app for a Filipino sari-sari store: checkout with barcode scanning, utang (credit) tracking with customer ledgers, stock tracking with low-stock alerts, and sales reports.

> **Status: design approved, implementation starting.** This repo currently holds the Laravel + React starter kit and the full design spec. The features below are what v1 is designed to do, not what's built yet.

## The design

The full spec is in [`docs/superpowers/specs/2026-09-25-sari-sari-pos-design.md`](docs/superpowers/specs/2026-09-25-sari-sari-pos-design.md). The parts I think matter most:

- **Works with no internet.** One codebase runs two ways: hosted on MySQL for stores with a usable connection, or installed on the store's own PC on SQLite, fully offline. Checkout, utang, inventory and reports never depend on an external service, and the code avoids MySQL-specific SQL so both databases behave the same.
- **Money stored as whole centavos.** ₱12.50 is stored as `1250`, so totals never pick up floating-point rounding errors. Pesos are formatted in one shared helper, only in the UI.
- **Built for the counter.** Laptop-first checkout with keyboard shortcuts, and a USB barcode scanner works because it behaves like a keyboard.
- **Utang that adds up.** Each customer has a ledger of credit sales and payments, an optional credit limit that warns at checkout, and a balance that's always computed from that history, never typed in.
- **Nothing silently disappears.** Every stock change goes into a movement log, sales are voided rather than deleted, and products are deactivated rather than removed, so history and reports stay complete.
- **Automatic backups** for offline installs, with an optional external drive and manual download.

### Planned v1 features

Checkout (cash and utang) · products and categories · stock movement log · restocking · manual stock adjustments · in-app low-stock alerts · customer ledgers, payments and credit limits · voiding sales · dashboard and reports · local backups

## Tech stack

Laravel 13 · React 19 + TypeScript · Inertia.js · Tailwind CSS + shadcn/ui · Recharts · MySQL or SQLite · Pest

---

By [Brandon Dylan Narito](https://brandonnarito.com)
