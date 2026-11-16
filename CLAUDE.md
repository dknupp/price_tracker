# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project status

Receipt Tracker is a tool for logging grocery receipts (one receipt = one trip) into SQLite to compare prices over time and across stores, and to catch receipt errors such as double charges. **There is no code yet.** The repo holds two design documents, and the code still has to be built from them:

- `receipt_tracker_db_spec.md` is the authoritative v1 spec: schema, views, load behavior, edge cases and acceptance criteria. Build to it, and update it when a design decision changes.
- `Receipt Tracker Feature Ideas.md` is a brainstorm-stage backlog. Ideas there are not specified, so don't build them unless asked.

## Planned stack and entry points (from the spec)

Python 3, standard library only: `sqlite3`, `argparse`, `csv`, `json`. **Add no third-party dependencies.** The schema is plain SQL so it can port to PostgreSQL later, so avoid SQLite-only features where a portable form exists.

- `load_trip.py --file trip.json [--db receipts.db] [--image photo.jpg ...] [--receipts-dir DIR] [--update TRIP_ID | --force]` loads one trip from JSON in the format of spec 7.8, or replaces an existing trip with `--update` (spec 7.9). It copies the photos and the JSON into the receipts folder (spec 7.7), which is never committed to git.
- `import_history.py --history trips.csv` bulk-loads many trips from a CSV (one row per line item). It seeds the database from the existing "Grocery Price Tracker" doc after a one-time conversion of that doc to CSV.
- There is no report program. Claude answers questions with SQL against the database, using the views (`v_unit_price`, `v_markdown`) for logic that is easy to get wrong.

No build, lint or test tooling exists yet. The acceptance criteria in spec section 10 are the test targets, and they use real receipts as fixtures (the Oct 27, 2026 Grocery Outlet $5-off-$25 and the Oct 30 Costco receipt's printed FSA total of $12.17).

## Invariants that must hold everywhere

These come from past errors on the project and are checked by the acceptance criteria:

- **Money is integer cents** (`*_cents` columns). Never store a float money value.
- **Store no identifiers for people or accounts.** No card digits, approval codes, member or loyalty numbers, phone numbers, names or prescription numbers in the database or the trip JSON (spec 13.6), even though the receipt prints them.
- **Never guess.** An unknown pack size, weight, brand or organic status is stored as NULL and shown as "?" (not yet known). Ask the user in the post-load review (spec 7.4) instead of filling in a value.
- **`line_items.raw_text` is never edited.** Corrections go to `products`, and a product fix applies to every past trip automatically.
- **Unit prices are derived** in `v_unit_price`, never stored. Compare only rows that share a `base_unit` (g, mL or ct). Weight oz and fl oz are separate units.
- **Keep price, discount and paid separate**, with `paid_cents = price_cents - discount_cents`. Separately printed discount lines (such as Costco's `/1901580 -3.00`) fold into the item's `discount_cents`. Trip-level promotions go in `trip_adjustments` as negative cents. `trips.savings_cents` is stored as printed and never used in checks.
- **Loads are all-or-nothing and idempotent.** A trip and its lines go in one transaction. A re-run must not crash or double-insert. Duplicate trips are detected by the first key the receipt supports: `(store_id, transaction_no)`; store, date and time; or store, date, total and the same lines (spec 7.1 step 2). A trip with no total yet is incomplete, and a new trip that may be part of one is stopped until it is added with `--update`.
- **Integrity-check failures don't abort the load.** They set `needs_review = 1` and print the exact mismatch in dollars.

## Data model in brief

Receipt text maps to a canonical product through `store_items`, which is keyed on `(store_id, item_code)` when the receipt prints a code and on `(store_id, receipt_text)` when it doesn't. Trader Joe's prints no item codes, so `item_code` is NULL and the text is the identifier. This mapping is what makes cross-store comparison possible: `line_items → store_items → products`, with `products.unit_measure → unit_conversions` for normalization. A `store_items.product_id` that is NULL means the item isn't identified yet, and the trip is flagged for review.

Receipt photos are read by Claude in conversation for v1, not by code in this project (spec section 11).
