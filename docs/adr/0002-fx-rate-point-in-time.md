# Point-in-time FX rates, cached locally, stored on the snapshot

Multi-currency Accounts (eToro often USD, apartment/mortgage likely EUR) need converting to a single Base Currency to aggregate net worth. FX Rates are fetched from a free external API (e.g. exchangerate.host / Frankfurter — no key required) for the specific date of a Value Snapshot, and cached locally in an `fx_rates` table so the app works offline after first fetch.

The rate used is stored directly on the Value Snapshot row, not just the converted amount, and not left to be joined against a rates table at read time. The Account's native-currency amount stays the source of truth; the stored rate makes the base-currency figure reproducible and auditable without re-deriving it from a rates table that may have since changed or been re-fetched.

Point-in-time was chosen over live/current-rate-applied-retroactively because a solo net-worth tracker's core value is the historical chart — a past net-worth figure silently shifting every time FX moves would undermine trust in the history. This trades simplicity (a live rate needs no per-snapshot storage) for stability of historical figures, judged worth it since the whole point of Value Snapshots is a stable series over time.
