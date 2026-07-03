<!--
SYNC IMPACT REPORT
==================
Version change: [TEMPLATE] → 1.0.0 (initial ratification — all placeholders replaced)

Modified principles:
  - [PRINCIPLE_1_NAME] → I. Prometheus Compliance (new)
  - [PRINCIPLE_2_NAME] → II. Cloudflare API Fidelity (new)
  - [PRINCIPLE_3_NAME] → III. Configuration-Driven Behavior (new)
  - [PRINCIPLE_4_NAME] → IV. Graceful Degradation (new)
  - [PRINCIPLE_5_NAME] → V. Simplicity & No Unnecessary Abstractions (new)

Added sections:
  - Technology Stack & Deployment
  - Development Workflow

Removed sections: none

Templates checked:
  ✅ .specify/templates/plan-template.md — "Constitution Check" gate present; no outdated refs
  ✅ .specify/templates/spec-template.md — no constitution-specific refs; compatible as-is
  ✅ .specify/templates/tasks-template.md — task categories compatible; no outdated refs
  ⚠  .specify/templates/commands/ — directory does not exist; no command files to update

Deferred TODOs: none
-->

# Cloudflare Exporter Constitution

## Core Principles

### I. Prometheus Compliance

All metrics exposed by this exporter MUST conform to the Prometheus data model and
exposition format. Specific rules:

- Metric names MUST use the `cloudflare_` prefix, followed by a descriptive snake_case
  identifier (e.g., `cloudflare_zone_requests_total`).
- Counter metrics MUST carry the `_total` suffix; gauges and histograms MUST NOT.
- Label sets MUST be consistent across all metrics in a family; adding a label to one
  series MUST be applied uniformly across the whole family.
- Help strings MUST be present, accurate, and human-readable.
- The `/metrics` endpoint MUST respond in the standard Prometheus text exposition format
  (content-type `text/plain; version=0.0.4`).

**Rationale**: Consumers scrape this exporter with standard Prometheus tooling.
Non-conformant metrics silently produce wrong data in dashboards and alerts.

### II. Cloudflare API Fidelity

Metrics MUST accurately reflect the data returned by the Cloudflare Analytics API.
Specific rules:

- All GraphQL queries MUST target the Cloudflare GraphQL Analytics API v4 and MUST be
  validated against the current schema before each release.
- Authentication MUST support both API token methods (user-level auto-discovery and
  account-scoped) as well as the legacy email + Global API Key method.
- Account-scoped tokens MUST require the `CF_ACCOUNTS` variable to be set; the exporter
  MUST log a clear error when this contract is violated.
- Metric values MUST never be altered beyond unit normalization (e.g., bytes → bytes;
  no silent rounding or aggregation across zones).

**Rationale**: Inaccurate metrics erode trust. Cloudflare's API evolves; staying
schema-aligned prevents silent data loss.

### III. Configuration-Driven Behavior

Every operational parameter MUST be configurable via environment variable or CLI flag.
Specific rules:

- No zone IDs, account IDs, timeouts, ports, or scrape intervals MAY be hardcoded.
- Each configuration key MUST have a documented default that is safe for production use.
- Environment variables and CLI flags MUST be kept in sync (one-to-one mapping via
  viper/cobra).
- Optional feature flags (e.g., `FREE_TIER`, `ENABLE_PPROF`, `ENABLE_EDGE_ERRORS_BY_PATH`)
  MUST default to `false` / disabled to prevent unintended data exposure.

**Rationale**: Operators deploy this exporter across many different Cloudflare account
configurations. Runtime flexibility eliminates the need to rebuild images for each setup.

### IV. Graceful Degradation

The exporter MUST remain operational when individual Cloudflare API calls fail or
partial plan limitations apply. Specific rules:

- A failure to collect one metric family MUST NOT prevent other metric families from
  being collected and exposed.
- All API errors MUST be logged at the configured log level and MUST NOT cause a
  process crash or HTTP 500 on the `/metrics` endpoint.
- When `FREE_TIER=true`, paid-plan metrics MUST be silently skipped, not errored.
- The exporter MUST honour `METRICS_DENYLIST` to allow operators to suppress specific
  metrics without modifying code.

**Rationale**: Production monitoring systems cannot tolerate an exporter that crashes on
transient API errors. Partial data is preferable to no data.

### V. Simplicity & No Unnecessary Abstractions

The codebase MUST remain minimal, readable, and free of speculative complexity.
Specific rules:

- New metric scrapers MUST follow the established pattern in `cloudflare.go`,
  `graphql.go`, and `prometheus.go` before introducing new packages or layers.
- Abstractions are only justified when they remove concrete duplication (≥ 3 identical
  callsites). A new interface or package requires written justification in the PR.
- YAGNI applies: no feature flags, adapters, or extension points for hypothetical
  future requirements.
- Code comments MUST explain *why*, not *what*. Self-documenting identifiers are
  preferred.

**Rationale**: This is a single-purpose exporter. Premature abstractions increase
maintenance burden and obscure the data-collection logic that operators need to audit.

## Technology Stack & Deployment

- **Language**: Go (see `go.mod` for the authoritative version constraint).
- **Prometheus client**: `prometheus/client_golang` — do not introduce a second metrics
  library.
- **Cloudflare SDK**: `cloudflare-go/v4` — API interactions MUST go through this SDK
  except where the SDK does not yet support a required GraphQL query.
- **CLI / config**: `cobra` + `viper` — all new flags MUST be registered through these
  libraries.
- **Logging**: `logrus` — all log output MUST use the project-level `log` instance; no
  `fmt.Println` for operational messages.
- **Deployment artefacts**: Docker image (multi-arch) and Helm chart under `charts/`.
  Both MUST be kept in sync with any new configuration keys added to the exporter.
- **Helm chart**: Chart version and app version MUST be updated together on each
  release. `values.yaml` MUST expose a documented entry for every new env-var added.

## Development Workflow

- **Building**: use `make` targets defined in `Makefile` for build, lint, and test.
- **Unit tests**: Go test files (`*_test.go`) MUST cover utility logic in `utils.go`.
  Run with `go test ./...`.
- **End-to-end tests**: `run_e2e.sh` provides integration coverage against a live or
  mocked Cloudflare API. E2E MUST pass before merging to `master`.
- **Linting**: `lintconf.yaml` governs linter rules. No new lint suppressions
  (`//nolint`) without a comment explaining the exception.
- **CI**: GitHub Actions workflows under `.github/workflows/` gate merges. Docker image
  is published on successful `master` builds and on version tags.
- **Releases**: Follow semantic versioning. A CHANGELOG entry is REQUIRED for every
  release. Breaking config-key changes increment MAJOR; new metrics increment MINOR;
  fixes increment PATCH.

## Governance

This constitution supersedes all other development conventions documented elsewhere in
the repository. Where a conflict exists, the constitution takes precedence.

**Amendment procedure**:
1. Propose the change as a PR with a clear rationale referencing the affected principle.
2. Increment `CONSTITUTION_VERSION` according to the versioning policy below.
3. Update `LAST_AMENDED_DATE` to the merge date.
4. Run the consistency propagation checklist (templates, README, Helm chart) before
   merging.

**Versioning policy**:
- MAJOR: Removal or incompatible redefinition of an existing principle.
- MINOR: Addition of a new principle or material expansion of an existing one.
- PATCH: Clarifications, wording improvements, typo fixes.

**Compliance review**: Every PR targeting `master` MUST include a "Constitution Check"
confirming no principles are violated. Use `.specify/templates/plan-template.md` as the
gate template.

**Version**: 1.0.0 | **Ratified**: 2026-07-03 | **Last Amended**: 2026-07-03
