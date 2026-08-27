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
A point-in-time record of an Account's value (amount, currency, date). Created event-based, whenever the value is updated — not on a fixed schedule. The series of snapshots is what makes net worth chartable over time.
_Avoid_: Valuation, History entry

**Transaction**:
A normalized row (date, amount, description) imported from a CSV statement into an Account. Distinct from a Value Snapshot: a snapshot is a value at a moment, a transaction is a discrete movement.

**External Flow**:
A Transaction tagged as money entering or leaving an Account from outside it (deposit, withdrawal, transfer, mortgage payment) — as opposed to the Account's value moving on its own (market appreciation, interest).

**Contribution** / **Growth**:
The two components of change between two Value Snapshots. Contribution is the sum of External Flow transactions in that period; Growth is the remainder (value delta minus contributions). Distinguishes "net worth went up because I added money" from "net worth went up because it appreciated."
