# Tasks: Add Support for Body Temperature Metric Ingestion, Storage, and Reporting

**Branch**: `TEST123PUB-127` | **Input**: `specs/TEST123PUB-127/plan.md` + `spec.md` + `analysis.md`
**Story**: [TEST123PUB-127](https://stg.jsw.ibm.com/browse/TEST123PUB-127)
**Child Stories**: [TEST123PUB-132](https://stg.jsw.ibm.com/browse/TEST123PUB-132) (ingestion) · [TEST123PUB-129](https://stg.jsw.ibm.com/browse/TEST123PUB-129) (kafka) · [TEST123PUB-130](https://stg.jsw.ibm.com/browse/TEST123PUB-130) (charting) · [TEST123PUB-131](https://stg.jsw.ibm.com/browse/TEST123PUB-131) (bff) · [TEST123PUB-133](https://stg.jsw.ibm.com/browse/TEST123PUB-133) (ui)

> **Analysis findings applied**: F-1 (batch semantics decision), F-3 (FastAPI Depends), F-4 (Java record/Optional), F-5 (URL state), F-6 (OTEL business metrics), F-7 (Docker Compose fixture)

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Establish cross-repo foundations — Avro schema, PostgreSQL DDL, Docker Compose test fixture — that all downstream work depends on.

- [ ] T001 Create Avro temperature event schema at `sapphire-kafka-pipeline/schemas/temperature-event-schema.avro` with fields: `user_id` (string), `value` (double), `unit` (enum: CELSIUS, FAHRENHEIT), `timestamp_iso8601` (string), `device_source` (string), `ingestion_source` (enum: DEVICE_PUSH, API), `measurement_method` (string, optional/nullable), `created_at` (string); set schema compatibility to BACKWARD
- [ ] T002 [P] Create PostgreSQL DDL migration `sapphire-kafka-pipeline/sql/001_create_temperature_hypertable.sql`: create table `temperature_metrics` with columns `user_id`, `value`, `unit`, `recorded_at` (timestamptz), `device_source`, `ingestion_source`, `measurement_method` (nullable), `dedup_hash` (text unique); convert to TimescaleDB hypertable on `recorded_at`; guard all DDL with `IF NOT EXISTS` (analysis R-8)
- [ ] T003 [P] Create aggregation views migration `sapphire-kafka-pipeline/sql/002_create_aggregation_views.sql`: daily, weekly, monthly continuous aggregates (`temperature_daily`, `temperature_weekly`, `temperature_monthly`) computing `min(value)`, `max(value)`, `avg(value)` per `user_id` + `device_source` bucket
- [ ] T004 [P] Create rollup trigger migration `sapphire-kafka-pipeline/sql/003_aggregation_refresh_policy.sql`: add TimescaleDB refresh policies for each continuous aggregate with appropriate lag and schedule intervals
- [ ] T005 [P] Create minimal `sapphire-kafka-pipeline/docker-compose-test.yml` with Kafka (bitnami/kafka), Schema Registry, and PostgreSQL+TimescaleDB for integration test environments (analysis F-7)

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Kafka Connect sink for temperature events must be in place before ingestion-api publishes live events; defines the contract the ingestion service writes to.

**⚠️ CRITICAL**: Phases 3–5 depend on Phase 1 being complete. Phase 3 (ingestion) can begin in parallel with Phase 2 once T001 (Avro schema) is done.

- [ ] T006 Create Kafka Connect sink connector config `sapphire-kafka-pipeline/connectors/postgres-sink-config-temperature.json`: subscribe to `temperature-events` topic; map Avro fields to `temperature_metrics` columns; set `insert.mode=insert`, `pk.mode=record_value`, `pk.fields=dedup_hash`; configure `transforms` for timestamp conversion
- [ ] T007 [P] Update `sapphire-kafka-pipeline/README.md` to document the temperature pipeline: Avro schema reference, topic name, connector config, DDL migration order, and local test instructions using `docker-compose-test.yml`

**Checkpoint**: Kafka → PostgreSQL pipeline for temperature events is defined and deployable.

---

## Phase 3: User Story 1 — Ingest Body Temperature from Devices and APIs (Priority: P1) 🎯 MVP

**Goal**: Accept single and batch temperature submissions; validate, deduplicate, and publish to Kafka. No UI or chart required — data in the pipeline is the deliverable.

**Independent Test**: `POST /api/temperatures` with valid payload → 201 + record retrievable. `POST /api/temperatures/batch` with 100 records → all 100 stored, no duplicates. Invalid value → 422 with error body specifying valid range. (Refs: TEST123PUB-132)

**Batch semantics decision (F-1)**: Adopt **partial-accept** semantics consistent with existing ingestion-api batch patterns — valid records in a batch are stored and acknowledged; invalid records are returned as per-item errors in the response body. The overall HTTP status is 207 Multi-Status when the batch contains a mix.

### sapphire-event-ingestion-api — Pydantic Models

- [ ] T008 [P] [US1] Create `sapphire-event-ingestion-api/src/models/temperature.py`: define `TemperatureUnit` enum (CELSIUS, FAHRENHEIT), `IngestionSource` enum (DEVICE_PUSH, API), `TemperatureRecord` Pydantic v2 BaseModel (all fields typed, `measurement_method: Optional[str] = None`), `TemperatureRecordBatch` model (records: `list[TemperatureRecord]`, max_length=100); add Javadoc-equivalent docstrings per Constitution gate 1

### sapphire-event-ingestion-api — Services

- [ ] T009 [P] [US1] Create `sapphire-event-ingestion-api/src/services/temperature_validator.py`: implement `validate_temperature(record: TemperatureRecord) -> None` raising `ValidationError` for out-of-range values, unrecognised units, and invalid timestamps; read range bounds from `TEMP_MIN_CELSIUS` / `TEMP_MAX_CELSIUS` env vars (default 34.0/42.0); convert Fahrenheit submissions to Celsius for range check; use named constants (no magic numbers, Constitution gate 2); inject config via FastAPI `Depends` (analysis F-3)
- [ ] T010 [P] [US1] Create `sapphire-event-ingestion-api/src/services/deduplication_service.py`: implement `compute_dedup_hash(user_id, device_source, timestamp, value) -> str` using `hashlib.sha256`; implement `is_duplicate(hash: str, store) -> bool` checking a Redis or in-memory set; inject the backing store via FastAPI `Depends` (analysis F-3)
- [ ] T011 [US1] Create `sapphire-event-ingestion-api/src/services/kafka_publisher.py`: implement `publish_temperature_event(record: TemperatureRecord, producer=Depends(get_kafka_producer)) -> None`; serialise record to Avro using the schema from T001; emit to topic `temperature-events`; use `Depends` for the Confluent Kafka producer instance (analysis F-3); add structured log entry (Constitution gate 13) with `trace_id` and `span_id` fields; emit OTEL span as child of incoming request span (Constitution gate 15)

### sapphire-event-ingestion-api — Routes

- [ ] T012 [US1] Create `sapphire-event-ingestion-api/src/api/temperature_routes.py`: implement `POST /api/temperatures` (single record) → validate → deduplicate → publish → return 201; implement `POST /api/temperatures/batch` (list of records, max 100) → validate each → deduplicate each → publish valid → return 207 with per-record status (partial-accept semantics); annotate all handlers `async def`; validate JWT via existing auth middleware; add OTEL span for each handler (Constitution gate 15); register custom metrics `temperature_records_ingested_total` (counter) and `temperature_validation_rejected_total` (counter) and `temperature_batch_size` (histogram) (analysis F-6)
- [ ] T013 [US1] Register `temperature_routes` router in `sapphire-event-ingestion-api/src/main.py` under prefix `/api`; confirm no other route conflicts; add startup log confirming route registration

### sapphire-event-ingestion-api — Documentation & Config

- [ ] T014 [P] [US1] Add `TEMP_MIN_CELSIUS`, `TEMP_MAX_CELSIUS` environment variable documentation to `sapphire-event-ingestion-api/README.md` under a "Configuration" section; add example `.env.example` entries

**Checkpoint**: User Story 1 fully functional. POST to `/api/temperatures` and `/api/temperatures/batch` validate, deduplicate, and publish temperature events to Kafka. All acceptance scenarios for US1 testable independently.

---

## Phase 4: User Story 2 — View Body Temperature Trend Charts (Priority: P2)

**Goal**: Store temperature events from Kafka into PostgreSQL; expose trend data (min/max/avg) via charting API; surface via GraphQL BFF; render a chart in the React UI with unit switching, loading, error, and empty states.

**Independent Test**: Pre-load temperature records for a test user → query `GET /api/charts/temperatures?userId=&granularity=day&dateFrom=&dateTo=` → verify min/max/avg match stored records. Query GraphQL `temperatureData` field → verify response shape. Open health dashboard → chart renders; switching C/F toggles displayed values; date-range filter updates chart; empty state shown when no data. (Refs: TEST123PUB-130, TEST123PUB-131, TEST123PUB-133)

### sapphire-charting-api — Domain Models

- [ ] T015 [P] [US2] Create `sapphire-charting-api/src/main/java/com/sapphire/charting/domain/TemperatureRecord.java` as a Java `record` type (Constitution gate F-4): fields `String userId`, `double value`, `String unit`, `Instant recordedAt`, `String deviceSource`; add Javadoc per Constitution gate 1
- [ ] T016 [P] [US2] Create `sapphire-charting-api/src/main/java/com/sapphire/charting/domain/TemperatureTrendSummary.java` as a Java `record` type (Constitution gate F-4): fields `String userId`, `String granularity`, `double minValue`, `double maxValue`, `double avgValue`, `String unit`, `Instant periodStart`, `Instant periodEnd`; add Javadoc

### sapphire-charting-api — Repository

- [ ] T017 [US2] Create `sapphire-charting-api/src/main/java/com/sapphire/charting/repository/TemperatureRepository.java`: JPA repository interface extending `JpaRepository`; add `findTrendSummary(String userId, String granularity, Instant dateFrom, Instant dateTo, Optional<String> deviceSource): List<TemperatureTrendSummary>` using a native TimescaleDB query against the continuous aggregate views from T003; all nullable parameters use `Optional<T>` (Constitution gate F-4); constructor injection only (no `@Autowired` fields)

### sapphire-charting-api — Service

- [ ] T018 [US2] Create `sapphire-charting-api/src/main/java/com/sapphire/charting/service/TemperatureTrendService.java`: implement `getTrend(userId, granularity, dateFrom, dateTo, deviceSource): Optional<List<TemperatureTrendSummary>>`; delegate to `TemperatureRepository`; apply cyclomatic complexity ≤ 10 per method (Constitution gate 3); return `Optional.empty()` rather than null for missing data (Constitution gate F-4); add structured log entries (Constitution gate 13)

### sapphire-charting-api — Controller

- [ ] T019 [US2] Create `sapphire-charting-api/src/main/java/com/sapphire/charting/controller/TemperatureChartController.java`: `@RestController @RequestMapping("/api/charts/temperatures")`; `GET` endpoint accepting `userId`, `granularity` (day/week/month), `dateFrom` (ISO-8601), `dateTo` (ISO-8601), `deviceSource` (optional); delegate to `TemperatureTrendService`; return 200 with trend list or 204 No Content when empty; add OTEL span (Constitution gate 15); emit custom metric `temperature_chart_requests_total` (Constitution gate F-6)

### sapphire-charting-api — Configuration & Docs

- [ ] T020 [P] [US2] Create `sapphire-charting-api/src/main/java/com/sapphire/charting/config/TemperatureChartConfiguration.java`: `@Configuration` bean defining physiological range constants loaded from `TEMP_MIN_CELSIUS` / `TEMP_MAX_CELSIUS` env vars; expose as `@Bean` for injection into service
- [ ] T021 [P] [US2] Add OpenAPI `@Operation` / `@ApiResponse` annotations to `TemperatureChartController`; update `sapphire-charting-api/README.md` with endpoint reference (Constitution gate 18)

### sapphire-bff-api — GraphQL Schema

- [ ] T022 [P] [US2] Create `sapphire-bff-api/src/schema/temperature.graphql`: define `type Temperature { id: ID! value: Float! unit: String! timestamp: String! deviceSource: String }` and `type TemperatureTrendData { minValue: Float! maxValue: Float! avgValue: Float! unit: String! granularity: String! periodStart: String! periodEnd: String! }`; add SDL comments per Constitution gate 1

- [ ] T023 [P] [US2] Create `sapphire-bff-api/src/schema/query-extensions.graphql`: extend `type Query` with `temperatureData(userId: ID!, granularity: String!, dateFrom: String!, dateTo: String!, deviceSource: String): [TemperatureTrendData!]`

### sapphire-bff-api — Service Client & Resolvers

- [ ] T024 [US2] Create `sapphire-bff-api/src/services/charting-service.js`: implement `getTemperatureTrend(userId, granularity, dateFrom, dateTo, deviceSource)` using `httpx`-style axios client to `CHARTING_API_BASE_URL` env var; add `trace_id` propagation via W3C `traceparent` header (Constitution gate 15); handle 204 as empty array; no inline URLs (Constitution gate)
- [ ] T025 [US2] Create `sapphire-bff-api/src/resolvers/temperature-resolvers.js`: implement resolver for `temperatureData` query; validate JWT claims before delegating (Constitution gate 10); use `chartingService.getTemperatureTrend()`; set Apollo cache policy `network-only` for mutable health data (Constitution gate 12); add DataLoader if user can request multiple date ranges in one query (Constitution gate F-4/P-10)

### sapphire-bff-api — Registration

- [ ] T026 [US2] Register `temperature.graphql` and `query-extensions.graphql` in BFF schema loader; register `temperature-resolvers.js` in resolver map; confirm schema compiles without conflicts

### Sapphire UI — Types & Utilities

- [ ] T027 [P] [US2] Create `Sapphire/src/features/temperature/types.ts`: TypeScript interfaces `TemperatureTrendData`, `TemperatureChartProps`, `TemperatureUnit` enum (`CELSIUS`, `FAHRENHEIT`); no `any` (Constitution gate 5)
- [ ] T028 [P] [US2] Create `Sapphire/src/utils/temperature-conversion.ts`: implement `celsiusToFahrenheit(c: number): number` and `fahrenheitToCelsius(f: number): number` using formula `°F = (°C × 9/5) + 32`; define conversion constants with named exports (Constitution gate 2); accuracy must satisfy SC-006 (≤ 0.1° error)
- [ ] T029 [P] [US2] Create `Sapphire/src/features/temperature/constants.ts`: named constants for `DEFAULT_UNIT`, `SUPPORTED_GRANULARITIES`, `CHART_COLORS_MIN`, `CHART_COLORS_MAX`, `CHART_COLORS_AVG`; no magic strings or numbers (Constitution gate 2)

### Sapphire UI — Apollo Hook

- [ ] T030 [US2] Create `Sapphire/src/features/temperature/useTemperatureData.ts`: Apollo `useQuery` hook for `temperatureData` query; accepts `userId`, `granularity`, `dateFrom`, `dateTo`, `deviceSource`; cache policy `network-only` (Constitution gate 12); returns `{ data, loading, error }`; read date-range and device-source filter from URL search params using `useSearchParams` (Constitution gate F-5 / Constitution III)

### Sapphire UI — Chart Component

- [ ] T031 [US2] Create `Sapphire/src/features/temperature/TemperatureChart.tsx`: functional component rendering min/max/avg line chart from `useTemperatureData`; display loading skeleton when `loading === true` (Constitution gate 9); display `<EmptyState>` from `src/components/EmptyState.tsx` when data array is empty (Constitution gate 9); display error boundary message when `error` is set (Constitution gate 9); unit toggle (C/F) reads/writes to URL state via `useSearchParams` (Constitution gate F-5); apply `temperature-conversion.ts` utilities for display-time unit conversion; no class components (React skill)
- [ ] T032 [US2] Create `Sapphire/src/features/temperature/TemperatureChart.module.css`: styles for chart container, unit toggle button, loading skeleton, and empty state; no inline styles in TSX

### Sapphire UI — Dashboard Integration

- [ ] T033 [US2] Add `<TemperatureChart>` to the health dashboard metrics grid in the appropriate dashboard layout file (follow existing metric component integration pattern); ensure it does not break dashboard load when temperature data is absent (SC-008)

**Checkpoint**: User Story 2 fully functional end-to-end. Chart renders in UI with real data from the pipeline; unit switching works; loading/empty/error states all handled.

---

## Phase 5: User Story 3 — Include Temperature in Health Analytics Export (Priority: P3)

**Goal**: Include temperature records in the existing health analytics export, honouring date-range and device-source filters.

**Independent Test**: Trigger health export for a date range containing temperature records → verify export output includes temperature section with value, unit, timestamp, device source. Records outside range excluded. No temperature records → section omitted, no error. (Refs: TEST123PUB-131)

### sapphire-bff-api — Export Extension

- [ ] T034 [P] [US3] Add `temperatureExport(userId: ID!, dateFrom: String!, dateTo: String!, deviceSource: String): TemperatureExportData` to `sapphire-bff-api/src/schema/query-extensions.graphql`; define `type TemperatureExportData { records: [Temperature!]! }` in `temperature.graphql`
- [ ] T035 [US3] Implement `getTemperatureExport(userId, dateFrom, dateTo, deviceSource)` in `sapphire-bff-api/src/services/analytics-service.js`: call existing export endpoint on `sapphire-charting-api` (or a dedicated export endpoint) with temperature filter; propagate W3C traceparent header (Constitution gate 15)
- [ ] T036 [US3] Add resolver for `temperatureExport` in `sapphire-bff-api/src/resolvers/temperature-resolvers.js`: validate JWT, delegate to `analytics-service.js`, return empty records array (not null) when no data (SC-007); cache policy `no-cache` for export queries (Constitution gate 12)

### sapphire-charting-api — Export Endpoint

- [ ] T037 [US3] Add `GET /api/charts/temperatures/export` endpoint to `TemperatureChartController.java`: accepts same filter params as trend endpoint; returns flat list of `TemperatureRecord` (all fields); return 204 when no records match filter; reuse `TemperatureTrendService` or add `getRecords()` method depending on ORM query complexity; add Javadoc and OpenAPI annotations (Constitution gate 18)

### Sapphire UI — Export Trigger

- [ ] T038 [US3] Update the existing health export trigger in `Sapphire` (follow existing metric export component pattern) to include temperature data in the export request parameters; confirm the export UI reads device-source filter and date-range from URL state (Constitution gate F-5)

**Checkpoint**: All three user stories independently functional. Export includes temperature data.

---

## Phase 6: Polish & Cross-Cutting Concerns

**Purpose**: OTEL instrumentation completeness, backward-compatibility consumer tests, documentation, and schema registration.

- [ ] T039 [P] Add OTEL instrumentation to `sapphire-event-ingestion-api`: verify `OTEL_EXPORTER_OTLP_ENDPOINT`, `OTEL_SERVICE_NAME=sapphire-event-ingestion-api`, `OTEL_DEPLOYMENT_ENVIRONMENT` are set in container config; confirm Kafka publish operations emit child spans (Constitution gate 15/16)
- [ ] T040 [P] Add OTEL instrumentation to `sapphire-charting-api`: add `@WithSpan` or manual span on `TemperatureTrendService.getTrend()`; instrument `TemperatureRepository` DB calls as child spans; confirm structured Logback JSON log output with `trace_id` + `span_id` (Constitution gate 13/15)
- [ ] T041 [P] Add OTEL instrumentation to `sapphire-bff-api`: confirm `temperature-resolvers.js` propagates W3C `traceparent` to downstream HTTP calls; add duration histogram metric for resolver execution (Constitution gate 14/15)
- [ ] T042 [P] Add OTEL instrumentation to `Sapphire` UI: confirm Apollo client forwards `traceparent` header on temperature queries per existing pattern (Constitution gate 15)
- [ ] T043 [P] Write consumer contract tests in `sapphire-charting-api/src/test/java/com/sapphire/charting/integration/TemperatureChartIntegrationTest.java` and `sapphire-event-ingestion-api/tests/integration/test_temperature_ingestion_flow.py` verifying that the existing `/api/health-metrics` and other pre-existing endpoints are unaffected by the temperature additions (Constitution gate 19)
- [ ] T044 [P] Commit `sapphire-kafka-pipeline/schemas/temperature-event-schema.avro` to the schema registry and verify BACKWARD compatibility against an empty baseline (no breaking change to existing schemas)
- [ ] T045 [P] Create `specs/TEST123PUB-127/backwards-compat-assessment.md`: classify all 6 API surfaces (PostgreSQL, Kafka topic, Avro schema, GraphQL, REST ingestion, REST charting) as non-breaking/additive; note consumer contract test coverage
- [ ] T046 [P] Update `sapphire-event-ingestion-api/README.md`, `sapphire-charting-api/README.md`, and `sapphire-bff-api/README.md` with purpose, usage, configuration, and known limitations for the temperature feature (Constitution gate 18)
- [ ] T047 [P] Add schema documentation for the temperature metric type to the platform schema docs referenced by SC-009 (update wherever existing metric type documentation is maintained)

---

## Dependencies & Execution Order

### Phase Dependencies

- **Phase 1 (Setup)**: No dependencies — start immediately. T001–T005 all parallelisable.
- **Phase 2 (Foundational)**: Depends on T001 (Avro schema) and T002 (DDL). Kafka Connect config (T006) needs both.
- **Phase 3 (US1 — Ingestion)**: Can begin as soon as T001 is done (T008–T010 parallelisable with Phase 2). T011 (Kafka publisher) depends on T001.
- **Phase 4 (US2 — Charts)**: `sapphire-charting-api` tasks (T015–T021) depend on T002/T003 (DDL + views). BFF tasks (T022–T026) depend on T015–T021. UI tasks (T027–T033) depend on T022–T026.
- **Phase 5 (US3 — Export)**: Depends on Phase 3 + Phase 4 being complete.
- **Phase 6 (Polish)**: Depends on Phases 3–5 complete.

### User Story Dependencies

| Story | Depends on | Can parallelise with |
|---|---|---|
| US1 (Ingestion) | Phase 1 (T001) | Phase 2 T006 |
| US2 (Charts) | Phase 1 complete + US1 data in pipeline | US1 service layer once schema is fixed |
| US3 (Export) | US1 + US2 complete | Phase 6 polish tasks |

### Within Each Repo (US2 execution order)
```
sapphire-charting-api:  T015/T016 [P] → T017 → T018 → T019 → T020/T021 [P]
sapphire-bff-api:       T022/T023 [P] → T024 → T025 → T026
Sapphire UI:            T027/T028/T029 [P] → T030 → T031 → T032 → T033
```

---

## Parallel Execution Examples

### Phase 1 — all tasks parallelisable
```
T001 Avro schema  ──┐
T002 DDL hypertable  ├── all independent, start together
T003 Agg views    ──┤
T004 Refresh policy─┤
T005 Docker Compose─┘
```

### Phase 3 (US1) — after T001
```
T008 Pydantic models ──┐
T009 Validator service ─┼── parallel after T001
T010 Dedup service  ───┘
T011 Kafka publisher (depends T008–T010)
T012 Routes (depends T011)
T013 main.py registration
T014 Docs/config ──── parallel anytime
```

### Phase 4 (US2) — cross-repo parallel streams
```
Stream A (charting-api):  T015/T016 → T017 → T018 → T019 → T020/T021
Stream B (bff-api):       T022/T023 → T024 → T025 → T026   [starts when T019 done]
Stream C (UI):            T027/T028/T029 → T030 → T031 → T032 → T033  [starts when T026 done]
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)
1. Complete Phase 1 (T001–T005)
2. Complete Phase 2 (T006–T007)
3. Complete Phase 3 US1 (T008–T014)
4. **STOP and validate**: POST temperature records, verify Kafka event published, record retrievable
5. Deploy/demo — temperature ingestion is live

### Incremental Delivery
1. Phase 1 + Phase 2 → pipeline wired
2. Phase 3 → ingestion works, temperature data flows into Kafka
3. Phase 4 → charts appear in UI ← core product value delivered
4. Phase 5 → export includes temperature
5. Phase 6 → observability, docs, consumer contract tests

---

## Format Summary

- **[P]**: Task can run in parallel with other [P] tasks in the same phase
- **[US1]/[US2]/[US3]**: Maps task to user story for traceability
- All file paths are absolute within their repo
- Analysis findings F-1 through F-7 addressed inline in relevant tasks
