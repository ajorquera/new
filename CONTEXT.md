# Net Worth

A personal net-worth tracking webapp: aggregates assets and liabilities across cash, brokerage positions, property, and debt, and understands spending from manually-imported statements.

## Language

**Account**:
Anything trackable that contributes to net worth — a cash balance, a single stock position, the apartment, the mortgage. One generic entity, classified by `kind` (cash, stock_position, property, mortgage, …), not a separate table per type. Asset/Liability is derived from `kind`, not stored independently. Carries its own currency and a kind-specific metadata blob (ticker, quantity, address, interest rate, …).
_Avoid_: Asset, Holding (as a standalone concept — a holding is an Account of kind stock_position), Position (same)

**Kind**:
The classifier on an Account that determines its Asset/Liability polarity and which metadata fields apply to it (e.g. stock_position implies ticker/quantity; mortgage implies interest_rate).

**Institution**:
The brokerage or bank an Account is held at (e.g. eToro, Revolut). A tag on the Account, not a container or parent entity — Accounts are flat.

**Value Snapshot**:
A point-in-time record of an Account's value (amount, currency, date). Created event-based, whenever the value is updated — not on a fixed schedule. The series of snapshots is what makes net worth chartable over time. When the Account's currency differs from the Base Currency, the snapshot also carries the FX Rate used to convert it, so the base-currency figure is fixed to that date rather than drifting with today's rate.
_Avoid_: Valuation, History entry

**Base Currency**:
The single currency net worth is aggregated and reported in (EUR). Every Account keeps its own native currency; conversion to Base Currency happens only when combining Accounts into a net-worth figure.

**FX Rate**:
An exchange rate between an Account's currency and the Base Currency, fetched from a free external rate API and cached locally. Looked up for the specific date of a Value Snapshot (point-in-time), not a live/current rate — so a past net-worth figure doesn't change as today's rates move.
_Avoid_: Conversion rate, exchange rate (as a live-only concept — an FX Rate here is always tied to a date)

**Transaction**:
A normalized row (date, amount, currency, description, optional fee) imported from a source statement file into an Account via an Import Schema. Distinct from a Value Snapshot: a snapshot is a value at a moment, a transaction is a discrete movement. Preserves its original imported row and a key derived from it, so re-running an Import Run over an overlapping statement never creates duplicates. Belongs to exactly one Import Run. A Transaction can carry multiple Categories; the full amount counts toward each tagged Category's spend total (not split across them).

**Import Schema**:
A reusable, user-selectable definition of how one specific file shape (e.g. "eToro Account Activity sheet", "Revolut Personal CSV", one particular bank's report layout) translates into Transactions: where the real table starts (sheet, header row), how each Transaction field is populated (from a column, or a fixed value — some sources never state a field explicitly, e.g. eToro's Account Summary sheet has no currency column because it's implicitly USD), how ambiguous values are parsed (explicit date format and decimal style, never auto-detected), which rows to skip (e.g. a bank's non-completed transactions), and which raw values mean a Transaction is an External Flow. One mechanism serves both the schemas shipped built-in for known institutions and ones a user authors from scratch for an unrecognized source — authoring is the same regardless of who created it. A Schema describes a file shape, not an Account: the same eToro Schema applies every time you import from that source, and the target Account is chosen separately at each import.
_Avoid_: Mapping, Importer (the Schema is the definition; importing is the act of applying it)

**Import Run**:
A record of one import: which Import Schema was applied, the source file, when, and how many Transactions resulted. Every Transaction it produced is linked to it, which is what makes reverting a bad import ("delete everything this run created") a single operation instead of a manual hunt.

**Category**:
A user-defined, flat spending label (Groceries, Rent, Dining, …) tagged onto a Transaction. Seeded with a starter set, freely added to/edited/deleted by the user. No hierarchy — a Transaction that spans two concepts (e.g. a Costco run) just carries both Categories rather than nesting one under the other.
_Avoid_: Kind (that's the Account classifier, a different axis — a Transaction's Category has nothing to do with its Account's Kind)

**Category Rule**:
A user-defined rule that auto-tags a Transaction with one Category when the Transaction's normalized (lowercased, trimmed) description contains the rule's match string, optionally scoped to one Account. Every Rule whose condition matches a given Transaction applies — there's no precedence between Rules; each contributing Rule adds its Category alongside any others (rule-sourced or manually tagged). Each Transaction-Category tag records which Rule produced it (or that it was added manually), so editing or deleting a Rule can retroactively add/remove only the tags *that Rule* produced, leaving manually-tagged and other-Rules'-tagged Categories untouched.
_Avoid_: Category (a Rule produces Category tags, it isn't one)

**External Flow**:
A Transaction tagged as money entering or leaving an Account from outside it (deposit, withdrawal, transfer, mortgage payment) — as opposed to the Account's value moving on its own (market appreciation, interest). Set at import time by the Import Schema's classification rule, not by Category Rules: it's a structural fact about the Transaction, a different axis from user-meaning spending Categories.

**Contribution** / **Growth**:
The two components of change between two Value Snapshots. Contribution is the sum of External Flow transactions in that period; Growth is the remainder (value delta minus contributions). Distinguishes "net worth went up because I added money" from "net worth went up because it appreciated."
