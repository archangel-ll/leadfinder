# leadfinder — Project Plan

> Status: **planning** (no code yet). This document is the source of truth for
> scope and design. We build only after the open decisions in §15 are settled.

## 1. Summary

leadfinder scrapes public business listings, normalizes and de-duplicates them,
and aggregates them into a searchable lead database with filtering and export.
Single-purpose: **find and organize B2B leads from public sources**.

## 2. Goals

- Pull leads (company + contact data) from one or more public sources.
- Normalize heterogeneous source data into one clean schema.
- De-duplicate aggressively so the same business isn't listed twice.
- Let a user search/filter leads and export the result (CSV/JSON).
- Make adding a new source a small, well-bounded amount of work (adapter pattern).

## 3. Non-goals (for v1)

- Email/outreach sequencing, CRM pipeline, calling — out of scope for v1.
- Scraping anything behind a login or paywall, or circumventing access controls.
- Real-time/continuous crawling. v1 runs on-demand or scheduled batches.
- Lead scoring / AI enrichment. Possible later; not v1.

## 4. Legal & ethical guardrails (non-negotiable, built in from day one)

- **Public data only.** No authenticated pages, no access-control bypass.
- **Respect `robots.txt`** and any source-specific crawl directives.
- **Per-source rate limiting + exponential backoff.** Never hammer a host.
- **Source allowlist.** A source must be explicitly added; no open crawling.
- **Prefer official APIs** where they exist (e.g. Google Places) over HTML scraping,
  even at a cost, when ToS favors it.
- **Honor takedown / opt-out.** Store provenance (`source`, `scraped_at`, `source_url`)
  on every lead so any record can be traced and removed.
- Each source adapter documents its legal basis (API ToS vs. public HTML) before merge.

## 5. Architecture

```
            ┌─────────────────────────────────────────┐
            │              Next.js app                 │
            │  ┌────────────┐        ┌──────────────┐  │
            │  │  UI (RSC)  │        │  API routes  │  │
            │  │  leads tbl │◄──────►│ /scrape /     │  │
            │  │  runs view │        │ leads /export │  │
            │  └────────────┘        └──────┬───────┘  │
            └───────────────────────────────┼──────────┘
                                             │
                       ┌─────────────────────┼───────────────────┐
                       │                     │                   │
               ┌───────▼───────┐    ┌────────▼────────┐   ┌──────▼──────┐
               │ Source adapters│    │  Normalizer +   │   │  Supabase   │
               │ (cheerio /     │───►│  dedup engine   │──►│  (Postgres) │
               │  Playwright)   │    │                 │   │             │
               └────────────────┘    └─────────────────┘   └─────────────┘
```

- **Adapters** fetch + parse a single source into a common `RawLead` shape.
- **Normalizer** maps `RawLead` → canonical `Lead`, computes a `dedup_key`.
- **Dedup engine** upserts on `dedup_key`, merging fields (prefer non-null/newer).
- **Storage** is Postgres via Supabase.

## 6. Tech stack

| Concern        | Choice                                   | Why |
|----------------|------------------------------------------|-----|
| App framework  | Next.js (App Router) + TypeScript        | One repo for UI + API; RSC for fast tables |
| DB             | Postgres via Supabase                    | Managed, RLS, easy auth; MCP tooling available |
| Static scrape  | `fetch` + `cheerio`                      | Cheap, fast for static HTML |
| Dynamic scrape | Playwright (chromium)                    | Needed for JS-rendered listings |
| Validation     | `zod`                                    | Validate RawLead + API payloads |
| Jobs (v1)      | API route + `scrape_runs` row + cron     | Keep it simple; no broker yet |
| Jobs (later)   | Queue (e.g. pg-boss / Supabase queue)    | When volume/concurrency demands it |
| Export         | CSV (papaparse) / JSON                   | Standard formats |
| Tests          | Vitest + Playwright test + recorded fixtures | Unit on normalizer/dedup; fixtures for adapters |

## 7. Data model

`sources`
- `id`, `slug`, `name`, `kind` (`api` | `html`), `config` (jsonb), `enabled`, `created_at`

`leads`
- `id`, `company_name`, `website`, `email`, `phone`, `address`, `city`, `region`,
  `postal_code`, `country`, `category`, `contact_name`, `contact_title`
- `source_id` (fk), `source_url`, `scraped_at`
- `dedup_key` (text, unique index), `raw` (jsonb, original payload)
- `created_at`, `updated_at`

`scrape_runs`
- `id`, `source_id` (fk), `status` (`queued`|`running`|`success`|`failed`),
  `started_at`, `finished_at`, `leads_found`, `leads_new`, `leads_updated`,
  `error` (text), `params` (jsonb)

Indexes: `leads(dedup_key)` unique; `leads(category)`, `leads(city,region)`,
trigram index on `company_name` for search; `scrape_runs(source_id, started_at)`.

## 8. Source adapter design

```ts
interface SourceAdapter {
  slug: string;
  kind: "api" | "html";
  // Yields raw leads for the given search params, respecting rate limits.
  run(params: ScrapeParams, ctx: AdapterContext): AsyncIterable<RawLead>;
}
```

- `AdapterContext` provides a rate-limited fetcher, logger, and `robots.txt` check.
- Adapters live in `src/sources/<slug>/`. Each ships a recorded HTML/JSON fixture
  + a parser unit test so we catch source layout changes.
- Adding a source = implement `SourceAdapter`, register it, add a `sources` row.

## 9. Normalization & dedup

- **Normalize:** lowercase/trim, canonicalize website to bare domain, E.164 phones,
  uppercase country codes, collapse whitespace in addresses.
- **dedup_key** (priority order): normalized domain → else normalized phone →
  else `slug(company_name)+postal_code`. Stored explicitly so logic can evolve.
- **Merge on conflict:** keep existing non-null fields, fill blanks from new record,
  prefer newer `scraped_at` for changed values, always append to `raw` history.

## 10. API surface (Next.js route handlers)

- `POST /api/scrape` — body `{ source, params }` → creates a `scrape_run`, kicks job.
- `GET  /api/runs` / `GET /api/runs/:id` — status + counts.
- `GET  /api/leads` — query: `q`, `category`, `city`, `source`, `page`, `pageSize`.
- `GET  /api/export?format=csv|json&<same filters>` — streamed download.

## 11. UI

- **Leads** page: server-rendered table, search box, category/city/source filters,
  pagination, row detail drawer (shows `source_url`, `scraped_at`, raw).
- **Runs** page: trigger a scrape (pick source + params), live status, result counts.
- **Export** button respects current filters.
- Minimal styling (Tailwind). Function over polish for v1.

## 12. Cross-cutting

- **Auth:** Supabase Auth, single-tenant for v1 (one workspace). RLS scaffolded so
  multi-tenant is a smaller lift later.
- **Observability:** structured logs per run; errors captured on the `scrape_runs` row;
  surfaced in the Runs UI.
- **Config/secrets:** `.env` (API keys per source, Supabase keys). Never committed.
- **Error handling:** adapter failures fail the run gracefully with a message, never
  crash the app; partial results already written are kept.

## 13. Testing strategy

- Unit: normalizer + dedup (the highest-risk logic) — table-driven tests.
- Adapter: parse against recorded fixtures; assert RawLead output. No live network in CI.
- E2E (smoke): trigger a run against a fixture source, assert rows land + dedup holds.

## 14. Milestones / build order

1. **M1 — Scaffold:** Next.js + TS, Supabase client, schema migration, env wiring, CI.
2. **M2 — First source end-to-end:** one adapter → normalize → dedup upsert. Unit tests.
3. **M3 — Leads UI:** searchable/filterable table + detail drawer.
4. **M4 — Runs:** trigger scrape from UI, status + counts.
5. **M5 — Export:** CSV/JSON of filtered leads.
6. **M6 — Second source:** validate the adapter abstraction; refactor if it strains.
7. **M7 — Hardening:** rate-limit tuning, robots checks, error UX, docs.

Each milestone is a separate commit set; we review before moving on.

## 15. Open decisions (must resolve before M2)

1. **First source(s).** Candidates: Google Places API (official, paid, ToS-clean),
   Yelp Fusion API, OpenStreetMap/Overpass (free, generous license), a niche/industry
   directory, Yellow Pages. **Recommendation:** start with an API-based source
   (Google Places or Overpass) to keep M2 clean and ToS-safe before tackling HTML.
2. **Supabase vs. local Postgres.** Recommendation: Supabase (managed, MCP tooling
   ready). Confirm, or pick local Docker Postgres.
3. **Single vs. multi-tenant** for v1. Recommendation: single-tenant, RLS-ready.
4. **Hosting target** (Vercel + Supabase assumed). Confirm.

## 16. Risks

- **Source layout drift** breaks HTML adapters → mitigated by fixture tests + alerting.
- **Rate limits / blocking** → conservative throttling, API-first sourcing.
- **Data quality / dedup false-merges** → explicit `dedup_key`, keep `raw` for audit.
- **Legal/ToS** → API-first, allowlist, provenance, opt-out support.
