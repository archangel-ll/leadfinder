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

## 16. Self-improvement loop

The system should get *better at scraping and at lead quality* over time instead of
silently rotting when a source changes its HTML. Three feedback loops, each with a
measurable signal and an automatic or semi-automatic correction.

### 16.1 Adapter drift detection → self-healing extraction

- **Baseline per source.** Each adapter records expected field fill-rates and
  leads-per-run on healthy runs (e.g. "≥90% of leads have a phone").
- **Detect.** A run that drops below threshold (fewer leads, collapsed fill-rates,
  parse exceptions) flags the adapter as `degraded` on its `scrape_runs` row.
- **Heal (LLM-assisted).** On a degraded run, fall back to an **LLM extractor**:
  pass the raw HTML to **Claude Opus 4.8** (`claude-opus-4-8`) and ask it to return
  a `RawLead[]` via **structured outputs** (`output_config.format` with a JSON
  schema — guarantees parseable output). This keeps leads flowing while the
  deterministic selectors are broken.
- **Promote.** A successful LLM extraction also asks Claude to emit the CSS/XPath
  selectors it effectively used. We surface those as a proposed patch to the
  adapter (human-reviewed before merge) — the deterministic parser is repaired from
  the model's findings, so the expensive LLM path is temporary, not permanent.

```python
# fallback extractor — runs only when an adapter is flagged `degraded`
import anthropic
client = anthropic.Anthropic()

resp = client.messages.create(
    model="claude-opus-4-8",
    max_tokens=16000,
    thinking={"type": "adaptive"},
    system=[{                      # stable prompt + schema → cache across pages
        "type": "text",
        "text": EXTRACTION_INSTRUCTIONS,
        "cache_control": {"type": "ephemeral"},
    }],
    output_config={"format": {"type": "json_schema", "schema": RAW_LEAD_ARRAY_SCHEMA}},
    messages=[{"role": "user", "content": page_html}],
)
```

Notes: the system prompt + schema are identical across every page, so a
`cache_control` breakpoint makes repeated extractions ~90% cheaper on the cached
prefix. For very large pages, switch to `client.messages.stream(...)` +
`get_final_message()` to avoid HTTP timeouts.

### 16.2 Lead-quality judge → source reprioritization

- An **LLM judge** (also `claude-opus-4-8`, structured output) scores a sample of
  new leads against a rubric: is this a real business, is contact info plausible,
  is it in the target category? Returns `{score, reasons}` per lead.
- Aggregate judge scores per source/category feed a **priority weight** on
  `sources`. High-quality sources get scraped more; low-quality ones get throttled
  or flagged for removal.
- Cheap variant: send judging in a nightly **Batch API** job (50% cost) since it's
  not latency-sensitive.

### 16.3 Evaluation harness (makes the loop trustworthy)

- **Golden fixtures** per source (recorded HTML + expected `RawLead`).
- Metrics tracked over time: extraction precision/recall vs. golden, dedup
  false-merge rate, judge-score distribution.
- Every adapter change (including LLM-proposed selector patches) is gated on the
  harness — so "self-improvement" can never silently regress quality.

> Guardrail: the LLM paths are **fallback and evaluation**, never the default
> hot path. Deterministic parsers run first; the model is invoked only on drift,
> on a sample for judging, or in batch. This caps cost and keeps runs fast.

## 17. Other recommendations worth folding in

- **Incremental / scheduled crawls.** Track `last_seen_at` per lead; re-scrape on a
  cadence and only upsert changes rather than re-ingesting everything. Cron via a
  simple scheduler now; a real queue later.
- **Page cache + conditional fetch.** Cache fetched HTML (with ETag/Last-Modified)
  so re-runs and the LLM fallback don't re-hit the source — cheaper and gentler.
- **Adapter health alerting.** When a source flips to `degraded`, surface it in the
  Runs UI and optionally notify (the loop in §16.1 is only useful if someone sees it).
- **Enrichment as a later phase.** Once aggregation is solid, add optional
  enrichment (email/phone validation, firmographics) behind the same adapter idea.
- **Idempotency + provenance everywhere.** Every lead already carries `source_url`
  + `scraped_at`; add a stable `dedup_key` and keep `raw` history so any record is
  traceable and removable (also satisfies the opt-out guardrail in §4).
- **Cost guardrails on the LLM loop.** Per-source budget caps, batch judging, and
  prompt caching keep the self-improvement loop from becoming a cost sink.

## 18. Risks

- **Source layout drift** breaks HTML adapters → mitigated by fixture tests + alerting.
- **Rate limits / blocking** → conservative throttling, API-first sourcing.
- **Data quality / dedup false-merges** → explicit `dedup_key`, keep `raw` for audit.
- **Legal/ToS** → API-first, allowlist, provenance, opt-out support.
