# Loader Implementation Design (draft)

Status: draft for [dknupp/price_tracker#9](https://github.com/dknupp/price_tracker/issues/9). Not agreed yet. Revised for spec 13.7 (all photos of a trip are loaded together, and every trip has a total, [#14](https://github.com/dknupp/price_tracker/issues/14)).

This document covers how `load_trip.py` is organized and built. What the loader does is defined in the spec (`price_tracker_design_spec.md`, sections 4, 7.1–7.7, 8, 9 and 10). Where this design needs a behavior change, it is listed in [Spec changes](#9-spec-changes) and goes into the spec when the design is agreed, so the spec stays the one place that defines behavior.

Target: Python 3.11 or later (3.13 is installed), SQLite 3.35 or later.

## 1. Code layout

A small package, with `load_trip.py` as a thin entry point that parses arguments, calls the package, prints the summary and sets the exit code.

```
load_trip.py                 command line only
price_tracker/
  schema.sql                 tables, views, unit_conversions seed rows
  migrations/                002_<name>.sql, 003_<name>.sql, ... (none yet)
  trip.schema.json           JSON Schema for the trip JSON (7.6)
  db.py                      connect, create or migrate the database
  trip_json.py               read and validate a trip JSON file -> Trip objects
  identifiers.py             scan a trip for identifiers that must not be stored (13.6)
  matching.py                duplicate and near-match rules (7.1 step 2)
  checks.py                  integrity checks (7.2)
  catalog.py                 stores, store_items, products: find, create, link (shared with #6)
  files.py                   receipts-folder names and copying (7.5)
  loader.py                  load_trip() and update_trip(): the transaction
  summary.py                 load summary as text or JSON (section 9)
tests/
  fixtures/                  made-up trip JSON files
  test_*.py
requirements.txt             jsonschema, pytest
```

The rule that keeps this testable: **`trip_json`, `identifiers`, `matching`, `checks` and `summary` never touch the database or the file system.** They take plain objects and return plain objects. Only `db`, `catalog`, `files` and `loader` do I/O.

- `Trip`, `Line`, `Proposal` and so on are frozen dataclasses built by `trip_json.py`. Everything after validation works with these, never with raw dicts.
- `loader.py` returns a `LoadResult` dataclass (outcome, trip id, lines loaded, unidentified lines, check failures, review questions, coupons, files written). `summary.py` renders it, and the exit code comes from its outcome. Nothing else prints.

## 2. Schema setup and changes

**New database.** `db.connect(path)` opens the file, turns on `PRAGMA foreign_keys = ON`, and if the database has no tables, runs `schema.sql` (tables, indexes, views, the nine `unit_conversions` rows) and records version 1. This happens on any run, so the first `load_trip.py` creates `receipts.db`.

**Version number.** A one-row table, `schema_version (version INTEGER NOT NULL)`, rather than SQLite's `PRAGMA user_version`, so it ports to PostgreSQL unchanged.

**Later changes: numbered migrations.** Each change is a file `migrations/NNN_<name>.sql`. On connect, if the database is older than the code, the loader:
1. copies `receipts.db` to `receipts.db.v<old>.bak` next to it,
2. applies each newer migration in order, each in its own transaction together with the version bump,
3. prints which migrations ran.

If the database is newer than the code, the loader refuses to run.

**Why not rebuild from the saved trip JSON.** Rebuilding would lose every review answer (brands, pack sizes, which product a line is, `is_fsa`), which live only in `products` and `store_items` (7.5). Reloading the JSON stays available as a repair for a loader bug, but it isn't how the schema changes.

**Transactions.** Connections are opened with `isolation_level=None`, and the loader issues `BEGIN IMMEDIATE`, `COMMIT` and `ROLLBACK` itself. This makes every transaction boundary visible in the code, rather than relying on the `sqlite3` module's implicit transactions, and `IMMEDIATE` takes the write lock before the duplicate check, so two loads at once can't both pass it.

## 3. Validating the trip JSON (7.6)

Two layers, both run before the database is opened. All errors are collected and reported together, not just the first.

**Structure: `jsonschema` with `trip.schema.json`.** Types, required fields (including `trip_date` and `total_cents`, 13.7), `YYYY-MM-DD` and `HH:MM` patterns, enums (`line_type`), and `additionalProperties: false` everywhere. The schema file also documents the format in a form that can be read when writing a trip JSON.

**Rules across fields: plain Python in `trip_json.py`.**
- `format_version` is 1.
- `line_no` values are 1, 2, 3 … with no gaps or repeats.
- `weight` and `weight_unit` are both present or both null; `weight > 0`.
- `weight_unit` and `interpretation` `unit_measure` are units in `unit_conversions`.
- Every `interpretation.lines[].line_no` names a line in `receipt.lines`, at most once.
- Adjustment amounts are negative.
- Every `*_cents` value is a JSON integer. Python reads `1899.0` as a float, and storing it would put a REAL in a money column, so it is rejected, not converted.
- `trip_date` is a real date (not 2026-02-30).

**Unknown fields are rejected.** A misspelled field (`discount_cent`) would otherwise be dropped silently and the line loaded with no discount. An unknown field is also where an identifier would slip in (`card_last4`).

**Error format.** One line per problem, with the JSON path, then exit code 2 and nothing written:

```
invalid trip JSON: trip.json
  receipt.total_cents: required; load the trip once every part of the receipt is photographed (13.7)
  receipt.lines[3].weight_unit: required when weight is given
  receipt.lines[5].price_cents: must be an integer number of cents, got 4.99
  interpretation.lines[0].line_no: 9 is not a receipt line
```

With `--json`, the same errors are a list of `{path, message}` objects.

**Identifier scan (13.6).** Identifiers are meant to be left out when the receipt is read, but that is done by hand, and a missed one would be stored and copied into the receipts folder, so the loader checks too. `identifiers.py` scans every free-text field (`raw_text`, `raw_discount_text`, `unreadable`, coupon descriptions, `interpretation` text, `notes`) for patterns that never belong in grocery text:
- masked card numbers (`XXXXXXXXXXXX1234`, `************1234`),
- email addresses,
- phone numbers,
- runs of 9 or more digits.

`item_code` and `transaction_no` are exempt, since they are digits by nature. A hit rejects the file (exit 2) and names the field and the pattern, not the matched value. Keyword rules such as `MEMBER` or `AUTH` are left out because receipts print "Member Savings" as an ordinary line. If a real receipt line ever trips a pattern, the pattern is narrowed rather than adding an override.

## 4. Load steps

`load_trip()` in order. Steps 1–2 happen before any transaction, so a file rejected there (a missing total, for example) writes nothing to the database or the receipts folder.

1. Read and validate the JSON, then run the identifier scan (section 3).
2. Check that each `--image` file exists and is readable.
3. Open the database and `BEGIN IMMEDIATE`.
4. Find or create the store (7.1 step 1).
5. Load the candidate trips: same store and same date, or same store and same `transaction_no`. Pass them and the new trip to `matching.classify()` (section 5). If it stops the load, `ROLLBACK` and return. Nothing is copied.
6. Insert the trip. Its `trip_id` is known from here on, inside the transaction.
7. Insert lines, adjustments and coupons, finding or creating `store_items` and products through `catalog.py` (7.1 step 4, 7.4).
8. Run `checks.run()` (7.2) and set `needs_review` and `notes`.
9. Copy files (section 6) and set `receipt_image` and `trip_json`.
10. `COMMIT`.

Any exception from step 4 on rolls back and removes the files step 9 created. `update_trip()` follows the same steps, with the differences in 7.7: the trip row is updated rather than inserted, its lines and adjustments are deleted and re-inserted, disputes are re-linked by `line_no` (or the update is refused), coupons are kept and new ones added, and the trip being updated is left out of the candidates.

## 5. Duplicate and near-match detection

`matching.classify(new, candidates, *, force)` takes a `TripFacts` for the new trip and for each candidate (store, date, time, transaction number, total, and the ordered lines as `(code or text, price_cents)`) and returns one of:

| Result | When | `--force` loads it? |
|---|---|---|
| `NoMatch` | nothing below applies | — |
| `AlreadyLogged(trip_id, key=1)` | same transaction number | never |
| `AlreadyLogged(trip_id, key=2 or 3)` | same date and time; or same date, total and lines | yes |
| `PossiblySameReceipt(trip_id)` | near match: equal totals, or at least 80% of lines matching | yes |

It applies the rules in the spec's order: the first key the receipt supports, then near matches (with their two exclusions: differing transaction numbers, or times more than 10 minutes apart). Every trip has a total (13.7), so key 3 and the equal-totals test compare two integers and never meet a NULL. `TripFacts.total_cents` is an `int`, not an optional, so the type says so too. The 80% line count is its own function, `line_match_ratio(a, b)`, since "each existing line can match only once" is the part most likely to be got wrong.

Because the function needs no database, most of the criteria in section 10 about duplicates are tested as plain function calls, with a few end-to-end tests to check that the loader wires it up.

## 6. Files and the transaction (7.5)

**Order: copy before commit, remove on failure.** The copies are made inside the transaction (step 9), after every database write and before `COMMIT`. If anything fails, the transaction is rolled back and every file this run created is deleted. A trip that is stopped at step 5, or rejected at step 1, copies nothing.

The one gap is a crash (power loss, killed process) between the copy and the commit. That leaves copies with no row pointing at them. This is the safer way round: committing first and copying after would leave rows pointing at files that don't exist. An orphan copy is harmless, and the next load that wants the same name stops with an error naming the file, rather than overwriting it.

**Names.** `files.base_name(trip, trip_id)` returns `<year>/<date>_<chain slug>_<key>`, where the key is the transaction number, else the time without the colon (`1432`), else `t<trip_id>`. The chain slug is lowercase with apostrophes removed and other non-letters as hyphens (`Trader Joe's` → `trader-joes`). The `t<trip_id>` case is only for a receipt that prints neither a transaction number nor a time. The trip id is available because the trip row is inserted (step 6) before the files are copied (step 9). The loader never overwrites an existing file. An existing file with the target name stops the load.

**All photos in one load.** Every `--image` of a load is copied in the order given: the first takes the base name and the rest get `_2`, `_3` (7.5, 13.7).

**Update.** New photos are added with the next free suffix. The new JSON is copied to `<base>.json.new` before the commit and moved over the old JSON with `os.replace` after it, so a failed update leaves the old JSON in place, as 7.7 requires. If the move fails after a successful commit, the summary warns that the saved JSON is out of date.

**The base name is fixed at the first load.** If an update corrects a misread transaction number or time, the files keep the name they were first given. Renaming would mean moving files the database already points to, for no gain, and the name is only for finding the files by eye.

## 7. Load summary and exit codes (section 9)

Plain text by default, for a person running it by hand. `--json` prints the same `LoadResult` as one JSON object, for the conversation that runs the loader, so nothing has to parse the wording. Both come from the same object, so they can't disagree.

```json
{
  "outcome": "loaded",
  "trip_id": 57,
  "store": "Costco Springfield #0042",
  "trip_date": "2026-10-30",
  "total_cents": 2816,
  "needs_review": true,
  "lines_loaded": 2,
  "lines_unidentified": [],
  "checks": [{"check": "subtotal", "ok": false,
              "message": "lines sum to $26.98, receipt subtotal is $27.98"}],
  "review_questions": [{"product_id": 12, "line_no": 2, "field": "pack_count",
                        "question": "Pack size of Kirkland Signature Ibuprofen 200 mg?"}],
  "coupons": [{"description": "$5 off $25, excludes alcohol", "valid_to": "2026-11-30"}],
  "files": ["2026/2026-10-30_costco_0042-04-2231.json", "2026/2026-10-30_costco_0042-04-2231.jpg"],
  "matched_trip_id": null,
  "errors": []
}
```

| Exit code | Outcome |
|---|---|
| 0 | `loaded` or `updated`, with or without `needs_review` |
| 2 | `invalid_input`: bad arguments, bad JSON (including a missing total), identifier found, image missing |
| 3 | `already_logged` |
| 4 | `possibly_same_receipt` |
| 5 | `update_refused`: trip not found, matches another trip, or a disputed line is gone |
| 1 | unexpected error (a bug); nothing is written |

`already_logged` gets its own non-zero code, not 0, so a script can tell that nothing was loaded. It prints no traceback and no error, which is what the first criterion in section 10 asks for. Code 2 matches what `argparse` already uses for bad arguments.

## 8. Tests

- **pytest**, with `tmp_path` giving each test its own database and receipts folder. A shared fixture returns a `load(json_name, *flags)` helper that calls `load_trip()` directly. A few tests run `load_trip.py` as a subprocess to check exit codes and `--json`.
- **Fixtures**: made-up trip JSON in `tests/fixtures/`, named after the criterion they serve (`subtotal_with_promotion.json`, `near_match_a.json` and `near_match_b.json`, `no_total.json`, `update_before.json` and `update_after.json`). Values are invented, as in the spec's example.
- **One test per criterion in section 10**, named after it, plus unit tests for `matching`, `checks`, `trip_json` and `identifiers`. The test for "a trip JSON with no total is rejected" checks the database and the receipts folder are both left as they were, and that it passes `--image` so a copy would show up if one were made.
- **Fixtures are checked too**: a test runs the identifier scan over every fixture, so a real receipt detail can't be committed by accident.
- **"A load that fails partway"** is tested by making `catalog` raise after the trip row and some lines are inserted (with `monkeypatch`), then checking that the database and the receipts folder are as they were. The same is done for a failure during the file copy.
- **"No money value is stored as a float"** is checked with `typeof()` on every `*_cents` column after loading all fixtures.

## 9. Spec changes

When this design is agreed, these go into the spec:

1. **`schema_version` table** (section 6): ten tables, not nine, and a short note on migrations in section 3.
2. **Exit codes and `--json`** (sections 4 and 9).
3. **Unknown fields and float cents are rejected** (7.6), and the identifier scan (7.6 and 13.6), as a new design decision record.
4. **The base name is fixed at the first load** and kept by `--update`, even when the update corrects the transaction number or time the name came from (7.5, 7.7).
5. **More than one photo per trip** (7.5): `trips.receipt_image` holds one path, but a trip can have `_2`, `_3` photos, and with 13.7 every long receipt does. Either the column holds the first photo and the rest are found by name, or a `trip_photos (trip_id, path)` table replaces it. Suggested: the table, so the database lists every photo, which keeps the "every trip is traceable to its photos" rule checkable with SQL.
6. **Updates that are near matches for another trip** (7.7 step 5): the spec refuses an update that matches another trip on a key, but doesn't say what happens if it is a near match for one. Suggested: refuse it the same way (exit 5), since `--force` can't be combined with `--update`. The user then corrects or removes the other trip first.
7. **Store details on later trips** (7.1 step 1): when an existing store has no branch or address and a new receipt prints one, fill in the NULL. Never overwrite a value already stored.

## 10. Overlap with #6 (review answers)

`catalog.py` holds every write to `stores`, `store_items` and `products`, so the loader and the review command share them:

- `find_or_create_store`, `find_or_create_store_item`, matching NULL to NULL with `IS` as the spec requires.
- `find_or_create_product`, which reuses an existing product that matches on the `ux_products` columns instead of failing on the unique index.
- `link_store_item(store_item_id, product_id)`, which also sets `products.is_fsa = 1` if any line for that item is FSA-marked (7.4). This is the only place `is_fsa` is set to 1.

**A point for #6.** To clear `needs_review`, the review command has to know why the trip was flagged. Most reasons can be worked out again from the rows (unidentified lines, unknown product fields, the subtotal and total checks, `unreadable` notes), but a printed paid amount that disagreed with the computed one can't, since `printed_paid_cents` isn't stored. So the reasons should be computed by one function both commands call, and the printed-paid mismatch needs to be stored somewhere, for example a `trip_review_reasons` table or a note on the line. That decision belongs to #6, but it affects what the loader writes.

## 11. Not decided here

- The shape of review questions beyond what the JSON example shows. They follow 7.3 and the `interpretation` rules in 7.6.
- The instructions for running the loader in conversation (`--json`, what to do for each exit code). They go into `CLAUDE.md` once the loader exists.
