# IAnamnesis — Overall Architecture

## Purpose

IAnamnesis is a clinical decision **support** tool for dentistry. It receives patient anamnesis data from a patient-facing totem, processes it through a deterministic rule engine backed by a curated medical knowledge base, and surfaces structured, evidence-linked alerts to the dentist before treatment begins.

**The system supports clinical decisions. It does not make them.** Every output is pre-authored by the medical team and traceable to a peer-reviewed source. The software is retrieval infrastructure; the medical knowledge base is the product.

---

## System Context Diagram (C4 Level 1)

```mermaid
flowchart TD
  Patient(["👤 Patient<br/>Fills anamnesis form on the clinic totem"])
  Dentist(["👤 Dentist<br/>Reviews structured alerts before procedure"])
  MedTeam(["👤 Medical Team<br/>Authors rules, correlations, paper references"])

  Platform["🖥️ IAnamnesis Platform<br/>Collects anamnesis, runs rule-based analysis,<br/>delivers structured clinical alerts"]

  Patient -->|"Submits anamnesis form (REST / Touch UI)"| Platform
  Platform -->|"Pushes structured alerts (WebSocket / SSE)"| Dentist
  MedTeam -->|"Authors knowledge base (Admin UI)"| Platform
```

---

## Container Diagram (C4 Level 2)

```mermaid
flowchart TD
  Patient(["👤 Patient"])
  Dentist(["👤 Dentist"])
  MedTeam(["👤 Medical Team"])

  subgraph Platform ["IAnamnesis Platform"]
    Totem["Patient Totem<br/>[Flutter — Android]<br/>Touch kiosk for anamnesis entry<br/>(Gertec SK210 or Elgin MK21)"]
    Dashboard["Dentist Dashboard<br/>[React + TypeScript / Web]<br/>Real-time alerts — runs in the dentist's browser"]
    Backoffice["Admin / Backoffice<br/>[React + TypeScript / Web]<br/>Knowledge base authoring"]

    API["API Layer<br/>[ASP.NET Core 8]<br/>REST + SignalR WebSocket"]
    Engine["Processing Engine<br/>[.NET Worker Service]<br/>Five-layer rule evaluation pipeline"]
    DB[("Database<br/>[PostgreSQL 16]<br/>Knowledge base + patient results")]
  end

  Patient -->|"Touch input"| Totem
  Totem -->|"POST /anamnesis"| API
  API -->|"Enqueue job (.NET Channels)"| Engine
  Engine -->|"Read rules / Write results (EF Core)"| DB
  API -->|"Read results on demand (EF Core)"| DB
  API -->|"Push alerts (SignalR WebSocket)"| Dashboard
  Dashboard -->|"On-demand refresh (REST GET)"| API
  MedTeam -->|"Authors rules"| Backoffice
  Backoffice -->|"CRUD knowledge base (REST)"| API
  Dashboard --> Dentist
```

---

## Request Flow

```mermaid
sequenceDiagram
  participant Patient
  participant Totem as Patient Totem
  participant API as API Layer (ASP.NET Core)
  participant Queue as Job Queue (.NET Channels)
  participant Engine as Processing Engine (.NET Worker)
  participant DB as PostgreSQL
  participant Dashboard as Dentist Dashboard

  Patient->>Totem: Fills anamnesis form
  Totem->>API: POST /anamnesis (raw inputs)
  API-->>Totem: 202 Accepted
  API->>Queue: Enqueue processing job

  Queue->>Engine: Dequeue job

  Engine->>DB: Layer 1 - Normalize inputs (lookup aliases)
  Engine->>DB: Layer 2 - Direct lookups (allergies, drugs)
  Engine->>DB: Layer 3 - Cross-correlate conditions
  Engine->>DB: Layer 4 - Apply group/demographic modifiers
  Engine->>DB: Layer 5 - Score severity
  Engine->>DB: Persist AnamnesiResult

  API->>Dashboard: Push structured result (SignalR WebSocket)
  Dashboard-->>Dashboard: Render alerts, evidence links, unresolved inputs
```

---

## Processing Engine — Evaluation Layers

The engine runs five deterministic layers in sequence. No probabilistic inference is used.

```mermaid
flowchart TD
  A[Raw Anamnesis Input] --> L1

  L1["Layer 1 - Normalization<br/>Map free-text to canonical entity IDs<br/>Unknown inputs flagged as unresolved"]
  L2["Layer 2 - Direct Lookup<br/>Single-table queries<br/>Allergy cross-reactions, drug restrictions"]
  L3["Layer 3 - Cross-Correlation<br/>Evaluate condition/drug/allergy combos<br/>against authored rules table"]
  L4["Layer 4 - Group Profiling<br/>Apply demographic modifiers<br/>(age range, sex, comorbidities)"]
  L5["Layer 5 - Severity Scoring<br/>Aggregate rule weights<br/>Order as Critical / Warning / Info"]

  L1 --> L2 --> L3 --> L4 --> L5

  L5 --> R1[Structured AnamnesiResult]
  L1 -- unresolved inputs --> R2[Unresolved Inputs - Flagged for manual review]
```

**Key constraint:** suggestions are never generated text. They are pre-authored rows retrieved by rule evaluation. The system is a retrieval engine, not a generation engine.

---

## Data Model

```mermaid
erDiagram
  conditions {
    uuid id PK
    string icd10_code
    string name
    string[] aliases
    string category
  }

  allergies {
    uuid id PK
    string substance
    uuid cross_reaction_group_id
    string severity_class
  }

  drugs {
    uuid id PK
    string name
  }

  drug_condition_contraindications {
    uuid drug_id FK
    uuid condition_id FK
  }

  drug_allergy_cross_refs {
    uuid drug_id FK
    uuid allergy_id FK
  }

  paper_references {
    uuid id PK
    string doi
    string citation
    string excerpt
    string credibility_tier
  }

  suggestions {
    uuid id PK
    string text
    string type
    string severity
    uuid source_correlation_id FK
  }

  correlations {
    uuid id PK
    uuid paper_reference_id FK
    uuid group_id FK
    string severity
  }

  correlation_conditions {
    uuid correlation_id FK
    uuid condition_id FK
  }

  correlation_allergies {
    uuid correlation_id FK
    uuid allergy_id FK
  }

  correlation_drugs {
    uuid correlation_id FK
    uuid drug_id FK
  }

  groups {
    uuid id PK
    string age_range
    string sex
    string[] prior_condition_categories
  }

  patients {
    uuid id PK
    uuid totem_id
    jsonb raw_inputs
    uuid[] resolved_entities
    string[] unresolved_inputs
    jsonb demographics
    timestamp created_at
  }

  anamnesis_results {
    uuid id PK
    uuid patient_id FK
    uuid[] fired_rule_ids
    uuid[] suggestion_ids
    string[] unresolved_inputs
    timestamp created_at
  }

  correlations ||--o{ correlation_conditions : "triggers on"
  correlations ||--o{ correlation_allergies : "triggers on"
  correlations ||--o{ correlation_drugs : "triggers on"
  correlations ||--o{ suggestions : "produces"
  correlations }o--|| paper_references : "referenced by"
  correlations }o--o| groups : "scoped to"
  drugs ||--o{ drug_condition_contraindications : ""
  drugs ||--o{ drug_allergy_cross_refs : ""
  conditions ||--o{ drug_condition_contraindications : ""
  allergies ||--o{ drug_allergy_cross_refs : ""
  patients ||--|| anamnesis_results : "has"
  anamnesis_results }o--o{ suggestions : "contains"
```

---

## Unresolved Input Handling

Inputs that do not match any known entity are:

1. Stored verbatim in `anamnesis_results.unresolved_inputs[]`
2. Rendered in the dashboard in a distinct visual zone: *"Patient reported: '[raw text]' — no automated analysis available. Manual review required."*
3. Reviewed periodically by the medical team to expand the normalization tables and knowledge base

This is the continuous improvement loop without ML. Over time, the unresolved input review queue drives knowledge base growth organically from real clinical data.

---

## Technology Stack

### Backend — C# / .NET

| Concern | Technology |
|---------|-----------|
| API Layer | ASP.NET Core 8 (Minimal APIs) |
| Background Processing | .NET Worker Service + in-process job queue (System.Threading.Channels); upgrade to MassTransit + RabbitMQ if multi-clinic scale requires it |
| Real-time push | ASP.NET Core SignalR (WebSocket with SSE fallback) |
| ORM / Data Access | Entity Framework Core 8 (code-first migrations) |
| Database | PostgreSQL 16 |
| Input Normalization | Custom fuzzy matcher over lookup tables (FuzzySharp or Lucene.NET for alias matching) |
| Validation | FluentValidation |
| Auth | ASP.NET Core Identity + JWT (clinic-scoped tokens) |
| Testing | xUnit + Testcontainers (PostgreSQL in-container integration tests) |

### Frontend — Android (Totem)

The patient-facing totem runs on Android hardware. Target devices (one will be selected for pilot):

| Hardware | Notes |
|----------|-------|
| **Gertec SK210** | Android-based self-service kiosk |
| **Elgin MK21** | Android-based self-service terminal |

| Concern | Technology |
|---------|-----------|
| Framework | Flutter (Dart) — single codebase targeting Android |
| UI | Flutter widget tree; touch-optimized for kiosk use |
| HTTP client | `dio` or `http` package — sends `POST /anamnesis` and polls status |
| Kiosk lockdown | Android kiosk mode (dedicated device / lock task); Flutter `SystemChrome` for full-screen |
| State management | Riverpod or Bloc |

### Frontend — Web (Dentist Dashboard + Backoffice)

Both the dentist dashboard and the admin backoffice run in the browser on the **doctor's own hardware** (workstation or laptop). No installation required on the clinic side.

| Concern | Technology |
|---------|-----------|
| UI Framework | React + TypeScript (Vite build) |
| Real-time client | `@microsoft/signalr` (npm) — SSE connection for live alert push |
| State management | React Context + hooks (or Zustand) |
| Routing | React Router |
| Dashboard | Live queue list, patient result panel, unresolved input zone |
| Backoffice | Rule/entity/suggestion CRUD forms; admin-only JWT scope |

### Infrastructure (Pilot)

| Concern | Technology |
|---------|-----------|
| Containerization | Docker + Docker Compose (single-server pilot) |
| Database | PostgreSQL in Docker or managed (Neon, Supabase) |
| Reverse proxy | Nginx |
| CI | GitHub Actions |

---

## Pilot Phasing

### Phase 1 — Bounded Knowledge Base
- 20 systemic conditions, 15 allergy groups, 50 correlations
- Single clinic deployment
- Goal: validate that the data model captures real clinical nuance and that dentists find the output actionable

### Phase 2 — Feedback Loop
- Expand knowledge base from unresolved input review and dentist feedback
- Add admin/backoffice UI for medical team to author and update rules
- Refine severity taxonomy based on real usage

### Phase 3 — Scale Assessment
- If pilot proves value, evaluate lightweight input normalization improvements (fuzzy classifier or embedding-based similarity for alias matching only — not generative AI)
- Evaluate multi-clinic architecture (tenant isolation, message broker)
- By Phase 3, real patient data informs any ML investment

---

## Key Architectural Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Rule engine vs. ML | Deterministic rule engine | Medical knowledge in dentistry is bounded and auditable. Rules are defensible in clinical and legal contexts; probabilistic inference is not. |
| Generated text vs. retrieved suggestions | Pre-authored retrieval | Suggestions authored by medical team, referenced to papers. No hallucination risk. Legally cleaner. |
| WebSocket / SSE vs. Webhook | SignalR (WebSocket + SSE fallback) | Dashboard is a persistent live view, not a one-off notification consumer. Push is more appropriate than polling. |
| Totem: Native Android vs. Web | Flutter (Android) | Clinic kiosk environment: Android hardware (Gertec SK210 / Elgin MK21) runs Flutter natively. Offline resilience, full kiosk lockdown via Android dedicated-device mode. Flutter gives full control over touch UX without a browser layer. |
| Dashboard + Backoffice: Desktop vs. Web | React (web) | Dentist and admin UIs run in a browser on the doctor's own hardware — a web app is simpler to deploy and update with no installation on the clinic workstation. |
| Single-server pilot | Docker Compose | Minimizes operational complexity while validating the concept. Architecture supports horizontal scaling later without structural changes. |
