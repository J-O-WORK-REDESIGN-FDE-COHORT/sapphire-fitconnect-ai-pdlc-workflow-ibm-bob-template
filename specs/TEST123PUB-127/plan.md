# Implementation Plan: Add Support for Body Temperature Metric Ingestion, Storage, and Reporting

**Branch**: `TEST123PUB-127` | **Date**: 2025-08-13 | **Spec**: [spec.md](spec.md)  
**Input**: Feature specification from `/specs/TEST123PUB-127/spec.md`

---

## Summary

TEST123PUB-127 adds body temperature as a first-class health metric across the Sapphire FitConnect platform through a multi-service feature spanning ingestion, storage, aggregation, and UI rendering. The implementation follows a phased, prioritized delivery approach:

- **P1 (Ingestion)**: Accept single and batch temperature submissions with validation, deduplication, and Kafka event publication via `sapphire-event-ingestion-api`.
- **P2 (Storage & Charting)**: Timeseries storage with daily/weekly/monthly rollups via Kafka pipeline; trend charting (min/max/avg) with filtering via `sapphire-charting-api`; GraphQL exposure via `sapphire-bff-api`; React chart component with unit switching in `Sapphire` UI.
- **P3 (Export)**: Temperature inclusion in health analytics export alongside existing metrics.

The technical approach leverages the existing Sapphire data pipeline (Kafka → PostgreSQL timeseries with TimescaleDB) and extends it with: (1) new Avro schema for temperature events, (2) PostgreSQL hypertable for temperature metrics, (3) aggregation rules in the Kafka pipeline, (4) new GraphQL fields in BFF, and (5) React chart component with loading/error/empty states.

---

## Technical Context

**Language/Version**: 
- Java 17 (Spring Boot 3.x) — `sapphire-event-ingestion-api`, `sapphire-charting-api`
- JavaScript/TypeScript (Node.js 18+, Express + Apollo GraphQL) — `sapphire-bff-api`
- TypeScript/React 18 — `Sapphire` UI
- Kafka (existing)
- PostgreSQL 15+ with TimescaleDB extension

**Primary Dependencies**:
- Spring Boot (REST endpoints, dependency injection, data access)
- Apache Kafka (event streaming, schema registry with Avro)
- PostgreSQL + TimescaleDB (timeseries storage, hypertables)
- Apollo GraphQL (BFF schema and resolvers)
- React + Apollo Client (UI chart component)

**Storage**: PostgreSQL 15+ with TimescaleDB extension; existing `timeseries_metrics` hypertable (or new partition if separate table preferred); Kafka topics for temperature events

**Testing**:
- Java: JUnit 5, Mockito, Testcontainers (Docker Compose for Kafka + PostgreSQL)
- TypeScript/React: Jest, React Testing Library, Apollo MockedProvider
- Integration: Docker Compose for multi-service validation

**Target Platform**: Linux containers (Kubernetes), cloud-native microservices

**Project Type**: Feature across 5 interconnected microservices with upstream/downstream dependencies

**Performance Goals**:
- Ingestion: Accept temperature submissions with acknowledgement within 500ms (p99) under normal load
- Data visible in charts within 30 seconds of submission (per SC-001)
- Charting API response within 3 seconds for day/week/month queries (per SC-004)
- Batch ingestion: Accept up to 100 records/request without N+1 query penalties

**Constraints**:
- Physiological range validation: 34.0–42.0°C (configurable via environment)
- Batch submission size: up to 100 records per request
- Idempotent deduplication: hash(user_id + device_source + timestamp + value)
- Unit conversion accuracy: ≤0.1° rounding error (per SC-006)
- Existing data pipeline patterns MUST be respected — no new patterns introduced

**Scale/Scope**: 
- Affected repositories: 5 (sapphire-event-ingestion-api, sapphire-kafka-pipeline, sapphire-charting-api, sapphire-bff-api, Sapphire UI)
- Affected domains: user telemetry ingestion, timeseries analytics, health metrics aggregation, GraphQL schema, React UI
- Estimated child stories: 5–7 (one per service + cross-cutting concerns)

---

## Constitution Check

**GATE: Must pass before Phase 1 design. Re-check after Phase 2 research completion.**

| # | Gate | Principle | Status |
|---|------|-----------|--------|
| 1 | All public functions/methods/classes have docstrings or Javadoc (intent, not implementation) | I. Code Quality | [ ] |
| 2 | No magic numbers or strings — named constants or enums used | I. Code Quality | [ ] |
| 3 | Cyclomatic complexity ≤ 10 per function/method confirmed via static analysis | I. Code Quality | [ ] |
| 4 | No commented-out code committed; feature flags or deletion used instead | I. Code Quality | [ ] |
| 5 | Stack-specific rules applied (Spring layering / PEP 8 + ruff / strict TS / BFF DataLoader) | I. Code Quality | [ ] |
| 6 | Coverage gates planned: Java 80%/100% domain, Python 80%, TS/React 70%, BFF resolvers 100% | II. Testing Standards | [ ] |
| 7 | Contract tests planned for all GraphQL schema changes and Kafka event schema changes | II. Testing Standards | [ ] |
| 8 | Test pyramid respected: unit (mocked I/O), integration (Docker Compose, pre-merge only), E2E | II. Testing Standards | [ ] |
| 9 | All data-fetching components handle loading skeleton / error boundary / empty state | III. UX Consistency | [ ] |
| 10 | Auth path is exclusively Keycloak OIDC/PKCE; no bypass routes in any environment | III. UX Consistency | [ ] |
| 11 | URL state is source of truth for filters, pagination, and selections | III. UX Consistency | [ ] |
| 12 | Apollo cache policies explicit; no implicit cache-first for mutable health data; cache TTL ≥ 30 s per session | I. Code Quality | [ ] |
| 13 | All services emit structured JSON logs (Logback+logstash / structlog / pino) with trace_id and span_id fields | IV. Observability | [ ] |
| 14 | OTEL metrics exported to Collector: request count, duration histogram, error rate, in-flight counter; custom business metrics added | IV. Observability | [ ] |
| 15 | Distributed traces emitted via OTEL SDK; W3C traceparent propagation used; DB, HTTP, and Kafka operations instrumented as child spans | IV. Observability | [ ] |
| 16 | OTEL_EXPORTER_OTLP_ENDPOINT, OTEL_SERVICE_NAME, and OTEL_DEPLOYMENT_ENVIRONMENT set in every container; no direct backend export from services | IV. Observability | [ ] |
| 17 | *(N/A for this feature)* LangGraph graph state, async nodes, persistent checkpointer, etc. | I. Code Quality | N/A |
| 18 | Markdown documentation covering purpose, usage, configuration, and known limitations committed in same PR as feature code; audit log entry added | V. Documentation & Audit | [ ] |
| 19 | Backward-compatibility assessment committed to `specs/TEST123PUB-127/` classifying schema changes as non-breaking/additive/breaking; consumer contract tests updated | VI. API Compatibility | [ ] |

---

## Project Structure

### Documentation (this feature)

```text
specs/TEST123PUB-127/
├── plan.md                      # This file (plan command output)
├── research.md                  # Phase 0 output: dependency analysis, schema research, existing patterns
├── data-model.md                # Phase 1 output: PostgreSQL hypertable schema, Avro event schema, API DTOs
├── contracts/
│   ├── avro-temperature-event.schema.json  # Temperature event schema
│   ├── graphql-temperature-fields.graphql  # New GraphQL schema fields for BFF
│   ├── rest-ingestion-contract.md          # Temperature ingestion endpoint contract
│   └── chart-api-contract.md               # Charting API contract
├── quickstart.md                # Phase 1 output: developer setup, local testing flow
├── backwards-compat-assessment.md # Phase 1 output: API compatibility analysis
└── tasks.md                     # Phase 2 output (/speckit.tasks command — NOT created by /speckit.plan)
```

### Source Code (by repository)

#### 1. `sapphire-event-ingestion-api` (Python FastAPI)

```text
sapphire-event-ingestion-api/
├── src/
│   ├── models/
│   │   └── temperature.py           # Pydantic models for temperature ingestion
│   ├── services/
│   │   ├── temperature_validator.py  # Validation logic (range, unit, timestamp)
│   │   ├── deduplication_service.py  # Idempotent deduplication (hash-based)
│   │   └── kafka_publisher.py        # Kafka event publishing
│   ├── api/
│   │   └── temperature_routes.py     # FastAPI routes: POST /temperatures (single + batch)
│   └── main.py
├── tests/
│   ├── unit/
│   │   ├── test_temperature_validator.py
│   │   ├── test_deduplication_service.py
│   │   └── test_kafka_publisher.py
│   └── integration/
│       └── test_temperature_ingestion_flow.py
└── requirements.txt
```

#### 2. `sapphire-kafka-pipeline` (Kafka Connect + PostgreSQL)

```text
sapphire-kafka-pipeline/
├── connectors/
│   └── postgres-sink-config-temperature.json  # Sink connector config for temperature events
├── schemas/
│   └── temperature-event-schema.avro         # Avro schema for temperature events
├── sql/
│   ├── 001_create_temperature_hypertable.sql  # TimescaleDB hypertable for temperatures
│   ├── 002_create_aggregation_views.sql      # Views for daily/weekly/monthly rollups
│   └── 003_aggregation_trigger.sql           # Trigger function for rollup computation
└── README.md
```

#### 3. `sapphire-charting-api` (Java Spring Boot)

```text
sapphire-charting-api/
├── src/main/java/com/sapphire/charting/
│   ├── domain/
│   │   ├── TemperatureTrendSummary.java       # Domain model for aggregated trends
│   │   └── TemperatureRecord.java             # Domain model for single reading
│   ├── service/
│   │   └── TemperatureTrendService.java       # Query logic for trend data
│   ├── controller/
│   │   └── TemperatureChartController.java    # REST endpoint: GET /charts/temperatures
│   ├── repository/
│   │   └── TemperatureRepository.java         # Data access (JPA/Reactive)
│   └── config/
│       └── TemperatureChartConfiguration.java # Configuration (physiological range, etc.)
├── src/test/java/com/sapphire/charting/
│   ├── service/
│   │   └── TemperatureTrendServiceTest.java
│   ├── controller/
│   │   └── TemperatureChartControllerTest.java
│   └── integration/
│       └── TemperatureChartIntegrationTest.java
└── pom.xml
```

#### 4. `sapphire-bff-api` (Node.js/Express + Apollo GraphQL)

```text
sapphire-bff-api/
├── src/
│   ├── schema/
│   │   ├── temperature.graphql       # GraphQL schema: Temperature, TemperatureTrendData types
│   │   └── query-extensions.graphql  # Extended Query type with temperature fields
│   ├── resolvers/
│   │   └── temperature-resolvers.js  # Resolver functions for temperature queries
│   ├── services/
│   │   ├── charting-service.js       # HTTP client to sapphire-charting-api
│   │   └── analytics-service.js      # HTTP client for export endpoint
│   └── middleware/
│       └── jwt-validation.js         # JWT claim validation (existing pattern)
├── tests/
│   ├── unit/
│   │   ├── temperature-resolvers.test.js
│   │   └── charting-service.test.js
│   └── integration/
│       └── temperature-graphql.test.js
└── package.json
```

#### 5. `Sapphire` (React + TypeScript UI)

```text
Sapphire/
├── src/
│   ├── features/
│   │   └── temperature/
│   │       ├── TemperatureChart.tsx          # Main chart component
│   │       ├── TemperatureChart.module.css   # Styling
│   │       ├── useTemperatureData.ts         # Apollo query hook
│   │       ├── types.ts                      # TypeScript interfaces
│   │       └── constants.ts                  # Named constants (units, ranges, etc.)
│   ├── components/
│   │   └── EmptyState.tsx                    # Shared empty state component (already exists)
│   ├── hooks/
│   │   └── useChartData.ts                   # Shared charting hook
│   └── utils/
│       └── temperature-conversion.ts         # Celsius ↔ Fahrenheit conversion utilities
├── tests/
│   ├── features/temperature/
│   │   ├── TemperatureChart.test.tsx
│   │   └── useTemperatureData.test.ts
│   └── utils/
│       └── temperature-conversion.test.ts
└── package.json
```

**Structure Decision**: Multi-repository feature with clear separation of concerns per microservice. Each repository retains its existing patterns (Spring layering for Java services, FastAPI dependency injection for Python, Express middleware for BFF, React hooks for UI). No new architectural patterns introduced.

---

## Implementation Phases

### Phase 0: Research (Completion before Phase 1 Design)

**Outputs**: `research.md` (to be created by research subtask)

1. **Dependency Analysis**
   - Survey existing Avro schema patterns in `sapphire-kafka-pipeline`
   - Identify available configuration mechanisms for physiological range defaults
   - Research existing batch submission handling patterns in `sapphire-event-ingestion-api`
   - Map PostgreSQL timeseries table structure and existing metric types

2. **Existing Pattern Documentation**
   - Document ingestion validation pattern (where validation errors are caught, how they're logged)
   - Document Kafka event publishing pattern (partitioning, error handling, retries)
   - Document timeseries aggregation pattern (existing rollup logic, trigger mechanisms)
   - Document GraphQL resolver pattern in BFF (DataLoader usage, JWT validation flow)
   - Document React chart component patterns (loading/error/empty states, Apollo cache policies)

3. **Infrastructure Requirements**
   - Confirm Kafka topic naming conventions and retention policies
   - Verify PostgreSQL replica lag and query performance assumptions
   - Confirm OTEL endpoint configuration and metric export patterns
   - Identify test environment setup (Docker Compose, fixtures, seed data)

### Phase 1: Design (Completion before Phase 2 Tasks Creation)

**Outputs**: `data-model.md`, `contracts/`, `quickstart.md`, `backwards-compat-assessment.md`

1. **Data Model Design** (`data-model.md`)
   - PostgreSQL schema: hypertable creation DDL, indexes for range queries
   - Avro event schema: field mappings, version strategy, compatibility rules
   - API DTOs: request/response shapes for ingestion and charting endpoints
   - GraphQL type definitions: Temperature, TemperatureTrendData, query fields

2. **API Contracts** (`contracts/`)
   - **Avro Temperature Event Schema**
     - Fields: user_id, value, unit (enum: CELSIUS, FAHRENHEIT), timestamp_iso8601, device_source, ingestion_source (enum: DEVICE_PUSH, API), measurement_method (optional), created_at
     - Schema version: initial version with compatibility rules for future extensions
   
   - **REST Ingestion Endpoint Contract** (`rest-ingestion-contract.md`)
     - POST `/api/temperatures` — single record submission
     - POST `/api/temperatures/batch` — batch submission (≤100 records)
     - Request/response body shapes, validation error codes, HTTP status codes
   
   - **GraphQL Schema Fields** (`graphql-temperature-fields.graphql`)
     - `type Temperature { id, value, unit, timestamp, deviceSource }`
     - `type TemperatureTrendData { minValue, maxValue, avgValue, unit, granularity }`
     - Query extension: `temperatureData(userId, dateRange, deviceSource): TemperatureTrendData`
   
   - **Charting API Contract** (`chart-api-contract.md`)
     - GET `/api/charts/temperatures?userId={id}&granularity={day|week|month}&dateFrom={iso}&dateTo={iso}&deviceSource={source}` — trend data endpoint
     - Response body: min, max, average values per time bucket, with metadata

3. **Backward Compatibility Assessment** (`backwards-compat-assessment.md`)
   - **PostgreSQL**: New hypertable (non-breaking) — no schema changes to existing metrics tables
   - **Kafka**: New temperature topic (non-breaking) — no changes to existing event schemas
   - **Avro**: Temperature event schema is net-new (additive change to overall schema registry)
   - **GraphQL**: New optional fields on Query type (additive, non-breaking)
   - **REST (Ingestion API)**: New endpoint (additive, non-breaking)
   - **REST (Charting API)**: New endpoint (additive, non-breaking)
   - **Consumer Contract Tests**: Verify that existing health metrics ingestion/charting endpoints are unaffected

4. **Developer Quickstart** (`quickstart.md`)
   - Prerequisites: Docker, Docker Compose, Java 17, Node.js 18+, Python 3.11+
   - Local setup: `docker-compose up` command to spin up Kafka + PostgreSQL + services
   - Data seeding: Script to load test temperature records
   - Testing workflow: How to trigger temperature ingestion, query charts, view GraphQL data
   - Example curl/GraphQL queries for each endpoint

---

## Work Breakdown Summary

### sapphire-event-ingestion-api (Python FastAPI)

| Task | Component | Effort | Dependencies |
|------|-----------|--------|--------------|
| Pydantic models for temperature ingestion | Models | 1d | None |
| Validation service (range, unit, timestamp) | Service | 2d | Constitution Check gate 1,2,3 |
| Deduplication service (hash-based) | Service | 2d | Validator service |
| Kafka publisher for temperature events | Service | 1d | Research: Kafka patterns |
| FastAPI routes (single + batch submission) | API | 2d | All services above, Constitution gate 5 |
| Unit tests (validators, deduplication, publishing) | Tests | 3d | Constitution gate 6 |
| Integration test (end-to-end ingestion flow) | Tests | 2d | Constitution gate 8, Docker Compose |
| **Subtotal** | — | **13d** | Phase 1 research complete |

### sapphire-kafka-pipeline (Kafka Connect + SQL)

| Task | Component | Effort | Dependencies |
|------|-----------|--------|--------------|
| Design & DDL for temperature hypertable | Schema | 1d | Research: existing table structure |
| Create aggregation views (day/week/month rollups) | Schema | 2d | Hypertable DDL |
| Create rollup computation trigger | Schema | 1d | Aggregation views |
| Kafka Connect sink connector config | Config | 1d | Research: Kafka patterns, Avro schema |
| Integration test (event flow Kafka → PostgreSQL) | Tests | 2d | Constitution gate 8 |
| **Subtotal** | — | **7d** | Phase 1 research, Avro schema from ingestion-api |

### sapphire-charting-api (Java Spring Boot)

| Task | Component | Effort | Dependencies |
|------|-----------|--------|--------------|
| Domain models (TemperatureTrendSummary, TemperatureRecord) | Domain | 1d | Data model design |
| Repository interface + JPA implementation | Repository | 2d | Constitution Check gate 1, PostgreSQL schema ready |
| Trend query service (min/max/avg aggregation logic) | Service | 3d | Repository, Constitution gate 2,3 |
| REST controller + endpoint | Controller | 2d | Service, Constitution gate 5 |
| Unit tests (service, repository) | Tests | 3d | Constitution gate 6 |
| Integration test (end-to-end charting flow) | Tests | 2d | Constitution gate 8, Docker Compose, Kafka pipeline ready |
| OpenAPI documentation | Docs | 1d | Constitution gate 18 |
| **Subtotal** | — | **14d** | Phase 1 design complete, Kafka pipeline ready |

### sapphire-bff-api (Node.js/Express + Apollo GraphQL)

| Task | Component | Effort | Dependencies |
|------|-----------|--------|--------------|
| GraphQL schema definition (Temperature types, Query extensions) | Schema | 1d | GraphQL contract design |
| Resolvers for temperature queries | Resolvers | 2d | Charting API ready, Constitution gate 5 |
| HTTP client to charting-api | Service | 1d | Service client patterns |
| Unit tests (resolvers, service client) | Tests | 2d | Constitution gate 6 |
| Integration test (GraphQL queries end-to-end) | Tests | 2d | Constitution gate 8, charting-api available |
| DataLoader setup (if needed for N+1 prevention) | Config | 1d | Constitution gate 12 |
| **Subtotal** | — | **9d** | Phase 1 design complete, charting-api ready |

### Sapphire UI (React + TypeScript)

| Task | Component | Effort | Dependencies |
|------|-----------|--------|--------------|
| TypeScript types and constants (units, ranges, colors) | Types | 1d | Constitution Check gates 1,2 |
| Temperature conversion utilities (C ↔ F) | Utils | 1d | Constitution gate 2 |
| Apollo query hook for temperature data | Hooks | 2d | BFF GraphQL ready, Constitution gate 12 |
| Chart component (min/max/avg visualization) | Component | 3d | React charting library (recharts/visx), Constitution gate 9 |
| Loading/error/empty state handling | Component | 2d | Existing EmptyState component, Constitution gate 9 |
| Unit conversion toggle (C/Fahrenheit) | Component | 1d | Conversion utils |
| Unit tests (component, hook, utils) | Tests | 3d | Constitution gate 6 |
| Integration/snapshot tests (chart rendering) | Tests | 2d | Constitution gate 8, MockedProvider |
| **Subtotal** | — | **15d** | Phase 1 design complete, BFF ready |

### Cross-Cutting

| Task | Effort | Dependencies |
|------|--------|--------------|
| OTEL instrumentation (all 5 services) | 5d | Constitution gate 13,14,15,16 |
| Backward-compatibility consumer contract tests | 3d | Constitution gate 19 |
| Documentation (purpose, usage, config, limitations) | 2d | Constitution gate 18 |
| Audit log entries | 0.5d | Constitution gate 18 |
| **Subtotal** | **10.5d** | Phase 1 design, all services ready |

---

## Risk Mitigation

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|-----------|
| Batch submission edge case handling not well-defined | Medium | High | **Action**: Clarify during Phase 0 research (existing batch patterns) or Phase 1 design gate; document explicit decision in `data-model.md` |
| PostgreSQL timeseries query performance under high cardinality | Low | Medium | **Action**: Phase 0 research into existing table indexing, query plans; Phase 1: define index strategy; Phase 2: load test with realistic data volumes |
| GraphQL N+1 queries for user → temperature trend data | Medium | Medium | **Action**: Constitution gate 12 enforces DataLoader; Phase 1 design includes DataLoader pattern for temperature resolver |
| React chart library performance with large date ranges | Medium | Low | **Action**: Phase 1 design specifies pagination/bucketing strategy; Phase 2 tasks include performance benchmarking |
| Unit conversion rounding errors exceeding 0.1° threshold | Low | Low | **Action**: Constitution gate 2 (named constants); Phase 2 tasks include unit test for conversion accuracy; SC-006 measured in test |
| OTEL instrumentation complexity | Low | Medium | **Action**: Constitution gate 13–16 enforces; Phase 0 research into existing patterns; reference existing service implementations |

---

## Entry and Exit Criteria

### Phase 1 Design (Entry Gates)

- [ ] Constitution Check gates 1–7, 9–16 baseline reviewed and approved by FDE
- [ ] Specification (spec.md) approved by Product Owner
- [ ] Research phase (Phase 0) complete with findings documented in `research.md`

### Phase 1 Design (Exit Gates / Phase 2 Entry Gates)

- [ ] `data-model.md` complete and reviewed by Architect
- [ ] Contract definitions in `contracts/` complete and approved by API owners (ingestion-api, charting-api, BFF)
- [ ] Backward-compatibility assessment (`backwards-compat-assessment.md`) complete and approved by Tech Lead
- [ ] `quickstart.md` complete with reproducible local test flow
- [ ] Phase 1 Design Checkpoint PR raised and approved (by FDE)

### Phase 2 Tasks (Entry Gates)

- [ ] Phase 1 design complete (exit gates passed)
- [ ] All Constitution Check gates passed
- [ ] Phase 2 tasks PR raised and approved by FDE

### Phase 2 Tasks (Exit Gates / Phase 3 Implementation Entry Gates)

- [ ] Child stories created in JIRA for each service + cross-cutting work
- [ ] Task list (`tasks.md`) complete with acceptance criteria tied to child stories
- [ ] Implementation queue and priority ordering established
- [ ] Phase 2 Tasks Checkpoint PR merged

---

## Next Steps

1. **Immediate**: Schedule Phase 0 Research subtask — confirm Kafka/PostgreSQL patterns, test environment setup, existing validation/aggregation implementations.
2. **Upon Phase 0 completion**: Proceed to Phase 1 Design — create data model, finalize contracts, produce backward-compatibility assessment.
3. **Upon Phase 1 completion**: Raise Phase 1 Design Checkpoint PR for FDE approval; upon approval, proceed to `/speckit.tasks` for Phase 2 task generation.

---

**Prepared by**: `/speckit.plan`  
**Created**: 2025-08-13  
**Version**: 1.0 (Draft)
