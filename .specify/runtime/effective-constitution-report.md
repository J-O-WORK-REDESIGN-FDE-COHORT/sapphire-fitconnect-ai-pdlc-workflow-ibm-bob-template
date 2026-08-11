# Constitution Resolution Report

**Timestamp:** 2025-08-13T00:00:00Z

## Sources

| Field | Value |
|---|---|
| Global source mode | `external` — context-studio (context_id: `ctx_325945189dd5`) |
| Global source name | "Constitution test" |
| Local source | `.specify/memory/constitution.md` |
| `global_found` | `true` — fetched successfully via `context-broker-vector-query` |
| `local_is_template` | `false` — local constitution is fully filled (v1.1.0) |
| Precedence rule applied | Local-over-global (Case A) |

## Status

**PASS** — Effective constitution generated with full global + local merge.

## Override Callouts

No explicit rule-level overrides identified in this run.

The global constitution (v2.2.0) covers **Section I (Code Quality)** with detailed per-stack sub-rules
for Java/Spring Boot, Python/FastAPI, REST APIs, LangGraph, TypeScript/React, and Node.js/BFF.

The local constitution (v1.1.0) covers the same Code Quality principle in summary form plus five
additional principles absent from the global:
- II. Testing Standards
- III. UX Consistency
- IV. Observability
- V. Documentation & Audit
- VI. API Compatibility *(new in v1.1.0)*

All local rules are **additive or complementary** to the global baseline — no contradictions found.

## Output Artifacts

| Artifact | Path | Written |
|---|---|---|
| Effective constitution | `.specify/runtime/effective-constitution.md` | ✅ |
| Global constitution snapshot | `.specify/runtime/global-constitution.md` | ✅ |
| Resolution report | `.specify/runtime/effective-constitution-report.md` | ✅ |

## Blocking Errors

None.
