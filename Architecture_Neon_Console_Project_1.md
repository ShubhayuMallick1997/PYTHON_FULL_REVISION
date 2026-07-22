# Healthcare Data Platform Architecture

## Architecture overview

> GitHub-safe architecture diagram: this section has no external image dependency and renders directly in GitHub Markdown.

```mermaid
flowchart TB
  SRC["Source Systems<br/>EHR · Lab · Scheduling · Payer"] --> INT["Integration Layer<br/>external_resources<br/>validate · map · deduplicate"]
  INT --> CLIN["Clinical Core<br/>patients · encounters · observations<br/>orders · medications · documents"]
  IAM["Security & Governance<br/>organizations · users · roles · audit_events"] -. tenant scope .-> CLIN
  CLIN --> BILL["Revenue Cycle<br/>charges → claims → payments"]
  CLIN --> OUT["Transactional Outbox<br/>outbox_events"]
  OUT --> DOWN["Downstream Services<br/>notifications · integrations · CDC"]
  CLIN --> ANALYTICS["Analytics<br/>daily_encounter_metrics"]
  BILL --> ANALYTICS
  ANALYTICS --> DASH["Dashboards & Reporting"]
```

The PNG and SVG files remain optional downloadable companions. To display them on GitHub, commit them in the same folder as this Markdown file.

The platform is a multi-tenant clinical data system on Neon Postgres. It preserves raw interoperability payloads, normalizes them into clinical entities, derives financial workflows, emits reliable events, and publishes reporting metrics.

## End-to-end pipeline

```mermaid
flowchart LR
  S["EHR · Lab · Scheduling · Payer"] --> I["Integration intake\nexternal_resources"]
  I --> V["Validate · deduplicate · map IDs"]
  V --> C["Clinical core\npatients · encounters · observations · orders"]
  C --> B["Revenue cycle\ncharges → claims → payments"]
  C --> O["Transactional outbox\noutbox_events"]
  O --> X["Downstream notifications / CDC consumers"]
  C --> A["Audit trail\naudit_events"]
  C --> M["Analytics materialized view"]
  B --> M
  M --> D["Dashboards · operational reporting"]
```

## Schema map

```mermaid
erDiagram
  ORGANIZATIONS ||--o{ FACILITIES : operates
  ORGANIZATIONS ||--o{ PATIENTS : owns
  PATIENTS ||--o{ COVERAGES : has
  PATIENTS ||--o{ ENCOUNTERS : attends
  FACILITIES ||--o{ ENCOUNTERS : hosts
  PRACTITIONERS ||--o{ ENCOUNTERS : attends
  ENCOUNTERS ||--o{ OBSERVATIONS : records
  ENCOUNTERS ||--o{ ORDERS : creates
  ORDERS ||--o| MEDICATION_REQUESTS : fulfills
  ENCOUNTERS ||--o{ CHARGES : produces
  CHARGES }o--o{ CLAIMS : submitted_on
  CLAIMS ||--o{ CLAIM_LINES : contains
  CLAIMS ||--o{ PAYMENTS : receives
```

## Data-volume profile

```mermaid
xychart-beta
  title "Production-like synthetic record targets"
  x-axis ["Patients", "Observations", "Events", "Appointments", "Encounters", "Claims", "Providers", "Facilities"]
  y-axis "Rows" 0 --> 5000
  bar [5000, 5000, 5000, 4500, 4200, 3000, 1500, 2000]
```

```mermaid
pie title "Operational data domains by planned volume"
  "Clinical" : 55
  "Integration & audit" : 25
  "Revenue cycle" : 15
  "Identity & facilities" : 5
```

## Layer responsibilities

| Layer | Schema | Responsibility |
|---|---|---|
| Identity and security | `iam` | Tenant scope, users, roles, facilities, immutable audit evidence |
| Clinical system of record | `clinical` | Patient identity, care episodes, clinical findings, orders, documents |
| Revenue cycle | `billing` | Charges, payer claims, adjudication outcomes, payments |
| Interoperability | `integration` | Source payload retention, idempotency, downstream event delivery |
| Analytics | `analytics` | Reporting-oriented aggregates separated from transactional workloads |

## Design rules

1. Every clinical and financial record is scoped to an organization, directly or through its patient/encounter relationship.
2. Raw source payloads are retained before normalization for traceability and replay.
3. Clinical writes and outbox events are produced atomically to prevent lost notifications.
4. Audit events record sensitive access and meaningful changes.
5. Dashboards read aggregates, not the operational clinical tables.

## Recommended next hardening steps

- Add row-level security policies using the active organization claim.
- Partition `audit_events`, `outbox_events`, and `observations` by time at larger scale.
- Add code-system reference tables for ICD, LOINC, SNOMED, and CPT mappings.
- Publish de-identified analytics views for non-clinical reporting users.
