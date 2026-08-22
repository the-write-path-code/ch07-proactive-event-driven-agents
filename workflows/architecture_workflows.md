# 🏛️ Architecture & System Workflows

This document provides a comprehensive visual and technical reference for all operational workflows in the **Agentic Staffing Dashboard**. It details data synchronization, AI agent orchestration, privacy-preserving spatial visualization, and telemetry tracing.

---

## 📑 Table of Contents
1. [Data Ingestion & Privacy ETL Pipeline](#1-data-ingestion--privacy-etl-pipeline)
2. [Agent Orchestration & Deterministic Tool Routing](#2-agent-orchestration--deterministic-tool-routing)
3. [Privacy Map Rendering & Event Loop](#3-privacy-map-rendering--event-loop)
4. [Telemetry, Tracing & Audit Trail](#4-telemetry-tracing--audit-trail)

---

## 1. Data Ingestion & Privacy ETL Pipeline

The Data Synchronization Pipeline (ETL) ingests raw Excel exports from the agency management system, verifies schema integrity, geocodes addresses incrementally, and writes privacy-safe Uber H3 spatial indices to the local SQLite database.

```mermaid
flowchart LR
    subgraph P1["1. Ingest and Validate"]
        direction TB
        Start([Manual Trigger or App Boot]) --> ReadExcel[Read CustomerData.xlsx and CaregiverData.xlsx]
        ReadExcel --> Validate{Validate Required Columns}
        Validate -->|Missing| Abort([Abort Sync Show Error in UI])
        Validate -->|Valid| FillOptional[Fill Missing Optional Columns with NA]
        FillOptional --> Exclusions[Apply Hardcoded Name Exclusions]
        Exclusions --> SurrogateKey[Compute Secure Hash Keys for Changes]
    end

    subgraph P2["2. Change Detection"]
        direction TB
        Compare{Compare with Existing DB}
        Compare -->|New or Address Changed| GeocodeQueue[Add to Geocoding Queue]
        Compare -->|No Address Change| Preserve[Preserve Existing H3 Index]
    end

    subgraph P3["3. Geocoding Pipeline"]
        direction TB
        CheckCache{Check Local Geocode Cache}
        CheckCache -->|Cached| FetchCache[Fetch Cached Lat Lng]
        CheckCache -->|Not Cached| CallGeocodio[Batch Call Geocodio API Address ONLY]
        CallGeocodio --> SaveCache[Save Lat Lng to Cache]
        SaveCache --> FetchCache
        FetchCache --> H3Conversion[Convert to H3 Hex Index Res 8]
        H3Conversion --> Discard[Discard Raw Coordinates Never Store Lat Lng]
    end

    subgraph P4["4. Database Rebuild"]
        direction TB
        RebuildDB[Rebuild SQLite Tables clients and staff]
        RebuildDB --> RebuildView[Rebuild vw_staff_capacity Calculating Available Hours]
        RebuildView --> End([Sync Complete Update UI Summary])
    end

    P1 --> P2
    P2 --> P3
    P3 --> P4

    style Start fill:#e1f5e1
    style End fill:#e1f5e1
    style Abort fill:#ffe6e6
    style CallGeocodio fill:#e6f3ff
    style H3Conversion fill:#f0e6ff
    style Discard fill:#ffe6e6
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

```mermaid
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

    style Start fill:#e1f5e1
    style Output fill:#e1f5e1
    style App fill:#fff4e6
    style ADK fill:#e6f3ff
    style ToolContext fill:#ffe6e6
    style UpdateUI fill:#e6e6fa
    style Compose fill:#f0e6ff
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

---

## 3. Privacy Map Rendering & Event Loop

The map visualization displays spatial proximity and staff capacity using obfuscated H3 polygons instead of pin-point GPS markers, keeping all PII rendering strictly client-side.

```mermaid
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

    style Start fill:#e1f5e1
    style SyncState fill:#e6f3ff
    style MapHex fill:#f0e6ff
    style ClickMap fill:#fff4e6
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
flowchart TB
    classDef flow fill:#E8F1FB,stroke:#2F5597,color:#111827,stroke-width:1.5px
    classDef state fill:#F3E8FF,stroke:#7E57C2,color:#111827,stroke-width:1.5px
    classDef telemetry fill:#FFF4D6,stroke:#B7791F,color:#111827,stroke-width:1.5px
    classDef gap fill:#FDE8E7,stroke:#C62828,color:#111827,stroke-width:1.5px,stroke-dasharray: 5 4

    subgraph A["Implemented Request and Telemetry Path"]
        direction TB
        U[User request]:::flow
        A1[ask_agent]:::flow
        A2[_ask_async and root_agent]:::flow
        T[Local tools]:::flow
        S[AGENT_CONTEXT and result context]:::state
        UI[Streamlit map update]:::state

        U --> A1 --> A2 --> T --> S --> UI

        O[Opik tracking<br/>tool and orchestration boundaries]:::telemetry
        L[Logging<br/>stdout INFO; file DEBUG]:::telemetry

        A1 -.-> O
        A2 -.-> O
        T -.-> O
        A1 -.-> L
        T -.-> L
    end

    subgraph B["Production Audit & Compliance Readiness"]
        direction LR
        G1[Inspect captured<br/>trace fields]:::gap
        G2[Join events with a<br/>durable request ID]:::gap
        G3[Define retention, access,<br/>redaction, and export]:::gap
        G4[Replace shared state for<br/>overlapping requests]:::gap
    end

    O -.-> G1
    O -.-> G2
    L -.-> G3
    S -.-> G4
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
