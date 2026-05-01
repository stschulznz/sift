# Sift

AI-assisted Microsoft 365 inbox triage. Bring order to chaos.

Sift is an open-source, self-hostable tool that uses an LLM to classify and
file Microsoft 365 / Outlook mail into actionable folders (Action, Read,
Reference, Newsletters, Notifications, To-Be-Deleted, etc.) on a schedule.
It is designed to handle backlog inboxes (10k to 1M+ messages) safely:
nothing is hard-deleted; everything moves through a quarantine folder
with configurable retention.

## Status

Pre-alpha. The v1 spec is being written now. Code follows shortly.

## Provider support (v1)

- GitHub Copilot (recommended default if you already have a Copilot
  subscription; usage is entitled, no per-call billing)
- Anthropic Claude
- Azure OpenAI
- Microsoft Foundry

Ollama and other providers are deferred to v2.

The LLM routing layer is a vendored snapshot of Project Solace's
production-tested ``llm-router``. See [VENDORED.md](VENDORED.md) for
details and the friction log.

## Tracking

See the v1 milestone tracking issue for the live plan and progress.
