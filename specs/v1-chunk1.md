## 1. Executive Summary

**What:** Sift v1 is an AI-assisted Microsoft 365 inbox triage tool, written in Python, that classifies and files Outlook mail into a configurable set of actionable folders (Action, Waiting-For, Read, Reference, Newsletters, Notifications, To-Be-Deleted, Review) on a schedule. It runs as a CLI plus YAML configuration on self-hosted infrastructure (bare metal, VM, or Docker container). Mail is moved, never hard-deleted; everything routed to "delete" lands in a quarantine folder with configurable retention. Classification uses an LLM via the vendored Project Solace router (`vendor/llm_router/`) across four providers: GitHub Copilot (default), Anthropic Claude, Azure OpenAI, and Microsoft Foundry. A pre-LLM rule layer, a sender-cache shortcut, and a reference-keeper safety net reduce LLM call volume and protect high-value mail (invoices, receipts, tax documents, contracts, statements).

**Why:** Power users on Microsoft 365 plans without Copilot for Outlook (Basic, Personal, Family, Business Basic) have no first-party tool for bulk triage of large inboxes. Backlogs of 10,000 to 1,000,000+ messages are unworkable in the native Outlook UI and unsafe for naive auto-rules. Sift exists to triage those backlogs in batched, resumable, observable runs that a self-hoster can supervise. Vendoring Solace's router (rather than depending on a not-yet-extracted package) keeps Sift v1 shippable now while accumulating real-world friction data that informs whether to extract the router into a shared package later.

**Who:**

- **End users:** Self-hosting power users on Microsoft 365 personal or business plans (no Copilot for Outlook), with backlog inboxes of 10k or more messages, comfortable running a Python CLI or Docker container and editing a YAML file.
- **Internal consumers (non-human):** None in v1. Sift v1 is a CLI tool with a config file. There is no HTTP API, no web UI, no webhook surface.
- **External systems Sift talks to:** Microsoft Graph (delegated OAuth via device-code flow), four LLM provider HTTP APIs (via the vendored router), an OS keychain (via `keyring`) or `.env` file for secrets, and a local SQLite database for run state and undo history.

**Success criteria (every item is falsifiable; every item is verified by an explicit test or runbook step before v0.1.0 ships):**

1. A user with a 100,000-message inbox SHALL be able to install Sift via Docker, complete OAuth device-code authentication against Microsoft Graph, and start a backfill run, in under 15 minutes of total wall-clock setup time.
2. The same 100,000-message backfill SHALL complete inside 12 hours of wall-clock time on a single host with one provider and the default `concurrency=4`, including paging, throttling, classification, and folder moves.
3. All four v1 providers (GitHub Copilot, Claude, Azure OpenAI, Microsoft Foundry) SHALL produce a non-error classification on the project's labelled 200-message test set, with macro F1 score of at least 0.80 on each provider when configured with its default model.
4. On the project's labelled test set of 50 invoices and receipts, fewer than 1% (zero or zero rounded) SHALL be classified as `Triage/To-Be-Deleted` by the default rule + reference-keeper + LLM pipeline.
5. Incremental triage of new mail (no backfill, only mail received since the last run) SHALL complete in under 30 seconds end-to-end on a typical inbox of fewer than 100 new messages, including Graph fetch, classification, and moves.
6. No Sift command SHALL hard-delete any message in v1. The only command capable of removing items from quarantine is `sift purge --quarantine`, and it SHALL require interactive confirmation (no `--yes` flag in v1).
7. Every move performed by Sift SHALL be recorded with enough metadata (run ID, message ID, source folder ID, target folder ID, classifier, confidence) to be exactly reversed by `sift undo --run-id ID`.
8. `sift undo --run-id ID` SHALL successfully reverse 100% of moves performed in that run, provided the messages still exist and the source folder still exists. Messages that have been manually moved or deleted by the user since the run SHALL be reported and skipped, not errored.
9. The vendored Solace router (`vendor/llm_router/`) SHALL run unmodified at its public API surface (the `BaseAdapter`, `GenerateRequest`, `GenerateResponse`, dispatch, and cost tracker contracts). All four v1 adapters SHALL pass the vendored adapter test suite.
10. A backfill run that is interrupted (kill, crash, host reboot) SHALL resume from its last checkpoint on the next `sift backfill --resume` invocation, with no message classified twice and no message skipped.
11. When Microsoft Graph returns HTTP 429, Sift SHALL honour the `Retry-After` header and SHALL NOT retry faster than the header allows. The throttle log SHALL show zero `RetryAfterIgnored` events across a backfill of 100,000 messages.
12. When the cost tracker has no pricing entry for an active `(provider, model)` pair, the per-run cost summary SHALL display `cost_usd=null` (not `0.00`), in line with the vendored router's NULL-not-zero contract (Solace FR-46).
13. No credential (Graph token, provider API key, GitHub Copilot `gho_*` token) SHALL appear in any log file, audit log entry, or stdout output. Static log scanning of a full backfill run's logs SHALL produce zero matches against credential regex patterns.
14. No full message body SHALL appear in the audit log. The audit log SHALL contain only metadata (message ID, sender, subject hash, classification, confidence, run ID).
15. `sift triage --dry-run` SHALL classify and log every targeted message without moving any of them. After a dry run, the inbox state SHALL be identical (byte-for-byte at the Graph metadata level) to the state before the run.

---

## 2. System Overview

### Architecture

```
+----------------------------------------------------------------------+
|                          Host (bare metal, VM, or Docker)             |
|                                                                       |
|  +----------------+                                                   |
|  |   Sift CLI     |                                                   |
|  |  (entrypoint)  |                                                   |
|  +-------+--------+                                                   |
|          |                                                            |
|          v                                                            |
|  +----------------+    +-----------------+    +-------------------+   |
|  |   Sift Core    |--->|  vendored       |--->| LLM Provider APIs |   |
|  |                |    |  llm_router     |    |                   |   |
|  | - Pipeline     |    |                 |    |  GitHub Copilot   |   |
|  | - Rule layer   |    |  - dispatch     |    |    (HTTPS)        |   |
|  | - Ref-keeper   |    |  - cost tracker |    |  Anthropic Claude |   |
|  | - Sender cache |    |  - 4 adapters   |    |    (HTTPS)        |   |
|  | - Backfill     |    |  - /livez,      |    |  Azure OpenAI     |   |
|  | - Undo/rollback|    |    /health,     |    |    (HTTPS)        |   |
|  | - Audit log    |    |    warm()       |    |  Microsoft Foundry|   |
|  +---+--------+---+    +-----------------+    |    (HTTPS)        |   |
|      |        |                               +-------------------+   |
|      |        |                                                       |
|      |        v                                                       |
|      |   +----------------+                                           |
|      |   | Microsoft Graph|------> graph.microsoft.com (HTTPS)        |
|      |   |   client       |                                           |
|      |   +----------------+                                           |
|      |                                                                |
|      v                                                                |
|  +----------------+         +----------------+                        |
|  |  SQLite state  |         | Secret store   |                        |
|  |  store         |         | (.env or       |                        |
|  |                |         |  OS keychain)  |                        |
|  | - run history  |         |                |                        |
|  | - undo journal |         | - Graph tokens |                        |
|  | - backfill     |         | - provider     |                        |
|  |   checkpoints  |         |   API keys     |                        |
|  | - sender cache |         | - Copilot      |                        |
|  +----------------+         |   gho_* token  |                        |
|                             +----------------+                        |
|                                                                       |
+----------------------------------------------------------------------+
```

The vendored router is invoked in-process as a Python library, not as a separate FastAPI service. Sift imports the dispatch entry points and adapter classes directly from `vendor/llm_router/`. The router's `/livez`, `/health`, and `warm()` machinery (Solace FR-41 to FR-43, FR-51, FR-52) is preserved at the Python level: Sift calls `warm()` on registered adapters at startup, and uses the wall-clock `health_check()` contract before the first classification call to fail fast on misconfigured providers. Sift does not expose an HTTP surface in v1.

### Technology Stack

| Component | Technology | Version |
|---|---|---|
| Language runtime | CPython | 3.13 or newer |
| Microsoft Graph HTTP client | `httpx` (async) | >= 0.27 |
| Microsoft Graph auth | `msal` (device-code flow) | >= 1.30 |
| Configuration | `pydantic-settings` plus YAML via `pyyaml` | `pydantic-settings` >= 2.5, `pyyaml` >= 6.0 |
| Data validation | Pydantic v2 | >= 2.5 |
| State store | SQLite via `sqlite-utils` | `sqlite-utils` >= 3.36 (stdlib `sqlite3` underneath) |
| Secret storage | `keyring` (OS keychain) and `python-dotenv` (.env fallback) | `keyring` >= 24.0, `python-dotenv` >= 1.0 |
| Scheduling | `apscheduler` (in-process) and external cron (both supported) | `apscheduler` >= 3.10 |
| CLI framework | `typer` plus `rich` for output | `typer` >= 0.12, `rich` >= 13.0 |
| Structured logging | `structlog` | >= 24.0 |
| LLM router | Vendored from `Projects/project-solace/llm-router/` (see `VENDORED.md` for source SHA) | snapshot, no semantic version |
| Tests | `pytest`, `pytest-asyncio`, recorded Graph fixtures via `respx` | `pytest` >= 8.0, `pytest-asyncio` >= 0.23, `respx` >= 0.21 |
| Packaging | `uv` for dependency resolution and virtualenv management | `uv` >= 0.4 |
| Container | Docker (optional but supported) | Docker 24+ or Podman 5+ |

### Components

| Component | Responsibility | Stateful? |
|---|---|---|
| Sift CLI | `typer`-based entrypoint. Parses subcommands (`auth login`, `triage`, `backfill`, `undo`, `status`, `purge`), loads configuration, invokes Sift Core. | No |
| Sift Config | Loads YAML configuration, layers environment overrides via `pydantic-settings`, validates with Pydantic v2. | No |
| Microsoft Graph Client | Async `httpx`-based client. Handles delegated OAuth via `msal` (device-code), token refresh, paging, 429 backoff with `Retry-After`, batched moves, and resumable batch state. | No (state lives in SQLite and the secret store) |
| Classification Pipeline | Orchestrates per-message processing: rule layer -> sender cache lookup -> reference-keeper -> LLM via vendored router -> classification result. | No (reads sender cache from SQLite) |
| Rule Layer | Evaluates regex rules, keyword rules, sender allow- and deny-lists from configuration. Short-circuits classification when a rule matches. | No |
| Reference-Keeper | Safety-net classifier. Routes messages with attachments and messages matching invoice/receipt/tax/contract/statement keywords to `Triage/Reference`, regardless of LLM output. | No |
| Sender Cache | After N high-confidence classifications from the same sender (configurable, default N=5, confidence threshold default 0.85), routes future messages from that sender without an LLM call. | Yes (SQLite table) |
| LLM Classifier | Builds the classification prompt, calls the vendored router's dispatch with the configured provider and model, parses the response into a `ClassificationResult`. | No |
| Vendored LLM Router | Snapshot of Solace's router. Provides dispatch, four adapters (GitHub Copilot, Claude, Azure OpenAI, Microsoft Foundry), cost tracking with NULL-not-zero, `warm()` lifecycle, wall-clock `health_check()`. | No |
| Cost Tracker | Vendored. Records per-call usage; emits one-shot `pricing_unknown` warning per `(provider, model)` per process; persists `cost_usd=NULL` for unknown pricing. | Yes (SQLite usage log table) |
| Backfill Engine | Iterates over the inbox in deterministic batches, writes a checkpoint per batch, and supports `--resume`. | Yes (SQLite checkpoint table) |
| Undo Engine | Reads the per-run journal of moves and reverses them. Reports messages that can no longer be reversed (deleted, manually moved). | Yes (SQLite undo journal) |
| Audit Logger | Structured log of every classification, move, skip, and error. No message bodies. No credentials. | Yes (file on disk; configurable rotation) |
| State Store | SQLite database holding sender cache, run history, undo journal, backfill checkpoints, and usage log. | Yes |
| Secret Store | `.env` file (mode 0600) or OS keychain via `keyring`. Holds Graph tokens, provider API keys, and the GitHub Copilot `gho_*` token. | Yes (file on disk or OS keychain) |

---

## 3. Requirements

### Functional Requirements

#### Microsoft Graph client

**FR-01: Delegated OAuth via device-code flow.**
When a user runs `sift auth login`, the system SHALL initiate a device-code OAuth flow via `msal.PublicClientApplication`, print the user code and verification URL to stdout, and poll Microsoft Graph until the user completes the flow or the code expires. On success, the access token, refresh token, and `expires_at` SHALL be persisted to the configured secret store (OS keychain by default, `.env` file as fallback). On failure or timeout, the system SHALL exit with a non-zero status code and a clear error message.

**FR-02: Token refresh.**
Before every Graph request, the system SHALL check whether the cached access token expires within the next 5 minutes. If yes, it SHALL silently refresh the token using the cached refresh token via `msal.PublicClientApplication.acquire_token_silent()`. If silent refresh fails, the system SHALL exit with status code 2 and a message instructing the user to re-run `sift auth login`. The system SHALL NOT prompt interactively during a non-interactive run (`triage`, `backfill`).

**FR-03: Paging through Graph collections.**
When listing messages, the system SHALL use Graph's `@odata.nextLink` paging. The default page size SHALL be 50 messages (`$top=50`); configurable up to 999 (Graph's documented limit). The system SHALL fetch one page at a time and SHALL NOT issue page N+1 until page N has been fully processed and checkpointed.

**FR-04: Throttle compliance with `Retry-After`.**
When Graph returns HTTP 429 or HTTP 503, the system SHALL read the `Retry-After` header. If the header is present and is an integer (seconds), the system SHALL sleep at least that many seconds before retrying. If the header is present as an HTTP-date, the system SHALL sleep until that date. If the header is absent, the system SHALL apply exponential backoff starting at 1 second, doubling per attempt, capped at 60 seconds, with up to 5 retries. After 5 retries the system SHALL fail the current operation and record it in the run journal as `throttled_giving_up`.

**FR-05: Configurable concurrency.**
The Graph client SHALL accept a `concurrency` setting (config key `graph.concurrency`, default 4, valid range 1 to 16). The client SHALL use an `asyncio.Semaphore` to limit in-flight Graph requests to that count. Concurrency higher than 16 is forbidden in v1 to avoid triggering Graph tenant-wide throttling.

**FR-06: Resumable batch state.**
Every Graph operation that mutates state (move, mark-read) SHALL be recorded in the run journal (SQLite) before the request is issued, with status `pending`. On successful response the row SHALL be updated to `done` with the response metadata. On failure the row SHALL be updated to `failed` with the error reason. If the process is killed mid-batch, the next run SHALL find any `pending` rows and either retry them or mark them `unknown` (depending on idempotency of the operation; moves are idempotent by message ID and target folder ID, so they SHALL be retried).

**FR-07: Batched moves.**
When moving messages, the system SHALL use Graph's `$batch` endpoint with up to 20 operations per batch (Graph's documented limit). Per-operation results SHALL be parsed individually so that one failed move does not roll back the other 19.

**FR-08: Folder discovery and creation.**
On every run, before any move, the system SHALL ensure the configured target folders exist (`Triage/Action`, `Triage/Waiting-For`, `Triage/Read`, `Triage/Reference`, `Triage/Newsletters`, `Triage/Notifications`, `Triage/To-Be-Deleted`, `Triage/Review`). Missing folders SHALL be created. Folder names are configurable; the parent path SHALL default to `Triage/` but is overridable.

#### Classification pipeline

**FR-09: Pipeline ordering.**
For every targeted message, the system SHALL execute classification in the following fixed order, short-circuiting as soon as a stage produces a definitive result:
1. Rule layer (FR-13 to FR-15)
2. Reference-keeper (FR-16, FR-17)
3. Sender cache lookup (FR-18, FR-19)
4. LLM classification via vendored router (FR-20 to FR-23)
5. Confidence gate (FR-24): low-confidence results route to `Triage/Review`

**FR-10: Per-message processing record.**
For every message processed, the system SHALL record: message ID, sender address, subject (truncated to 256 chars), source folder ID, target folder ID, classifier name (`rule`, `reference_keeper`, `sender_cache`, `llm`, `low_confidence_review`), confidence (0.0 to 1.0), LLM provider and model (if applicable), prompt and completion token counts (if applicable), `cost_usd` (or NULL per Solace FR-46), and run ID.

**FR-11: Dry-run mode.**
When `--dry-run` is set, the system SHALL execute the full pipeline (including LLM calls and cost tracking) but SHALL NOT issue any Graph mutation request. Every would-be move SHALL be logged with the same record format as a real run, with an additional `dry_run=true` field. The run SHALL appear in run history with status `dry_run` so that it can be reviewed but not undone.

**FR-12: Run ID.**
Every invocation of `sift triage` or `sift backfill` SHALL generate a UUIDv4 run ID at startup. Every record produced by the run (audit log, undo journal, usage log) SHALL include the run ID. The run ID SHALL be printed to stdout at start and end of the run.

#### Folder taxonomy and rule layer

**FR-13: Default folder taxonomy.**
The default target labels SHALL be exactly: `Action`, `Waiting-For`, `Read`, `Reference`, `Newsletters`, `Notifications`, `To-Be-Deleted`, `Review`. Each label maps to a Graph folder under the configured parent (default `Triage/`). The user MAY override label names and parent in configuration; if overridden, the configuration SHALL provide a complete mapping (no partial overrides).

**FR-14: Sender allow- and deny-lists.**
The system SHALL evaluate sender allow- and deny-lists from configuration before any other rule. Allow-list match SHALL force routing to a specified target folder (default `Action`). Deny-list match SHALL force routing to `To-Be-Deleted`. Matches SHALL be case-insensitive and SHALL support exact address (`alice@example.com`), whole domain (`@example.com`), and subdomain (`@*.example.com`). Allow-list SHALL win over deny-list if both match (this SHALL be documented as the intended behaviour).

**FR-15: Regex and keyword rules.**
After sender-list evaluation, the system SHALL evaluate user-defined regex rules and keyword rules from configuration in the order written. Each rule has a `match` (regex pattern or list of keywords), a `field` (`subject`, `from`, `to`, `body_preview`), and a `target` (one of the v1 labels). The first matching rule SHALL determine the target. Regex SHALL be compiled once per run (not per message). The body field used SHALL be Graph's `bodyPreview` (first 255 characters), never the full body, to avoid loading large message bodies into memory during the rule pass.

#### Reference-keeper safety net

**FR-16: Attachment-based routing.**
If a message has one or more file attachments (Graph `hasAttachments=true` and at least one attachment of type `fileAttachment`), the reference-keeper SHALL route it to `Triage/Reference` with classifier=`reference_keeper` and confidence=1.0, overriding the LLM and sender cache. Inline images (`isInline=true`) SHALL NOT trigger this rule on their own.

**FR-17: Keyword-based routing.**
If a message's subject or `bodyPreview` contains any of the following keywords (case-insensitive whole-word match, configurable list), the reference-keeper SHALL route to `Triage/Reference` with confidence=1.0: `invoice`, `receipt`, `tax`, `taxation`, `contract`, `statement`, `bill`, `policy`, `agreement`, `purchase order`, `remittance`. Users MAY extend or replace this list in configuration. Reference-keeper SHALL override the LLM but SHALL be overridable by an explicit allow-list entry that targets a non-Reference folder.

#### Sender cache

**FR-18: Sender cache write.**
After the LLM classifies a message with confidence at or above the cache threshold (config `cache.confidence_threshold`, default 0.85), the system SHALL increment a counter for `(sender_address, target_label)` in the sender cache table. The counter SHALL persist across runs.

**FR-19: Sender cache read (shortcut).**
At the start of LLM classification, if the sender cache shows that `(sender_address, target_label)` has been observed at least N times (config `cache.min_observations`, default 5) and the same target label is the modal target for that sender (more than 80% of the sender's classified messages), the system SHALL skip the LLM call and route directly to that target. Classifier name SHALL be recorded as `sender_cache`. Confidence SHALL be the historical confidence average for that sender, capped at 0.99.

**FR-20: Sender cache invalidation on user undo.**
When `sift undo --run-id ID` reverses a move that was originated by `sender_cache`, the system SHALL decrement the counter for `(sender_address, target_label)` by the number of reversed messages and SHALL log a `cache_invalidated` event. If the counter falls below `cache.min_observations`, future messages from that sender SHALL again go through the LLM.

#### LLM classification via vendored router

**FR-21: Provider configuration.**
The active provider SHALL be read from configuration (`llm.provider`, default `github_copilot`). The active model SHALL be read from configuration (`llm.model`); if unset, the system SHALL apply per-provider defaults: GitHub Copilot `gpt-4o`, Claude `claude-sonnet-4-5`, Azure OpenAI `gpt-4o`, Microsoft Foundry `gpt-4o`. The user MAY override the model per provider in configuration.

**FR-22: Adapter `warm()` at startup.**
At startup, the system SHALL call `warm()` (Solace FR-52) on the active provider's adapter as a fire-and-forget asyncio task. For the GitHub Copilot adapter, this triggers the device-flow token exchange and capability fetch so that the first classification call does not pay the cold-start penalty. The system SHALL NOT block startup on `warm()` completion; failures SHALL be logged at WARNING and SHALL NOT abort the run.

**FR-23: Adapter `health_check()` before first call.**
Before the first classification call of a run, the system SHALL invoke the active adapter's `health_check()` (Solace FR-51), which is bounded by a wall-clock 4 second deadline. If the adapter reports unhealthy, the system SHALL exit with status code 3 and a message naming the provider and the reason. The system SHALL NOT silently fall back to a different provider in v1.

**FR-24: Classification prompt and response schema.**
The classification prompt SHALL include: the message sender, subject, `bodyPreview`, attachment indicator, and the list of configured target labels with one-line descriptions. The LLM SHALL respond with a JSON object matching the `ClassificationResult` schema: `{"label": "<one of the configured labels>", "confidence": <float 0.0-1.0>, "reasoning": "<short justification, max 280 chars>"}`. If the LLM response cannot be parsed as JSON, or `label` is not in the configured list, the system SHALL retry once with an explicit "JSON only" reminder; on second failure the message SHALL be routed to `Triage/Review` with classifier=`llm_parse_failed`.

**FR-25: Confidence gate.**
If the LLM-reported confidence is below `llm.review_threshold` (default 0.6), the message SHALL be routed to `Triage/Review` with classifier=`low_confidence_review`. The original LLM-suggested label SHALL be recorded for telemetry but SHALL NOT determine the move target.

**FR-26: Cost tracking via vendored tracker.**
Every LLM call SHALL be recorded by the vendored cost tracker. The system SHALL persist `prompt_tokens`, `completion_tokens`, `cost_usd` (which MAY be NULL per Solace FR-46), `provider`, `model`, and `run_id` to the SQLite usage log. The system SHALL NOT substitute `0.0` for `NULL` in any aggregation, summary, or report.

#### Backfill

**FR-27: Backfill window.**
`sift backfill --since YYYY-MM-DD` SHALL classify every message in the inbox received on or after the given date. If `--since` is omitted, backfill SHALL classify every message in the inbox regardless of date. `--until YYYY-MM-DD` MAY be provided to upper-bound the window.

**FR-28: Backfill batching and checkpointing.**
Backfill SHALL iterate the target window in batches of `backfill.batch_size` messages (default 200), ordered by `receivedDateTime` ascending. After each batch is fully processed (classified + moved + journalled), the system SHALL write a checkpoint row containing the run ID, the high-water `receivedDateTime`, the high-water message ID, and the count of messages processed in the batch.

**FR-29: Backfill resume.**
`sift backfill --resume [--run-id ID]` SHALL look up the latest checkpoint for the named run (or the most recent backfill run if `--run-id` is omitted) and SHALL continue from the high-water mark. If no checkpoint exists for the named run, the command SHALL exit with status code 4 and an explanatory message.

**FR-30: Backfill stop conditions.**
Backfill SHALL stop when one of: (a) the inbox window is exhausted, (b) the user sends SIGINT or SIGTERM (clean shutdown after the current batch finishes and is checkpointed), (c) the configured `backfill.max_messages` ceiling is reached, or (d) an unrecoverable error (provider exhausted, Graph auth permanently failed) is encountered. In all cases the system SHALL write a final checkpoint and exit cleanly.

#### Undo

**FR-31: Per-run undo.**
`sift undo --run-id ID` SHALL read every `done` row in the run journal for that run ID and SHALL issue the reverse Graph move (target folder ID -> source folder ID) for each. Reversed moves SHALL themselves be journalled under a new run ID with classifier=`undo` so that an undo can itself be undone (one level of redo).

**FR-32: Undo of unreachable messages.**
If a message ID in the undo target no longer exists, or has been moved to a folder that is not the recorded target folder, the system SHALL skip that message and record a `skipped_unreachable` event. The undo command SHALL NOT fail because some messages cannot be reversed; it SHALL report a summary at the end (`Reversed: X, Skipped (deleted): Y, Skipped (moved): Z`).

**FR-33: Undo dry-run.**
`sift undo --run-id ID --dry-run` SHALL list the moves it would issue without performing them. The output SHALL be machine-parseable (one move per line, with run ID, message ID, source folder, target folder).

#### CLI

**FR-34: Subcommand surface.**
The CLI SHALL expose exactly the following subcommands in v1, no more, no fewer: `sift auth login`, `sift triage --dry-run`, `sift triage --run`, `sift backfill --since YYYY-MM-DD [--until YYYY-MM-DD] [--resume] [--run-id ID]`, `sift undo --run-id ID [--dry-run]`, `sift status`, `sift purge --quarantine`. Any other subcommand SHALL exit with status code 64 (`EX_USAGE`) and the help text.

**FR-35: `sift status` output.**
`sift status` SHALL print: configured provider and model, last run ID and timestamp, last run outcome, count of messages in `Triage/To-Be-Deleted` and their oldest age, count of messages in `Triage/Review`, sender cache size, total persisted `cost_usd` (or `NULL` if unknown for any entry), and the next scheduled run (if any). It SHALL NOT make any LLM call. It MAY make Graph calls (folder counts).

**FR-36: `sift purge --quarantine` confirmation.**
`sift purge --quarantine` SHALL list every message in `Triage/To-Be-Deleted` older than `quarantine.retention_days` (default 30) and SHALL prompt interactively (typed confirmation phrase) before issuing any Graph delete. Non-interactive runs (no TTY) SHALL exit with status code 5 and an explanatory message; there is no `--yes` flag in v1.

**FR-37: Configuration loading.**
On every CLI invocation, the system SHALL load configuration in the following order, with later sources overriding earlier: (a) packaged defaults, (b) `~/.config/sift/config.yaml`, (c) `./sift.yaml` in the current working directory, (d) environment variables prefixed `SIFT_`, (e) command-line flags. Unknown keys in YAML SHALL produce a WARNING but SHALL NOT fail validation.

#### Audit log and observability

**FR-38: Audit log content.**
The audit log SHALL be a structured JSON-lines file at `audit.path` (default `~/.local/share/sift/audit.log`). Each line SHALL contain: ISO-8601 timestamp, run ID, event type, message ID (if applicable), sender address, subject hash (SHA-256 of the subject, never the subject itself), classifier, confidence, source folder ID, target folder ID, provider and model (if applicable), and outcome (`moved`, `dry_run`, `skipped`, `error`). The audit log SHALL NEVER contain message bodies, attachment content, OAuth tokens, API keys, or full subject lines.

**FR-39: Audit log rotation.**
The audit log SHALL be size-rotated at `audit.max_bytes` (default 50 MiB) with `audit.backup_count` (default 5) historical files retained. Rotation SHALL use `logging.handlers.RotatingFileHandler` semantics.

**FR-40: Run summary on stdout.**
At the end of every run, the system SHALL print a summary table to stdout: messages processed, messages moved per target folder, sender-cache shortcuts taken, LLM calls made, total prompt and completion tokens, total `cost_usd` (or `NULL` if any contributing entry was NULL), wall-clock duration, throttle events encountered.

### Non-Functional Requirements

**NFR-01: Backfill throughput.**
With `concurrency=4` and one provider, backfill SHALL sustain at least 4 classified-and-moved messages per second over a 1,000 message run, measured wall-clock on a host with 4 vCPU and 8 GiB RAM, ignoring time spent inside `Retry-After` sleeps.

**NFR-02: Incremental triage latency.**
Incremental `sift triage --run` against an inbox with 50 new messages SHALL complete in under 30 seconds wall-clock at p95 over 20 consecutive runs, again ignoring time spent inside `Retry-After` sleeps.

**NFR-03: Per-batch backfill latency.**
A single backfill batch of `backfill.batch_size=200` messages SHALL be processed (classified plus moved plus checkpointed) in under 5 minutes wall-clock at p95, when the active provider is healthy and Graph is not throttling.

**NFR-04: Memory footprint.**
Resident set size during a backfill SHALL stay under 512 MiB on the same 4 vCPU / 8 GiB reference host, measured at p99 over a 100,000 message run. Streaming and paging SHALL be used; full-message-body fetches SHALL NOT happen during classification.

**NFR-05: Startup time.**
`sift triage --dry-run --help` (no Graph calls) SHALL return in under 1 second on the reference host. `sift triage --run` (with valid cached Graph token) SHALL reach the first Graph request in under 5 seconds, including config load, secret fetch, adapter `warm()` scheduling, and `health_check()`.

**NFR-06: SQLite footprint.**
The SQLite state store SHALL stay under 1 GiB after a 1,000,000 message backfill (assuming default audit and journal retention). Indexes SHALL be defined on `(run_id)`, `(message_id)`, and `(sender_address, target_label)`.

**NFR-07: Throttle compliance.**
Across a 100,000 message backfill, the system SHALL produce zero `RetryAfterIgnored` events in the throttle log. The 99th-percentile retry-after compliance error (sleep_actual minus sleep_required) SHALL be under 250 ms.

**NFR-08: Provider failover behaviour.**
When the active provider returns an unrecoverable error mid-run (auth invalid, quota exhausted, repeated 5xx after retries), the run SHALL stop within 60 seconds, write a final checkpoint, and exit with a non-zero status code naming the provider and reason. The run SHALL NOT silently continue and SHALL NOT silently switch to another provider in v1.

**NFR-09: Logging overhead.**
Audit logging SHALL add no more than 5% wall-clock overhead at NFR-01 throughput. Structured logging SHALL use `structlog`'s `BytesLoggerFactory` or equivalent batched writer to avoid per-line `fsync()`.

**NFR-10: Test coverage.**
`pytest` SHALL achieve at least 80% line coverage on Sift code (`sift/` package, excluding `vendor/`) and at least 90% on the rule layer, reference-keeper, sender cache, undo, and Graph client backoff logic.

**NFR-11: Determinism in tests.**
The full `pytest` suite SHALL run offline. No test SHALL hit Microsoft Graph or any LLM provider over the network. Recorded fixtures via `respx` SHALL stand in for HTTP. The suite SHALL complete in under 60 seconds on the reference host.

**NFR-12: Documentation completeness.**
The repository SHALL contain at minimum: `README.md` (install, quickstart), `docs/configuration.md` (every config key documented with default and valid range), `docs/operations.md` (backfill runbook, undo runbook, purge runbook, troubleshooting), and `VENDORED.md` (vendoring contract, kept current with the friction log).

### Security Requirements

**SR-01: Graph token storage.**
Microsoft Graph access and refresh tokens SHALL be stored in the OS keychain via `keyring` by default. If the OS keychain is unavailable (typical in headless Linux containers without a session bus), the tokens SHALL be stored in a `.env`-style file at `~/.local/share/sift/.tokens` with file mode 0600 (`-rw-------`). The system SHALL verify and enforce the 0600 mode on every read; a wider mode SHALL trigger a hard error and abort.

**SR-02: Provider API key storage.**
Provider API keys (Claude, Azure OpenAI, Microsoft Foundry) SHALL be sourced from environment variables (`ANTHROPIC_API_KEY`, `AZURE_OPENAI_API_KEY`, `AZURE_OPENAI_ENDPOINT`, `FOUNDRY_CLIENT_ID`, `FOUNDRY_CLIENT_SECRET`, `FOUNDRY_TENANT_ID`, `FOUNDRY_ENDPOINT`), or from an `.env` file at file mode 0600, or from the OS keychain. They SHALL NEVER be read from configuration YAML.

**SR-03: GitHub Copilot token storage.**
The GitHub Copilot `gho_*` token SHALL be stored only in the OS keychain or in a file at mode 0600. The system SHALL refuse to start with a Copilot token file at any wider mode. The token SHALL NEVER be embedded in configuration YAML, environment defaults, or any image layer.

**SR-04: No credentials in logs.**
No credential (Graph access token, Graph refresh token, provider API key, Copilot `gho_*` token, OAuth client secret) SHALL appear in any log line, audit log entry, structured log field, panic traceback, or stdout output. Log emitters SHALL apply a redaction filter that scans serialised log payloads for credential regex patterns and replaces matches with `[REDACTED]` before write.

**SR-05: No message bodies in audit log.**
The audit log SHALL NEVER contain a message body, attachment content, attachment file name, or full subject line. Subjects SHALL be hashed (SHA-256) before logging. Sender addresses MAY be logged in clear (they are necessary to debug routing decisions); recipient lists SHALL NOT be logged.

**SR-06: Least-privilege Graph scopes.**
The Graph OAuth flow SHALL request only the scopes required for v1: `Mail.ReadWrite`, `MailboxSettings.Read`, `offline_access`, and `User.Read`. The system SHALL NOT request `Mail.Send`, `Mail.Send.Shared`, `Mail.ReadWrite.Shared`, or any directory or admin scope.

**SR-07: Outbound TLS only.**
All HTTPS calls (Graph, all four providers, OAuth endpoints) SHALL use TLS 1.2 or higher with certificate validation enabled. The system SHALL NOT expose any inbound network listener in v1; there is no HTTP server, no webhook receiver, no health endpoint exposed externally.

**SR-08: SQLite file permissions.**
The SQLite state store file (`~/.local/share/sift/state.db` by default) SHALL be created with file mode 0600 and a parent directory at mode 0700. The system SHALL verify these modes on every open and SHALL refuse to open a state store with wider permissions.

**SR-09: Configuration validation at boundary.**
All configuration loaded from YAML, environment, or CLI SHALL be validated by Pydantic v2 models at load time. Unknown keys SHALL produce WARNING. Invalid types or out-of-range values SHALL produce a hard validation error and a non-zero exit before any Graph or LLM call is made.

**SR-10: No hard-delete in v1.**
Excluding `sift purge --quarantine` (interactive confirmation required, no `--yes` in v1, FR-36), no Sift command SHALL issue a Graph DELETE on a message. All "delete" routing SHALL move the message to the quarantine folder. Static analysis (`ripgrep` for Graph DELETE call sites) SHALL be part of CI to enforce this.

**SR-11: Vendored router pinning.**
The vendored `vendor/llm_router/` snapshot SHALL be pinned to the source SHA recorded in `VENDORED.md`. The CI suite SHALL fail if the contents of `vendor/llm_router/` change without a corresponding update to `VENDORED.md`.

**SR-12: Dependency provenance.**
All Python dependencies SHALL be declared with version pins (or compatible-release specifiers) in `pyproject.toml` and resolved by `uv` into a `uv.lock` checked into the repository. CI SHALL run `pip-audit` (or equivalent) on every PR and SHALL fail on any known high-severity advisory in a direct dependency.

### Open questions for chunk 1

None at this time. All scope inputs from the v1 tracking issue and the vendoring contract are reflected. Open items raised during chunks 2 to 4 (data model, component specifications, error matrix, test plan, etc.) will be tracked there.
