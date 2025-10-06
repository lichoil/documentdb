# RFC-002: Unified Testing Framework for DocumentDB

## Summary

Introduce a unified, cross-layer testing framework for the DocumentDB project that standardizes testing across:
- Layer A: PostgreSQL extension regression tests (psql)
- Layer B: Rust-based Gateway Mongo-compatible tests(integratio test)
- Layer C: End-to-end scenarios gateway → Postgres+DocumentDB(os/pg version matrix coverage)
- Layer D: Compatibility test(jstest/python script)
- Layer E: Basic performance baselines

This framework streamlines contributor workflows, improves reliability, and integrates cleanly with CI for PR validation and scheduled performance checks.

## Motivation

Contributors may face friction when validating changes across layers, and CI lacks a single orchestration. A unified approach will:
- Reduce onboarding time and confusion
- Catch cross-layer regressions earlier
- Provide reproducible local and CI runs
- Establish the foundation for future perf and distributed tests

## Goals

- One command to run all tests locally (extensions, gateway, e2e, compatibility, optional perf)
- Clear conventions for adding tests at any layer
- CI pipelines for PR validation and scheduled perf baselines
- Minimal flakiness with deterministic fixtures and assertions

## Non-Goals

- Full benchmark suite with statistical analysis (future, separate tast)

## Design Overview

### Layers
- Layer A: PostgreSQL regression tests via `make check` in `pg_documentdb_core` and `pg_documentdb`.
- Layer B: Gateway Mongo API tests via Rust integration tests (`pg_documentdb_gw/tests/`).
- Layer C: End-to-end integration tests: gateway forwarding to Postgres+DocumentDB with realistic datasets in os/pg version matrix.
- Layer D: Compatibility tests: mongoDB jstest/python script test.
- Layer E: Performance baselines: parallel throughput and query latency, recorded with environment metadata.

### Environments
- Local: Docker-based hosting Postgres + installed extension and gateway.
- CI: GitHub Actions matrix for key OS/Postgres versions; fast path (A+B) per PR, scheduled perf.

### Orchestration
- Provide cross-platform scripts:
  - `scripts/test_all.sh` or specific single layer test by assigning a layer parameter.
- Responsibilities:
  - Build components
  - Start Postgres with DocumentDB
  - Start gateway
  - Run tests by layer
  - Aggregate results (XML/JSON summary)

### Test Data and Fixtures
- Store under `tests/fixtures/`; utilities to load into Postgres and Mongo collections.
- for test init, data will be loaded and initialized(data insertion, index creation, etc).
- Keep small, targeted datasets; avoid nondeterminism.

## CI Integration

- PR pipeline:
  - Build extensions and gateway
  - Run Postgres regression tests (Layer A)
  - Run Gateway integration tests (Layer B)
  - Run E2E (Layer C) test covering different os/posrgres version, client drivers.
  - Run compatibility test(Layer D)
  - Run perf baselines (Layer E)

  - Alert for test regression/compatibility drop
  - Alert on >20% perf degradation compared to last baseline

## Performance Baselines

- Scenarios:
  - Query latency: Insert throughput for small/medium docs
  - Query latency: find, lookup, range, simple aggregation
- Resource
  - resource level(CPU core/memoery/disk IO) setup for docker?
- Output:
  - CSV/JSON in `perf/results/` with timestamp, versions, environment info
- Thresholds:
  - CI sanity check and alerting via GitHub Actions

## Drawbacks

- Increased maintenance for scripts and fixtures across platforms
- CI runtime cost

## Security and Secrets

- No secrets in repo; `.env.example` files for configs
- Ports and credentials configurable via env vars

## Timeline(TBD)

- Milestone 1 (2–4 weeks):
  - Implement runner scripts and wire Postgres tests
  - Add minimal Gateway integration test with mongosh
  - Provide docker-compose for e2e
- Milestone 2 (4–6 weeks):
  - Add compat test
  - Add baseline perf tests and CI schedule
  - summary reporting for CI
- Milestone 3(?)
  - Add scheduled full perf benchmark.
  - Auto generate perf change trend page(visualization).

## Success Metrics

- Single-command local runs succeed on new clones
- PR CI runs (A+B) consistently pass/fail with clear logs
- Perf baselines produced weekly with trend visibility

## Open Questions

- scope of test coveraged in pr trigged action, full test might take longer time.
- Desired thresholds for perf alerting(20%)?
- compatibility can be merged to E2E test(as one test aspect)?
- setup a separate docker for client/backend for sagregation of resource consumption?