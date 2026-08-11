<!--
SYNC IMPACT REPORT
==================
Version change: 1.0.0 → 1.1.0 (MINOR — new principle added)

Added principles:
  - VI. API Compatibility (NEW — from /speckit.constitution invocation)

Modified principles: none

Added sections: none

Removed sections: none

Templates requiring updates:
  ✅ .specify/templates/plan-template.md — Gate row 19 added for Principle VI
  ✅ .specify/templates/spec-template.md — No change required
  ✅ .specify/templates/tasks-template.md — No change required

Follow-up TODOs:
  - TODO(RATIFICATION_DATE): Set the actual team ratification date once agreed
-->

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
