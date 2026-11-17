# Price Tracker

Price Tracker logs grocery receipts into a SQLite database, with one receipt per shopping trip. You can then compare what you paid for a product over time and across stores, and catch receipt problems such as double charges or price mismatches.

The workflow: you send a photo of a receipt in a Claude conversation (it works from a phone). Claude reads the photo and writes the trip as JSON, then loads it with `load_trip.py`. To answer questions, Claude runs SQL against the database.

## Status

**Design stage. There is no code yet.**

- [`price_tracker_design_spec.md`](price_tracker_design_spec.md) is the authoritative v1 spec. It covers the schema, views, load behavior, edge cases and acceptance criteria.
- [`Price Tracker Feature Ideas.md`](Price%20Tracker%20Feature%20Ideas.md) is a backlog of ideas that are not specified yet.

## Planned usage

The loader doesn't exist yet ([#9](https://github.com/dknupp/price_tracker/issues/9)). Per the spec (section 4), it will load one trip like this:

```bash
python load_trip.py --file trip.json --image receipt.jpg
```

Options include `--db` (defaults to `receipts.db`), `--receipts-dir` (where photos and trip JSON are copied), and `--update TRIP_ID` to replace a trip already in the database. There is no report program. Claude answers questions by querying the database and its views.

## Privacy

- Real receipts, receipt photos and the database are never committed to this repository. Photos and trip JSON go in a receipts folder outside git (spec 7.5).
- The database and the trip JSON store no identifiers for people or accounts: no card digits, approval codes, member or loyalty numbers, phone numbers, names or prescription details (spec 13.6).
- Test fixtures are made-up trips, never real receipts.

## Requirements

Python 3. Third-party packages, if any, will go in a `.venv/` and be listed in `requirements.txt` once the code exists (spec section 3).
