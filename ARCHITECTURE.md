# Architecture

AI Audience Intelligence & Retargeting Platform (working name: **Audience IQ**).

This document covers items 1–13 of the build brief's "First Task": repository analysis, proposed architecture, module breakdown, API design, security model, roadmap, external API requirements, risks and blockers. The database design lives in [DATABASE.md](DATABASE.md).

---

## 1. Repository analysis

| Item | Finding |
|---|---|
| Existing repository | None. The workspace was empty and no repository was supplied, so this is a green-field build. |
| Existing framework / DB / auth / UI / APIs | None to reuse. |
| Coding standards | Adopted: PSR-12 via Laravel Pint, strict types in domain code, PHPStan level 6 target (Larastan). |
| Deployment setup | None. Docker Compose and a cPanel-compatible "lite" mode are defined in [DEPLOYMENT.md](DEPLOYMENT.md). |
| Security issues | None inherited. |
| Framework version | **Laravel 13** (`^13.0`, PHP ≥ 8.3). The brief says "Laravel 12+"; Laravel 12 left bug-fix support in August 2026, so starting on 12 would mean an immediate upgrade. |

---

## 2. Guiding principles

1. **Intelligence layer, not a pixel.** Meta Pixel / Google tag remain the activation channels. We decide *who* and *why*; the ad platforms do identity matching.
2. **Deterministic first, AI second.** Metrics, scores, audience rules and attribution are computed in code and SQL. AI only classifies, explains, names, groups and recommends, and every AI output must cite evidence rows computed deterministically (see §7.4).
3. **Tenant isolation is defence-in-depth.** Every tenant row carries `organization_id`; isolation is enforced by a global scope, by policies, and by tests. Any one layer failing must not leak data.
4. **Privacy by construction.** Anonymous IDs, consent-gated processing, no fingerprinting, no sensitive-attribute inference, configurable retention, DSAR export/delete.
5. **Hot path is cheap.** The collection endpoint does no ORM work and no synchronous DB writes; it validates, enqueues and returns `202`.
6. **Adapters at every external boundary** (ad platforms, AI provider, CRM, SEO data providers) so API changes stay local.

---

## 3. System overview

```mermaid
flowchart LR
  subgraph Client website
    SDK[tracker.js<br/>anonymous ids, consent]
    PX[Meta Pixel / Google tag<br/>(client's own)]
  end

  SDK -- beacon /collect --> EDGE[Collector endpoint<br/>validate + bot filter]
  EDGE -- LPUSH --> RQ[(Redis ingest stream)]
  RQ --> ING[Ingest workers<br/>batch insert]
  ING --> RAW[(MySQL raw tables<br/>partitioned by month)]
  ING --> VS[Visitor state updater]
  VS --> SCORE[Scoring engine]
  SCORE --> AUD[Audience engine]
  RAW --> ROLL[Hourly / daily rollups]
  ROLL --> API[REST API v1]
  AUD --> API
  API --> SPA[React dashboard]

  subgraph Integrations
    META[Meta Marketing API / CAPI]
    GADS[Google Ads API]
    GSC[Search Console API]
    GA4[GA4 Data API]
    CRM[CRM adapters]
  end
  AUD -- segment rules --> META
  AUD -- segment rules --> GADS
  SDK -. "segment signal (ad consent only)" .-> PX
  GSC --> ROLL
  GADS --> ROLL
  META --> ROLL
  CRM --> ING

  ROLL --> AI[AI layer<br/>evidence-grounded]
  AI --> API
```

### Runtime components

| Component | Tech | Scales by |
|---|---|---|
| Web/API | Laravel 13 on PHP-FPM behind Nginx | Stateless; add nodes behind LB |
| Collector | Same codebase, dedicated route group without session/CSRF/cookies middleware; optionally a separate pool | Add nodes; Redis absorbs bursts |
| Queues | Redis + Horizon; queues: `ingest`, `scoring`, `audiences`, `integrations`, `ai`, `reports`, `default` | Add workers per queue |
| Scheduler | `schedule:run` on one node (with `onOneServer()`) | n/a |
| Primary DB | MySQL 8 (InnoDB, utf8mb4) | Read replicas for dashboards; partitioned event tables |
| Cache/locks | Redis | Redis Cluster / separate instances for cache vs queue |
| Object storage | S3-compatible (reports, exports, logos) | n/a |
| Frontend | React + TypeScript + Vite + Tailwind + Recharts SPA, served by Laravel | CDN |

---

## 4. Tenancy model

```
Platform (super admins)
 └── Organization type=agency          ← AGENCY_ADMIN, billing owner
      └── Organization type=client     ← CLIENT_ADMIN, CLIENT_USER
           └── Website → tracking, visitors, events, audiences, campaigns, reports
```

A business that is not an agency gets an **agency organization with a single client** created at signup, so there is one code path.

- `organizations(type: agency|client, parent_id)`. Client orgs always have an agency parent; agencies have none.
- `organization_users(organization_id, user_id, role)` where role ∈ `agency_admin | client_admin | client_user`.
- `users.is_super_admin` for platform staff (not an org role, so it can never be granted by an agency).
- **Data-owning tenant = the client organization.** Every tenant table carries `organization_id` of the client (denormalized even when derivable through `website_id`) so the global scope and composite indexes work without joins.

### Access rules

| Role | Can access |
|---|---|
| SUPER_ADMIN | Platform staff flag on `users` (granted only by `php artisan aiq:super-admin`). Uses `/api/v1/admin/*`, and may select any organization as tenant; every such access to an org they aren't a member of is audit-logged (`admin.tenant_accessed`). |
| AGENCY_ADMIN | Their agency org + all client orgs whose `parent_id` = their agency. |
| CLIENT_ADMIN | Their client org; manage users, websites, integrations, audiences. |
| CLIENT_USER | Their client org; read + limited write (cannot manage users, integrations, API keys, billing). |

### Enforcement layers

1. **Tenant context** (`App\Support\Tenancy\TenantContext`): resolved per request by `ResolveTenant` middleware from the `X-Organization-Id` header or the `{organization}` route parameter, and verified against the user's accessible orgs. API keys pin the context to their own org.
2. **Global scope** (`BelongsToOrganization` trait): every tenant model auto-filters by the current tenant and auto-fills `organization_id` on create. If no tenant is set, tenant models **return nothing** (fail closed) unless code explicitly opts out with `withoutTenantScope()`, which is reserved for queue jobs that set context themselves and for super-admin services.
3. **Policies** on every model, checked in controllers (`authorize`) and Form Requests.
4. **Queue jobs** carry `organization_id` and re-establish `TenantContext` in `handle()` via the `TenantAware` job middleware.
5. **Tests**: a cross-tenant matrix test iterates every tenant API route and asserts 404/403 for a foreign org (see `tests/Feature/Tenancy`).

Selecting an organization you cannot access returns **403 with the same message whether or not it exists**. Resource IDs from another tenant (websites, API keys, invitations…) return **404** because the scoped lookup never finds them. Either way, nothing confirms that another tenant's record exists.

---

## 5. Module breakdown

Code is organised by domain under `app/Domain/*` (models, services, DTOs, events, jobs, policies), with thin HTTP controllers in `app/Http`.

| Module | Responsibility | Phase |
|---|---|---|
| **Tenancy** | Organizations, memberships, roles, tenant context, invitations | 1 |
| **Identity** | Users, Sanctum auth, API keys, 2FA (later), sessions | 1 |
| **Audit** | Append-only audit log of security-relevant actions | 1 |
| **Websites** | Websites, site keys, allowed tracking domains, installation status | 1 |
| **Privacy** | Consent records, retention policies, DSAR export/delete, anonymisation jobs | 1 (policies) / 2 |
| **Tracking SDK** | `tracker.js` (vanilla TS → ES2017, < 20 KB min) | 2 |
| **Ingestion** | `/collect` endpoint, validation, bot filtering, Redis buffer, batch writer, dedup | 2 |
| **Visitor Intelligence** | Visitor/session state, page classification (pricing/product/service rules), timeline | 2–3 |
| **Analytics** | Rollups, dashboard metrics, funnels, traffic sources, conversions | 3 |
| **Attribution** | First/last/linear/position-based; data-driven only above sample thresholds | 3 |
| **Scoring** | Configurable signal weights → intent, engagement, readiness, conversion probability, each with explanation | 4 |
| **Audiences** | Rule DSL → SQL compiler, predefined audiences, memberships, overlap, discovery | 4 |
| **Integrations** | Adapter interfaces + Meta, Google Ads, GSC, GA4, CRM adapters; token vault | 5–6 |
| **Activation** | Audience → platform destination workflow, status monitoring | 6 |
| **AI** | Provider abstraction, evidence packs, classification, recommendations, analyst chat | 7 |
| **Competitors** | Public-site crawler (robots.txt-respecting), SEO provider adapter, gap analyses | 8 |
| **Automation** | Alerts, scheduled reports (PDF/CSV), nightly discovery | 9 |
| **Billing** | Plans (DB-driven), subscriptions, metering, limits | 10 (schema in 1) |
| **White label** | Branding, custom domains | 10 |
| **Admin** | Super-admin console, feature flags, system health | 1 (skeleton) → 10 |

---

## 6. Data flow and performance design

### 6.1 Ingestion hot path

1. SDK batches events (max 20 or 5 s, flushed on `pagehide` via `navigator.sendBeacon`) to `POST /collect/v1`.
2. Collector middleware stack is minimal: no session, cookies, CSRF or auth. It resolves `site_key` → website config from Redis cache (60 s TTL), checks the `Origin` against allowed domains, rejects oversized payloads (> 32 KB), and applies a per-site rate limit.
3. Payload is schema-validated, bots are filtered (UA list + behaviour rules, configurable per site), and consent state is applied: `denied` → drop everything except an aggregate counter.
4. The validated batch goes onto a Redis stream `ingest:{shard}` and the endpoint returns `202`. Target p99 < 20 ms.
5. `IngestBatchWorker` pops ~500–2,000 events, dedups on `(website_id, event_uuid)` via Redis SET with TTL, and does multi-row `INSERT` into the raw tables in one transaction.
6. Visitor/session counters are upserted (`INSERT … ON DUPLICATE KEY UPDATE`), and visitors are marked dirty for re-scoring.

### 6.2 Storage strategy

- Raw `events` and `page_views` are **RANGE partitioned by month** on `occurred_at`. MySQL does not allow foreign keys on partitioned tables, so integrity is enforced in the ingest layer and these tables carry `organization_id` + `website_id` as plain indexed columns.
- Retention deletes become `ALTER TABLE … DROP PARTITION` for platform-wide windows, plus batched deletes per tenant for shorter tenant policies.
- Dashboards never query raw tables. Rollups are `stats_hourly_*` and `stats_daily_*` (by website × dimension). Raw queries are only allowed for visitor timelines (indexed by visitor) and for scheduled jobs.
- **Scale path:** the analytics read layer sits behind `AnalyticsStore` and `EventStore` interfaces. MySQL covers the 100 → ~1,000 website tier. Past roughly 500 M events, or heavy ad-hoc segmentation, add a ClickHouse implementation of the same interfaces without touching domain code (see Risks).

### 6.3 Scoring and audiences

- Dirty visitors are re-scored in batches by `ScoreVisitorsJob` every minute and nightly in full (score decay is time-based).
- The scoring engine is **pure PHP over a `VisitorFeatures` DTO**: deterministic, unit-tested, with per-organization weights stored as JSON (`scoring_models`) and each contribution recorded as an explanation line.
- `conversion_probability` uses a logistic model trained per website only when it has ≥ N conversions (default 50). Otherwise it is `null` with reason "insufficient data". Nothing is fabricated.
- Audience rules are a JSON AST (`all/any/not` groups of conditions), compiled to parameterized SQL against `visitors` + `visitor_features`. Membership is materialised in `audience_memberships` with `entered_at`/`exited_at` for growth history.

---

## 7. AI architecture

### 7.1 Where AI is used

Classification labels (intent class, search-query intent), pattern explanation, recommendation text, audience naming, topic clustering of GSC queries and competitor content, and the analyst chat.

### 7.2 Where AI is never used

Counts, rates, lifts, CPA/ROAS, scores, audience membership, attribution, and statistical significance. These come from SQL/PHP.

### 7.3 Provider

The `AiProvider` interface is implemented first with the Laravel AI SDK (Laravel 13) configured for Anthropic Claude. The provider and model come from `.env`, so it can be swapped without code changes. Every call is logged to `ai_requests` (tokens, latency, org, purpose) for metering and the admin AI-usage view.

### 7.4 Evidence grounding (anti-hallucination contract)

1. A deterministic **EvidenceBuilder** produces a JSON evidence pack of named metrics with IDs, e.g. `{"id":"m3","metric":"lead_rate","segment":"pricing_visitors","value":0.071,"baseline":0.017,"n":1840}`.
2. The model receives only the evidence pack and must return structured JSON: `title, insight, evidence_refs[], confidence, recommended_action, expected_objective`.
3. A **validator** rejects any output that cites unknown evidence IDs, contains numbers not present in the referenced evidence (numeric-token check), or has confidence above what the sample-size rule allows.
4. Below the minimum-sample thresholds, the system returns "Insufficient data" without calling the model at all.
5. The analyst chat uses tool calls against a fixed set of read-only, tenant-scoped query tools (no free-form SQL). Answers carry the tool results they used.

### 7.5 Audience discovery (nightly)

Candidate segments come from combinations of up to three behavioural predicates. Each needs a minimum segment size (default 100 visitors and ≥ 10 conversions). Lift is tested with a two-proportion z-test using a Benjamini–Hochberg correction across candidates. Survivors become `ai_recommendations` with status `proposed`. Nothing is activated without a human approving it.

---

## 8. API design (REST, versioned)

- Base: `/api/v1`. JSON, snake_case, ISO-8601 UTC timestamps, cursor pagination (`?cursor=`), and errors in `{ "message", "errors" }` form (Laravel validation shape).
- Auth:
  - **Dashboard SPA**: Sanctum cookie session (first-party), CSRF protected.
  - **Personal/integration tokens**: Sanctum bearer tokens with abilities.
  - **Server-to-server API keys**: `aiq_live_<prefix>_<secret>`. Only a SHA-256 hash is stored; keys are scoped to one org with abilities and an optional IP allowlist. Sent as `Authorization: Bearer`.
- Tenant selection: `X-Organization-Id` header for user tokens (validated); implicit for API keys.
- Rate limits: per token and per org, with tracking ingestion limited separately per site.

| Area | Endpoints (v1) | Phase |
|---|---|---|
| Auth | `POST /auth/register`, `POST /auth/login` (SPA session), `POST /auth/tokens`, `DELETE /auth/tokens/current`, `POST /auth/logout`, `GET /me` | 1 ✅ |
| Organizations | `GET /organizations`, `GET/PATCH /organizations/{id}`, `POST /organizations/{agency}/clients` | 1 ✅ |
| Members | `GET /organizations/{id}/members`, `PATCH/DELETE /organizations/{id}/members/{user}` | 1 ✅ |
| Invitations | `GET/POST /organizations/{id}/invitations`, `DELETE /organizations/{id}/invitations/{inv}`, `POST /invitations/accept` | 1 ✅ |
| Websites | `GET/POST /websites`, `GET/PATCH/DELETE /websites/{id}`, `POST /websites/{id}/rotate-key` | 1 ✅ |
| API keys | `GET/POST /api-keys`, `DELETE /api-keys/{id}`, `GET /key` (introspection with an API key) | 1 ✅ |
| Audit | `GET /organizations/{id}/audit-logs` | 1 ✅ |
| Retention | `GET/PUT /data-retention-policy` | 1 ✅ |
| Admin | `GET /admin/organizations`, `PATCH /admin/organizations/{id}/status`, `GET /admin/plans`, `PATCH /admin/plans/{code}` | 1 ✅ |
| Collection | `POST /collect/v1` (public, site key), `GET /tracker.js` | 2 |
| Visitors | `GET /visitors`, `GET /visitors/{id}`, `GET /visitors/{id}/timeline` | 3 |
| Sessions/Events | `GET /sessions`, `GET /events`, `POST /events` (server-side events via API key) | 2–3 |
| Conversions | `GET/POST /conversions` | 3 |
| Analytics | `GET /analytics/overview`, `/funnels`, `/sources`, `/pages` | 3 |
| Audiences | `GET/POST /audiences`, `GET/PATCH/DELETE /audiences/{id}`, `GET /audiences/{id}/members`, `POST /audiences/preview`, `POST /audiences/overlap` | 4 |
| Integrations | `GET /integrations`, `GET /integrations/{provider}/connect` (OAuth redirect), `GET /oauth/{provider}/callback`, `DELETE /integrations/{id}` | 5–6 |
| Activations | `POST /audiences/{id}/activations`, `GET /activations/{id}` | 6 |
| Campaigns | `GET /campaigns`, `GET /campaigns/{id}/metrics` | 5–6 |
| AI | `GET /recommendations`, `POST /recommendations/{id}/accept`, `POST /analyst/messages` | 7 |
| Competitors | `GET/POST /competitors`, `GET /competitors/{id}/analysis` | 8 |
| Reports/Alerts | `GET/POST /reports`, `GET /reports/{id}/download`, `GET/PATCH /alerts` | 9 |
| Webhooks | `GET/POST /webhooks` (outbound subscriptions), `POST /hooks/inbound/{source}` | 9 |
| Admin (later) | users, integrations, usage, AI usage, feature flags, system health | 10 |

---

## 9. Security model

| Concern | Control |
|---|---|
| AuthN | Sanctum (SPA cookie + tokens); bcrypt/argon2id passwords; login throttling; email verification; 2FA (TOTP) in Phase 10 |
| AuthZ | Policies per model + permission matrix (`OrganizationRole::permissions()`); policies also require the record to belong to the *active* tenant; super-admin console behind `super_admin` middleware |
| Tenant isolation | Tenant context + fail-closed global scope + policies + 404 on foreign IDs + cross-tenant test matrix |
| Secrets | `.env` only; OAuth tokens and webhook secrets stored with Laravel `encrypted` casts (AES-256-GCM via `APP_KEY`); key rotation via `APP_PREVIOUS_KEYS` |
| API keys | Shown once; SHA-256 hash stored; prefix for lookup; last-used tracking; revocation |
| Collector abuse | Origin allowlist per website, payload size limit, per-site and per-IP rate limits, bot filter, schema validation; site keys are public by design and grant write-only access to their own site |
| Webhooks in | HMAC signature verification (provider-specific), replay window, idempotency key table |
| Webhooks out | HMAC-SHA256 signed payloads, retries with backoff, delivery log |
| Web | CSRF, strict CSP, HSTS, `X-Content-Type-Options`, `Referrer-Policy`, `frame-ancestors` (SecurityHeaders middleware); React escapes output; no `dangerouslySetInnerHTML` |
| SQL | Eloquent/Query Builder bindings only; the audience compiler whitelists fields and operators and binds every value |
| Audit | Append-only `audit_logs` (actor, org, action, subject, IP, UA, changes) for auth, membership, key, integration, export, delete and admin actions |
| Privacy | IP truncated before storage (/24 v4, /48 v6) and used only for coarse geo; no fingerprinting; consent gating; retention jobs; DSAR export/delete by visitor ID |
| Sensitive inference | Prohibited by design: scoring features are an allow-list of behavioural signals; AI prompts include a policy block and outputs are screened for sensitive-category terms |

---

## 10. Implementation roadmap

| Phase | Scope | Exit criteria |
|---|---|---|
| **1 Foundation** | Laravel 13 app, Sanctum auth, orgs (agency/client), roles, tenant context, policies, websites, API keys, audit log, retention policy, plans/subscriptions schema, security headers, Docker, docs | Tenant-isolation test matrix green; CI runs Pint + tests |
| **2 Tracking** | `tracker.js`, `/collect`, Redis buffer, batch writer, visitors/sessions/page views/events, consent, internal search capture, bot filter, data-quality counters, tracking health | 1,000 events/s sustained on one node in a load test; SDK < 20 KB |
| **3 Analytics** | Rollups, dashboard, visitor explorer + timeline, funnels, traffic sources, conversions, attribution models | Dashboard p95 < 300 ms from rollups |
| **4 Audience engine** | Page classification rules, scoring engine + explanations, rule builder + compiler, predefined audiences, memberships, overlap, discovery statistics | Deterministic scoring tests; compiler fuzz tests |
| **5 Google** | OAuth, GSC sync, Google Ads reporting, GA4 (optional), rule-based remarketing lists | Live sandbox/test-account sync |
| **6 Meta** | OAuth, pixels, ad accounts, campaign insights, CAPI forwarding (consented), Website Custom Audiences via segment signal | Activation end-to-end in a test ad account |
| **7 AI** | Provider, evidence builder, validator, classifications, recommendations, analyst chat | Hallucination validator rejects fabricated numbers in tests |
| **8 Competitors** | Crawler, tech detection, content/landing-page gap, SEO provider adapter, audience hypotheses | Respects robots.txt; provider data labelled with its source |
| **9 Automation** | Alerts, scheduled reports (PDF/CSV), outbound webhooks, nightly discovery | |
| **10 SaaS** | Billing provider, metering enforcement, white label, custom domains, admin console, 2FA | |

---

## 11. External API requirements

These were checked against official documentation where reachable on 2026-10-08. Platform rules change often; each adapter's doc page (`META-INTEGRATION.md`, `GOOGLE-INTEGRATION.md`) must be re-verified before that phase starts.

### Meta (Phase 6)

- A Meta developer app (Business type) with Facebook Login for Business. Permissions needed: `ads_read`, `ads_management`, `business_management`.
- **App Review + Business Verification** are required for Advanced Access, which is needed to serve other businesses' accounts. Allow weeks of lead time.
- Each ad account must accept the **Custom Audience Terms** before website audiences can be created.
- **How activation works without identifying anyone:** when a visitor has granted *advertising* consent and the client's Meta Pixel is present, `tracker.js` emits a Pixel custom event (e.g. `AIQSegment` with an `aiq_segment` parameter) through the client's own `fbq`. We then create a **Website Custom Audience** through the Marketing API with a rule on that event parameter. Meta does all matching. We never send or receive identities for anonymous visitors.
- Website Custom Audience retention is limited (the API docs state 1–180 days, the FAQ says up to 365; verify) and audiences have an unpublished minimum size before they can be targeted.
- **Conversions API** is only used for events the client is entitled to send (e.g. a consented lead with an email the visitor submitted), with SHA-256 hashed identifiers and `event_id` deduplication against the Pixel.
- **Special Ad Categories** (housing, employment, credit/financial products, social issues) restrict Custom Audience and lookalike usage. This matters for construction/real-estate clients. The activation UI must surface it.
- Marketing API versions are deprecated on a rolling schedule, so the version is set in config (`META_GRAPH_VERSION`).

### Google (Phase 5)

- A Google Cloud OAuth client. Scopes: `webmasters.readonly` (GSC), `adwords` (Google Ads), `analytics.readonly` (GA4). The Ads scope typically triggers **OAuth app verification** for production use.
- **Google Ads developer token** (Basic access needs an application) and, for agencies, an MCC `login-customer-id`.
- **Website remarketing** uses rule-based user lists (`UserListService` with `FlexibleRuleUserListInfo`) on a custom parameter that `tracker.js` passes to the client's existing Google tag when ad consent is granted. Prepopulation is available. Google's policy forbids building audiences from sensitive information.
- **Customer Match** (hashed emails/phones) needs account eligibility based on policy-compliance and payment history, explicit `ad_user_data`/`ad_personalization` consent, SHA-256 normalised hashing, and Google recommends ≥ 5,000 members. Since 1 April 2026, accounts without recent Customer Match use are pointed to the **Data Manager API** instead, so the adapter must support that path.
- **EEA traffic** requires Consent Mode v2 signals.
- **Search Console data** is aggregate and anonymised (rare queries are withheld), covers roughly 16 months, and has row limits. It is never presented as per-visitor search history.

### Competitor data (Phase 8)

- Public pages are fetched with a declared user agent, respecting robots.txt and crawl delays.
- Keyword and backlink data need a licensed provider (e.g. DataForSEO, Semrush or Ahrefs APIs). This is a paid dependency to choose.
- The **Meta Ad Library API** requires identity verification. As we understand it, it returns ads about social issues/elections/politics, plus broader coverage only for ads delivered in the EU (DSA). For Sri Lankan commercial competitors it will likely return nothing, so this must be verified before promising "advertising insights".
- Google results are not scraped (Terms of Service); a SERP data provider is used if needed.

### AI provider (Phase 7)

- An Anthropic API key (or another provider via the AI SDK). Usage is metered per organization.

---

## 12. Risks

| Risk | Impact | Mitigation |
|---|---|---|
| MySQL as the event store at "hundreds of millions of events" | Slow segmentation and heavy rollups | Partitioning + rollups now; `EventStore` interface so ClickHouse can be added later |
| Meta/Google approval lead times | Phases 5–6 blocked for weeks | Start app review and developer-token applications during Phase 2 |
| Activation depends on the client's Pixel/Google tag being installed | No audience delivery | Tracking Health checks for `fbq`/`gtag` presence; clear setup guide |
| Small audiences below platform thresholds | Activation "succeeds" but cannot serve | Show platform size status and never promise delivery |
| Special Ad Categories (housing, credit) | Retargeting may be disallowed for some clients | Category flag on the client, with warnings in the activation flow |
| Consent misconfiguration | Legal exposure | Default to `unknown` = no ad signals; consent health alerts |
| AI fabricating numbers | Loss of trust | Evidence-pack contract + numeric validator + "Insufficient data" short-circuit |
| cPanel hosting cannot run Horizon or long-running workers | Ingest backlog | "Lite mode": database queue + cron-driven `queue:work --stop-when-empty`, documented capacity ceiling; Docker/VPS for production |
| Ad-blockers block `tracker.js` | Under-counting | Support first-party subdomain serving (CNAME/proxy path) configured by the client; never evade consent |
| Legal regimes (Sri Lanka PDPA No. 9 of 2022, GDPR for EU visitors, others) | Compliance obligations | Retention, DSAR, consent records, DPA templates; legal review is outside engineering scope |

---

## 13. Blockers

1. **Dependency installation in this build environment.** Packagist was blocked by the build session's network policy, so `composer install` and the PHP test suite could not run there. All PHP was syntax-checked, every framework class import was verified against the Laravel 13 source, the framework-independent logic was executed, and the dashboard was type-checked, built and rendered. Run `composer install && php artisan test` (or the included CI workflow) before building on Phase 1.
2. **Meta app + Business Verification** and **Google Ads developer token + OAuth verification** must be applied for by the business owner. Engineering cannot do these.
3. **Choice of SEO data provider** (paid) for keyword and backlink gaps.
4. **Billing provider choice** (e.g. Stripe, or PayHere for Sri Lanka) for Phase 10.
5. **Hosting target**: confirm VPS/Docker for production. cPanel is supported only in lite mode.
