# DRAFT: Future extraction issue body

DO NOT FILE this on the Project Solace tracker until BOTH of these
are true:

1. Sift v1 has shipped (the v1 milestone tracking issue is closed).
2. The friction log in ``../VENDORED.md`` has three or more entries.

Rule of three: extract on the third consumer (or when accumulated
friction justifies it). Two consumers is not enough signal.

When ready, paste this issue body into a new issue on
``stschulznz/infra``, refine, and file.

---

# Extract ``llm-router/`` from Project Solace into a reusable package

## Summary

Extract the Project Solace ``llm-router/`` service into a standalone,
reusable Python package so multiple projects (Solace, Sift, future
consumers) can depend on the same routing layer without vendoring
or drift.

## Why now

- A second project (Sift) is in production and has accumulated a
  friction log against the vendored snapshot. See ``VENDORED.md`` in
  the Sift repo for the list.
- A third consumer is being scoped (or already exists), making the
  cost of extraction cheaper than the cost of further duplication.
- Each friction-log entry represents an assumption that needs a clean
  extension point in the extracted package. They form the spec for
  what needs to be pluggable.

## Proposed scope

Extract these from ``Projects/project-solace/llm-router/`` into a new
package (working title ``llm-router-py``):

- All seven provider adapters (or whichever subset has multi-consumer
  demand at the time of extraction).
- Router dispatch, Pydantic models, cost tracker, smart-routing logic,
  ``/v1/generate``, ``/v1/models``, ``/v1/models/capabilities``,
  ``/livez``, ``/health``, ``warm()`` lifecycle hook.
- Pluggable interfaces for: cost persistence (Solace = Postgres,
  Sift = SQLite, others = no-op), Copilot auth cache (different paths
  per consumer), and pricing-table overrides (consumer-specific SKUs).

## Migration plan

1. Cut ``v0.1.0`` of the package.
2. Publish to PyPI (or private index).
3. Solace replaces ``llm-router/`` with a thin shim that pip-installs
   the package and wires Solace-specific overrides.
4. Sift replaces ``vendor/llm_router/`` with the same dependency.
5. Confirm both consumers still pass their integration tests.

## Friction log to incorporate

(Fill in from ``VENDORED.md`` in the Sift repo at extraction time.)

## Open questions

- License: MIT or Apache 2.0?
- Repo location: dedicated ``llm-router-py`` repo or monorepo?
- Smart-routing scope: ship in v0.1 or split into an optional extra?
