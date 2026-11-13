# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project status

Receipt Tracker is a tool for logging grocery receipts (one receipt = one trip) into SQLite to compare prices over time and across stores, and to catch receipt errors such as double charges. **There is no code yet.** The repo holds two design documents, and the code still has to be built from them:

- `receipt_tracker_db_spec.md` is the authoritative v1 spec: schema, views, load behavior, edge cases and acceptance criteria. Build to it, and update it when a design decision changes.
- `Receipt Tracker Feature Ideas.md` is a brainstorm-stage backlog. Ideas there are not specified, so don't build them unless asked.

## Planned stack and entry points (from the spec)

Python 3, standard library only: `sqlite3`, `argparse`, `csv`, `json`. **Add no third-party dependencies.** The schema is plain SQL so it can port to PostgreSQL later, so avoid SQLite-only features where a portable form exists.

- `load_trip.py --file trip.json [--db receipts.db] [--image photo.jpg]` loads one trip from JSON shaped like the `trips` and `line_items` tables.
- `import_doc.py --history trips.csv` bulk-loads many trips from a CSV (one row per line item). It seeds the database from the existing "Grocery Price Tracker" doc after a one-time conversion of that doc to CSV.
- `report.py --unit-prices <product> | --frequent [N] | --fsa [--year YYYY] | --duplicates` runs reports, which can be exported as CSV or JSON.

No build, lint or test tooling exists yet. The acceptance criteria in spec section 10 are the test targets, and they use real receipts as fixtures (the Oct 30, 2026 Costco duplicate tuna, the Oct 27 Grocery Outlet $5-off-$25, and the Costco ibuprofen FSA total of $12.17).

## Invariants that must hold everywhere

These come from past errors on the project and are checked by the acceptance criteria:

- **Money is integer cents** (`*_c` columns). Never store a float money value.
- **Never guess.** An unknown pack size, weight, brand, organic status or printed expiration date is stored as NULL and shown as "?" (not yet known). Ask the user in the post-load review (spec 7.4) instead of filling in a value.
- **`line_items.raw_text` is never edited.** Corrections go to `products`, and a product fix applies to every past trip automatically.
- **Unit prices are derived** in `v_unit_price`, never stored. Compare only rows that share a `base_unit` (g, mL or ct). Weight oz and fl oz are separate units.
- **Keep price, discount and paid separate**, with `paid_c = price_c - discount_c`. Separately printed discount lines (such as Costco's `/1901580 -3.00`) fold into the item's `discount_c`. Trip-level promotions go in `trip_adjustments`.
- **Loads are all-or-nothing and idempotent.** A trip and its lines go in one transaction. A re-run must not crash or double-insert. Duplicate trips are detected by `(store_id, transaction_no)`, or by store, date and total when there is no transaction number.
- **Integrity-check failures don't abort the load.** They set `needs_review = 1` and print the exact mismatch in dollars.

## Data model in brief

Receipt text maps to a canonical product through `store_items`, which is keyed on `(store_id, item_code, receipt_text)`. Trader Joe's prints no item codes, so `item_code` is NULL and the text is the identifier. This mapping is what makes cross-store comparison possible: `line_items → store_items → products`, with `products.unit_measure → unit_conversions` for normalization. A `store_items.product_id` that is NULL means the item isn't identified yet, and the trip is flagged for review.

Receipt photos are read by Claude in conversation for v1, not by code in this project (spec section 11).
