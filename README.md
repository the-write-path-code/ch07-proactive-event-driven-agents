# Chapter 7: Proactive and Event-Driven Agents

Companion code for *Building Safe Agentic AI for Enterprise Systems* by Mohit Aggarwal.

This repository implements a privacy-preserving staffing dashboard for Directors of Nursing. It helps identify available caregivers near a client without placing raw client addresses, caregiver addresses, or exact coordinates in the language model's context.

The system separates geographic processing from agent reasoning. An extract-transform-load (ETL) step geocodes source addresses, converts coordinates to H3 cells, and minimizes the retained data. Deterministic Python tools then calculate proximity and update the map through a side channel. The model can request a staffing query, but it cannot write SQL, retrieve raw locations, or drive the map from generated prose.

## What You Will Run

| Chapter section | Demonstration | What it shows |
| --- | --- | --- |
| 7.1 | Privacy-aware data preparation | Address inputs are geocoded, converted to H3 cells, and removed from the agent-facing data model. |
| 7.2 | Spatial privacy with H3 | A staffing query uses H3 grid distance rather than exact latitude and longitude. |
| 7.3 | LLM bypass side channel | Deterministic tools write map state directly for the interface to render. The map does not parse locations from model output. |
| 7.4 | Deterministic tool-calling | The agent calls registered Python tools rather than generating SQL or receiving database access. |
| 7.5 | Tracing and auditability | Loguru and optional Opik traces record the operation path and tool results. |

The repository demonstrates a design pattern for protected operational data. It is not a substitute for a complete healthcare privacy, security, retention, or access-control program.

## Production Warning

H3 reduces location precision; it does not make location data anonymous. A cell can still be sensitive when it is combined with names, time windows, workload data, external maps, or other auxiliary information. Treat H3 indexes and proximity results as sensitive operational data, especially when they relate to clients, caregivers, or care delivery.

The source spreadsheet inputs can contain addresses and other personally identifiable information (PII). Keep them out of Git, use synthetic or properly approved test data for local demonstrations, and do not send them to a hosted model, tracing platform, or unapproved third-party service.

## Prerequisites

- Git
- [uv](https://docs.astral.sh/uv/)
- Python 3.9 or later, matching the repository's current project requirement
- A Google Gemini API key for the conversational staffing assistant
- A Geocodio API key for address geocoding during ETL
- Optional: an Opik API key for tracing

The application uses SQLite locally. Docker is not required for the standard dashboard path.

## Quick Start

### 1. Install uv

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

### 2. Clone and synchronize the repository

```bash
git clone https://github.com/the-write-path-code/ch07-proactive-event-driven-agents.git
cd ch07-proactive-event-driven-agents
uv sync
```

The repository currently does not include a committed `uv.lock`. `uv sync` resolves dependencies from `pyproject.toml`. Before public book release, generate and commit `uv.lock` so readers can reproduce the tested dependency set.

### 3. Create local configuration

```bash
cp .env.example .env
```

Set the required credentials:

```dotenv
GOOGLE_API_KEY=your-google-ai-studio-key
GEOCODIO_API_KEY=your-geocodio-key
```

Optional tracing settings:

```dotenv
OPIK_API_KEY=your-opik-key
OPIK_PROJECT_NAME=agentic-healthcare-staffing
```

Do not commit `.env` or any source spreadsheets containing addresses, names, phone numbers, or other sensitive data.

### 4. Start the dashboard

```bash
uv run streamlit run src/app.py
```

Streamlit opens the dashboard at the local URL printed in the terminal, normally `http://localhost:8501`.

## Configuration

| Variable | Required? | Purpose |
| --- | --- | --- |
| `GOOGLE_API_KEY` | Yes for the assistant | Google Gemini credential used for the agent-facing conversation layer |
| `GEOCODIO_API_KEY` | Yes for address geocoding | Geocodes source addresses during ETL before location data is minimized |
| `OPIK_API_KEY` | No | Enables optional Opik tracing |
| `OPIK_PROJECT_NAME` | No | Names the Opik project used for traces |

The agent should receive only the data that its registered tools return. Do not add raw addresses, phone numbers, names, latitude values, longitude values, or raw SQL capability to a model prompt, tool result, or trace without a reviewed data-handling decision.

> **Tip**
>
> Run the application first with synthetic source spreadsheets. Verify the database schema and tool-return payloads before loading any approved operational dataset.

## Run the Chapter Demonstrations

### 1. Run the Privacy-Aware ETL Path, Sections 7.1 and 7.2

Place the approved demonstration inputs in the repository's data directory. The dashboard's refresh path runs the ETL process that:

1. Reads the client and caregiver source records.
2. Builds an address only for the geocoding step.
3. Converts geocoded coordinates to H3 cells at Resolution 8.
4. Stores minimized client and staff records for staffing queries.
5. Retains an opaque change-detection hash so future source changes can be detected without reusing address data in the agent path.

The system uses H3 grid distance to find caregivers near a client. A requested mile radius is converted to a conservative H3 grid threshold before the tool evaluates available staff.

### 2. Run a Deterministic Staffing Query, Sections 7.2 through 7.4

Start the dashboard and use one of the supplied examples:

```text
Find a PCA near Client C005.
How many RNs are available?
Find LPNs within 15 miles of Client C001.
What about PCAs instead?
```

The model interprets the request and invokes a registered tool. The tool queries the local database, calculates H3 grid distance, applies role and availability filters, and returns a restricted result shape. The model does not construct SQL and does not receive a raw client address, caregiver address, phone number, or coordinate.

### 3. Inspect the Map Side Channel, Section 7.3

A staffing tool writes map selections and result state to the application-side context. Streamlit reads that deterministic state after the tool call and renders the map.

This distinction matters. The interface does not extract locations, filters, or map updates from generated model prose. A model response can explain the result, but it does not control the geographic display.

### 4. Enable Optional Tracing, Section 7.5

Set `OPIK_API_KEY` and the project name in `.env`, then run the dashboard normally. Before enabling tracing with non-synthetic data, review the trace payloads and retention policy. A trace that includes a tool result can become a second copy of sensitive operational data.

## Expected Results

A normal staffing query should produce:

- A limited list of eligible caregivers or a clear no-match response.
- Approximate proximity information derived from H3 cells.
- Role and availability information needed for the staffing decision.
- A deterministic map update based on the tool result.
- A conversational explanation that does not become the source of truth for the map.

The important boundary is what the model cannot obtain. It should not receive or infer raw GPS coordinates, street addresses, direct database access, or unrestricted user records.

## Run the Tests

```bash
uv run pytest
```

Run the test suite before changing the ETL pipeline, H3 resolution, retention rules, tool return fields, map context, tracing, or agent instructions. The tests should establish that:

- Raw coordinate fields do not appear in the agent-facing client and staff records.
- The ETL process strips fields outside the approved retained set.
- Change-detection hashing does not expose raw address strings through its stored output.
- Agent tools return only their documented fields.
- The map reads deterministic side-channel state rather than generated text.
- Error and no-match paths have defined result contracts.

## Repository Layout

```text
.
├── README.md
├── pyproject.toml
├── .env.example
├── src/
│   ├── app.py                         # Streamlit dashboard
│   ├── agent.py                       # ADK agent and registered staffing tools
│   └── logger.py                      # Loguru configuration
├── etl/
│   └── sync.py                        # Address processing, H3 conversion, and SQLite sync
├── data/                              # Local input spreadsheets and generated database; keep sensitive inputs out of Git
├── logs/                              # Local application logs
├── workflows/
│   └── architecture_workflows.md      # Mermaid diagrams and lifecycle documentation
└── tests/
```

## Architecture Diagrams and Supporting Documents

The `workflows/architecture_workflows.md` document contains the diagrams used in Chapter 7:

- The address-to-H3 ETL flow and data minimization boundary.
- Deterministic agent-tool routing.
- The map side channel and Streamlit rendering path.
- Tracing and audit flow.

Read the data-ingestion diagram before changing field retention. The most important question is not whether a field is useful. It is whether the agent, database, user interface, or trace system actually needs that field after the geocoding step has finished.

## Safety and Operational Limits

- H3 cells reduce precision but remain sensitive location data when combined with other records or external sources.
- Source spreadsheets and the geocoding cache can contain raw addresses. Keep them outside version control and restrict access to them.
- The model must not generate SQL or gain direct database access. Enforce the database boundary through registered tools.
- Tool return values and traces require the same privacy review as API responses. A field that is safe in a local SQLite table may not be safe in a model context or third-party tracing service.
- The current map context is application state. If the system accepts overlapping requests, replace shared module-level state with request-scoped storage and test the concurrency behavior.
- This repository demonstrates data minimization and tool boundaries. It does not certify HIPAA compliance or replace legal, security, or organizational controls.

## Troubleshooting

### The dashboard cannot start

Synchronize dependencies, then start Streamlit from the repository root:

```bash
uv sync
uv run streamlit run src/app.py
```

### Geocoding fails

Confirm that `GEOCODIO_API_KEY` is set and that the source inputs contain the fields the ETL process expects. Do not log full addresses while debugging. Use synthetic data or redacted identifiers in error reports.

### The assistant cannot answer a staffing request

Confirm that `GOOGLE_API_KEY` is set, the ETL process completed, and the local SQLite data store contains the expected H3-indexed records. Inspect tool results before changing agent prompts.

### The map does not reflect the tool result

Inspect the deterministic side-channel state written by the tool and read by the Streamlit application. Do not fix a map update by parsing the model's response text.

### An H3 proximity result looks wrong

Check the configured H3 resolution, the miles-to-grid conversion, and whether the client and staff records were geocoded from the intended source inputs. H3 grid steps are an approximation and should be validated against the operating use case.

## Related Chapters

- Chapter 2 introduces the Agentic Context Layer and the need to partition state and responsibilities.
- Chapter 6 applies typed extraction and controlled handoffs to multimodal and potentially sensitive inputs.
- Chapter 8 extends tool-boundary design through the Model Context Protocol.
- Chapter 14 adds fail-closed action gates, approval holds, and human review for regulated workflows.
- Chapter 15 turns safety and data-boundary checks into continuous regression tests.

## License and Errata

See `LICENSE` for licensing terms. Report documentation or code issues through this repository's GitHub issue tracker.
