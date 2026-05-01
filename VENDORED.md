# Vendored from Project Solace

The LLM routing layer in ``vendor/llm_router/`` (created in the next
milestone) is a snapshot of ``Projects/project-solace/llm-router/`` from
the ``stschulznz/infra`` repository.

This is a **vendored snapshot, not a dependency**. Sift v1 ships with
the snapshot frozen. If Project Solace adds features Sift needs, we
re-snapshot deliberately, never automatically.

When the friction list at the bottom of this file shows three or more
distinct Solace-specific assumptions we had to work around, file the
extraction issue on the Solace tracker and convert this directory into
a real package dependency. Until then, vendoring is the right answer.

## Source commit

- Repo: ``stschulznz/infra``
- Path: ``Projects/project-solace/llm-router/``
- Reference SHA: ``0c562c760fc34c588fd716760ea2f2a6b91accd2`` (head of
  ``main`` at the time of this commit; the actual snapshot SHA will be
  recorded once vendoring lands)

## What is in scope to vendor

For Sift v1:

- ``adapters/base.py`` (``BaseAdapter`` ABC, health-state contract)
- ``adapters/github_copilot.py``
- ``adapters/claude.py``
- ``adapters/azure_openai.py``
- ``adapters/microsoft_foundry.py``
- ``models.py`` (``GenerateRequest``, ``GenerateResponse``, ``ChatMessage``,
  etc.)
- ``router/dispatch.py`` (provider dispatch)
- Cost tracker (with the post-#179 NULL-not-zero hardening)
- Liveness / readiness split (``/livez`` vs ``/health``) and the
  ``warm()`` lifecycle hook

## What is NOT vendored

- The Project Solace orchestrator
- The MCP memory server
- The Planner / Worker / Judge harness
- Adapters not used in v1 (``ollama``, ``openrouter``, ``chatgpt``)
- Anything that depends on the Solace Postgres schema

## Solace-specific things removed during vendoring

(none yet, fill in as they happen during the vendoring milestone)

## Friction log

Append every time you have to work around a Solace-specific assumption
while building Sift. One line each: date plus a short description.

When this list reaches three entries, stop and reconsider extraction
to a shared package.

- (no entries yet)
