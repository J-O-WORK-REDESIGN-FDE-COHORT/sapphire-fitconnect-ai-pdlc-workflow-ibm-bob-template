<!-- EFFECTIVE CONSTITUTION
     Generated  : 2025-08-13T00:00:00Z
     Global     : external (context-studio, context_id: ctx_325945189dd5)
     Local      : .specify/memory/constitution.md
     Precedence : local-over-global
     If confused, give precedence to the local constitution.
-->

# Effective Constitution

**Generated:** 2025-08-13T00:00:00Z
**Precedence:** Local constitution overrides global on conflict.
**Conflict report:** `.specify/runtime/effective-constitution-report.md`

> If a rule in Part 1 conflicts with a rule in Part 2, Part 2 wins — always.

---

## Resolved Rules — Authoritative Reference

> **Read this section first.** These are the final, binding rules for every
> topic where global and local conflict. Agents MUST apply these rules.
> Do not apply the corresponding Part 1 rule for any topic listed here.

No explicit rule-level conflicts were identified between the global and local constitutions in this run.

---

## PART 1 — Global Baseline

> Source: external (context-studio, context_id: ctx_325945189dd5)
> Rules superseded by Part 2 are documented in the conflict report.

# Constitution

## Core Principles

### I. Code Quality

All public functions, methods, and classes MUST have a docstring or Javadoc comment
explaining intent, not implementation. No magic numbers or strings — use named constants
or enums. Functions MUST do one thing; if a block inside a function needs a comment to
explain what it does, that block MUST be extracted into a named function or method.

Cyclomatic complexity MUST NOT exceed 10 per function/method, enforced via static analysis
in CI. Commented-out code MUST NOT appear in any commit; use feature flags or delete it.

**Java / Spring Boot**

- Follow standard Spring layering: Controller → Service → Repository. Business logic in
  controllers or repositories is FORBIDDEN.
- Use `record` types for immutable DTOs. Use `sealed interface` for discriminated domain
  types.
- Prefer constructor injection. Field injection via `@Autowired` is FORBIDDEN in
  non-test production code.
- Define domain-specific checked exceptions for recoverable errors. Unchecked exceptions
  are reserved for programming errors only.
- `Optional<T>` MUST be returned for nullable values. Returning `null` from any public
  method is FORBIDDEN.
- Java compilation MUST pass before merge (for example, `mvn -q -DskipTests compile` or
  `./gradlew compileJava` in the affected repo).

**Python / FastAPI**

- Follow PEP 8. `ruff` MUST be used for linting and formatting (line length 100).
- Pydantic v2 models MUST be used for all request/response schemas and internal data
  contracts.
- All FastAPI route handlers MUST be `async`. `httpx` MUST be used for async HTTP calls;
  the `requests` library is FORBIDDEN in service code.
- Every function signature MUST be fully type-annotated including return types. Bare
  `except:` clauses are FORBIDDEN; catch specific exception types only.
- Shared resources (DB sessions, HTTP clients, configuration) MUST be injected via
  FastAPI `Depends`.
- Python compilation checks MUST pass before merge (for example,
  `python -m compileall -q .` in the affected repo).

**REST API Contracts (all services)**

- All REST API contracts MUST be resource-based. Endpoint paths MUST use plural
  resource nouns (for example, `/users`, `/orders/{orderId}/items`) and MUST NOT
  use verb-style action paths (for example, `/createUser`, `/getOrders`,
  `/calculateScore`).
- CRUD semantics MUST map to standard HTTP methods (`GET`, `POST`, `PUT`, `PATCH`,
  `DELETE`) on resources. RPC-style action tunneling over REST paths is FORBIDDEN
  unless explicitly approved in the feature specification with documented rationale.
- Resource identifiers MUST be path parameters and filtering/pagination inputs MUST
  be query parameters. Request bodies for `GET` endpoints are FORBIDDEN.
- REST contract artifacts (OpenAPI specs, endpoint docs, and tests) MUST reflect the
  same resource model and naming conventions as implemented routes.

**LangGraph / AI Agents (sapphire-wellness-agent, sapphire-wellness-coach)**

- Graph state MUST be defined as a `TypedDict` with fully `Annotated` fields specifying
  the appropriate reducer (e.g., `operator.add` for append-only lists). Raw `dict` state
  is FORBIDDEN.
- Each graph node MUST be a single-responsibility `async` function accepting and
  returning state. Node functions MUST NOT perform DB or HTTP I/O without injected async
  clients (passed via `RunnableConfig` extras or constructor-injected dependencies).
- Conditional routing MUST use named routing functions returning string literals that
  match registered edge targets. Anonymous `lambda` routing callables are FORBIDDEN.
- All production LangGraph agents MUST use a persistent checkpointer
  (`AsyncPostgresSaver` or equivalent). `MemorySaver` is permitted in unit tests and
  local development only.
- Tool definitions MUST use Pydantic v2 `BaseModel` subclasses for their input schema.
  Bare `dict` or untyped `args_schema` is FORBIDDEN.
- Graphs MUST be compiled once at application startup and reused across requests.
  Compiling a new `StateGraph` per request is FORBIDDEN.
- Human-in-the-loop pauses MUST be implemented using LangGraph's `interrupt_before` or
  `interrupt_after` mechanism. Polling loops or arbitrary `asyncio.sleep` for approval
  waiting are FORBIDDEN.
- Node failures MUST be caught and encoded into a typed `error` field in the state.
  Unhandled exceptions that propagate out of a node and crash the graph are FORBIDDEN.
- LangGraph agent execution MUST emit trace data via LangSmith or an OTEL-compatible
  callback. Silent graph execution without observable trace output is FORBIDDEN in
  non-local environments.
- Unit tests MUST test each node in isolation with minimal constructed state dicts.
  Integration tests MUST run the compiled graph end-to-end using `MemorySaver` with
  deterministic LLM stubs or recorded cassettes.

**TypeScript / React (Sapphire UI)**

- Strict TypeScript MUST be enabled project-wide. `any` is FORBIDDEN. `ts-ignore`
  requires an explanatory comment and a linked ticket reference.
- Class components are FORBIDDEN; use functional components only.
- Feature code MUST be co-located: one directory per feature containing the component,
  its hook, its types, and its tests.
- Apollo cache policies MUST be explicit on every query. Implicit `cache-first` for
  mutable health data is FORBIDDEN.
- Raw `fetch` calls inside components are FORBIDDEN. All API interaction MUST go through
  the Apollo client or a typed service module.
- TypeScript compilation MUST pass before merge (for example, `tsc --noEmit` or the repo's equivalent compile script).

**Node.js / BFF (sapphire-bff-api)**

- All GraphQL resolvers MUST validate JWT claims before delegating to backend REST calls.
- Inline SQL and raw REST URLs are FORBIDDEN. Use typed resolver helpers and
  environment-configured service clients.
- `DataLoader` MUST be used for any field resolver that could trigger N+1 calls.

**Version**: 2.2.0 | **Ratified**: 2026-03-21 | **Last Amended**: 2026-04-02

---

## PART 2 — Local Constitution (Authoritative)

> Source: `.specify/memory/constitution.md`
> Rules here take precedence over Part 1 wherever a conflict exists.

# Sapphire FitConnect Constitution

## Core Principles

### I. Code Quality

Every feature MUST meet baseline code quality standards before it is considered shippable.

- All public functions, methods, and classes MUST have docstrings or Javadoc describing intent (not implementation).
- Magic numbers and strings are FORBIDDEN — named constants or enums MUST be used instead.
- Cyclomatic complexity MUST NOT exceed 10 per function or method; violations MUST be confirmed via static analysis.
- Commented-out code MUST NOT be committed; feature flags or outright deletion MUST be used instead.
- Stack-specific rules MUST be applied: Spring layering / PEP 8 + ruff / strict TypeScript / BFF DataLoader patterns.
- Apollo cache policies MUST be explicit; no implicit cache-first for mutable health data; cache TTL MUST be ≥ 30 s per session.

### II. Testing Standards

Automated tests are a non-negotiable delivery gate, not an afterthought.

- Coverage gates MUST be met: Java 80% overall / 100% domain layer, Python 80%, TypeScript/React 70%, BFF resolvers 100%.
- Contract tests MUST be planned for all GraphQL schema changes and Kafka event schema changes.
- The test pyramid MUST be respected: unit tests (mocked I/O), integration tests (Docker Compose, pre-merge only), E2E tests.

### III. UX Consistency

Every user-facing feature MUST provide a coherent, accessible, and secure experience.

- All data-fetching components MUST handle loading skeleton, error boundary, and empty state.
- The auth path MUST be exclusively Keycloak OIDC/PKCE; bypass routes are FORBIDDEN in any environment.
- URL state MUST be the source of truth for filters, pagination, and selections.

### IV. Observability

Every service MUST be observable from day one — not retrofitted after deployment.

- All services MUST emit structured JSON logs (Logback+logstash / structlog / pino) containing `trace_id` and `span_id` fields.
- OTEL metrics MUST be exported to the Collector: request count, duration histogram, error rate, in-flight counter; custom business metrics MUST be added where applicable.
- Distributed traces MUST be emitted via the OTEL SDK using W3C traceparent propagation; DB, HTTP, and Kafka operations MUST be instrumented as child spans.
- `OTEL_EXPORTER_OTLP_ENDPOINT`, `OTEL_SERVICE_NAME`, and `OTEL_DEPLOYMENT_ENVIRONMENT` MUST be set in every container; direct backend export from services is FORBIDDEN.

### V. Documentation & Audit

Every new feature MUST be fully documented in Markdown format with a proper audit log.

- A `docs/` or `spec/` Markdown document MUST accompany every new feature, covering: purpose, usage, configuration, and known limitations.
- An audit log entry MUST be created for each feature, capturing: feature name, author, date, summary of changes, and any breaking changes introduced.
- Documentation MUST be committed in the same PR as the feature code — documentation-only follow-up PRs are NOT acceptable.
- Audit log entries MUST follow the format: `## [YYYY-MM-DD] Feature: <name> — Author: <github-handle>`.
- Documentation MUST be written in plain English, use correct Markdown headings, and be free of placeholder text before merge.

### VI. API Compatibility

Every API change MUST include a backward-compatibility assessment before the change is merged.

- A backward-compatibility assessment MUST be produced for every change to a REST endpoint, GraphQL schema, or Kafka event schema; it MUST classify the change as **non-breaking**, **additive**, or **breaking**.
- Breaking changes are FORBIDDEN without an explicit migration plan approved in the feature spec; the plan MUST cover: client migration path, deprecation timeline, and versioning strategy.
- Additive changes (new optional fields, new endpoints) MUST NOT remove, rename, or change the type of any existing field or parameter.
- The compatibility assessment MUST be committed to `specs/<branch>/` alongside the feature plan and referenced in the plan's Constitution Check gate.
- Consumer contract tests MUST be updated or added to verify that existing consumers are unaffected by the change before merge.

## Development Workflow

All feature work MUST follow the PDLC gate sequence enforced by this sidekick:

1. **Specify** — Feature spec written and spec PR approved by Product Owner before planning begins.
2. **Plan** — Implementation plan and contracts produced; plan PR approved by FDE before tasking begins.
3. **Tasks** — Task list generated and tasks PR approved by FDE before implementation begins.
4. **Implement** — Code written against approved tasks; PRs raised per repo with traceability to JIRA story.
5. **Document** — Markdown documentation and audit log entry MUST be included in the implementation PR (Principle V).

Approvals MUST be GitHub PR approvals. Chat or verbal confirmation does NOT satisfy a gate. The submitter MUST NOT self-approve their own gate PR.

## Governance

This constitution supersedes all other project-level practices. Any conflict between a team convention and a principle stated here MUST be resolved in favour of this constitution.

**Amendment procedure**:
1. Raise a PR against `.specify/memory/constitution.md` with the proposed change and a written rationale.
2. Amendment MUST be approved by at least one Architect and the Product Owner before merge.
3. After merge, run `/constitution.resolve` to regenerate `.specify/runtime/effective-constitution.md`.
4. Bump `CONSTITUTION_VERSION` following semantic versioning: MAJOR for removals/redefinitions, MINOR for additions, PATCH for clarifications.

All PRs MUST verify compliance with the Constitution Check gate in `plan-template.md` before implementation begins.

**Version**: 1.1.0 | **Ratified**: TODO(RATIFICATION_DATE) | **Last Amended**: 2025-08-13
