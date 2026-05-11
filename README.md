# Voyage Privé × Mixpanel — Architecture Recommendation

> **Post-POC architecture proposal** for aligning Mixpanel with Voyage Privé's existing Snowplow and BigQuery stack. The goal: a single source of truth for product data, with Mixpanel as the self-serve analysis layer for product teams.

---

## Contents

1. [Background](#1-background)
2. [Current State & Problems](#2-current-state--problems)
3. [Target Architecture](#3-target-architecture)
4. [Ingestion Path — Two Options](#4-ingestion-path--two-options)
5. [Mobile Strategy](#5-mobile-strategy)
6. [Data Layer Audit](#6-data-layer-audit)
7. [Governance Model](#7-governance-model)
8. [Implementation Roadmap](#8-implementation-roadmap)

---

## 1. Background

The POC has demonstrated that Mixpanel can power the product analytics use cases Voyage Privé needs - funnel analysis, segmentation, PM self-serve insight, retention. The data is there. The question being raised now is the right one: **what does a production-ready architecture look like, and where does Mixpanel sit within it?**

Vincent's instinct, that Snowplow and BigQuery should remain the source of truth, with Mixpanel as the analysis layer reading from them — is the correct framing. This document proposes how to get there, and in what order.

> **Core principle:** BigQuery is the source of truth. Mixpanel is the lens. Any fix, enrichment or new property is made once in BigQuery, and Mirror mode propagates it automatically — no re-instrumentation, no dual maintenance.

---

## 2. Current State & Problems

The POC architecture is functional — the data we have been analysing proves it. But it is not production-ready as a long-term foundation. Three issues need to be resolved before building further on top of it.

| # | Problem | Why it matters |
|---|---------|---------------|
| 1 | **The data layer hasn't been audited** | Missing context, unnested ARRAY columns, undefined UTMs and gaps in business properties suggest the instrumentation predates any product analytics intent. Messy data in, messy data in. |
| 2 | **The ingestion journey is too long** | GTM → Snowplow → BigQuery → Warehouse Connector means multiple failure points and meaningful latency. Mirror runs daily, so Mixpanel is always ~24 hours behind live product behaviour. |
| 3 | **No mobile app instrumentation** | The mobile app has no event tracking today. 78% of search events come from Phone users, yet the in-app journey is invisible. |

> ⚠️ **The data layer audit is the critical path.** Until the BigQuery view produces clean, flat, correctly-typed data with proper identity mapping, any other architectural change just moves the same problems faster. The audit comes first.

---

## 3. Target Architecture

The proposed architecture keeps Snowplow and BigQuery as the collection and enrichment backbone, with Mixpanel reading from BigQuery via the Warehouse Connector in Mirror mode. No new pipeline complexity until the foundation is solid.

```
┌─────────────────────────────────────────────┐
│  Collection Layer                           │
│  VP Data Layer → Google Tag Manager         │
│  custom events + properties · web only      │
└────────────────────┬────────────────────────┘
                     │  fires via Snowplow JS tracker
                     ▼
┌─────────────────────────────────────────────┐
│  Enrichment Layer                           │
│  Snowplow Pipeline                          │
│  schema validation · context enrichment     │
│  UTM parsing · PII / consent handling       │
└────────────────────┬────────────────────────┘
                     │  enriched events → raw table
                     ▼
┌─────────────────────────────────────────────┐
│  ★ SOURCE OF TRUTH                          │
│  BigQuery — snowplow_events.events          │
│                                             │
│  • Flattened SQL view                       │
│  • user_id + device_id identity columns     │
│  • Business context extracted from arrays   │
│  • UTM normalised · bot traffic filtered    │
└────────────────────┬────────────────────────┘
                     │  Mirror mode · daily sync
                     │  automatic retroactive updates
                     ▼
┌─────────────────────────────────────────────┐
│  Analysis Layer                             │
│  Mixpanel (EU) — Warehouse Connector        │
│                                             │
│  • Funnels · Retention · Flows              │
│  • Session Replay (booking flow)            │
│  • Lexicon governance                       │
│  • PM self-serve dashboards                 │
│  • MCP-powered programmatic querying        │
└─────────────────────────────────────────────┘
```

**The key principle:** no enrichment or data modelling happens inside Mixpanel. Mixpanel receives clean, pre-modelled data from BigQuery. Any fix is made once in the BigQuery view, and Mirror reflects it automatically — including historically. This is the Mirror advantage that directly addresses VP's goal of one source of truth.

---

## 4. Ingestion Path — Two Options

Vincent's suggestion of forwarding events from Snowplow directly to Mixpanel post-enrichment is architecturally valid and [supported natively by Mixpanel](https://docs.mixpanel.com/docs/tracking-methods/integrations/snowplow). But the right choice depends on timing.

### Option A — BigQuery → Warehouse Connector *(recommended now)*

Continue the current architecture. BigQuery is master. All enrichment, flattening, identity resolution and cleaning happens there. Mixpanel reads from BigQuery via Mirror. BigQuery also serves BI tools, data science and any future analytics consumers.

**Pros:**
- No new pipeline work required
- Mirror handles retroactive fixes automatically
- BigQuery remains the audit trail and compliance record
- A single change propagates to all consumers
- Simplest to govern — one data model to maintain

**Cons:**
- Daily sync — Mixpanel is ~24h behind live behaviour
- Not suitable for real-time alerting use cases

---

### Option B — Snowplow → Mixpanel HTTP forwarding *(consider post-audit)*

Configure Snowplow to forward enriched events directly to Mixpanel's ingestion API in near real-time, bypassing the daily sync cycle entirely.

**Pros:**
- Real-time dashboards and alerting
- Removes daily sync latency

**Cons:**
- Requires Snowplow pipeline configuration change
- Messy data upstream = messy data in Mixpanel, but faster
- Creates two consumers of Snowplow output to maintain
- Historical backfill still requires BigQuery anyway

> **The decision rule:** Option A is correct today. Option B becomes worth evaluating once the data layer audit is complete and the BigQuery view is producing reliable output. Running Option B on top of unaudited data just delivers bad data faster.

### A note on Google Tag Manager

GTM as a collection mechanism introduces a layer of indirection that is hard to audit and version-control. As VP grows its event taxonomy, migrating toward server-side tracking — sending events from the application layer directly to Snowplow's collector endpoint — would give more reliable, auditable instrumentation. The data layer audit should make clear whether this is needed, or whether GTM is working correctly and the gaps are purely downstream.

---

## 5. Mobile Strategy

No mobile app instrumentation today is a material blind spot. Three options in order of recommended priority:

### Option 1 — Snowplow Mobile SDK *(recommended)*
Add Snowplow's native iOS/Android tracker to the app. Events enrich and land in the same BigQuery table, then flow to Mixpanel via the existing Warehouse Connector. Keeps BigQuery as the single source of truth and maintains the architecture cleanly. Most implementation effort, but the correct long-term answer.

### Option 2 — Mixpanel iOS/Android SDK *(fast start)*
Instrument the native app directly and send events to Mixpanel in parallel with web. Fast to deploy, but creates a second ingestion path that diverges from BigQuery. Useful to get mobile data quickly while the Snowplow mobile implementation is being planned.

### Option 3 — Mixpanel Autocapture on mobile web *(interim only)*
Add Mixpanel's JS SDK with autocapture to the mobile web experience. No native app changes needed. Very limited in scope — cannot capture in-app interactions — but captures mobile web sessions quickly at near-zero instrumentation cost.

> **Pragmatic path:** Deploy Option 3 now to capture mobile web sessions immediately. Run Option 1 (Snowplow mobile SDK) as a planned project alongside the data layer audit. This gives VP mobile insight within days while native instrumentation is properly scoped.

---

## 6. Data Layer Audit

The following issues were identified during the POC by analysing Voyage Privé's live Mixpanel project. All fixes are upstream — changes to the BigQuery SQL view that feeds the Warehouse Connector. Because Mirror mode is active, each fix propagates automatically once applied in BigQuery.

| Issue | Fix location | Impact | Priority |
|-------|-------------|--------|----------|
| UTM medium undefined or polluted (88% of events) | BigQuery view + Snowplow enrichment | All traffic source analysis blocked | 🔴 Critical |
| Raw Snowplow ARRAY context columns not flattened | BigQuery view — `SAFE_OFFSET(0)` extraction | `sale_id`, `booking_price`, `payment_method`, `session_index` inaccessible | 🔴 Critical |
| `select_booking_option` event has zero data | Snowplow tracking or BigQuery `WHERE` clause | Checkout step 1→2 journey invisible | 🔴 Critical |
| `session_index` contains non-numeric values | BigQuery view — `SAFE_CAST AS INT64` | New vs returning user segmentation unreliable | 🟠 High |
| Device type fragmented + bot traffic included | BigQuery view — `CASE` normalisation + `is_bot` filter | Device breakdowns inflated and inconsistent | 🟠 High |
| `sale_name`, `booking_price`, `payment_method` null | BigQuery view + confirm connector sync | Revenue and deal category analysis blocked | 🟠 High |
| Multi-market URL paths not normalised (DE/AT/BE) | BigQuery view OR Mixpanel Custom Events | German and Dutch markets appear near-zero in funnels | 🟡 Medium |

### Identity fix

The most important fix — and the one that unlocks the three POC use cases — is the identity mapping. The events connector currently has only `user_id` mapped. `device_id` (`domain_userid`) is missing. Without both:

```sql
-- Required in the BigQuery view
user_id        -- authenticated identifier (null for anonymous / no cookie consent)
domain_userid AS device_id  -- Snowplow cookie ID (present for all users)
```

When Mixpanel receives both on the same event, it automatically merges the anonymous and authenticated identities, connecting pre-login browsing to post-login purchase. The connector needs to be deleted and recreated with both columns mapped.

---

## 7. Governance Model

The architecture makes BigQuery the source of truth technically. Governance makes it the source of truth organisationally. Without it, different teams build different definitions of the same metric, trust in the data erodes, and PMs revert to asking the data team for SQL — the exact outcome VP is trying to avoid.

### BigQuery layer *(data team owns)*
- Canonical SQL view — reviewed and version-controlled
- Approved column list — no unmapped arrays or raw Snowplow fields
- Change process — any schema change reviewed before sync
- Identity model — `user_id` + `device_id` mapping documented and owned
- Bot filtering — `is_bot` column maintained and applied

### Mixpanel Lexicon layer *(data owner + PMs)*
- All core events verified and named in plain language
- Properties described — no raw Snowplow names visible to PMs
- Tags applied by domain: Booking, Search, Marketing, Identity
- Raw / technical properties hidden from the default property picker
- New events go through Event Approval before appearing in reports

> Mixpanel's hub-and-spoke governance model fits VP's structure: a central data owner (Mathieu / Ahmed's team) owns the BigQuery view and Lexicon standards. Product team leads (Vincent's team) own their domain's events and dashboards. PMs are spoke consumers — they explore and build within governed, verified data, without needing SQL.

---

## 8. Implementation Roadmap

Sequenced to fix the foundation before adding capability. Each phase unlocks the next.

### Phase 1 — Foundation: Data Audit *(now → 4–6 weeks)*
- [ ] Fix BigQuery view — flatten all ARRAY contexts with `SAFE_OFFSET(0)`
- [ ] Resolve identity mapping — add `device_id` to connector, re-import events table
- [ ] Normalise `utm_medium` and `utm_source` columns
- [ ] Fix `session_index` type (`SAFE_CAST AS INT64`) and consolidate `device_type`
- [ ] Investigate `select_booking_option` zero-data gap
- [ ] Validate data in Mixpanel post re-import

### Phase 2 — Expand: Mobile + Governance *(6–12 weeks)*
- [ ] Deploy Mixpanel autocapture on mobile web (interim)
- [ ] Scope and build Snowplow mobile SDK (iOS + Android)
- [ ] Build Lexicon — verify core events, add descriptions and domain tags
- [ ] Establish change management process for BigQuery view
- [ ] Build canonical PM dashboards by team and market
- [ ] Enable Session Replay on the booking flow

### Phase 3 — Optimise: Real-time + Experimentation *(3–6 months)*
- [ ] Evaluate Snowplow → Mixpanel real-time HTTP forwarding (Option B)
- [ ] Connect Mixpanel Experiments to A/B test measurement layer
- [ ] Revenue Analytics — map `payment_amount` to Mixpanel revenue tracking
- [ ] MCP-enabled programmatic analysis for the data team
- [ ] Feature Flags integration for experiment rollout
- [ ] Expand to additional business units

> **Phase 1 delivers the majority of the POC use cases.** The three use cases submitted by VP — login to sales listing conversion, search bar impact, full funnel analysis — are all achievable once the data audit is complete. Phases 2 and 3 add mobile coverage, real-time capability and experimentation measurement on top of a solid foundation.

---

## References

- [Mixpanel Warehouse Connector docs](https://docs.mixpanel.com/docs/tracking-methods/warehouse-connectors)
- [Snowplow → Mixpanel integration](https://docs.mixpanel.com/docs/tracking-methods/integrations/snowplow)
- [Warehouse best practices](https://docs.mixpanel.com/docs/tracking-best-practices/warehouse-best-practices)
- [ID management — warehouse perspective](https://docs.mixpanel.com/docs/tracking-methods/id-management/identifying-users-simplified)
- [Mixpanel Lexicon governance](https://docs.mixpanel.com/docs/data-governance/lexicon)

---

*Prepared by Mixpanel Solutions · Post-POC recommendation · 2026*
