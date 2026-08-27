# Stack options research

Research for issue #11, blocking #3 (tech stack decision). This is a **neutral tradeoff comparison**, not a recommendation — no option below is declared a winner; the human decides in #3.

Scope note: the app is a solo/single-user hobby webapp (no multi-tenancy), rebuilt from scratch (not reusing code — `my-worth` at `/Users/andres/personal/my-worth` is referenced only as a "familiar stack" data point: React 18 + Vite 6 + Hono/`@hono/node-server` + SQLite via `@libsql/client` + Drizzle + Netlify, TypeScript, Vitest). The data model (from `CONTEXT.md` / ADR 0001) is a generic `Account` entity with a `kind` field and a per-kind JSON metadata blob, event-based `value_snapshots`, and normalized `transactions` tagged as external-flow for a contribution-vs-growth split. Standing decisions locked elsewhere: manual CSV import only (no live bank/broker API sync), an auto-categorization rules engine (needs a rules-config UI) in v1, multi-currency support required, and real financial data sensitivity (account balances, positions, mortgage details).

---

## 1. Frontend framework

**Candidates compared:** React + Vite (the `my-worth` reference stack), SvelteKit, SolidStart. Considered against the app's UI surface: a CSV-import flow (upload, preview, column mapping), dashboard/report views with charts, and a form-heavy rules-engine config UI with conditional logic.

| | React + Vite | SvelteKit | SolidStart |
|---|---|---|---|
| What it is | UI library (React) + build tool (Vite); routing/data-fetching are *not* built in — React's own docs say building from scratch "does require that you make choices on which tools to use for routing, data fetching" and recommends a framework for new apps ([react.dev/learn/creating-a-react-app](https://react.dev/learn/creating-a-react-app)) | Full meta-framework: SvelteKit itself provides file-based routing, SSR/CSR/prerendering, and data loading ([svelte.dev/docs/kit/introduction](https://svelte.dev/docs/kit/introduction)) | Full meta-framework built on SolidJS + Vite, "router agnostic" (ships an official router, or use TanStack Router) with fine-grained reactivity extended server-side ([start.solidjs.com/getting-started/what-is-solidstart](https://start.solidjs.com/getting-started/what-is-solidstart)) |
| Rendering model | Component-based UI, described via "components... piece of the UI that has its own logic and appearance" ([react.dev/learn](https://react.dev/learn)); rendering strategy (CSR/SSR/SSG) is left to whichever framework wraps it (Next.js, React Router v7, TanStack Start, or a plain Vite SPA with no SSR at all) | Configurable per-route: SSR, CSR, and prerendering, with the docs describing "configurable rendering to handle different parts of your app" ([svelte.dev/docs/kit/introduction](https://svelte.dev/docs/kit/introduction)) | Fine-grained reactivity (Solid's signal model) carried through to the server — "Fine-grained reactivity goes fullstack" ([start.solidjs.com](https://start.solidjs.com/getting-started/what-is-solidstart)) |
| Routing | Not part of React itself. React's own docs list React Router v7 ("paired with Vite to create a full-stack React framework"), Next.js App Router, TanStack Start (beta), and Expo as the maintained options ([react.dev/learn/creating-a-react-app](https://react.dev/learn/creating-a-react-app)) | Built-in file-based router, described in the intro as "a router that updates your UI when a link is clicked" ([svelte.dev/docs/kit/introduction](https://svelte.dev/docs/kit/introduction)) | Router-agnostic: ships an official Solid Router but explicitly also supports TanStack Router ([start.solidjs.com](https://start.solidjs.com/getting-started/what-is-solidstart)) |
| Data loading / forms | Left to the chosen framework/library layer — not addressed by React's own docs (the Quick Start guide covers only local state via `useState` and props) ([react.dev/learn](https://react.dev/learn)) | Two built-in primitives: `load` functions and form actions, both documented as core SvelteKit concepts ([svelte.dev/docs/kit/introduction](https://svelte.dev/docs/kit/introduction)) | "Strongly parallelized data loading" with request/resource deduplication and co-located "Server Actions", plus "Single-Flight Mutations" to avoid server waterfalls ([start.solidjs.com](https://start.solidjs.com/getting-started/what-is-solidstart)) |
| Build tool / bundling | Vite: near-instant dev server via native ESM ("dev server startup was nearly instant, regardless of application size"), production builds still bundled because "shipping it in production is still inefficient due to additional network round trips from nested imports" — Vite is moving its production bundling to Rolldown, a Rust-based unification of its previous esbuild+Rollup pipelines ([vite.dev/guide/why](https://vite.dev/guide/why)) | Also built on Vite, inheriting the same dev-server/HMR model, plus SvelteKit-specific "build optimizations to load only the minimal required code" ([svelte.dev/docs/kit/introduction](https://svelte.dev/docs/kit/introduction)) | Also built on Vite; docs highlight deployability "to every platform with a Vite plugin," listing 20+ targets including Netlify ([start.solidjs.com](https://start.solidjs.com/getting-started/what-is-solidstart)) |
| Learning-curve framing (framework's own words) | react.dev's Quick Start is scoped to "80% of the React concepts that you will use on a daily basis," but the docs are explicit that routing/data-fetching are additional decisions on top ([react.dev/learn](https://react.dev/learn)) | Docs recommend the interactive tutorial for newcomers; prior Svelte knowledge "will help" but isn't required — positions itself as comparable in shape to "Next" or "Nuxt" for anyone coming from another ecosystem ([svelte.dev/docs/kit/introduction](https://svelte.dev/docs/kit/introduction)) | Landing-page docs don't make an explicit learning-curve claim; the reactivity model (signals) differs from React's re-render model, which is a conceptual shift for anyone coming from React |
| Ecosystem for forms/charts | Largest ecosystem by virtue of React's age/adoption (not itself a documented claim — noted here as context, not a sourced fact) | Smaller ecosystem; SvelteKit's own `load`/form-action primitives cover a good share of form needs without a separate library | Smaller ecosystem than React's; relies on Solid-specific or framework-agnostic (web-standard) libraries |

**Tradeoffs for this app specifically:**
- The rules-engine config UI is form-heavy with conditional logic. SvelteKit's built-in form actions and SolidStart's co-located server actions both give an opinionated, framework-native answer to "how do I submit and validate a form" out of the box; a plain React+Vite SPA has no built-in equivalent and needs an added routing/forms layer (e.g. React Router or a form library) chosen separately — which is exactly the tradeoff react.dev itself flags when it says a from-scratch React app "does require that you make choices on which tools to use for routing, data fetching" ([react.dev/learn/creating-a-react-app](https://react.dev/learn/creating-a-react-app)).
- Charts/dashboards are not addressed as a built-in capability by any of the three frameworks' own docs fetched here — none of React, SvelteKit, or SolidStart ship a charting solution; this is an ecosystem-library concern independent of framework choice.
- For a solo-maintained hobby app, "familiar stack" (the `my-worth` React+Vite precedent) carries a maintenance-cost argument that is orthogonal to any framework's documented technical capabilities — reusing known tooling avoids a second learning curve, at the cost of not benefiting from a framework's built-in routing/data/forms story.
- SolidStart's signals-based reactivity and SvelteKit's compiler-based reactivity are both a conceptual departure from React's render-and-diff model; the switching cost is a maintenance-solo factor not captured in either framework's own docs.

---

## 2. Database

**Candidates compared:** SQLite / libSQL (embedded, as used by the `my-worth` reference stack via `@libsql/client`) vs. PostgreSQL (client/server), with Turso noted as a hosted-libSQL middle ground.

### Fit for local/solo/zero-ops use

SQLite's own docs frame its target use case directly:

> "SQLite does not compete with client/server databases. SQLite competes with fopen()." — and: "Because an SQLite database requires no administration, it works well in devices that must operate without expert human support." ([sqlite.org/whentouse.html](https://www.sqlite.org/whentouse.html))

The same page describes when a client/server engine (like Postgres) is the better fit: "If there are many client programs sending SQL to the same database over a network, then use a client/server database engine instead of SQLite... avoid using SQLite in situations where the same database will be accessed directly ... simultaneously from many computers over a network" ([sqlite.org/whentouse.html](https://www.sqlite.org/whentouse.html)). It also notes SQLite "only supports one writer at a time per database file," calling this a non-issue for most workloads since "a write transaction only takes milliseconds" ([same page](https://www.sqlite.org/whentouse.html)). For a single-user app with no concurrent-writer or tenant-isolation requirement, both of these SQLite-vs-Postgres differentiators (network-multi-client access, write concurrency) are largely moot per the locked "solo/single-user, no multi-tenancy" decision.

libSQL — the SQLite variant used by `@libsql/client` in the reference stack — describes itself as "an open source, open contribution fork of SQLite, created and maintained by Turso," created because "SQLite famously doesn't accept external contributors." It commits to file-format and API compatibility: "libSQL will always be able to ingest and write the SQLite file format" and "will keep 100% compatibility with the SQLite API, but may add additional APIs" ([github.com/tursodatabase/libsql](https://github.com/tursodatabase/libsql)). Turso's own docs describe Turso Database as "the next evolution of SQLite, built for modern applications," "fully backwards compatible with SQLite," offering embedded replicas for "local reads and writes with push/pull sync to the cloud," with a managed "Turso Cloud" layer for branching/backups/analytics ([docs.turso.tech/introduction](https://docs.turso.tech/introduction)) — i.e., a hosted path for SQLite/libSQL if a fully local file isn't the deploy target.

### JSON metadata blob (the per-`kind` Account metadata field)

| | SQLite (JSON1) | PostgreSQL (JSON/JSONB) |
|---|---|---|
| Storage | Text JSON by default; binary "JSONB" storage format available since SQLite 3.45.0, described as "smaller and faster than text JSON — potentially several times faster" for internal processing, though still "O(N) time complexity for most operations ... just like text JSON" ([sqlite.org/json1.html](https://www.sqlite.org/json1.html)) | `json` stores "an exact copy of the input text, which processing functions must reparse on each execution"; `jsonb` is "stored in a decomposed binary format that makes it slightly slower to input... but significantly faster to process, since no reparsing is needed" ([postgresql.org/docs/current/datatype-json.html](https://www.postgresql.org/docs/current/datatype-json.html)) |
| Querying | 28 scalar functions/operators plus `json_each`/`json_tree` table-valued functions for walking JSON, e.g. `json_extract(x, '$.path')`, `->`/`->>` operators ([sqlite.org/json1.html](https://www.sqlite.org/json1.html)) | Rich operator set on `jsonb`: `?`/`?|`/`?&` (key existence), `@>` (containment), `@?`/`@@` (jsonpath) ([postgresql.org/docs/current/datatype-json.html](https://www.postgresql.org/docs/current/datatype-json.html)) |
| Indexing JSON fields | SQLite's own docs explicitly disclaim fast lookup: "SQLite's JSONB format makes no such claim" (of O(1) element lookup) — general-purpose indexes can still be built on `json_extract()` expressions, but this isn't addressed on the page fetched ([sqlite.org/json1.html](https://www.sqlite.org/json1.html)) | `jsonb` supports GIN indexes (`jsonb_ops` default, or the smaller/faster `jsonb_path_ops`), plus btree/hash for whole-document equality; `json` supports no indexing ([postgresql.org/docs/current/datatype-json.html](https://www.postgresql.org/docs/current/datatype-json.html)) |
| Docs' own guidance | Not phrased as a recommendation either way | "In general, most applications should prefer to store JSON data as `jsonb`, unless there are quite specialized needs" ([postgresql.org/docs/current/datatype-json.html](https://www.postgresql.org/docs/current/datatype-json.html)) |

Given the ADR's own tradeoff framing — the generic-Account/JSON-metadata design is chosen "at the cost of DB-level query strictness on those fields — an acceptable trade for a solo hobby project with no query-performance requirements" (ADR 0001) — the indexing/query-performance gap between the two engines' JSON handling is a smaller factor here than it would be at higher scale or with heavier ad-hoc JSON querying.

### Multi-currency numeric storage

| | SQLite | PostgreSQL |
|---|---|---|
| Native decimal type | None. SQLite has only INTEGER and REAL (8-byte IEEE-754 floating point) numeric storage classes; "NUMERIC" is a type *affinity* (a conversion hint), not a distinct fixed-precision storage class. Text↔REAL conversion preserves "about 15.95 significant decimal digits" ([sqlite.org/datatype3.html](https://www.sqlite.org/datatype3.html)) | Dedicated `numeric`/`decimal` type: "especially recommended for storing monetary amounts and other quantities where exactness is required... yield exact results where possible," with precision up to "131072 digits before the decimal point; up to 16383 digits after" ([postgresql.org/docs/current/datatype-numeric.html](https://www.postgresql.org/docs/current/datatype-numeric.html)) |
| Tradeoff noted in the source | SQLite's docs don't discuss currency storage directly; storing values as TEXT or as integer minor-units (cents) is the common workaround for exact decimal amounts on SQLite, given the REAL-only floating-point option | Postgres's own docs flag a performance cost for exactness: "calculations on `numeric` values are very slow compared to the integer types, or to the floating-point types" ([postgresql.org/docs/current/datatype-numeric.html](https://www.postgresql.org/docs/current/datatype-numeric.html)) |

Multi-currency amounts need either exact decimal storage or an integer-minor-units convention; Postgres has a built-in exact-decimal type documented for this purpose, while SQLite has no equivalent native type and would rely on an application-level convention (integer cents, or text-encoded decimals) to get equivalent exactness.

---

## 3. Deploy target

**Candidates compared:** Netlify (serverless/JAMstack host), self-host (home server or VPS), fully local (localhost only, no deploy). Tailscale and Cloudflare Tunnel are noted as a "remote access to a home server without exposing it publicly" middle ground. This app holds real financial data (account balances, positions, mortgage details), which the task treats as a genuine factor, not just a convenience tradeoff.

### Netlify

Netlify Functions run in "an ephemeral runtime environment" ([docs.netlify.com/build/functions/overview](https://docs.netlify.com/build/functions/overview/)) and are also documented as immutable once deployed ("an update to a function on your production branch won't change the version that was deployed in a branch deploy" — same page). Netlify's own data-and-storage docs steer persistent-data needs toward two managed primitives instead of a function-local file: **Netlify Database** (managed Postgres, "optimized for complex queries and relationships") and **Netlify Blobs** (key/value store, explicitly framed as "a data store for functions" so that e.g. Background Functions can "persist the output of those computations") ([docs.netlify.com/build/data-and-storage/overview](https://docs.netlify.com/build/data-and-storage/overview/), [Netlify Blobs docs](https://docs.netlify.com/build/data-and-storage/netlify-blobs/)). Netlify Blobs' own docs describe it as optimized for "frequent reads and infrequent writes" with eventual consistency by default ("Updates and deletions propagate globally within 60 seconds," with an opt-in strongly-consistent mode), last-write-wins concurrency, and no SQL query capability — the docs point complex/relational/concurrent workloads at "third-party databases" instead ([Netlify Blobs docs](https://docs.netlify.com/build/data-and-storage/netlify-blobs/)). None of the Netlify pages fetched here mention SQLite-file storage as a supported persistence option; the ephemeral-function-runtime model plus the Blobs/Database docs' own framing point away from a bundled SQLite file surviving on Netlify's own infrastructure. A locally-embedded SQLite/libSQL file therefore does not fit Netlify's function model directly — the `my-worth` reference stack's `@libsql/client` on Netlify presumably works by pointing at a remote libSQL/Turso endpoint rather than a Netlify-local file (consistent with Turso's own description of itself as the hosted/synced path for libSQL, [docs.turso.tech/introduction](https://docs.turso.tech/introduction)), though that specific wiring isn't itself documented on the Netlify pages fetched.

Netlify's pricing page lists a Free tier (Functions, AI models, Blob storage, 300 monthly credits), Personal at $9/mo (1,000 credits) and Pro at $20/mo (3,000 credits, 3+ concurrent builds), with usage billed against credits — e.g. "Bandwidth" at "20 credits per GB," "Production deploys" at "15 credits each" ([netlify.com/pricing](https://www.netlify.com/pricing/)). Streaming functions specifically carry "a 60-second execution limit and a 20 MB response size limit" ([docs.netlify.com/build/functions/api](https://docs.netlify.com/build/functions/api/)); the standard (non-streaming) function default timeout is referenced by the docs but wasn't found stated on the pages fetched.

Data-sensitivity framing: hosting a personal-finance dataset (balances, positions, mortgage terms) on a third-party managed platform means that platform's security posture, data-retention, and breach-exposure profile become part of the app's own risk surface — a factor distinct from Netlify's technical capabilities, and one the Netlify docs fetched here don't themselves address (this is a general observation, not a sourced claim from Netlify's docs).

### Self-host (home server / VPS)

A VPS (e.g. DigitalOcean's "Droplet") is described in the provider's own docs as "Linux-based virtual machines (VMs) that run on top of virtualized hardware," deployable independently or as part of larger infrastructure ([docs.digitalocean.com/products/droplets](https://docs.digitalocean.com/products/droplets/)). Entry-level pricing on DigitalOcean's own pricing page starts at "$4.00" per month for "512 MiB" RAM / "1 vCPU" / "10 GiB" SSD ([digitalocean.com/pricing/droplets](https://www.digitalocean.com/pricing/droplets)) — persistent disk, so a SQLite file (or Postgres) can live on-instance without the ephemeral-storage question that applies to Netlify Functions. A VPS or home server puts OS patching, backups, and exposure/firewall configuration in the operator's own hands rather than a managed platform's — this is the standard self-host tradeoff (zero-ops from a code-deploy perspective, but not zero-ops from an infrastructure/security perspective).

For remote access to a home server without exposing it directly to the public internet, two documented middle-ground approaches:
- **Tailscale**: creates "a peer-to-peer mesh network (known as a tailnet)" over WireGuard, giving "encrypted point-to-point connections" so that "only devices on your private network can communicate with each other" ([tailscale.com/kb/1151/what-is-tailscale](https://tailscale.com/kb/1151/what-is-tailscale)) — no inbound ports opened on the home network at all; access is restricted to devices enrolled in the tailnet.
- **Cloudflare Tunnel**: a local daemon (`cloudflared`) "creates outbound-only connections to Cloudflare's global network," so the firewall can "allow only these outbound connections and block all inbound traffic" while still serving "HTTP web servers, SSH servers, remote desktops, and other protocols" through Cloudflare ([developers.cloudflare.com/cloudflare-one/connections/connect-networks](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/)) — traffic can be made public through Cloudflare's edge, or restricted to authenticated users via Cloudflare Access, depending on configuration.

Both keep the home server's inbound ports closed while still allowing remote access; Tailscale scopes access to devices on the operator's own tailnet by default, while Cloudflare Tunnel can additionally front the service for broader/public access if desired. Neither eliminates the "operator is responsible for patching the box" tradeoff inherent to self-hosting.

### Fully local (no deploy)

Running only on `localhost` removes the network-exposure question entirely — no third-party platform holds the data, and no port is opened for remote access. The tradeoff is the converse of the above: no remote/mobile access to the app away from the machine it runs on, and the operator is the sole backup mechanism (no managed platform's durability guarantees apply). This option wasn't researched against a specific "official docs" source since it is the absence of a deploy target rather than a product with its own documentation; it's included here only for completeness of the comparison as instructed by the research scope.

### Summary of the deploy tradeoff shape

| | Netlify | Self-host (VPS/home server) | Fully local |
|---|---|---|---|
| Remote/mobile access | Yes, by default | Yes, if opened up (directly, or via Tailscale/Cloudflare Tunnel) | No |
| Ops burden | Managed platform; app-level code only | Operator patches OS, manages backups, manages exposure | None (no server to run) |
| Cost | Free tier available; usage-based credits beyond it ([netlify.com/pricing](https://www.netlify.com/pricing/)) | From ~$4-5/mo for a small VPS, or free if repurposing existing home hardware ([digitalocean.com/pricing/droplets](https://www.digitalocean.com/pricing/droplets)) | Free (existing hardware) |
| SQLite-file persistence fit | Not a documented supported pattern for Netlify Functions' ephemeral runtime — Netlify's own storage docs steer toward Netlify DB (Postgres) or Blobs instead ([docs.netlify.com/build/data-and-storage/overview](https://docs.netlify.com/build/data-and-storage/overview/)) | Native fit — persistent disk, matches SQLite's own "competes with fopen()" framing ([sqlite.org/whentouse.html](https://www.sqlite.org/whentouse.html)) | Native fit, same as self-host |
| Data-sensitivity surface | Financial data held on a third-party managed platform | Financial data held on hardware/infrastructure the operator controls (and secures) | Financial data never leaves the local machine |

---

## Sources

**Frontend**
- [react.dev/learn](https://react.dev/learn) — React Quick Start
- [react.dev/learn/creating-a-react-app](https://react.dev/learn/creating-a-react-app) — framework recommendations
- [vite.dev/guide/](https://vite.dev/guide/) — Vite Getting Started
- [vite.dev/guide/why](https://vite.dev/guide/why) — Vite philosophy / Rolldown
- [svelte.dev/docs/kit/introduction](https://svelte.dev/docs/kit/introduction) — SvelteKit introduction
- [start.solidjs.com/getting-started/what-is-solidstart](https://start.solidjs.com/getting-started/what-is-solidstart) — SolidStart overview

**Database**
- [sqlite.org/whentouse.html](https://www.sqlite.org/whentouse.html) — SQLite appropriate use cases
- [sqlite.org/json1.html](https://www.sqlite.org/json1.html) — SQLite JSON1 extension
- [sqlite.org/datatype3.html](https://www.sqlite.org/datatype3.html) — SQLite storage classes/type affinity
- [postgresql.org/docs/current/datatype-json.html](https://www.postgresql.org/docs/current/datatype-json.html) — Postgres JSON/JSONB
- [postgresql.org/docs/current/datatype-numeric.html](https://www.postgresql.org/docs/current/datatype-numeric.html) — Postgres NUMERIC/DECIMAL
- [github.com/tursodatabase/libsql](https://github.com/tursodatabase/libsql) — libSQL README
- [docs.turso.tech/introduction](https://docs.turso.tech/introduction) — Turso introduction

**Deploy**
- [docs.netlify.com/build/functions/overview](https://docs.netlify.com/build/functions/overview/) — Netlify Functions overview
- [docs.netlify.com/build/functions/api](https://docs.netlify.com/build/functions/api/) — Netlify Functions API reference / limits
- [docs.netlify.com/build/data-and-storage/overview](https://docs.netlify.com/build/data-and-storage/overview/) — Netlify data/storage overview
- [docs.netlify.com/build/data-and-storage/netlify-blobs](https://docs.netlify.com/build/data-and-storage/netlify-blobs/) — Netlify Blobs
- [netlify.com/pricing](https://www.netlify.com/pricing/) — Netlify pricing
- [docs.digitalocean.com/products/droplets](https://docs.digitalocean.com/products/droplets/) — DigitalOcean Droplets
- [digitalocean.com/pricing/droplets](https://www.digitalocean.com/pricing/droplets) — DigitalOcean Droplet pricing
- [tailscale.com/kb/1151/what-is-tailscale](https://tailscale.com/kb/1151/what-is-tailscale) — Tailscale overview
- [developers.cloudflare.com/cloudflare-one/connections/connect-networks](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/) — Cloudflare Tunnel

**Reference stack (context only, not itself a decision input)**
- `/Users/andres/personal/my-worth/package.json` — the "familiar stack" data point (React 18 + Vite 6 + Hono + `@libsql/client` + Drizzle + Netlify)
