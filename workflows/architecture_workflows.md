# 🏛️ Architecture & System Workflows

This document provides a comprehensive visual and technical reference for all operational workflows in the **Agentic Staffing Dashboard**. It details data synchronization, AI agent orchestration, privacy-preserving spatial visualization, and telemetry tracing.

---

## 📑 Table of Contents
1. [Core Architectural & Privacy Principles](#-core-architectural--privacy-principles)
2. [Data Ingestion & Privacy ETL Pipeline](#1-data-ingestion--privacy-etl-pipeline)
3. [Agent Orchestration & Deterministic Tool Routing](#2-agent-orchestration--deterministic-tool-routing)
4. [Privacy Map Rendering & Event Loop](#3-privacy-map-rendering--event-loop)
5. [Telemetry, Tracing & Audit Trail](#4-telemetry-tracing--audit-trail)

---

## 🛡️ Core Architectural & Privacy Principles

1. **No Raw Coordinates in Database:** Exact GPS coordinates are considered highly sensitive. They exist only transiently in memory during ingestion and are permanently discarded. The database only stores Uber H3 hexagonal indices (Resolution 8).
2. **Minimal PII Retention:** Addresses, ZIP codes, and birth dates are strictly purged from the database after geographical indexing and hashing steps are complete.
3. **Opaque Hashing for Sync:** To sync Excel updates without storing addresses, the system computes deterministic SHA-256 hashes of the `Name + Address` string (`_match_hash`).
4. **LLM Sandboxing:** The AI (Google Gemini via Google ADK) never executes raw SQL and never sees the whole database. It only receives targeted, minimized contextual data returned by strictly defined Python tools.
5. **Deterministic Map Control:** The map UI never relies on the LLM to output structured JSON to reposition the map. Instead, it uses a deterministic "Side-Channel" written directly by Python tools during execution.

---

## 1. Data Ingestion & Privacy ETL Pipeline

The Data Synchronization Pipeline (ETL) ingests raw Excel exports from the agency management system, verifies schema integrity, geocodes addresses incrementally, and writes privacy-safe Uber H3 spatial indices to the local SQLite database.

```mermaid
%%{init: {"theme": "base", "themeVariables": {"fontFamily": "Arial, Helvetica, sans-serif", "fontSize": "30px", "primaryTextColor": "#000000", "lineColor": "#4B5563"}}}%%
flowchart LR
    subgraph Col1 ["Phase 1 & 2: Ingestion & Changes"]
        direction TB
        R1["<b>1. Ingestion</b><br/>CustomerData.xlsx &<br/>CaregiverData.xlsx"]
        R2["<b>2. Validation & Scrubbing</b><br/>Enforce required columns<br/>Drop extraneous PII"]
        R3["<b>3. Surrogate Key</b><br/>Compute SHA-256<br/>_match_hash string"]
        R4["<b>4. Change Detection</b><br/>Compare _match_hash<br/>with existing SQLite DB"]
        R1 --> R2 --> R3 --> R4
    end

    subgraph Col2 ["Phase 3 & 4: Privacy & Rebuild"]
        direction TB
        R5["<b>5. Geocoding Cache</b><br/>Check local cache or<br/>query Geocodio API"]
        R6["<b>6. Spatial Indexing</b><br/>Convert coordinates to<br/>Uber H3 Res 8 (~0.73 km²)"]
        R7["<b>7. Discard Raw Coords</b><br/>Purge Lat/Lng & street<br/>addresses permanently"]
        R8["<b>8. SQLite DB Rebuild</b><br/>Commit clients & staff<br/>Rebuild vw_staff_capacity"]
        R5 --> R6 --> R7 --> R8
    end

    Col1 -->|"New / Changed"| Col2

    classDef proc fill:#EBF5FF,stroke:#2563EB,color:#000000,stroke-width:1.5px
    classDef check fill:#FEF9C3,stroke:#CA8A04,color:#000000,stroke-width:1.5px
    classDef purge fill:#FEE2E2,stroke:#DC2626,color:#000000,stroke-width:1.5px
    classDef db fill:#DCFCE7,stroke:#15803D,color:#000000,stroke-width:1.5px

    class R1,R3,R5,R6 proc
    class R2,R4 check
    class R7 purge
    class R8 db
```

### Key Components

1. **Data Ingestion & Validation (`etl/sync.py`)**:
   - Ingests `CustomerData.xlsx` and `CaregiverData.xlsx` via pandas.
   - Enforces required columns (`First Name`, `Last Name`, `Address 1`, `City`, `State`, `Zip`).
   - Silently drops extraneous PII columns before processing.

2. **Opaque Tracking (SHA-256 Surrogate Keys)**:
   - Rows are tracked across imports without storing raw addresses in the database.
   - Computes a deterministic surrogate key: `SHA-256(fname | lname | address_string)`.
   - Compares incoming keys against `_match_hash` in the database to detect address modifications.

3. **Incremental Geocoding & Local Cache**:
   - Queries the **Geocodio API** to convert address strings into coordinates.
   - Transmits *only* sanitized address strings over the network (no patient or caregiver names).
   - Caches coordinate results locally in SQLite to prevent duplicate API lookups.

4. **Privacy via H3 Hexagonal Grid**:
   - Converts transient latitude/longitude coordinates to **Uber H3 Resolution 8** indices (~0.73 km² cell area).
   - Raw coordinates are immediately purged from memory and are never written to the SQLite database.

5. **Capacity View Calculation**:
   - Rebuilds the `vw_staff_capacity` database view to compute available hours (`Weekly_Capacity_Hours - Scheduled_Hours`) for fast SQL filtering by agent tools.

---

## 2. Agent Orchestration & Deterministic Tool Routing

The AI Staffing Assistant bridges conversational queries from the Director of Nursing (DON) to the secured SQLite database using **Google ADK** (Agent Development Kit) with deterministic tool execution.

### Runtime Interaction Sequence

```mermaid
%%{init: {"theme": "base", "themeVariables": {"fontFamily": "Arial, Helvetica, sans-serif", "fontSize": "32px", "actorFontSize": "34px", "noteFontSize": "28px", "messageFontSize": "32px", "primaryTextColor": "#000000", "lineColor": "#4B5563"}}}%%
sequenceDiagram
    autonumber
    participant User
    participant UI as Streamlit UI
    participant ADK as ADK Agent
    participant LLM as Gemini LLM
    participant DB as SQLite DB

    User->>UI: "Find PCAs near Claire"
    UI->>ADK: ask_agent(query)
    ADK->>LLM: Query + Tool Signatures

    LLM-->>ADK: lookup_client("Claire")
    ADK->>DB: Query Client C001
    DB-->>ADK: H3 Index: 882a...
    ADK-->>LLM: Found C001 @ 882a...

    LLM-->>ADK: find_nearby_staff(C001, 10mi)
    activate ADK
    ADK->>DB: Query grid_dist <= K
    DB-->>ADK: Return 2 PCAs
    Note over ADK,DB: SIDE-CHANNEL WRITE:<br/>Record client & radius to<br/>local AGENT_CONTEXT memory
    ADK-->>LLM: Found 2 PCAs
    deactivate ADK

    LLM-->>ADK: Synthesized Answer
    ADK-->>UI: Answer + Context Payload
    Note over UI: Read side-channel context &<br/>trigger st.rerun() for map redraw
    UI->>User: Display Chat + Centered Map
```

### Tool Execution Flowchart

```mermaid
%%{init: {"theme": "neutral", "themeVariables": {"fontFamily": "Arial, Helvetica, sans-serif", "fontSize": "11px", "actorFontSize": "11px", "noteFontSize": "10px", "messageFontSize": "10px"}}}%%
flowchart TD
    Start([User Chat Input]) --> App[Streamlit UI ask_agent]

    App --> ADK[Google ADK Runner gemini-3.1-flash-lite]

    ADK --> ParsePrompt{Intent Parser}

    ParsePrompt -->|Needs Client Info| Tool1[Client Lookup Tool finds Client ID and H3]
    ParsePrompt -->|Spatial Search| Tool2[Nearby Staff Tool finds Staff in Radius]
    ParsePrompt -->|Role/Hour Filter| Tool3[Hours Filter Tool filters Staff by Availability]

    Tool1 --> ToolContext[AGENT_CONTEXT Side Channel Dictionary]
    Tool2 --> ToolContext
    Tool3 --> ToolContext

    ToolContext --> UpdateUI[UI Map Markers and Tables Bypass LLM Hallucinations]

    Tool1 --> DB1[(Secure SQLite DB Clients Table)]
    Tool2 --> DB2[(Secure SQLite DB H3 Grid Distance)]
    Tool3 --> DB3[(Secure SQLite DB Capacity View)]

    DB1 --> ReturnTool1[Tool Result Client ID and Roles]
    DB2 --> ReturnTool2[Tool Result List of Staff IDs]
    DB3 --> ReturnTool3[Tool Result Filtered Staff IDs]

    ReturnTool1 --> Compose[LLM Synthesizes Answer]
    ReturnTool2 --> Compose
    ReturnTool3 --> Compose

    Compose --> Output([Markdown Chat Response])

```

### Key Components

1. **In-Memory Session Runner**:
   - Preserves conversational context across chat turns using ADK's `InMemoryRunner`.
   - Isolates chat histories by `session_id`.

2. **Strict Pydantic Input Schemas**:
   - All tools enforce type validation through Pydantic schemas:
     - `ClientLookupInput`: Validates client name queries.
     - `NearbyStaffInput`: Validates radius bounds and role constraints.
     - `StaffFilterInput`: Validates role and capacity hour thresholds.

3. **H3 Spatial Traversal Rules**:
   - `find_nearby_staff` translates radial mile distances into H3 k-ring bounds (e.g., 5 miles ≈ 10 k-rings at Resolution 8).
   - Evaluates neighbor distances natively in Python via `h3.grid_distance()` without requiring spatial database extensions.

4. **Deterministic Side-Channel Map State**:
   - As Python tools execute, they write targeted spatial state (`client_id`, `radius`, `matched_staff_ids`) into a local `AGENT_CONTEXT` side-channel dictionary.
   - The UI updates its map directly from this side channel, completely preventing LLM hallucination of coordinates, distances, or names.

5. **Why the Side-Channel Matters**:
   - If the system relied on the LLM to emit formatted JSON to control the UI, the LLM could hallucinate client names, invent radii, or output malformed syntax that crashes the app.
   - By having deterministic Python tools (`find_nearby_staff`) write directly to `AGENT_CONTEXT` *while executing*, the map reflects the exact database query ground truth. The LLM never touches map rendering parameters.

---

## 3. Privacy Map Rendering & Event Loop

The map visualization displays spatial proximity and staff capacity using obfuscated H3 polygons instead of pin-point GPS markers, keeping all PII rendering strictly client-side.

```mermaid
%%{init: {"theme": "neutral", "themeVariables": {"fontFamily": "Arial, Helvetica, sans-serif", "fontSize": "11px", "actorFontSize": "11px", "noteFontSize": "10px", "messageFontSize": "10px"}}}%%
flowchart TD
    Start([Side Channel Update from Agent System]) --> ReadDict[Read AGENT_CONTEXT for Staff IDs]

    ReadDict --> SyncState[Stash IDs in Streamlit Session State]

    SyncState --> QueryDB{Intersect with Local SQLite DB}

    QueryDB -->|Fetch H3 Indexes| MapHex[Render Hexagonal Folium Overlays]
    QueryDB -->|Fetch PII| RenderTable[Render Data Table Names Phones Hours]

    MapHex --> Colors[Apply Heatmap Colors by Available Hours]
    MapHex --> Tooltip[Bind Hover Tooltips Client PII is local only]

    RenderTable --> Sort[Sort by Distance Miles]

    Colors --> MapComponent[st_folium component]
    Tooltip --> MapComponent

    MapComponent --> ClickMap[User clicks Hexagon]
    ClickMap --> Override[Override Agent Context Center newly clicked client]

    Override --> Start

```

### Key Components

1. **The Discard Protocol**:
   - Raw latitude and longitude coordinates are never stored in SQLite. Only Resolution 8 H3 hex indices persist.

2. **Hexagonal Clustering (Folium / `st_folium`)**:
   - The map layer queries `h3.cell_to_boundary()` for each index and renders an obfuscated polygon area (~0.73 km²).
   - Provides macro-level spatial awareness for staffing coordinators while mathematically preventing street-level identification of client or caregiver homes.

3. **Visual Heatmap & Role Signatures**:
   - Caregiver hexagon fills reflect capacity depth (darker green indicating higher `Available_Hours`).
   - Distinct border color accents differentiate credentials (`PCA`, `LPN`, `RN`).

4. **Bi-Directional User Interaction Loop**:
   - Clicking a client or staff hexagon on the Folium map updates `st.session_state` and triggers `st.rerun()`.
   - The app overrides active agent context directly from local state without invoking external LLM APIs.

---

## 4. Telemetry, Tracing & Audit Trail

Observability spans user requests, ADK agent tool routing, and external API interactions, providing operational traceability while maintaining patient data isolation.

```mermaid
%%{init: {"theme": "base", "themeVariables": {"fontFamily": "Arial, Helvetica, sans-serif", "fontSize": "22px", "primaryTextColor": "#000000", "lineColor": "#4B5563"}}}%%
flowchart TB
    subgraph A ["Implemented Request and Telemetry Path"]
        direction TB
        U["User Request"]
        A1["ask_agent Entrypoint"]
        A2["_ask_async & root_agent"]
        T["Local Agent Tools"]
        S["AGENT_CONTEXT Side-Channel"]
        UI["Streamlit Map Update"]

        U --> A1 --> A2 --> T --> S --> UI

        O["Opik Distributed Tracing<br/>(Tool & Orchestration Spans)"]
        L["Centralized Logging<br/>(stdout INFO / file DEBUG)"]

        A1 -.-> O
        A2 -.-> O
        T -.-> O
        A1 -.-> L
        T -.-> L
    end

    subgraph B ["Production Audit & Compliance Readiness"]
        direction LR
        G1["Inspect Captured<br/>Trace Fields"]
        G2["Join Events with<br/>Durable Request ID"]
        G3["Define Retention &<br/>Redaction Policies"]
        G4["Isolate Shared State<br/>for Concurrency"]
    end

    O -.-> G1
    O -.-> G2
    L -.-> G3
    S -.-> G4

    classDef flow fill:#EBF5FF,stroke:#2563EB,color:#000000,stroke-width:1.5px
    classDef state fill:#FEF9C3,stroke:#CA8A04,color:#000000,stroke-width:1.5px
    classDef telem fill:#EDE9FE,stroke:#7C3AED,color:#000000,stroke-width:1.5px
    classDef audit fill:#DCFCE7,stroke:#15803D,color:#000000,stroke-width:1.5px

    class U,A1,A2,T flow
    class S,UI state
    class O,L telem
    class G1,G2,G3,G4 audit
```

### Key Components

1. **Opik Distributed Tracing**:
   - Wraps agent entry points (`ask_agent`, `_ask_async`) and tool execution functions with `@track` decorators.
   - Captures latency, token consumption, and tool routing performance.

2. **Dual-Tier Centralized Logging (`src/logger.py`)**:
   - Structured logging powered by Loguru.
   - `stdout`: Formatted `INFO` level events for developer visibility.
   - File log: Daily rotating, compressed `DEBUG` logs in `logs/` for diagnostics.

3. **Production Audit & Compliance Verification**:
   - **Trace Field Inspection**: Ensures Opik spans only record sanitized IDs and spatial parameters, strictly excluding unmasked PHI/PII.
   - **Durable Request IDs**: Injects request-scoped UUIDs across log events and LLM spans for correlation.
   - **Log Retention & Redaction**: Automatically rolls and ages out historical local logs.
   - **Context Concurrency Isolation**: Isolates `AGENT_CONTEXT` to session memory to ensure multi-tab safety.
