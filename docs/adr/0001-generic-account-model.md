# Generic Account model instead of a fixed type per kind

Cash, brokerage positions, the apartment, and the mortgage are all modeled as one generic `Account` entity classified by a `kind` field, rather than a hardcoded set of separate types (Cash, Stock, Apartment, Mortgage) each with their own table. Kind-specific attributes (ticker, quantity, address, interest rate, …) live in a JSON metadata blob on the Account rather than strict per-kind columns or side-tables.

Chosen because net-worth trackers tend to grow their tracked kinds over time (crypto, vehicles, other loans), and a fixed set forces a schema migration each time; a `kind` enum plus a metadata blob absorbs new kinds and fields with no migration, at the cost of DB-level query strictness on those fields — an acceptable trade for a solo hobby project with no query-performance requirements.

Institutions (eToro, Revolut) are a tag on the Account, not a container/parent entity — Accounts stay flat; a brokerage holding multiple positions is multiple flat Accounts sharing an `institution` tag, not one container Account with nested children.
