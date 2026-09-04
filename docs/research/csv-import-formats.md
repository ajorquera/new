# CSV import formats research

Research for issue #40 ("Import CSV formats: eToro/Revolut/bank statements (research)"), a child of wayfinder map issue #1, feeding the future import-mapping design ticket #41. This is **pure fact-gathering**, not a design proposal — no import schema, column-mapping scheme, or normalization approach is proposed below; the goal is only to lay out what real exports from eToro, Revolut, and typical bank statements actually look like, so #41 has concrete facts to design against.

Scope note on source confidence: eToro's and Revolut's own help centers are both JavaScript-rendered Salesforce/Intercom-style knowledge bases that return an empty shell to a plain HTTP fetch (verified directly — see below) and are also thin on exact column-level specs even when read via a rendered browser. Where the official help center only confirms *that* a feature exists (e.g. "Account Statement has a Closed Positions tab") but not its exact column headers, this doc supplements with well-corroborated **secondary/community sources** — open-source parsers that read real exported files, personal-finance-tool docs, and finance-industry write-ups. Every secondary source is flagged inline as such, with a lower-confidence caveat.

---

## 1. eToro

### Export format

eToro's statement is called the **Account Statement**, downloaded from Settings/Account. It is Excel-based, not a flat CSV:

> Export the report in Excel format (.xlsx required). ([help.waltio.com/en/articles/3748833-etoro-file](https://help.waltio.com/en/articles/3748833-etoro-file), secondary/community source — Waltio's own integration-help article for importing eToro data)

The current statement UI is organized into distinct sections/tabs — not a single flat sheet — corroborated by a search-indexed snippet of eToro's own help center and independently by an eToro-focused fintech news write-up:

> eToro's account statement includes several tabs: Summary, Closed Positions, Account Activity (previously called "Transactions Report"), and Financial Summary ([fxnewsgroup.com/forex-news/retail-forex/etoro-enhances-account-statement-feature](https://fxnewsgroup.com/forex-news/retail-forex/etoro-enhances-account-statement-feature/), secondary source — trade-press article describing an eToro product update)

> The "Account Activity" section generates a list of all the transactions made in the account within the selected time frame, including deposits, withdrawals, positions opened and closed, and fees. The "Closed positions" section displays all manually closed trades, as well as closed copied trades, including ISIN codes, instrument types and names. ([help.etoro.com/My-Account/70325632/What-is-the-eToro-Account-Statement.htm](https://help.etoro.com/My-Account/70325632/What-is-the-eToro-Account-Statement.htm) — text as indexed by a search engine; the live page itself is a client-side-rendered Salesforce community page that returned an empty HTML shell when fetched directly during this research, so this wording is taken from a search-result snippet of the same URL rather than a directly-rendered page)

When exported to Excel, the underlying workbook is multi-sheet. A community-maintained conversion tool that parses real downloaded files documents at least two named worksheets and their exact column headers (see below); a separate community tool references an "Account Summary" and "Financial Summary" sheet pairing, suggesting the exact sheet set/naming has changed across eToro statement format revisions over time (consistent with eToro's own "new account statement" product-update history):

> eToro's exported account statement is a multi-sheet Excel workbook. ([github.com/weirdapps/etoro_statement](https://github.com/weirdapps/etoro_statement), secondary source — README of an open-source eToro-statement parser)

The eToro statement also has a personal-details footer/header component added for tax-filing purposes:

> eToro incorporated ... personal details (name and address) to facilitate "filing the statement with your local authorities" ([fxnewsgroup.com](https://fxnewsgroup.com/forex-news/retail-forex/etoro-enhances-account-statement-feature/), secondary source)

### Key column names (per sheet)

The most detailed, version-specific column data available comes from `etoro-edavki`, an open-source Python tool built specifically to parse real eToro Excel statements for Slovenian tax reporting. It defines explicit column-header constants per sheet and documents three successive format revisions (secondary source — community parser reading actual exported files, high specificity, not eToro's own documentation):

**"Account Activity" sheet** — format evolution documented in the tool as:

| Format version | Columns |
|---|---|
| 2022 | Date, Type, Details, Amount, Realized Equity Change, Realized Equity, Balance, Position ID, NWA |
| 2023 | Date, Type, Details, Amount, **Units**, Realized Equity Change, Realized Equity, Balance, Position ID, **Asset type**, NWA |
| 2025.2 (current) | Date, Type, Details, Amount, **Units / Contracts**, Realized Equity Change, Realized Equity, Balance, Position ID, Asset type, NWA |

(source: [github.com/masbug/etoro-edavki/blob/main/etoro_edavki.py](https://github.com/masbug/etoro-edavki/blob/main/etoro_edavki.py), secondary source)

**"Closed Positions" sheet** (v2025.2.15 columns, per the same tool):

> Position ID, Action, Long / Short, Amount, Units / Contracts, Open Date, Close Date, Leverage, Spread Fees (USD), Market Spread (USD), Profit(USD), Profit(EUR), FX rate at open (USD), FX rate at close (USD), Open Rate, Close Rate, Take profit rate, Stop loss rate, Overnight Fees and Dividends, Copied From, Type, ISIN, Notes ([github.com/masbug/etoro-edavki](https://github.com/masbug/etoro-edavki/blob/main/etoro_edavki.py), secondary source)

A separate community write-up of eToro's own "Dividend Tab" (added as a statement feature) lists its columns as: Date of Payment, Instrument Name, Net Dividend Amount (in USD), Withholding Tax (%), Withholding Tax (USD), Position ID, Type (CFD or underlying asset position), ISIN Code ([fastbull.com/brokersview](https://www.fastbull.com/brokersview/news/etoro-introduces-dividend-section-to-account-statements-6601/), secondary source describing an eToro product announcement).

### Date format

The `etoro-edavki` parser auto-detects between two date-format families depending on account locale, both `DD/MM` or `DD.` (day-first) ordered, with or without a time-of-day component:

> `"%d/%m/%Y %H:%M:%S"` (English, with seconds), `"%d.%m.%Y %H:%M:%S"` (Slovenian, with seconds), `"%d/%m/%Y"` (English, date only), `"%d.%m.%Y"` (Slovenian, date only) ([github.com/masbug/etoro-edavki](https://github.com/masbug/etoro-edavki/blob/main/etoro_edavki.py), secondary source)

This is day-first in every observed variant (no MM/DD ordering seen), but the separator character (`/` vs `.`) and whether a time component is present both vary by account/locale settings, and the tool has to auto-detect which one it received rather than assume a fixed format.

### Currency / multi-currency handling

eToro's own statement expresses monetary columns in USD regardless of the account's funding currency, with an explicit `(USD)` suffix baked into several column headers (e.g. "Profit(USD)", "FX rate at open (USD)") and a parallel EUR-labeled column for at least the Profit field:

> All amounts are in USD (`ETORO_CURRENCY = "USD"`), with conversion to EUR handled separately using daily exchange rates ([github.com/masbug/etoro-edavki](https://github.com/masbug/etoro-edavki/blob/main/etoro_edavki.py), secondary source)

The presence of both `FX rate at open (USD)` and `FX rate at close (USD)` columns on the Closed Positions sheet indicates eToro records the FX conversion rate applied at position open and close as separate fields, rather than only a converted amount — relevant for any position whose underlying instrument trades in a non-USD currency.

### Transaction-type encoding

eToro uses an explicit `Type` column on both the Account Activity and Closed Positions sheets (not sign-inference alone). Confirmed value markers include "Payment caused by dividend" appearing in the Details field for dividend rows on Account Activity ([github.com/masbug/etoro-edavki](https://github.com/masbug/etoro-edavki/blob/main/etoro_edavki.py), secondary source); the full enumerated set of `Type` values was not found documented anywhere (official or community) during this research — only individual values referenced in passing (dividend payments, open/close position rows, deposits/withdrawals per the official help-center snippet above, fees per the "Account Activity ... fees" description). This is a gap: no source enumerates the complete `Type` vocabulary.

### Quirks

- **Sign convention**: quantity/amount sign follows the standard buy(+)/sell(−) convention observed in the community parser: `if trade["quantity"] > 0` → purchase, else → sale ([github.com/masbug/etoro-edavki](https://github.com/masbug/etoro-edavki/blob/main/etoro_edavki.py), secondary source).
- **Leverage**: Closed Positions carries an explicit `Leverage` column; the parser multiplies amounts by leverage when leverage > 1, implying the raw `Amount` field is margin (not notional) unless adjusted.
- **Symbol/instrument identification**: the `Details` field on Account Activity encodes the instrument as free text that community tooling splits on `/` to recover a symbol — there is no separate ticker/symbol column on that sheet (the Closed Positions and Dividend sheets do carry an `ISIN` column instead).
- **Copy-trading provenance**: a `Copied From` (aka "Copy From") column records whether a position was opened via eToro's copy-trading feature and by whom.
- **Format instability across time**: three different Account Activity column sets were documented across 2022/2023/2025.2 exports from the same tool's changelog, meaning column presence (e.g. `Units` vs `Units / Contracts`, `Asset type` presence) is a function of *when* the statement was generated, not just a fixed spec — an import mapping keyed to exact header strings risks breaking across eToro's own format revisions.
- **Locale variants**: decimal separator (period vs comma) and date separator (`/` vs `.`) both vary with the exporting account's locale, per the same tool's auto-detection logic.
- eToro's official help center content could not be fetched as rendered text (Salesforce community JS SPA, confirmed empty server response) — official confirmation of exact column-level facts above should be treated as indirect (via search-engine-indexed snippets of the same pages) rather than a first-hand read of eToro's documentation.

---

## 2. Revolut

### Export format

Revolut's statement export is a flat, single-table CSV (or PDF), generated per currency account/pocket, via the app or web:

> A personal Revolut account usually only offers a PDF and CSV overview. You can export Revolut transactions to CSV or PDF from both the mobile app and the web, per currency account, monthly or for a custom range. ([bankxlsx.com/blog/can-i-export-revolut-transactions-to-csv-or-excel](https://bankxlsx.com/blog/can-i-export-revolut-transactions-to-csv-or-excel), secondary source, but the underlying steps it describes match Revolut's own in-app flow)

> Go to Accounts and choose Statements. Choose the date range... Choose the file type. We support PDF and CSV. Click on 'Generate' to download the file or 'Share' to send it via email ([help.revolut.com](https://help.revolut.com), via search-engine-indexed snippet of Revolut's own help documentation — the live help.revolut.com pages returned a bot-check/challenge page rather than article content when fetched directly during this research, so exact wording is taken from indexed snippets, not a directly rendered page)

Because export is **per currency account**, a Revolut user holding balances in multiple currencies produces one CSV per currency pocket rather than one consolidated multi-currency statement — i.e., multi-currency is handled by *separate single-currency files*, not a multi-currency column layout within one file. This is inferred from the "per currency account" export step above and not itself an explicit Revolut statement about file structure, so treat as a lower-confidence structural inference pending direct verification against a real export.

Revolut's own docs note a time-zone nuance in what the CSV contains:

> By default, all statements are generated using UTC times. When statements are downloaded in CSV format, the statement can include both local and UTC times for transactions. ([help.revolut.com](https://help.revolut.com), via search-engine-indexed snippet)

### Key column names

The exact Revolut Personal CSV header row is corroborated by two independent sources: a spreadsheet-integration vendor quoting the literal header row, and an open-source `ofxstatement` parser plugin whose source code hard-codes the same header names as a file-format signature check:

> Soubor ve formátu CSV se sloupci v angličtině: Type, Product, Started Date, Completed Date, Description, Amount, Fee, Currency, State, Balance. ("CSV file with columns in English: ...") ([dativery.com/en/apps/revolut-personal-csv](https://www.dativery.com/en/apps/revolut-personal-csv/), secondary source)

Confirmed independently in the parser's own signature-detection code:

```python
required_columns = [
    "Type", "Started Date", "Completed Date", "Description",
    "Amount", "Currency", "Balance", "Product",
]
```
([github.com/mlaitinen/ofxstatement-revolut](https://github.com/mlaitinen/ofxstatement-revolut), secondary source — source code fetched directly, reading a real exported file's header row at runtime, so this is effectively first-hand evidence of the actual file format even though it's not Revolut's own documentation)

For **Revolut Business** (a distinct export, not the personal-account one, included here only for contrast since some solo users may hold both):

> Revolut Business CSV files include columns such as Date started (UTC), Date completed (UTC), ID, Type, Description, Reference, Payer, Card number, Orig currency, Orig amount, Payment currency, Amount, Fee, Balance, and Account ([help.revolut.com](https://help.revolut.com) business docs, via search-engine-indexed snippet)

Note the Business format's `Orig currency` / `Orig amount` vs `Payment currency` / `Amount` split — an explicit two-currency-pair column layout for currency-converted transactions — versus the Personal CSV's single `Currency`/`Amount` pair.

### Date format

The `ofxstatement-revolut` parser (reading real Personal CSV exports) hard-codes:

```python
date_format = "%Y-%m-%d %H:%M:%S"
```
([github.com/mlaitinen/ofxstatement-revolut](https://github.com/mlaitinen/ofxstatement-revolut/blob/master/src/ofxstatement/plugins/revolut.py), secondary source, read from live source)

i.e. ISO-8601-like `YYYY-MM-DD HH:MM:SS`, with both a "Started Date" and "Completed Date" column populated per transaction. However, a separate, actively-maintained `ofxstatement-revolut` package (PyPI `2.0.3`) exposes `date_format` as a **user-supplied configuration option** rather than a fixed constant, and its release notes describe support for "the new CSV format" as of one version alongside an older "iOS CSV format from September 2019 and earlier," implying Revolut's own CSV date formatting has changed across app versions/export paths over time and isn't guaranteed stable ([pypi.org/project/ofxstatement-revolut/2.0.3](https://pypi.org/project/ofxstatement-revolut/2.0.3/), secondary source). A Revolut community-forum thread titled "Statement Export - CSV Date Formatting" (bug report) also suggests date-format behavior in the export has been a point of user confusion, though the thread's discussion content itself could not be directly fetched during this research (the URL redirected to Revolut's general help portal rather than the specific thread).

### Currency / multi-currency handling

Personal CSV: single `Currency` column + single signed `Amount` column per row (no separate orig/payment currency split) — multi-currency activity across pockets is represented by exporting separate per-currency files rather than one file with multiple currency columns (see Export format above). Business CSV: explicit `Orig currency`/`Orig amount` vs `Payment currency`/`Amount` column pairs on the same row for currency-converted transactions, i.e. a currency-conversion transaction row carries both the original and settled currency/amount inline.

### Transaction-type encoding

Revolut uses an explicit `Type` column (not sign-inference alone). The enumerated `Type` values actually used in exports are documented — as code, not prose — in the `ofxstatement-revolut` plugin's mapping table, built from reading real files:

```python
TRANSACTION_TYPES = {
    "TRANSFER": "XFER",
    "CARD_PAYMENT": "POS",
    "CARD_REFUND": "POS",
    "TOPUP": "DEP",
    "EXCHANGE": "XFER",
    "ATM": "ATM",
    "FEE": "FEE",
}
```
([github.com/mlaitinen/ofxstatement-revolut](https://github.com/mlaitinen/ofxstatement-revolut/blob/master/src/ofxstatement/plugins/revolut.py), secondary source, read from live source code — this is the most concrete evidence found of Revolut's actual `Type` enum values, though it is not an official/exhaustive list published by Revolut itself, and the mapping's fallback (`TRANSACTION_TYPES.get(value, "POS")`) implies other, unlisted Type values exist in the wild that this particular tool doesn't recognize)

A separate `ofxstatement-revolut` PyPI changelog corroborates `ATM`, `EXCHANGE`, and `FEE` as distinct type-handling cases added in version 2.0.3 ("Use ATM types for ATM events, XFER type for EXCHANGE operations, and add FEE type") ([pypi.org/project/ofxstatement-revolut/2.0.3](https://pypi.org/project/ofxstatement-revolut/2.0.3/), secondary source).

### Quirks

- **Explicit `State` column with pending/non-pending states**: transaction lifecycle state is a separate enumerated column, not inferred. Confirmed values referenced across sources: `COMPLETED`, `PENDING`, `REVERTED`, `DECLINED`, and (per Revolut's own transaction-lifecycle description of a "failed" case) `FAILED` — via search-indexed snippets of help.revolut.com and Revolut's own blog post on pending transactions ([revolut.com/blog/post/what-does-a-pending-transaction-mean](https://www.revolut.com/blog/post/what-does-a-pending-transaction-mean/), and help.revolut.com pages on reverted/declined card payments). The `ofxstatement-revolut` parser explicitly **drops any row where `State != "COMPLETED"`** — i.e. a real-world import tool treats pending/reverted/declined rows as noise to filter, not data to import as-is:
  > `# Ignore pending charges` / `if line[c["State"]] != "COMPLETED": return None` ([github.com/mlaitinen/ofxstatement-revolut](https://github.com/mlaitinen/ofxstatement-revolut/blob/master/src/ofxstatement/plugins/revolut.py), secondary source, read from live source)
- **Separate `Fee` column, not folded into `Amount`**: a transaction's fee is a distinct numeric column alongside `Amount`; the same parser synthesizes an *additional* separate ledger line for the fee (payee "Revolut", type "FEE") rather than treating it as part of the original transaction's amount — meaning fee handling can be modeled either as "one transaction with a fee sub-amount" or "two transactions" depending on how the field is interpreted.
- **Thousands separators in numeric fields**: the same parser strips apostrophe (`'`) characters from `Amount`/`Fee` values before parsing (`.replace("'", "").strip()`), suggesting at least some Revolut CSV variants (locale-dependent, e.g. Swiss formatting) use `'` as a thousands separator rather than none/comma.
- **Header row plus product filter, no footer/preamble rows observed**: the parser reads the CSV's first line directly as the header (`self.cur_record <= 1: return None` skips only row 1) with no mention of preamble or footer rows in any source found — unlike eToro's multi-sheet workbook, Revolut's CSV appears to be a plain single header row + data rows, though this wasn't independently confirmed against a real downloaded sample.
- **`Product` column**: a `Product` field (e.g. distinguishing "Current" from other Revolut product pockets) is present and used by at least one importer to filter which rows belong to which product/account — relevant if a Revolut export can contain rows spanning more than one product within a single file.
- Revolut's own help center could not be fetched as rendered content directly (Cloudflare bot-check challenge page returned instead of article content on every attempt during this research); all "official" Revolut facts above are via search-engine-indexed snippets of the same help.revolut.com pages, not a first-hand read.

---

## 3. Typical / generic bank statement CSV downloads

**This section explicitly does not describe one canonical format** — there isn't one. It characterizes documented, representative patterns and variance across retail banks and the personal-finance tooling built to cope with that variance. The variance itself — not any single bank's format — is the key finding for #41 to design against.

Plaid, a company whose entire business is normalizing bank data across thousands of institutions, states the scale of the underlying problem plainly:

> the data financial institutions natively provide is messy and convoluted—and far from normalized ([plaid.com/blog/how-plaid-parses-transaction-data](https://plaid.com/blog/how-plaid-parses-transaction-data/) — Plaid's own blog, describing e.g. the same Chick-fil-A purchase appearing as `"chick-fil-A 3848489"`, `"POS DEBIT Chick Fil A 4/5"`, or `"Authorized purchase Chkfila 333222121 NY NY"` depending on the issuing bank)

GoCardless (an EU account-information/open-banking provider operating under the PSD2-mandated Berlin Group NextGenPSD2 API standard — i.e. a case where banks *are* supposed to expose data in one standardized schema) makes the same point even for that "standardized" API layer:

> Although financial institutions are supposed to follow the same recommendations in terms of data structures, this is not always the case... you should anticipate situations where the amount of data delivered differs between banks — for example, some banks may include `ownerName` with the account details, some may opt not to do this. ([docs.gocardless.com/bank-account-data](https://docs.gocardless.com/bank-account-data/), GoCardless's own developer documentation)

Even the "standardized" Berlin Group schema GoCardless exposes distinguishes `bookingDate`/`bookingDateTime` from `valueDate`/`valueDateTime` as separate date fields per transaction, plus a bank-specific escape hatch (`additionalDataStructured`) for fields that don't fit the common schema ([docs.gocardless.com/bank-account-data/transactions](https://docs.gocardless.com/bank-account-data/transactions), GoCardless developer docs) — i.e. even a regulator-mandated common API doesn't fully collapse the "which date is *the* date" question, let alone raw retail-bank CSV downloads which have no such mandate at all.

### Representative column-layout patterns

| Pattern | Description | Example (secondary/community sourcing) |
|---|---|---|
| Single signed `Amount` column | Debits negative, credits positive, one column | Monzo (UK): "signed Amount column (debits are negative, credits are positive)" ([search-indexed characterization](https://www.bank-statements.co/convert/monzo-bank-statement-to-excel), secondary) |
| Separate `Money In` / `Money Out` (or `Debit`/`Credit`) columns | Two columns, only one populated per row | Starling (UK): "Date, Description, Money In, and Money Out, with a Balance column" ([secondary source](https://accounter.co.za/tools/international-bank-statement-converter/starling), secondary); a US retail-bank example also documents "Amount (either one signed column or separate Debit/Credit)" as both existing in the wild ([bankxlsx.com](https://bankxlsx.com/blog/can-i-export-bank-of-america-transactions-to-csv-or-excel), secondary) |
| Running `Balance` column | Post-transaction account balance included per row | Documented as present for Starling and optionally for Bank of America-style exports, but explicitly *not* guaranteed: "some exports include a running balance, others won't" ([bankxlsx.com](https://bankxlsx.com/blog/can-i-export-bank-of-america-transactions-to-csv-or-excel), secondary) |
| `Posted Date` vs `Transaction Date` | Two distinct date columns for when a transaction occurred vs when it settled | "Posted Date (sometimes also Transaction Date)... Posted Date is recommended for reconciliation; Transaction Date available but less reliable" ([bankxlsx.com](https://bankxlsx.com/blog/can-i-export-bank-of-america-transactions-to-csv-or-excel), secondary) — directly analogous to the `bookingDate`/`valueDate` split GoCardless documents for the regulated open-banking API layer |
| Category field usually absent from raw exports | Bank's own website/app category tags typically don't carry into the CSV | "Categories shown on the website rarely carry into CSV" ([bankxlsx.com](https://bankxlsx.com/blog/can-i-export-bank-of-america-transactions-to-csv-or-excel), secondary) |

### Date format variance

No single convention. Observed/documented variants:
- `YYYY-MM-DD` (ISO 8601) — e.g. Monzo, per the secondary source above; also YNAB's own import guidance recommends this format specifically "as it avoids day/month confusion" between US and non-US conventions.
- `DD/MM/YYYY` and `MM/DD/YYYY` — both explicitly accepted (ambiguously, since the tool can't always tell them apart) by YNAB's CSV importer, which documents accepting `DD/MM/YY, DD/MM/YYYY, DD/MM//YYYY, MM/DD/YY, MM/DD/YYYY, MM/DD//YYYY` ([support.ynab.com](https://support.ynab.com/en_us/formatting-a-csv-file-an-overview-BJvczkuRq), YNAB's own support documentation, via search-indexed snippet — the live Zendesk/Intercom-style page is client-rendered and did not return article text to a direct fetch during this research).
- `DD.MM.YYYY` (dot-separated, day-first) — documented for German retail banks (e.g. Sparkasse), alongside semicolon field delimiters and comma decimal separators, per community CSV-conversion-tool documentation: delimiter `;`, encoding `ISO-8859-1`, date format `%d.%m.%Y`, decimal separator `,` ([kontocsv.de](https://www.kontocsv.de/en/sparkasse), secondary/community source).

### Decimal and thousands separators

Not universal. US/UK-style exports use `.` for decimals with no (or comma) thousands separators; continental-European exports (per the German-bank example above) use `,` for decimals and `.` or nothing for thousands, often paired with `;` as the field delimiter (since `,` is already the decimal marker and can't also be the CSV delimiter). YNAB's own import guidance is explicit that its importer expects "dots for decimals with no thousands separators," implying non-conforming exports (e.g. the German comma-decimal style) must be reformatted before import into that tool.

### Transaction-type encoding

Generic retail bank CSVs largely **do not** have an explicit enumerated `Type` column comparable to Revolut's or eToro's — the BankXLSX characterization of Bank of America notes only Date/Description/Amount/Check-Number/Balance, with transaction "type" (purchase, transfer, direct debit, etc.) left implicit in the free-text `Description` field, not a separate coded column. This is the inverse of Revolut/eToro's explicit-`Type`-column pattern and is itself a key variance point: an import-mapping design cannot assume a `Type` column exists at all for this category of source, whereas it can for Revolut and eToro.

### Quirks

- **Encoding variance**: German-bank example above uses `ISO-8859-1`; nothing guarantees UTF-8 across all banks. YNAB's guidance separately calls out "save as UTF-8 to avoid encoding issues with special characters" as advice to *users*, implying banks' native exports aren't reliably UTF-8 already.
- **No blank rows / merged cells before the header, per YNAB's own import constraints**: YNAB's documented CSV requirements state "no blank rows above headers, and no merged cells," which is worth noting precisely because it implies some real bank exports historically *did* have such preamble content that had to be stripped before import — otherwise the constraint wouldn't need stating.
- **QIF/OFX exist as bank-provided alternatives to CSV** for some institutions, with OFX in particular carrying dedicated structured fields (payee, memo, check number) that a flat CSV would need separate columns to replicate: "OFX and QBO files have dedicated fields for each piece of data (NAME, MEMO, CHECKNUM), so they tend to preserve more detail" ([meetglimpse.com](https://meetglimpse.com/software-guides/convert-financial-files/), secondary/vendor source) — relevant context in case a "bank statement" input to the future import system turns out to be OFX/QIF rather than CSV.
- **Open-banking APIs still disagree on "the" date and on optional fields**, per the GoCardless quotes above, even where a shared regulatory standard exists — a raw retail CSV download (no such standard) should be assumed to vary at least as much, and the "which date field is authoritative" ambiguity (booking vs value, posted vs transaction) recurs across every source examined for this section.

---

## Cross-institution comparison

Facts relevant to what ticket #41 will need to design against, per source:

| Fact | eToro | Revolut (Personal) | Generic bank CSV |
|---|---|---|---|
| Export format | Excel (.xlsx), **multi-sheet** (Summary / Closed Positions / Account Activity / Financial Summary / Dividends) | Flat **CSV** (or PDF), **single table**, one file per currency pocket | Varies: CSV most common; some banks offer OFX/QIF instead; always single-table when CSV |
| Explicit `Type`/transaction-type column? | Yes — `Type` column on Account Activity and Closed Positions sheets, but no source enumerates its full value set | Yes — `Type` column with a documented (if possibly incomplete) enum: `TRANSFER`, `CARD_PAYMENT`, `CARD_REFUND`, `TOPUP`, `EXCHANGE`, `ATM`, `FEE` | Usually **no** — type is typically implicit in a free-text `Description`/`Payee` field, not a coded column |
| Date format | Day-first only observed (`DD/MM/YYYY` or `DD.MM.YYYY`), separator and time-of-day presence vary by locale; format has changed across statement versions | ISO-like `YYYY-MM-DD HH:MM:SS`, though at least one parser treats the exact format as configurable/non-guaranteed across app-version eras | No single convention: ISO `YYYY-MM-DD`, `DD/MM/YYYY`, `MM/DD/YYYY`, and dot-separated `DD.MM.YYYY` (locale-dependent) all documented in the wild |
| Multiple date fields per row? | Yes — `Open Date`/`Close Date` (Closed Positions); separate `FX rate at open`/`FX rate at close` | Yes — `Started Date` and `Completed Date` | Common — `Posted Date` vs `Transaction Date`, or `bookingDate` vs `valueDate` in the regulated open-banking equivalent |
| Sign convention for amount | Positive = buy/open, negative = sell/close (community-parser-documented); leverage multiplies the raw amount | Single signed `Amount` column | Split: some banks single signed `Amount`; others separate `Debit`/`Credit` (or `Money In`/`Money Out`) columns |
| Currency representation | Statement amounts denominated in **USD** regardless of account funding currency, with separate FX-rate columns at open/close; a parallel EUR profit column also exists | Single `Currency` + `Amount` pair per file; multi-currency handled via **separate files per currency pocket**, not multi-currency columns in one file (Business variant instead uses paired `Orig currency`/`Orig amount` vs `Payment currency`/`Amount` columns on one row) | No fixed pattern found; not covered in depth by sources examined here |
| Decimal / thousands separators | Locale-dependent (period vs comma), auto-detected by the community parser rather than fixed | At least one locale variant uses `'` (apostrophe) as a thousands separator | Locale-dependent: `.` decimal (US/UK) vs `,` decimal + `;` delimiter (continental Europe, e.g. German banks) |
| Pending vs completed / lifecycle state | Not clearly documented as a separate column in sources found (statement appears to reflect settled/closed activity only) | Yes — explicit `State` column: `COMPLETED`, `PENDING`, `REVERTED`, `DECLINED`, `FAILED`; at least one real-world importer discards all non-`COMPLETED` rows | Not typically present as a coded field in a plain CSV download (a downloaded statement generally already reflects posted/settled transactions only) |
| Header/footer/preamble rows | Multi-sheet workbook structure itself is the main structural quirk; per-sheet header row position not fully characterized in sources found | Single header row (row 1) confirmed by parser logic; no preamble/footer rows found documented | YNAB's own import constraints ("no blank rows above headers, no merged cells") imply such preamble content is a real-world occurrence for at least some bank exports |
| Format stability over time | **No** — Account Activity's own column set changed across 2022 / 2023 / 2025.2 statement revisions (documented) | Partial — CSV format has changed since a "September 2019 and earlier" iOS-era format per a parser's compatibility notes | Not assessed directly, but the general open-banking evidence (GoCardless) suggests even regulated schemas leave room for per-bank field-presence variance, which plain CSV has no constraint against at all |
| Fee representation | Not a separate documented column beyond `Spread Fees (USD)`/`Market Spread (USD)` on Closed Positions | Separate `Fee` column alongside `Amount` (not folded into it) | Not typically a separate column; fees usually appear as their own transaction rows within `Description` |

---

## Sources

**eToro**
- [help.etoro.com/My-Account/70325632/What-is-the-eToro-Account-Statement.htm](https://help.etoro.com/My-Account/70325632/What-is-the-eToro-Account-Statement.htm) — eToro's own help center (content taken from a search-engine-indexed snippet; the live page is a client-rendered Salesforce community SPA that returned an empty HTML shell to a direct fetch during this research, confirmed via `curl`)
- [help.waltio.com/en/articles/3748833-etoro-file](https://help.waltio.com/en/articles/3748833-etoro-file) — secondary source, third-party crypto-tax tool's eToro import guide
- [fxnewsgroup.com/forex-news/retail-forex/etoro-enhances-account-statement-feature](https://fxnewsgroup.com/forex-news/retail-forex/etoro-enhances-account-statement-feature/) — secondary source, trade press covering an eToro statement feature update
- [fastbull.com/brokersview/news/etoro-introduces-dividend-section-to-account-statements-6601](https://www.fastbull.com/brokersview/news/etoro-introduces-dividend-section-to-account-statements-6601/) — secondary source, trade press
- [github.com/masbug/etoro-edavki/blob/main/etoro_edavki.py](https://github.com/masbug/etoro-edavki/blob/main/etoro_edavki.py) — secondary source, open-source parser read directly from source (highest-specificity source found for exact column headers and date formats)
- [github.com/weirdapps/etoro_statement](https://github.com/weirdapps/etoro_statement) — secondary source, open-source statement-processing tool README
- [github.com/earlisreal/eJournal](https://github.com/earlisreal/eJournal) — secondary source, open-source trading-journal importer

**Revolut**
- [help.revolut.com](https://help.revolut.com) (various personal/business statement and transaction-state articles) — Revolut's own help center (content taken from search-engine-indexed snippets; the live pages returned a Cloudflare bot-check challenge page to every direct fetch attempt during this research, confirmed via `curl`)
- [revolut.com/blog/post/what-does-a-pending-transaction-mean](https://www.revolut.com/blog/post/what-does-a-pending-transaction-mean/) — Revolut's own blog
- [bankxlsx.com/blog/can-i-export-revolut-transactions-to-csv-or-excel](https://bankxlsx.com/blog/can-i-export-revolut-transactions-to-csv-or-excel) — secondary source
- [dativery.com/en/apps/revolut-personal-csv](https://www.dativery.com/en/apps/revolut-personal-csv/) — secondary source, quotes the literal Personal CSV header row
- [github.com/mlaitinen/ofxstatement-revolut](https://github.com/mlaitinen/ofxstatement-revolut) (source file: `src/ofxstatement/plugins/revolut.py`) — secondary source, open-source parser read directly from source; highest-specificity source found for Revolut's actual `Type` enum, date format, and `State`-filtering behavior
- [pypi.org/project/ofxstatement-revolut/2.0.3](https://pypi.org/project/ofxstatement-revolut/2.0.3/) — secondary source, package changelog documenting format evolution over time

**Generic bank CSV**
- [plaid.com/blog/how-plaid-parses-transaction-data](https://plaid.com/blog/how-plaid-parses-transaction-data/) — Plaid's own blog (first-party for Plaid's own normalization approach; used here as evidence of cross-bank format variance)
- [docs.gocardless.com/bank-account-data](https://docs.gocardless.com/bank-account-data/) and [docs.gocardless.com/bank-account-data/transactions](https://docs.gocardless.com/bank-account-data/transactions) — GoCardless's own developer documentation (first-party for GoCardless's API; describes real-world bank-to-bank variance even under a shared regulatory schema)
- [support.ynab.com/en_us/formatting-a-csv-file-an-overview-BJvczkuRq](https://support.ynab.com/en_us/formatting-a-csv-file-an-overview-BJvczkuRq) — YNAB's own support documentation (content taken from a search-engine-indexed snippet; the live Zendesk/Intercom-style page is client-rendered and did not return article text to a direct fetch during this research, confirmed via `curl`)
- [bankxlsx.com/blog/can-i-export-bank-of-america-transactions-to-csv-or-excel](https://bankxlsx.com/blog/can-i-export-bank-of-america-transactions-to-csv-or-excel) — secondary source, representative US retail-bank CSV characterization
- [bank-statements.co/convert/monzo-bank-statement-to-excel](https://www.bank-statements.co/convert/monzo-bank-statement-to-excel) and [accounter.co.za/tools/international-bank-statement-converter/starling](https://accounter.co.za/tools/international-bank-statement-converter/starling) — secondary sources, representative UK retail-bank CSV characterizations
- [kontocsv.de/en/sparkasse](https://www.kontocsv.de/en/sparkasse) — secondary source, representative German retail-bank (Sparkasse) CSV characterization (semicolon delimiter, comma decimal, `DD.MM.YYYY` dates, ISO-8859-1 encoding)
- [meetglimpse.com/software-guides/convert-financial-files](https://meetglimpse.com/software-guides/convert-financial-files/) — secondary/vendor source, OFX/QIF/CSV format comparison
