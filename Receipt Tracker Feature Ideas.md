# Receipt Tracker: Feature Ideas

A running list of features to build into Receipt Tracker, the grocery trip and price tracking tool. These are brainstorm-stage ideas, not fully specified.

## 1. Post-trip receipt review

After a receipt is logged, the tool runs a review pass and asks the user about anything missing or unclear.

- **Missing brand names:** If an item has an obviously missing brand (for example, "Chili with beans" or "Canned chicken"), ask the user for the brand and fill it in. Also confirm guessed expansions of abbreviations, like "PE" for Purely Elizabeth.
- **Multi-packs:** Ask whether any items are multi-packs (cans, rolls, bottles, packages) so a true per-unit price can be computed. Costco items especially need this because the receipt doesn't print pack sizes.
- **N/A option:** Let the user enter "N/A" in Unit Price when a per-unit price doesn't apply or isn't known, so the field doesn't have to be guessed or left ambiguous. In the database spec, an unknown value is stored as empty and shown as "N/A", never guessed.

## 2. In-store duplicate check on the receipt

Right at checkout, the user photographs the receipt and the tool immediately scans it for duplicate line items.

- **Why:** On the October 30, 2026 Costco trip, the Kirkland Signature albacore tuna was charged twice, but only one 8-pack was bought. Catching that while still in the store means the user can get a refund at the service desk instead of going back later.
- **Flag same-item repeats:** Highlight lines with the same item code or name appearing more than once, and ask whether the quantity is really right.
- **Design sketch (removed from the v1 spec):** a `line_items.suspected_duplicate` flag (0/1), set on load when another line in the same trip has the same item code (or the same text when there is no code) and the same price. Flagged lines appear in the load summary, and the user confirms one (creating a `disputes` row) or clears it. It was dropped from v1 because buying several of the same item is common and true double charges are rare. Open issues carried over: some stores print one line per unit, so repeats are normal there; and it's unclear whether to flag both lines or only the later one. A `--duplicates` report listed flagged lines.

## 3. Shelf price versus receipt price comparison

While shopping, the user uploads photos of shelf price tags. After checkout, the tool compares the advertised shelf prices against what was actually charged on the receipt.

- **Match photos to receipt lines:** Pair each shelf photo with its receipt item.
- **Flag mismatches:** Report any item charged at a different price than the shelf tag, including missed sale prices or discounts, so the user can ask for a correction before leaving.
- **Storage sketch (removed from the v1 spec):** a `shelf_prices` table with store, product (NULL until matched), shelf price in cents, photo path, and when it was seen. Open question carried over: should shelf photos be matched to products by hand, or suggested by the tool and confirmed?

## 4. FSA item flagging

Whenever a receipt is logged, the tool always flags any FSA-eligible items, so none get missed at tax time or when submitting claims.

- **Flag on every receipt:** Mark FSA-eligible lines automatically, using the receipt's own markers where they exist (for example, the Costco receipt on October 30, 2026 listed the Kirkland Signature ibuprofen as FSA-eligible, with a separate FSA total of $12.17 including tax).
- **Ask about likely candidates:** If a receipt doesn't mark FSA items but a line looks like one (medicine, first aid, sunscreen, and similar), ask the user whether it should be flagged.
- **Show an FSA total per trip:** Add the FSA-eligible amount, with tax, to each trip's footer so it's easy to claim.
- **Already in v1 (recorded only):** `line_items.fsa_marked` (the receipt's per-line mark), `trips.fsa_total_cents` (a printed FSA total) and `products.is_fsa` (NULL unknown, 1 once any receipt marks the product, 0 only when the user says so). Nothing in v1 reads them.
- **Still to design:** a report or computed FSA total, and asking about unmarked items that look eligible. The hard part is each line's share of tax. Approach discussed: store no tax rate (receipts print the amount, not the rate, and one trip can have more than one rate); split the printed `tax_cents` across taxable lines in proportion to the amount paid, with rounding that makes the shares add up exactly. A trip with any line of unknown taxability can't be split, and receipts that print two tax lines aren't handled. A printed `fsa_total_cents`, when present, is the check for any computed total.

## 5. Shopping list price optimizer

The user supplies an intended shopping list, and the tool uses the history of past trips to work out the cheapest place to buy it.

- **Match list items to history:** Map each item on the list to products in past receipts, using brand, pack size, and unit size so the comparison is fair (for example, tuna per can, not per package).
- **Compare stores:** For each store, price the list using the most recent price paid, and show the total per store. Flag items the store has never been seen to carry, since the history can't price them there.
- **Recommend a plan:** Show the cheapest single store, and optionally a split plan (which items to buy where), with the savings from splitting weighed against the extra trip.
- **Account for pack size and bulk:** Warn when the cheapest per-unit price needs a bulk purchase, such as Costco 8-packs, so the user isn't surprised by a bigger bill.
- **Compare across units:** The database spec converts each size to a common base (grams, millilitres, or count), so a 1 lb butter and four 4 oz sticks, or a single can and an 8-pack, can be compared per ounce, per can, or per pound.
- **Account for discounts:** Show whether a price was a markdown (such as the Costco chili at $4.80 off) so the user knows it may not repeat.
- **Show how stale prices are:** Display the date of the price used, and flag old ones.

## 6. Product type and grade matching

When comparing prices, the tool should only compare products that are really the same kind of thing, not just the same category.

- **Record the grade or type:** Capture attributes beyond brand and size, such as extra virgin vs. refined olive oil, whole vs. low-fat milk, decaf vs. regular coffee, or organic vs. conventional. Organic is now its own field on products in the database spec (yes, no, or unknown), left unknown until the user says; the other attributes still need a design. The Oct 30, 2026 Costco Kirkland refined olive oil and the Safeway Bertolli extra virgin were easy to confuse.
- **Ask during review:** In the post-trip review, prompt for the grade or type when a product is in a category where it matters.
- **Compare like with like:** In price comparisons and the shopping list optimizer, group matching products together, and clearly label comparisons across different grades (for example, "refined vs. extra virgin, not directly comparable").
- **Allow substitutions on purpose:** Let the user mark which types they consider interchangeable, so a cheaper acceptable substitute can still be suggested.

## 7. Item number lookup from past trips

Every logged item keeps the store's own item number, so future receipts can be filled in automatically from what was learned on earlier trips.

- **Remember the item number:** Store the item number printed on the receipt, tied to that store, along with the brand, size, pack count, and grade the user confirmed. Costco and Safeway print item numbers. Trader Joe's prints none, so those items would match on receipt text instead.
- **Auto-fill on later trips:** When a new receipt has an item number the tool has seen before, fill in the brand, size, pack count, and unit price right away, even if the receipt itself only prints an abbreviation. A Grocery Outlet receipt with a known item number would not need any follow-up questions.
- **Ask only about new items:** In the post-trip review, ask questions only for item numbers it has never seen, so the extra work happens up front and shrinks over time.
- **Show what was assumed:** Mark auto-filled details as carried over from a past trip, so the user can spot a changed product, such as a new pack size under the same item number.
- **Handle changes:** If the price or package changes under the same item number, flag it and ask whether the size or pack count also changed.
- **Fall back to receipt text:** For stores without item numbers, match on the exact receipt text for the same store, and ask for confirmation the first time a match is used.
- **Safeguards for text matching:** Text is a weaker identifier than an item number, so it needs extra care. Text can drift ("Brussels sprouts, organic" vs. "Brussels Sprouts 1lb"), one generic string can cover two products ("Granola"), and a wrong match silently fills in a wrong size. To limit that: match only on exact text within the same store, never across stores; ask for confirmation the first time a text match is used and remember the answer; mark auto-filled details as coming from a text match so they can be told apart from item-number matches; and let the user split a text match when it turns out to cover two products. The exact text kept on every receipt line makes any match easy to re-check against the receipt.

## 8. Recipe suggestions from recent purchases

The tool suggests recipes based on what was bought on recent trips. This is not pantry or inventory tracking. It simply assumes that recent purchases are still on hand.

- **Use recent trips only:** Draw on items from the last trip or two, and let the user say "I already used the salmon" to remove things.
- **Prefer perishables first:** Rank fresh items like herbs, fish, and produce higher, since they spoil soonest.
- **Anchor on staples:** Build ideas around items that show up often in the history, such as canned tuna, chili, spaghetti, and salad kits.
- **Show what's missing:** Tell the user which ingredients they don't have, such as "everything but a lemon," and offer to add them to a shopping list. This connects to the shopping list price optimizer.
- **Respect preferences:** Allow the user to mark dietary needs or dislikes so suggestions stay relevant.

## 9. Distribution: letting users bring their own Claude access

If this becomes a product other people run themselves, the developer can't supply Claude credits. This section captures the options for who pays for Claude.

- **Bring your own API key (supported route):** The user gets a key from Anthropic's developer console and pastes it into the app. Usage bills to their own account. This works for both a website and a phone app.
- **Signing in with a regular Claude account:** As far as is known, third-party apps can't use a user's claude.ai login or subscription to power the app. The consumer plan and the developer API are separate, with separate billing. Confirm against Anthropic's current terms before relying on this, since policies change.
- **Website considerations:** Avoid routing the key through the developer's own server. Keep it in the user's browser and call the API from there, or store it encrypted if it must be kept.
- **Phone app considerations:** Keep the key in the device's secure storage, such as the iOS Keychain or Android Keystore.
- **Cost transparency:** Tell users that usage is pay-per-use, and show a rough cost per receipt scan.
- **MCP route:** Put the data behind an MCP server, and let users connect it to their own Claude app. They use Claude under their own subscription, and the developer never handles credentials.

## 10. Scaling notes (thought experiment: about 1,000 users)

This is a study project, but it helps to imagine real-world use. These are rough estimates, not measurements.

- **Database storage is small:** A trip is about 25 lines and a row is a few hundred bytes. At roughly four trips per user per month, 1,000 users add about 100,000 rows a month, or tens of megabytes. Postgres is not the likely bottleneck.
- **Receipt photos are the real storage cost:** At a few hundred kilobytes each, they add up to around a gigabyte a month. Keep them in object storage such as S3, and store only the path in the database.
- **Database CPU is modest:** Queries are small joins over one user's own history. A single modest Postgres instance with an index on user and date should handle thousands of users.
- **Multi-user changes:** Add a user id column to the tables so each person sees only their own data.
- **Likely bottlenecks:** Claude calls for reading receipts, which are slow and rate limited, so queue them in the background; photo upload and processing bandwidth; and the shared product catalog, if item numbers are pooled across users.
- **Shared catalog caution:** Pooling item numbers between users is valuable but needs rules for wrong or conflicting entries.
- **Bring-your-own-key effect:** If users supply their own API keys, the Claude cost and rate limits are theirs, so one person's burst can't hurt anyone else.

## 11. Model-agnostic design: let users choose their own LLM

The tool should not be tied to one AI provider. Users could bring whichever model they already use, such as Claude, ChatGPT, or Gemini, through their own API account.

- **Data layer is naturally neutral:** MCP is an open protocol, so a server that exposes simple tools (add trip, update product size, find the cheapest tuna) can be used by any model that supports tool calling.
- **App-side model calls need an interface:** For tasks the app runs itself, such as reading a receipt photo, define one small interface (photo in, structured line items out) and write a thin adapter for each provider. Libraries that wrap several providers behind one interface can save most of the work.
- **Expect differences between models:** Receipt image reading accuracy varies, especially on crumpled or cut-off receipts. Tool calling formats and quirks differ. Cost and rate limits stay on each user's own account.
- **Put validation in code, not in the model:** The integrity checks in the database spec, such as line totals matching the receipt subtotal and the duplicate-line check, catch a weaker model's mistakes no matter which provider is used.
- **Show which model read each receipt:** Record the provider and model used on each trip, so errors can be traced and re-checked if a better model is chosen later.

## 12. Expiration dates for food items

Add an expiration date column for food items, so each purchase can carry a use-by or best-by date.

- **Capture the printed date:** Record the expiration date read from the package for each food item, either when it's logged or during the post-trip review, since receipts don't print it. This goes in its own "printed expiration" field and stays empty unless someone actually entered it.
- **Seed an estimate by item type:** Based on what the item is, pre-fill a separate "estimated expiration" date from a typical shelf life, such as a few days for fresh fish and herbs, a couple of weeks for milk, or years for canned goods. The user can confirm or change it. Keeping the estimate in its own field means it is never mistaken for the date printed on the package, and entering a printed date later doesn't erase the estimate.
- **Example question:** "Which items have I purchased recently that might go bad soon?" Claude would estimate roughly how long each perishable item lasts and fill in the estimate when the trip is loaded.
- **Open issue from the spec review:** the typical shelf life per item type needs a source, such as a table by category or Claude's own estimate at load time. An estimate stored at load time also goes stale if the item's category is later corrected.
- **Make it optional:** Many items (household goods, deposits, fees) have no expiration date, so the field should be allowed to stay empty, not guessed.
- **Connect to other features:** The date could feed the recipe suggestions, so items closest to expiring are used first, and the shopping list, so the tool doesn't suggest buying something that's already stocked and still good.
- **Database change:** Two new nullable date columns on the line items table, `expiration_printed` and `expiration_estimated`, since the dates belong to a specific purchase, not to the product in general. A view returns the printed date when there is one and falls back to the estimate, and also says which of the two it used. That view was drafted in the v1 database spec and then moved back here: this is a v2 feature.

## 13. Markdown column and regular price tracking

Show the markdown as its own visible column, so it's clear how much of a price was a sale and what the regular price was.

- **Show the regular price:** For each line, display the regular (pre-discount) price, the markdown amount, and what was paid. Example: the Land O'Lakes unsalted butter at Safeway on Sept 20, 2026 cost $3.99, but that was a $3.00 markdown from the regular $6.99.
- **Add a markdown percent:** Show the discount as a percent of the regular price, such as 43% off, so deals are easy to compare across items.
- **Compare on both prices:** In price comparisons and the shopping list optimizer, show the sale price and the regular price side by side, so a one-time markdown isn't mistaken for the normal price. The Costco chili at $2.33 a can was $3.13 without its $4.80 markdown.
- **Flag how often an item goes on sale:** Over time, note which items are usually bought at a markdown and which are not, to guess whether a sale price is likely to come back.
- **Database note:** The line items table already stores `price_c` (regular), `discount_c` (markdown), and `paid_c`, so this is mostly a reporting and display change, not a new stored column. The database spec adds a markdown view that lists discounted lines with the regular price, markdown amount, amount paid, and percent off.

## 14. Verbatim receipt transcript

Save a transcript of every printed line of the receipt, in order and exactly as printed, alongside the structured trip data.

- **Why:** The trip JSON records only the fields designed so far. A transcript keeps everything else the receipt prints (register and lane numbers, the "items sold" count, survey codes), so a field added later can be pulled from saved transcripts instead of reopening hundreds of photos. It is also searchable and much faster to reread than a photo.
- **Where it goes:** A `receipt.transcript` field in the trip JSON (a list of printed lines), so it is saved in the receipts folder next to the photo and reloaded by `--update`. It stays out of the database until something needs to query it.
- **Rules:** Copy each line verbatim, keeping the spacing so the receipt's fixed-width columns stay aligned. Nothing is interpreted or corrected. Lines holding identifiers for people or accounts (card digits, member numbers, names, prescription details) are replaced with `[omitted]`, as in spec 13.6. Unreadable parts are marked, not guessed.
- **Limits:** A transcript is Claude's reading, not the evidence. A misread in it can only be caught by checking the photo, and barcodes, faded sections and handwriting don't come through. So while the photo is kept, the photo is the record and the transcript is a convenience.
- **Becomes important if photos are deleted:** If photos are ever deleted after a retention period (spec section 12), the transcript is the only remaining record of everything the receipt printed.

## Open questions

- Whether the "N/A" shown for an unknown Unit Price should look different from a cell that is simply blank, now that the database stores unknown values as empty.
- Whether the shelf-photo flow should run through the voice assistant, the text chat, or both.
