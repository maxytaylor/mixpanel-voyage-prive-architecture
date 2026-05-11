# Voyage Privé × Mixpanel - Architecture Recommendation

> **Post-POC architecture proposal** for aligning Mixpanel with Voyage Privé's existing Snowplow and BigQuery stack. The goal: a single source of truth for product data, with Mixpanel as the self-serve analysis layer for product teams.

---

## Contents

1. [Background](#1-background)
2. [Current State](#2-current-state)
3. [Target Architecture](#3-target-architecture)
4. [Ingestion Path](#4-ingestion-path)
5. [Mobile Strategy](#5-mobile-strategy)
6. [Data Layer Audit](#6-data-layer-audit)
7. [Governance Model](#7-governance-model)
8. [Implementation Roadmap](#8-implementation-roadmap)

---

## 1. Background

The POC has demonstrated that Mixpanel can power the product analytics use cases Voyage Privé needs - funnel analysis, segmentation, PM self-serve insight, retention. The data is there. The question being raised now is the right one: **what does a production-ready architecture look like, and where does Mixpanel sit within it?**

Vincent's instinct - that Snowplow and BigQuery should remain the source of truth, with Mixpanel as the analysis layer reading from them - is the correct framing. This document proposes how to get there cleanly.

> **Core principle:** BigQuery is the source of truth. Mixpanel is the lens. Any fix, modelling change or new property is made once in BigQuery, and Mirror mode propagates it automatically - no re-instrumentation, no dual maintenance.

---

## 2. Current State

The POC architecture is sound in principle and has proven itself during the evaluation. The path from Snowplow through BigQuery to Mixpanel is the right one. What needs attention is not the architecture itself, but the quality of what flows through it. Three things need to be addressed to move from a working POC to a production-ready setup.

| # | Area | What needs to happen |
|---|------|---------------------|
| 1 | **Data layer quality** | VP's data layer predates product analytics intent. The BigQuery modelling view needs a full audit - missing context, unnested arrays and incomplete business properties need to be resolved before the architecture can be relied upon for product decisions. |
| 2 | **Event collection approach** | The current reliance on client-side GTM introduces fragility and makes instrumentation harder to audit and version-control as the product evolves. A move toward server-side or direct Snowplow SDK tracking is worth planning. |
| 3 | **Mobile instrumentation** | The native mobile app has no event tracking today. Given that 78% of search interactions come from phone users, this is a meaningful gap to address in the next phase. |

> The architecture is right. The work is in the data flowing through it.

---

## 3. Target Architecture

The proposed architecture keeps Snowplow and BigQuery as the collection and modelling backbone, with Mixpanel reading from BigQuery via the Warehouse Connector in Mirror mode. The responsibilities of each layer are distinct and should stay that way.

```
┌─────────────────────────────────────────────────┐
│  Collection Layer                               │
│  Application → Snowplow JS / Server-side SDK    │
│  custom events + properties                     │
│  (GTM today - direct SDK recommended over time) │
└─────────────────────┬───────────────────────────┘
                      │  raw events to Snowplow collector
                      ▼
┌─────────────────────────────────────────────────┐
│  Enrichment Layer                               │
│  Snowplow Pipeline                              │
│  schema validation · context enrichment         │
│  UTM parsing · PII handling · consent           │
└─────────────────────┬───────────────────────────┘
                      │  enriched events → raw table
                      ▼
┌─────────────────────────────────────────────────┐
│  ★ SOURCE OF TRUTH                              │
│  BigQuery - snowplow_events.events              │
│                                                 │
│  • SQL modelling view                           │
│  • ARRAY contexts flattened to scalar columns   │
│  • user_id + device_id identity columns         │
│  • UTM values normalised                        │
│  • Device type consolidated · bot filtered      │
└─────────────────────┬───────────────────────────┘
                      │  Mirror mode
                      │  hourly sync or API-triggered
                      │  automatic retroactive updates
                      ▼
┌─────────────────────────────────────────────────┐
│  Analysis Layer                                 │
│  Mixpanel (EU) - Warehouse Connector            │
│                                                 │
│  • Funnels · Retention · Flows                  │
│  • Session Replay (booking flow)                │
│  • Lexicon governance                           │
│  • PM self-serve dashboards                     │
│  • MCP-powered programmatic querying            │
└─────────────────────────────────────────────────┘
```

### How the layers divide responsibility

**Snowplow handles enrichment** - schema validation, context attachment, UTM parsing and PII handling all happen in the Snowplow pipeline before data reaches BigQuery. This is Snowplow's core function and should not be replicated downstream.

**BigQuery handles modelling** - what happens in BigQuery is preparation for Mixpanel's data model, not re-enrichment: flattening ARRAY contexts into scalar columns, casting types correctly, normalising inconsistent values, filtering bot traffic, and surfacing the identity columns Mixpanel needs. This is a SQL modelling view maintained and version-controlled by the data team.

**Mixpanel handles analysis** - no data transformation happens in Mixpanel. It receives clean, pre-modelled data and surfaces it through funnels, retention analysis, flows and dashboards. The Lexicon layer governs what PMs see and how properties are labelled.

### On the collection layer

VP currently uses Google Tag Manager to fire events to the Snowplow collector. GTM works and is not a reason to change anything in the short term. However as VP's event taxonomy grows, GTM's client-side nature introduces limitations worth planning around: events fire based on DOM interactions which is fragile, and instrumentation is harder to version-control than application-layer code. Server-side events sent directly from the application to Snowplow's collector are more reliable and easier to test and audit. The data layer audit will clarify whether GTM configuration is a contributing factor to current data quality issues.

---

## 4. Ingestion Path

The Warehouse Connector in Mirror mode is the correct ingestion path for VP's architecture. It keeps BigQuery as the single source of truth and means any fix or modelling improvement made upstream propagates to Mixpanel automatically - including historically.

### Sync frequency

Mirror mode supports **hourly syncs** and can also be **triggered via the Mixpanel API** for more frequent or event-driven updates. This significantly reduces the latency between live product behaviour and Mixpanel, and means the warehouse-first approach is suitable for VP's product analytics needs without any additional pipeline complexity.

```
# Trigger a sync on demand via the API
POST https://eu.mixpanel.com/api/v2/warehouse-data/sync/{sync_id}/trigger
```

For VP's use cases - funnel analysis, conversion tracking, session-level retention - hourly or API-triggered syncs provide more than sufficient data freshness. There is no need to introduce a parallel real-time ingestion path alongside the warehouse connector for these purposes.

---

## 5. Mobile Strategy

The native mobile app has no event tracking today. The correct approach for VP's architecture - and the only one that preserves the single source of truth goal - is the **Snowplow mobile SDK** (iOS and Android).

Adding the Snowplow mobile SDK to the native app means mobile events enrich through the same pipeline and land in the same BigQuery table as web events, then flow to Mixpanel via the existing Warehouse Connector with no additional configuration. No parallel ingestion paths, no identity reconciliation challenges, no architectural compromise.

This is a Phase 2 workstream, to be scoped and delivered once the data layer audit (Phase 1) is complete and the BigQuery modelling view is stable.

---

## 6. Data Layer Audit

The following issues were identified during the POC by analysing VP's live Mixpanel project. All fixes belong in the BigQuery SQL modelling view that feeds the Warehouse Connector. Because Mirror mode is active, each fix propagates automatically once applied in BigQuery.

| Issue | Fix location | Impact | Priority |
|-------|-------------|--------|----------|
| UTM medium undefined or polluted (88% of events) | BigQuery view + confirm Snowplow UTM enrichment is firing | All traffic source analysis blocked | 🔴 Critical |
| Raw Snowplow ARRAY context columns not flattened | BigQuery view - `SAFE_OFFSET(0)` extraction per field | `sale_id`, `booking_price`, `payment_method`, `session_index` inaccessible | 🔴 Critical |
| `select_booking_option` event has zero data | Snowplow tracking or BigQuery `WHERE` clause | Checkout step 1→2 journey invisible | 🔴 Critical |
| `session_index` contains non-numeric values | BigQuery view - `SAFE_CAST AS INT64` | New vs returning user segmentation unreliable | 🟠 High |
| Device type fragmented + bot traffic included | BigQuery view - `CASE` normalisation + `is_bot` flag | Device breakdowns inflated and inconsistent | 🟠 High |
| `sale_name`, `booking_price`, `payment_method` null | BigQuery view - confirm ARRAY extraction is complete | Revenue and deal category analysis blocked | 🟠 High |
| Multi-market URL paths not normalised (DE/AT/BE) | BigQuery view `page_category` column OR Mixpanel Custom Events | German and Dutch markets appear near-zero in funnels | 🟡 Medium |

A companion document with exact SQL examples for each fix, referencing VP's actual Snowplow column names and context schemas, is available separately.

---

## 7. Governance Model

The architecture makes BigQuery the source of truth technically. Governance makes it the source of truth organisationally. Without it, different teams build different definitions of the same metric, trust erodes, and PMs revert to asking the data team for SQL - the exact outcome VP is trying to avoid.

### BigQuery layer *(data team owns)*
- Canonical SQL modelling view - reviewed and version-controlled
- Approved column list - no unmapped arrays or raw Snowplow field names exposed
- Change process - any schema change reviewed before connector sync runs
- Bot filtering - `is_bot` column maintained and applied consistently

### Mixpanel Lexicon layer *(data owner + PMs)*
- All core events verified and named in plain language
- Properties described - no raw Snowplow names visible to PMs
- Tags applied by domain: Booking, Search, Marketing, Identity
- Raw and technical properties hidden from the default property picker
- New events go through Event Approval before appearing in reports

> Mixpanel's hub-and-spoke governance model fits VP's structure well. A central data owner (Mathieu and Ahmed's team) owns the BigQuery modelling view and Lexicon standards. Product team leads (Vincent's team) own their domain events and dashboards. PMs are spoke consumers - they explore and build within governed, verified data, without needing SQL or data team involvement for every question.

---

## 8. Implementation Roadmap

Sequenced to fix the foundation before adding capability. Each phase unlocks the next.

> **Note on timescales:** The durations below are indicative and subject to further scoping. Actual timelines will vary depending on the complexity of the data layer audit findings, team availability, existing data engineering capacity, and the scope of any Snowplow pipeline changes required. Phases may run in parallel where dependencies allow, or extend where audit findings reveal additional work.

### Phase 1 - Foundation: Data Audit *(indicative: 4–6 weeks)*
- [ ] Audit and fix BigQuery modelling view - flatten all ARRAY contexts with `SAFE_OFFSET(0)`
- [ ] Normalise `utm_medium` and `utm_source` columns
- [ ] Fix `session_index` type (`SAFE_CAST AS INT64`) and consolidate `device_type`
- [ ] Investigate `select_booking_option` zero-data gap in Snowplow tracking
- [ ] Configure Mirror sync frequency - hourly or API-triggered
- [ ] Validate full data quality in Mixpanel post-fix
- [ ] Build Lexicon - verify core events, add descriptions and domain tags

### Phase 2 - Expand: Mobile + Governance *(indicative: 6–12 weeks)*
- [ ] Scope and instrument Snowplow mobile SDK (iOS + Android)
- [ ] Validate mobile events flowing through BigQuery to Mixpanel
- [ ] Establish formal change management process for the BigQuery modelling view
- [ ] Build canonical PM dashboards by team and market
- [ ] Enable Session Replay on the booking flow
- [ ] Evaluate direct Snowplow SDK tracking to replace GTM client-side instrumentation

### Phase 3 - Optimise: Experimentation + Expansion *(indicative: 3–6 months)*
- [ ] Connect Mixpanel Experiments to A/B test measurement layer
- [ ] Revenue Analytics - map `payment_amount` to Mixpanel revenue tracking
- [ ] MCP-enabled programmatic analysis for the data team
- [ ] Feature Flags integration for experiment rollout and measurement
- [ ] Expand analytics coverage to additional business units

> **Phase 1 delivers the majority of the POC use cases.** The three use cases submitted - login to sales listing conversion, search bar impact, full funnel analysis - are fully achievable once the data audit is complete. Phases 2 and 3 add mobile coverage, experimentation measurement and broader organisational rollout on top of a solid, audited foundation.

---

## References

- [Mixpanel Warehouse Connector docs](https://docs.mixpanel.com/docs/tracking-methods/warehouse-connectors)
- [Snowplow integration](https://docs.mixpanel.com/docs/tracking-methods/integrations/snowplow)
- [Warehouse best practices](https://docs.mixpanel.com/docs/tracking-best-practices/warehouse-best-practices)
- [Mixpanel Lexicon governance](https://docs.mixpanel.com/docs/data-governance/lexicon)
- [Mirror mode sync documentation](https://docs.mixpanel.com/docs/tracking-methods/warehouse-connectors#mirror)

---

*Prepared by Mixpanel Solutions · Post-POC recommendation · 2026*
