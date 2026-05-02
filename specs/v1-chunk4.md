## 12. Implementation Plan

### 12.1 File structure

The Sift v1 repository at v0.1.0 ship time SHALL match the following layout. One-line annotations document the intent of every top-level node and every Python module; tests mirror `src/sift/` one-for-one.

```
sift/
├── pyproject.toml                       # uv-managed; PyPI name "sift-mail" (chunk 3b 10.2); Python >= 3.13.
├── uv.lock                              # Locked dependency tree; checked in for reproducibility.
├── README.md                            # Install, quickstart, link to docs/. Rewritten for v0.1.0.
├── LICENSE                              # MIT (matches Solace).
├── VENDORED.md                          # Solace router snapshot SHA, copy date, divergence policy (chunk 1 SR-11).
├── CHANGELOG.md                         # Keep-a-Changelog format. v0.1.0 entry on tag.
├── .gitignore                           # Standard Python + .venv + .sift/ + secrets.local + *.db.
├── .python-version                      # "3.13" (uv pin).
├── .editorconfig                        # 4-space indent, LF, UTF-8.
├── pricing-overrides.yaml               # Sample pricing-override file (chunk 2 5.2). Loaded only if path set.
│
├── vendor/
│   └── llm_router/                      # Verbatim Solace snapshot per VENDORED.md (chunk 1 2 / 3b 9.2).
│       ├── __init__.py                  # Re-exports dispatch and models (`from sift.vendor.llm_router import dispatch, models`).
│       ├── adapters/
│       │   ├── base.py                  # BaseAdapter ABC, health_check(), warm() (Solace 6.5).
│       │   ├── github_copilot.py        # Default provider (chunk 1 SR-03 token cache).
│       │   ├── claude.py
│       │   ├── azure_openai.py
│       │   └── microsoft_foundry.py
│       ├── models.py                    # ChatMessage, GenerateRequest, GenerateResponse, GenerateChunk, TokenUsage, LLMProviderName, UsageRecord.
│       ├── router/
│       │   └── dispatch.py              # Provider dispatch entry; consumed via `dispatch.generate(...)`.
│       ├── cost/
│       │   ├── tracker.py               # NULL-not-zero (Solace FR-46) cost tracker; accepts UsageSink.
│       │   └── pricing.py               # Built-in pricing table; merged with overrides at startup.
│       └── tests/                       # Vendored adapter test suite; runs unmodified (chunk 1 SC-9).
│
├── src/
│   └── sift/
│       ├── __init__.py                  # Exports __version__; reads from importlib.metadata.
│       ├── __main__.py                  # `python -m sift` entry; delegates to sift.cli.main.
│       │
│       ├── models/
│       │   ├── __init__.py
│       │   ├── base.py                  # STRICT / STRICT_OPEN ConfigDict (chunk 2 4.1).
│       │   ├── mail.py                  # EmailAddress, MailMessage, MailFolder (chunk 2 4.3).
│       │   ├── classification.py        # Classification, ClassificationSource enum, Confidence (chunk 2 4.4).
│       │   ├── state.py                 # Run, Action, ActionLogEntry, Checkpoint (chunk 2 4.6).
│       │   ├── cost.py                  # UsageRecord pass-through, RunCostSummary (chunk 2 4.7).
│       │   ├── errors.py                # SiftError hierarchy (chunk 2 4.8).
│       │   └── router.py                # Sift-side wrappers around vendored router models (chunk 2 4.5).
│       │
│       ├── config/
│       │   ├── __init__.py              # SiftConfig top-level model (chunk 2 4.2.1).
│       │   ├── loader.py                # Layered loader: defaults < ~/.config/sift/config.yaml < ./sift.yaml < env < CLI (FR-37).
│       │   ├── graph.py                 # GraphAuthConfig (chunk 2 4.2.2).
│       │   ├── llm.py                   # LLMConfig and per-provider sub-configs (chunk 2 4.2.3).
│       │   ├── folders.py               # FolderMap (chunk 2 4.2.4).
│       │   ├── retention.py             # RetentionConfig (chunk 2 4.2.5).
│       │   ├── rules.py                 # RuleConfig, regex / keyword / attachment rules (chunk 2 4.2.6).
│       │   ├── sender_cache.py          # SenderCacheConfig (chunk 2 4.2.7).
│       │   └── pricing.py               # pricing_overrides_path resolution (chunk 2 4.2.1, 6.9).
│       │
│       ├── secrets/
│       │   ├── __init__.py              # get_secret() facade (chunk 1 SR-02).
│       │   ├── keyring_backend.py       # OS keychain (preferred).
│       │   ├── env_backend.py           # SIFT_*_API_KEY fallback, locked-down .env file.
│       │   └── chain.py                 # keyring -> env chain with structured-event logging.
│       │
│       ├── security/
│       │   ├── __init__.py
│       │   ├── permissions.py           # assert_locked_down(path); enforces 0700 dir / 0600 file (SR-08).
│       │   ├── redaction.py             # Credential regex scanner; used by audit-log writer and CI (SR-04).
│       │   └── sender_hash.py           # HMAC-SHA-256 of sender for audit log (chunk 1 SC-14).
│       │
│       ├── log/
│       │   ├── __init__.py              # Logger factory.
│       │   ├── audit.py                 # Append-only NDJSON audit log; rotation (FR-39).
│       │   ├── events.py                # Structured event vocabulary (chunk 2 5.1.16).
│       │   └── redact_filter.py         # Last-line-of-defence redaction filter (SR-04).
│       │
│       ├── db/
│       │   ├── __init__.py              # `connect(path)` returns sqlite3.Connection with WAL mode.
│       │   ├── schema.py                # Embedded DDL for all tables.
│       │   ├── migrations/              # Numbered SQL files: 0001_init.sql, 0002_purge_log.sql, ...
│       │   └── store.py                 # StateStore class (chunk 2 5.3.3, 6 multiple sections).
│       │
│       ├── lock/
│       │   └── __init__.py              # File-lock helper (fcntl on POSIX, msvcrt on Windows). Used by run-lock + DB-writer-lock (chunk 2 6.10).
│       │
│       ├── graph/
│       │   ├── __init__.py
│       │   ├── auth.py                  # Device-code flow via msal (chunk 1 FR-01, SR-01).
│       │   ├── client.py                # GraphClient: paging, throttling, retry-after, per-run shared httpx.AsyncClient (chunk 2 6.2).
│       │   ├── folders.py               # Folder discovery + ensure-exists (chunk 1 FR-08).
│       │   ├── messages.py              # List, fetch, move; batch endpoint $batch wrapper.
│       │   └── throttle.py              # Token-bucket throttle, Retry-After honour, throttle-event logging.
│       │
│       ├── llm/
│       │   ├── __init__.py
│       │   ├── client.py                # LLMRouterClient interface (chunk 2 5.3.1).
│       │   ├── adapter.py               # RouterAdapter (concrete; wraps vendored dispatch).
│       │   ├── prompts.py               # build_classification_prompt(); deterministic templates.
│       │   ├── label_desc.py            # Per-label one-liners injected into the prompt.
│       │   ├── parser.py                # JSON repair: strict parse -> brace-balance recovery -> regex extract -> abort.
│       │   └── bootstrap.py             # Builds router; injects CostPersistenceAdapter, PricingOverrideLoader, CopilotAuthCachePath (chunk 3b 9.2).
│       │
│       ├── cost/
│       │   ├── __init__.py
│       │   ├── persistence.py           # CostPersistenceAdapter implementing UsageSink; sets agent_name="sift" (chunk 1 FR-26, chunk 3b 9.2).
│       │   ├── overrides.py             # PricingOverrideLoader: YAML -> vendored pricing table.
│       │   └── summary.py               # RunCostSummary builder; NULL-not-zero (chunk 1 SC-12).
│       │
│       ├── rules/
│       │   ├── __init__.py              # RuleEngine.evaluate(message) -> Optional[Classification].
│       │   ├── regex.py                 # Compiled-regex matcher.
│       │   ├── keyword.py               # Boundary-aware keyword matcher.
│       │   └── attachments.py           # MIME / extension matcher (chunk 1 FR-16).
│       │
│       ├── reference_keeper/
│       │   ├── __init__.py              # ReferenceKeeper.classify(message) -> Optional[Classification]; mandatory override (chunk 2 6.6).
│       │   ├── invoice.py               # Invoice / receipt heuristics + LLM secondary check.
│       │   ├── tax.py                   # Tax document heuristics.
│       │   ├── contract.py              # Contract heuristics.
│       │   └── statements.py            # Bank/credit statement heuristics.
│       │
│       ├── sender_cache/
│       │   ├── __init__.py              # SenderCache.get/put; thresholds N=6, confidence >= 0.9 (chunk 2 6.4).
│       │   ├── store.py                 # Backed by `sender_cache` SQLite table.
│       │   └── reset.py                 # Reset on undo / on user override; full-reset on `sift sender-cache reset`.
│       │
│       ├── pipeline/
│       │   ├── __init__.py              # Pipeline.classify(message) -> Classification.
│       │   ├── ordering.py              # FR-09 ordering: rule -> reference_keeper -> sender_cache -> llm -> low_confidence_review.
│       │   ├── llm_stage.py             # Builds prompt, calls LLM, parses, validates label.
│       │   └── low_confidence.py        # Confidence-floor check; routes to Triage/Review NEVER To-Be-Deleted (chunk 1 FR-25).
│       │
│       ├── backfill/
│       │   ├── __init__.py
│       │   ├── engine.py                # Backfill orchestration (chunk 1 FR-27 .. FR-30; chunk 2 6.7).
│       │   ├── checkpoint.py            # Batch-boundary checkpointing; resume support.
│       │   └── windowing.py             # received-from / received-to / max-messages window logic.
│       │
│       ├── undo/
│       │   ├── __init__.py
│       │   ├── engine.py                # Per-run reverse moves (chunk 1 FR-31 .. FR-33; chunk 2 6.8).
│       │   ├── idempotency.py           # Idempotency key per action; safe-replay.
│       │   └── purge_check.py           # Refuses if a quarantine purge has run since the target run (chunk 2 6.8 step 2).
│       │
│       ├── purge/
│       │   ├── __init__.py
│       │   └── quarantine.py            # `sift purge --quarantine`; interactive confirm; SR-13 retention rules; exit 5 if non-interactive without --yes (chunk 1 FR-36).
│       │
│       ├── scheduler/
│       │   ├── __init__.py              # Async loop wrapping `sift triage --run`.
│       │   ├── apscheduler_backend.py   # AsyncIOScheduler; missed-fire policy = coalesce-and-skip (chunk 3b 9.3).
│       │   └── lock_guard.py            # Composes with sift.lock to refuse overlapping runs.
│       │
│       └── cli/
│           ├── __init__.py
│           ├── main.py                  # typer app entry; dispatches subcommands.
│           ├── auth.py                  # `sift auth login`, `sift auth status`, `sift auth logout`.
│           ├── triage.py                # `sift triage`, `--dry-run`, `--run`, `--foreground`.
│           ├── backfill.py              # `sift backfill`, `--resume`.
│           ├── undo.py                  # `sift undo --run-id ID`, `--dry-run`.
│           ├── purge.py                 # `sift purge --quarantine`, `--yes` gating (chunk 3a SR-13).
│           ├── status.py                # `sift status`.
│           ├── llm.py                   # `sift llm test`, `sift llm models`, `sift llm pricing`.
│           ├── config_cmd.py            # `sift config validate`, `sift config show` (allow-no-providers permitted only on validate).
│           ├── schedule.py              # `sift schedule` (resolved per chunk 3b OQ #1; included in v1).
│           ├── sender_cache.py          # `sift sender-cache reset`, `sift sender-cache show`.
│           └── version.py               # `sift version`; embeds vendored router SHA.
│
├── tests/
│   ├── conftest.py                      # Shared fixtures; loads test_data/.
│   ├── unit/                            # 1:1 mirror of src/sift/.
│   │   ├── test_config_loader.py
│   │   ├── test_secrets_chain.py
│   │   ├── test_audit_log.py
│   │   ├── test_db_store.py
│   │   ├── test_graph_throttle.py
│   │   ├── test_pipeline_ordering.py
│   │   ├── test_reference_keeper.py
│   │   ├── test_sender_cache.py
│   │   ├── test_rules_regex.py
│   │   ├── test_rules_keyword.py
│   │   ├── test_rules_attachments.py
│   │   ├── test_backfill_checkpoint.py
│   │   ├── test_undo_idempotency.py
│   │   ├── test_undo_purge_check.py
│   │   ├── test_cost_persistence.py
│   │   ├── test_pricing_overrides.py
│   │   ├── test_low_confidence_routing.py
│   │   ├── test_llm_parser.py
│   │   └── test_lock_acquire.py
│   ├── integration/                     # Mock Graph + recorded LLM responses.
│   │   ├── test_triage_dry_run.py
│   │   ├── test_triage_full.py
│   │   ├── test_backfill_resume.py
│   │   ├── test_undo_full.py
│   │   ├── test_purge_quarantine.py
│   │   └── test_scheduler_loop.py
│   ├── live/                            # Marked @pytest.mark.live; opt-in via env (chunk 3b 11 intro).
│   │   ├── test_graph_auth_live.py
│   │   ├── test_provider_smoke_live.py
│   │   └── test_end_to_end_live.py
│   └── test_data/
│       ├── inbox-1000-backfill.json
│       ├── labelled-200.json
│       ├── labelled-50-invoices.json
│       ├── pricing-overrides-test.yaml
│       ├── failing-keyring-fixture.py
│       └── graph-mock-recorder/
│
├── config/
│   ├── sift.example.yaml                # Fully annotated sample config.
│   └── pricing-overrides.example.yaml   # Sample pricing override; same schema as vendored router.
│
├── docker/
│   ├── Dockerfile                       # python:3.13-slim base; non-root sift UID 10001 (chunk 3b 10.1).
│   ├── docker-compose.yml               # Reference compose (chunk 3b 10.1).
│   ├── healthcheck.sh                   # `sift status --health`; exit 0 if last run within 2 * schedule interval.
│   └── entrypoint.sh                    # Drops to UID 10001; resolves XDG paths.
│
├── systemd/
│   ├── sift.service                     # User unit; PrivateTmp, ProtectSystem=strict (chunk 3b 10.3).
│   └── sift.timer                       # OnCalendar=*:0/15; Persistent=true.
│
├── docs/
│   ├── install.md                       # Container, pipx, systemd paths.
│   ├── configuration.md                 # YAML schema; per-key examples; ENV var equivalents.
│   ├── operations.md                    # Cron / systemd / container scheduler comparison; runbook for routine tasks.
│   ├── troubleshooting.md               # Auth failures, throttling, cost = NULL, lock contention.
│   ├── architecture.md                  # Overview diagram; vendored boundary; pipeline ordering.
│   ├── security.md                      # Threat model summary; SR-01 .. SR-14 cross-reference.
│   ├── undo-and-purge.md                # Step-by-step examples.
│   └── faq.md
│
├── notes/                               # Engineering notes; not shipped on PyPI.
│   ├── solace-snapshot-log.md           # Each re-snapshot date / commit / diff summary.
│   └── deferred-features.md             # v1.x candidates from chunk 3 OQ resolutions.
│
└── .github/
    ├── workflows/
    │   ├── ci.yml                       # uv sync, ruff, mypy --strict, pytest, pip-audit, vendored test suite.
    │   ├── nightly-live.yml             # @pytest.mark.live job (manual or schedule); not blocking.
    │   ├── release.yml                  # Tag-triggered: build sdist + wheel; publish to PyPI; build container; sign.
    │   └── codeql.yml                   # Static analysis.
    └── ISSUE_TEMPLATE/
        └── bug_report.md
```

### 12.2 Build sequence

Sift v1 is built in dependency order. Each row is a discrete unit of work; rows in the same step number MAY run in parallel. Agent assignments use the suggested roster: Python Backend Developer (PBD), DevOps Engineer (DOE), Security Specialist (SEC), Linux Expert (LX), Writing Quality Specialist (WQS).

| Order | File(s) / Artefact | Component | Depends On | Agent |
|-------|--------------------|-----------|------------|-------|
| 1 | `vendor/llm_router/` snapshot, `VENDORED.md` | Vendoring | Solace router commit chosen | PBD |
| 2 | `vendor/llm_router/tests/` run on green | Vendoring | Step 1 | PBD |
| 3 | `pyproject.toml`, `uv.lock`, `.python-version` | Packaging | Step 1 (vendored deps known) | PBD |
| 4 | `src/sift/models/base.py` | Core models | Step 3 | PBD |
| 5 | `src/sift/models/{mail,classification,state,cost,errors,router}.py` | Core models | Step 4 | PBD |
| 6 | `src/sift/config/{__init__,loader,graph,llm,folders,retention,rules,sender_cache,pricing}.py` | Config | Step 5 | PBD |
| 7 | `src/sift/security/{permissions,redaction,sender_hash}.py` | Security primitives | Step 5 | SEC |
| 8 | `src/sift/secrets/{keyring_backend,env_backend,chain}.py` | Secrets chain | Step 7 | SEC |
| 9 | `src/sift/log/{audit,events,redact_filter}.py` | Logging | Step 7 | PBD |
| 10 | `src/sift/lock/__init__.py` | Lock primitive | Step 3 | PBD |
| 11 | `src/sift/db/{schema,migrations/0001_init.sql,store}.py` | State store | Steps 5, 10 | PBD |
| 12 | `src/sift/db/migrations/0002_purge_log.sql` and store hooks | State store | Step 11 | PBD |
| 13 | `src/sift/graph/{auth}.py` | Graph auth | Steps 6, 8 | PBD |
| 14 | `src/sift/graph/{throttle,client,folders,messages}.py` | Graph client | Steps 9, 13 | PBD |
| 15 | `src/sift/cost/{persistence,overrides,summary}.py` | Cost wiring | Steps 11, vendored router cost contract | PBD |
| 16 | `src/sift/llm/{client,adapter,prompts,label_desc,parser,bootstrap}.py` | LLM facade | Steps 6, 15 | PBD |
| 17 | `src/sift/rules/{regex,keyword,attachments}.py` | Rule layer | Step 6 | PBD |
| 18 | `src/sift/reference_keeper/*.py` | Reference keeper | Steps 16, 17 | PBD |
| 19 | `src/sift/sender_cache/{store,reset}.py` | Sender cache | Step 11 | PBD |
| 20 | `src/sift/pipeline/{ordering,llm_stage,low_confidence}.py` | Pipeline | Steps 16, 17, 18, 19 | PBD |
| 21 | `src/sift/backfill/{engine,checkpoint,windowing}.py` | Backfill | Steps 14, 20 | PBD |
| 22 | `src/sift/undo/{engine,idempotency,purge_check}.py` | Undo | Steps 14, 11 | PBD |
| 23 | `src/sift/purge/quarantine.py` | Quarantine purge | Steps 14, 12 | PBD |
| 24 | `src/sift/scheduler/{apscheduler_backend,lock_guard}.py` | Scheduler | Steps 10, 21 | PBD |
| 25 | `src/sift/cli/{main,auth,triage,backfill,undo,purge,status,llm,config_cmd,schedule,sender_cache,version}.py` | CLI | Steps 13 .. 24 | PBD |
| 26 | `tests/unit/*` | Unit tests | Mirrors steps 4 .. 25 | PBD |
| 27 | `tests/integration/*` | Integration tests | Step 26 | PBD |
| 28 | `tests/live/*` (gated) | Live tests | Step 27 | PBD |
| 29 | `tests/test_data/*` | Test fixtures | Step 5 | PBD |
| 30 | `docker/Dockerfile`, `entrypoint.sh`, `healthcheck.sh` | Container | Step 25 | DOE |
| 31 | `docker/docker-compose.yml` | Container | Step 30 | DOE |
| 32 | `systemd/sift.service`, `systemd/sift.timer` | systemd | Step 25 | LX |
| 33 | `.github/workflows/ci.yml` | CI | Step 27 | DOE |
| 34 | `.github/workflows/codeql.yml` | CI | Step 33 | SEC |
| 35 | `.github/workflows/nightly-live.yml` | CI | Step 28 | DOE |
| 36 | `.github/workflows/release.yml` | Release | Steps 30, 33 | DOE |
| 37 | `pricing-overrides.yaml`, `config/sift.example.yaml`, `config/pricing-overrides.example.yaml` | Configuration samples | Steps 6, 15 | PBD |
| 38 | `docs/install.md`, `docs/configuration.md`, `docs/operations.md` | Docs | Steps 30, 31, 32 | WQS |
| 39 | `docs/troubleshooting.md`, `docs/security.md`, `docs/undo-and-purge.md`, `docs/architecture.md`, `docs/faq.md` | Docs | Steps 14, 22, 23 | WQS |
| 40 | `README.md` rewrite, `CHANGELOG.md` v0.1.0 entry | Docs | Step 38 | WQS |
| 41 | Final pip-audit + secret-scan dry run on full repo | Validation | Steps 33, 34 | SEC |
| 42 | Tag v0.1.0; release workflow runs | Release | Step 36, step 41 | DOE |

### 12.3 Parallelizable work

Once the listed prerequisites complete, the following step groups MAY run in parallel:

- Steps 4 and 7 (model base config, security primitives) after step 3.
- Steps 8 and 9 (secrets chain, logging) after step 7.
- Steps 13 and 17 (Graph auth, rule layer) after step 8 / step 6.
- Steps 18, 19 (reference keeper, sender cache) after their respective prerequisites; both can proceed in parallel before step 20.
- Step 26 (unit tests) tracks the source files; per-module unit-test files MAY be authored in parallel by multiple agents.
- Steps 30 and 32 (container, systemd) after step 25; both deployment surfaces are independent.
- Step 33 (CI) and step 36 (release) workflow files are independent of each other; both depend on tests and packaging being green.
- Step 38 and step 39 (docs) are independent and MAY be authored in parallel.

---

## 13. Implementation Sub-Issues

The following sub-issues SHALL be created on the `stschulznz/sift` tracker and dispatched per workspace conventions. Each sub-issue references the FR / NFR / SR / AC identifiers established in earlier chunks. Acceptance criteria are falsifiable.

### Sub-Issue 1: Vendoring milestone

**Assigned agent:** Python Backend Developer
**Scope:** `vendor/llm_router/`, `VENDORED.md`, vendored test suite run.
**Details:**
- Choose the Solace router commit (latest green on Solace `main`) and snapshot the entire `llm_router/` package into `vendor/llm_router/` verbatim.
- Record snapshot SHA, copy date, and the SHA-256 of the snapshot tarball in `VENDORED.md`.
- Add the re-export shim in `vendor/llm_router/__init__.py` so that `from sift.vendor.llm_router import dispatch, models` works (chunk 3b 9.2).
- Run the vendored adapter test suite under Sift's CI matrix; do not modify any vendored source.
- Smoke test: a 5-message classification run against GitHub Copilot using the vendored router unmodified.
**Acceptance:** SC-9 met (all four vendored adapters pass the vendored test suite). `VENDORED.md` lists snapshot SHA, copy date, and divergence policy. The smoke run succeeds with exit 0 and one row appears in `usage_log` with `agent_name = "sift"` (FR-26). SR-11 verified.

### Sub-Issue 2: Configuration and Pydantic models

**Assigned agent:** Python Backend Developer
**Scope:** `src/sift/models/`, `src/sift/config/`, `tests/unit/test_config_loader.py`.
**Details:**
- Implement every Pydantic v2 model from chunk 2 section 4 with `STRICT` config (chunk 2 4.1).
- Implement layered loader (defaults < `~/.config/sift/config.yaml` < `./sift.yaml` < `SIFT_*` env < CLI flags) per FR-37.
- Reject unknown YAML keys (`extra="forbid"`) with exit code 1 and a Pydantic error report (chunk 3a SR-09).
- Allow zero providers only for `sift config validate` (chunk 3b 11; chunk 3a SR-14 surface).
**Acceptance:** AC-CONFIG-* tests pass. Unknown key in `sift.yaml` exits 1 with the offending key reported. Naive datetime in YAML rejected. `sift config validate` with zero providers succeeds; any other subcommand with zero providers exits 3.

### Sub-Issue 3: SQLite state store and migrations

**Assigned agent:** Python Backend Developer
**Scope:** `src/sift/db/`, `tests/unit/test_db_store.py`.
**Details:**
- Implement schema for `runs`, `actions`, `action_log`, `sender_cache`, `usage_log`, `purge_log`, `checkpoints` tables (chunk 2 4.6, 4.7; chunk 3b 9.2 cost; chunk 2 6.8 OQ #4 resolved with a dedicated `purge_log` table).
- WAL mode, `synchronous=NORMAL`, single-writer lock via `sift.lock` (chunk 2 6.10).
- Numbered SQL migrations in `src/sift/db/migrations/`; runtime applies pending migrations on startup.
- File mode 0600 enforced via `sift.security.permissions.assert_locked_down` (SR-08).
**Acceptance:** Two concurrent writers are serialised (no `database is locked` errors); a third reader proceeds without blocking writers under WAL. Migration from a 0001-only DB to current head succeeds without data loss. SR-08 verified.

### Sub-Issue 4: Microsoft Graph client

**Assigned agent:** Python Backend Developer
**Scope:** `src/sift/graph/`, `tests/unit/test_graph_throttle.py`, `tests/integration/test_triage_full.py` Graph paths.
**Details:**
- Device-code OAuth via `msal` (chunk 1 FR-01, SR-01); token cached at `~/.cache/sift/graph-token.json` mode 0600.
- One `httpx.AsyncClient` per run (chunk 2 6.2).
- Retry-After honoured to the second; retries SHALL NOT exceed `Retry-After` (chunk 1 FR-04, NFR-07; AC-THROTTLE-*).
- Concurrency capped by `concurrency` config; default 4 (chunk 3a SR-13).
- Folder discovery and create-if-missing (FR-08).
- `$batch` move endpoint when batch size > 1.
**Acceptance:** AC-THROTTLE-01, AC-THROTTLE-02 pass. Backfill of the 1000-message fixture against the recorder mock completes with zero `RetryAfterIgnored` events. Token refresh transparent across a > 1-hour run. SR-01 verified.

### Sub-Issue 5: Rule layer, reference keeper, sender cache

**Assigned agent:** Python Backend Developer
**Scope:** `src/sift/rules/`, `src/sift/reference_keeper/`, `src/sift/sender_cache/`, `tests/unit/test_rules_*.py`, `tests/unit/test_reference_keeper.py`, `tests/unit/test_sender_cache.py`.
**Details:**
- RuleEngine evaluates regex / keyword / attachment rules (chunk 1 FR-15, FR-16, FR-17).
- Reference keeper is a mandatory override (chunk 2 6.6); it can route to Reference even when an earlier rule did not fire, and can override an LLM `To-Be-Deleted` decision for invoice / receipt / tax / contract / statement signals.
- Sender cache thresholds: N >= 6 prior LLM classifications, all matching, confidence floor 0.9 (chunk 2 6.4); reset on undo and on user override.
**Acceptance:** AC-RULES-*, AC-REFKEEPER-* pass. SC-4 met (< 1% of the 50-invoice fixture routed to To-Be-Deleted). Sender-cache-induced false-positives in a deliberate adversarial fixture do not propagate after one undo.

### Sub-Issue 6: Classification pipeline

**Assigned agent:** Python Backend Developer
**Scope:** `src/sift/pipeline/`, `src/sift/llm/{prompts,parser,adapter}.py`, `tests/unit/test_pipeline_ordering.py`, `tests/unit/test_llm_parser.py`, `tests/unit/test_low_confidence_routing.py`.
**Details:**
- Ordering: rule -> reference_keeper -> sender_cache -> LLM -> low_confidence_review (chunk 1 FR-09; chunk 2 OQ #1 deferred, `reply_to_self` not implemented in v1).
- LLM stage uses deterministic prompt template, label list, per-label one-liner, JSON-schema response shape.
- Parser: strict JSON parse first; on failure, attempt brace-balance recovery; on second failure, regex extract the label field; on third failure, raise `LLMResponseUnparseable` and route the message to Triage/Review.
- Confidence floor configurable; messages below the floor SHALL route to Triage/Review and SHALL NEVER route to To-Be-Deleted (chunk 1 FR-25).
**Acceptance:** SC-3 met (macro F1 >= 0.80 on `labelled-200.json` for each provider). AC-PIPELINE-ORDER-* pass. Truncated JSON fixture does not move the message; one structured event `LLMResponseRecovered` or `LLMResponseUnparseable` is emitted per case.

### Sub-Issue 7: Backfill engine

**Assigned agent:** Python Backend Developer
**Scope:** `src/sift/backfill/`, `tests/integration/test_backfill_resume.py`.
**Details:**
- Implement window resolution (received-from / received-to / max-messages), batched fetch and processing, batch-boundary checkpointing (chunk 1 FR-27 .. FR-30; chunk 2 6.7).
- Resume reads the last checkpoint and replays no message twice and skips no message (chunk 1 SC-10).
- Stops cleanly on SIGINT and SIGTERM; checkpoint flushed on signal.
**Acceptance:** SC-10 met. Killing the process at 50% completion and re-invoking with `--resume` produces identical final state to an uninterrupted run. AC-BACKFILL-* pass.

### Sub-Issue 8: Undo engine

**Assigned agent:** Python Backend Developer
**Scope:** `src/sift/undo/`, `tests/unit/test_undo_idempotency.py`, `tests/unit/test_undo_purge_check.py`, `tests/integration/test_undo_full.py`.
**Details:**
- Per-run reverse moves using `action_log` (chunk 1 FR-31 .. FR-33; chunk 2 6.8).
- Idempotency: replay of an already-reversed action is a no-op with INFO event.
- Refuse undo if a `sift purge --quarantine` event has run since the target run; lookup against the `purge_log` table (chunk 2 OQ #4 resolved as option (b)).
- `--dry-run` lists every reverse move without performing it.
**Acceptance:** SC-7, SC-8 met. AC-UNDO-* pass. Replay of a completed undo run produces zero Graph mutations and exits 0. Undo across a purge boundary refuses with exit code 1 and the structured event `UndoRefusedPurgeBoundary`.

### Sub-Issue 9: Cost tracker wiring

**Assigned agent:** Python Backend Developer
**Scope:** `src/sift/cost/`, `tests/unit/test_cost_persistence.py`, `tests/unit/test_pricing_overrides.py`.
**Details:**
- Implement Sift's `CostPersistenceAdapter` against the vendored router's `UsageSink` protocol (chunk 3b 9.2). Set `agent_name = "sift"` on every `UsageRecord`.
- Implement `PricingOverrideLoader`: parse `pricing_overrides_path` YAML and merge into the vendored pricing table at startup.
- `cost_usd = NULL` for unknown pricing (chunk 1 SC-12; Solace FR-46). NEVER write 0.0.
- Per-run `RunCostSummary` displays `cost_usd=null` when any contributing record was NULL (chunk 1 FR-47-equivalent).
**Acceptance:** AC-COST-01, AC-COST-02 pass. A test with no pricing row and no override file produces `cost_usd=null` in `usage_log` and the per-run summary shows `cost_usd=null`. Adding a row in the override file changes the summary to a numeric value on the next run.

### Sub-Issue 10: Scheduler and lock guard

**Assigned agent:** Python Backend Developer
**Scope:** `src/sift/scheduler/`, `src/sift/lock/`, `tests/unit/test_lock_acquire.py`, `tests/integration/test_scheduler_loop.py`.
**Details:**
- File-lock at `$XDG_DATA_HOME/sift/run.lock` mode 0600; `fcntl.flock` POSIX, `msvcrt.locking` Windows (chunk 2 6.10).
- AsyncIOScheduler loop wrapping `sift triage --run`; missed-fire policy = coalesce-and-skip (chunk 3b 9.3).
- Stale-lock reclamation when PID is no longer running on the same host (WARNING event).
**Acceptance:** AC-LOCK-* pass. Two concurrent invocations: the second exits 1 with the lock-contention message. AC-SCHEDULER-MISSED-FIRE passes after a simulated 10-minute pause.

### Sub-Issue 11: CLI surface

**Assigned agent:** Python Backend Developer
**Scope:** `src/sift/cli/`, `tests/unit/test_cli_*.py`.
**Details:**
- Implement every subcommand listed in chunk 2 5.1 using typer.
- Include `sift schedule` in v1 (chunk 3b OQ #1 resolved: include).
- `sift purge --quarantine` with no TTY and no `--yes` SHALL exit 5 (chunk 3a SR-13).
- `sift status` outputs both human-readable and `--json` shapes; structured event vocabulary per chunk 2 5.1.16.
- `sift version` embeds the vendored router SHA from `VENDORED.md`.
**Acceptance:** AC-CLI-* pass. `sift purge --quarantine < /dev/null` exits 5. `sift schedule --interval 15m` runs and obeys the lock. `sift version` includes the vendored SHA.

### Sub-Issue 12: Docker and Podman packaging

**Assigned agent:** DevOps Engineer
**Scope:** `docker/Dockerfile`, `docker/entrypoint.sh`, `docker/healthcheck.sh`, `docker/docker-compose.yml`.
**Details:**
- Base `python:3.13-slim`; non-root user `sift` UID 10001 GID 10001 (chunk 3b 10.1).
- Compose file maps `~/.config/sift`, `~/.cache/sift`, `~/.local/share/sift` to host volumes; container long-running command `sift schedule` (chunk 3b OQ #1 resolved).
- `healthcheck.sh` invokes `sift status --health`; healthy if last successful run was within 2 * schedule interval.
- Verify image runs unmodified under `podman compose up`.
**Acceptance:** AC-DEPLOY-01 passes. Container starts, completes device-code auth, runs the in-container scheduler loop, and remains healthy. Podman parity verified manually on a Bazzite host.

### Sub-Issue 13: systemd unit and timer

**Assigned agent:** Linux Expert
**Scope:** `systemd/sift.service`, `systemd/sift.timer`.
**Details:**
- User unit (`~/.config/systemd/user/`); `OnCalendar=*:0/15`, `Persistent=true` (chunk 3b 10.3).
- `PrivateTmp=true`, `ProtectSystem=strict`, `NoNewPrivileges=true`, `ReadWritePaths` only for XDG state dirs.
- Refuse on world-writable config (chunk 3b 10.3 line 849).
**Acceptance:** AC-DEPLOY-02 passes. Unit refuses to start when `~/.config/sift/config.yaml` mode is 0644 group-writable; warning logged. Timer triggers `sift triage --run` every 15 minutes; missed fires collapsed by `Persistent=true`.

### Sub-Issue 14: CI and quality gates

**Assigned agent:** DevOps Engineer
**Scope:** `.github/workflows/ci.yml`, `.github/workflows/codeql.yml`, `.github/workflows/nightly-live.yml`.
**Details:**
- CI matrix: Linux + macOS + Windows; Python 3.13.
- Run `uv sync`, `ruff check`, `ruff format --check`, `mypy --strict src/sift`, `pytest tests/unit tests/integration`, the vendored adapter test suite, `pip-audit`, secret-scan via `gitleaks` or `truffleHog`.
- Nightly live job (`@pytest.mark.live`) runs against the maintainer's tenant; not blocking (chunk 3b OQ #4 deferred).
- CodeQL on push to `main` and on PRs (note: solo repo runs main-direct per workspace policy; CodeQL still attaches to push events).
**Acceptance:** A failing `mypy --strict` blocks merge. `pip-audit` finding above HIGH blocks merge. NFR-10 met (>= 80% coverage on unit tests). Vendored adapter test suite green.

### Sub-Issue 15: Documentation rewrite

**Assigned agent:** Writing Quality Specialist
**Scope:** `README.md`, `docs/install.md`, `docs/configuration.md`, `docs/operations.md`, `docs/troubleshooting.md`, `docs/security.md`, `docs/undo-and-purge.md`, `docs/architecture.md`, `docs/faq.md`, `CHANGELOG.md`.
**Details:**
- README rewrite covers what Sift is, who it is for, install commands for container / pipx / systemd, and links to docs.
- `install.md` covers all three deployment paths (container, pipx, systemd) with full worked examples.
- `troubleshooting.md` covers: device-code auth fails, throttling-stuck, `cost = NULL` interpretation, lock contention, sender-cache poisoning recovery.
- `security.md` cross-references SR-01 .. SR-14 with one paragraph per SR.
- `undo-and-purge.md` walks the operator through a real `sift undo --run-id` and a real `sift purge --quarantine` interaction.
- All docs run through `slop-detector` and `document-review` skills before merge.
**Acceptance:** No banned phrases per workspace style rules. `slop-detector` audit of every file in `docs/` reports a clean run. CHANGELOG entry for v0.1.0 lists every FR / NFR / SR met.

### Sub-Issue 16: Test suite organisation

**Assigned agent:** Python Backend Developer
**Scope:** `tests/conftest.py`, `tests/unit/`, `tests/integration/`, `tests/live/`, `tests/test_data/`.
**Details:**
- Unit / integration / live split per chunk 3b 11; `pytest -m "not live"` is the default CI invocation; `pytest -m live` requires `SIFT_LIVE=1`.
- Fixtures: `inbox-1000-backfill.json`, `labelled-200.json`, `labelled-50-invoices.json`, `pricing-overrides-test.yaml`, `failing-keyring-fixture.py`, `graph-mock-recorder/` (chunk 3b 11.14).
- All test addresses synthetic (`@example.com`, `@example.org`, `@spam.example`).
**Acceptance:** `pytest` default run is hermetic; no network. `pytest -m live` is opt-in and gated on `SIFT_LIVE=1`. SC-3 and SC-4 measured against fixtures; results recorded in CI artefact.

### Sub-Issue 17: Security validation pass

**Assigned agent:** Security Specialist
**Scope:** Cross-cutting; produces an audit report committed to `notes/security-validation-v0.1.0.md`.
**Details:**
- Re-verify SR-01 .. SR-14 against the implemented code; produce a SR-by-SR pass / fail table.
- Run `pip-audit`, `gitleaks`, `bandit -r src/`, and a manual review for: credential leakage paths, hard-delete code paths (must be zero in v1), audit-log redaction filter coverage, sender-hash use in audit log, file-mode enforcement at every state-store path.
- Confirm OWASP Top 10 alignment for the CLI's external surfaces (config parsing, OAuth callback, message parsing).
**Acceptance:** SR-01 .. SR-14 each marked `pass` with evidence. Zero high-severity findings from `pip-audit`. Zero credential matches from `gitleaks` against the full git history.

### Sub-Issue 18: PyPI publish workflow

**Assigned agent:** DevOps Engineer
**Scope:** `.github/workflows/release.yml`.
**Details:**
- Trigger on tag `v*.*.*`. Build sdist + wheel via `uv build`; sign artefacts; publish to PyPI as `sift-mail` (chunk 3b 10.2; chunk 3b OQ #3 resolved: keep `sift-mail`).
- Build container image (multi-arch: linux/amd64, linux/arm64); push to GHCR; sign with `cosign`.
- Generate SBOM via `cyclonedx-py`; attach as release asset.
- Create GitHub release with auto-generated notes from CHANGELOG.
**Acceptance:** Tagging `v0.1.0` produces a published `sift-mail==0.1.0` on PyPI, a signed container image on GHCR, and a release with CHANGELOG body and SBOM attached. `pipx install sift-mail` from a clean machine succeeds and `sift version` prints `0.1.0` plus the vendored SHA.

---

## 14. Anti-Patterns

Each anti-pattern below shows the WRONG approach and the CORRECT approach. Prevent every WRONG below from entering the codebase.

**Anti-pattern 1: LLM API keys in YAML.**

```yaml
# WRONG: ./sift.yaml
llm:
  azure_openai:
    api_key: "sk-abc...xyz"
```

This puts a long-lived secret on disk in a file that is commonly committed to dotfile repos and synced to cloud storage. SR-02 forbids it.

```yaml
# CORRECT: ./sift.yaml
llm:
  azure_openai:
    api_key_env: "SIFT_AZURE_OPENAI_API_KEY"   # OR omit and rely on keyring
```

Resolution chain (chunk 1 SR-02): keychain via `keyring` first, then `SIFT_<PROVIDER>_API_KEY` env, then a locked-down `~/.config/sift/secrets.env` (mode 0600). Never the YAML file directly.

**Anti-pattern 2: One `httpx` client per Graph call.**

```python
# WRONG
async def fetch(message_id: str) -> dict:
    async with httpx.AsyncClient() as client:
        return (await client.get(url)).json()
```

This creates a fresh TCP + TLS handshake per message. Across a 100k-message backfill the connection churn alone misses NFR-01.

```python
# CORRECT
class GraphClient:
    def __init__(self) -> None:
        self._client = httpx.AsyncClient(http2=True, timeout=httpx.Timeout(30.0))
    async def fetch(self, message_id: str) -> dict:
        return (await self._client.get(url)).json()
    async def aclose(self) -> None:
        await self._client.aclose()
```

One client per run; closed in the run's outer `try / finally`.

**Anti-pattern 3: Skipping `Retry-After`.**

```python
# WRONG
if resp.status_code == 429:
    await asyncio.sleep(1.0)        # ignores Retry-After
    return await retry()
```

Microsoft Graph returns precise wait values; ignoring them gets the tenant deeper into throttling and violates FR-04 / NFR-07.

```python
# CORRECT
if resp.status_code == 429:
    delay = parse_retry_after(resp.headers.get("Retry-After"))
    log_event("RetryAfterHonoured", retry_after_seconds=delay)
    await asyncio.sleep(delay)
    return await retry()
```

Throttle log records every wait; AC-THROTTLE-* asserts zero `RetryAfterIgnored` events across a backfill.

**Anti-pattern 4: Hard-deleting messages directly.**

```python
# WRONG
await graph.delete(f"/me/messages/{message_id}")
```

SR-10 forbids hard-delete in v1. The message is gone from the user's mailbox with no recovery path.

```python
# CORRECT
await graph.move(message_id, target_folder_id=cfg.folders.quarantine)
state.record_action(run_id, message_id, source_folder_id, cfg.folders.quarantine, ...)
```

Every "delete" is a move to the quarantine folder. Removal from quarantine happens only via `sift purge --quarantine` after retention has elapsed.

**Anti-pattern 5: `gho_*` token in YAML or logs.**

```yaml
# WRONG
llm:
  github_copilot:
    oauth_token: "gho_abc...xyz"
```

```text
# WRONG audit-log line
event=LLMRequest provider=github_copilot oauth_token=gho_abc...xyz prompt_tokens=812
```

The Copilot OAuth token is a long-lived bearer. SR-03 / SR-04 forbid this on disk and in logs.

```yaml
# CORRECT
llm:
  github_copilot:
    token_path: "~/.cache/sift/copilot-token.json"   # mode 0600 enforced
```

The audit-log redaction filter (`sift.log.redact_filter`) strips any string matching the credential regex set; CI scans logs against the same set.

**Anti-pattern 6: `cost_usd = 0.0` for unknown pricing.**

```python
# WRONG
cost_usd = pricing.get(provider, model)  # returns 0.0 if missing
state.record_usage(... cost_usd=cost_usd)
```

SC-12 and Solace FR-46 require NULL for unknown pricing. Writing 0.0 silently understates cost and breaks the per-run summary.

```python
# CORRECT
price = pricing.lookup(provider, model)            # may return None
cost_usd = price.compute(usage) if price else None
state.record_usage(... cost_usd=cost_usd)          # NULL preserved through to SQLite
```

The per-run summary surfaces `cost_usd=null` whenever any contributing record was NULL (chunk 3b 11 cost AC).

**Anti-pattern 7: Single-shot LLM call without JSON repair.**

```python
# WRONG
resp = await router.generate(prompt)
label = json.loads(resp.text)["label"]  # raises on truncated JSON, kills the run
```

Provider responses are sometimes truncated, sometimes wrap JSON in markdown fences, sometimes append commentary. A single `json.loads` makes one LLM hiccup fail an entire batch.

```python
# CORRECT
def parse(text: str) -> dict:
    try:
        return json.loads(text)
    except json.JSONDecodeError:
        recovered = balance_braces(text)
        try:
            return json.loads(recovered)
        except json.JSONDecodeError:
            label = regex_extract_label(text)
            if label:
                log_event("LLMResponseRecovered", strategy="regex_extract")
                return {"label": label, "confidence": 0.0}
            raise LLMResponseUnparseable(text)
```

Three repair strategies; on final failure the message routes to Triage/Review with a structured event (Sub-Issue 6 acceptance).

**Anti-pattern 8: Sender cache that does not reset on undo.**

```python
# WRONG
async def undo(run_id: str) -> None:
    for action in state.actions_for(run_id):
        await graph.move(action.message_id, action.source_folder_id)
    # sender_cache never touched
```

If the run that is being undone polluted the sender cache (e.g., 6 wrong classifications for `bills@example.com`), every future message from that sender continues to short-circuit to the wrong folder.

```python
# CORRECT
async def undo(run_id: str) -> None:
    affected_senders: set[str] = set()
    for action in state.actions_for(run_id):
        await graph.move(action.message_id, action.source_folder_id)
        affected_senders.add(action.sender_hash)
    sender_cache.invalidate_many(affected_senders)
    log_event("SenderCacheInvalidated", count=len(affected_senders), reason="undo")
```

Undo is the user's correction signal. Trust it.

**Anti-pattern 9: Backfill without checkpointing.**

```python
# WRONG
async def backfill(window) -> None:
    async for batch in graph.iterate(window):
        for message in batch:
            classification = pipeline.classify(message)
            await graph.move(message.id, classification.target)
    # if the process dies at 73%, the next run starts at 0%
```

A 100k-message backfill that crashes at 73% and re-runs from 0 wastes ~10 hours and double-classifies every message it already processed.

```python
# CORRECT
async def backfill(window, resume: bool = False) -> None:
    cursor = state.last_checkpoint(window) if resume else None
    async for batch in graph.iterate(window, after=cursor):
        for message in batch:
            classification = pipeline.classify(message)
            await graph.move(message.id, classification.target)
        state.checkpoint(window, batch.last_received_at, batch.last_message_id)
```

Checkpoints land at batch boundaries (chunk 2 6.7); resume reads the last checkpoint and replays no message twice.

**Anti-pattern 10: Confidence-based fallback to To-Be-Deleted.**

```python
# WRONG
if classification.confidence < cfg.confidence_floor:
    classification.label = "To-Be-Deleted"  # safe-looking but destructive
```

Low confidence means the system does not know. Routing unknown messages to a quarantine-bound folder is precisely the wrong default. FR-25 forbids this.

```python
# CORRECT
if classification.confidence < cfg.confidence_floor:
    classification.label = "Triage/Review"
    classification.source = ClassificationSource.LOW_CONFIDENCE_REVIEW
    log_event("LowConfidenceRouted", from_label=classification.label_proposed, confidence=classification.confidence)
```

Triage/Review is the only safe default for low-confidence outcomes.

**Anti-pattern 11: Reference-keeper as advisory.**

```python
# WRONG
ref_hint = reference_keeper.consult(message)
classification = llm.classify(message)
if ref_hint and classification.confidence < 0.5:
    classification.label = ref_hint
```

This makes a safety net optional. An invoice that the LLM decisively classifies as `To-Be-Deleted` (high confidence, wrong) bypasses the safety net entirely.

```python
# CORRECT
ref = reference_keeper.classify(message)
if ref:
    return ref                                   # mandatory override (chunk 2 6.6)
classification = llm.classify(message)
if classification.label == Folder.TO_BE_DELETED and reference_keeper.is_protected(message):
    classification.label = Folder.REFERENCE      # secondary safety net
    log_event("ReferenceKeeperOverride")
```

Protection of invoices, receipts, tax docs, contracts, and statements is non-negotiable.

**Anti-pattern 12: Logging full message bodies in audit.**

```python
# WRONG
audit.write({"event": "Classified", "subject": message.subject, "body": message.body})
```

SC-14 forbids full bodies in the audit log. Even subjects can leak; SR-04 requires sender hashes (HMAC-SHA-256) instead of plain addresses.

```python
# CORRECT
audit.write({
    "event": "Classified",
    "message_id": message.id,
    "sender_hash": sender_hash(message.from_),
    "subject_hash": subject_hash(message.subject),  # short HMAC for dedup, not the subject text
    "label": classification.label.value,
    "confidence": classification.confidence,
    "run_id": run_id,
})
```

The audit log must reconstruct what happened without revealing what the messages said.

**Anti-pattern 13: `sift purge --quarantine` non-interactive without `--yes` and no TTY.**

```bash
# WRONG (in cron)
sift purge --quarantine >> /var/log/sift/cron.log 2>&1
```

In cron there is no TTY. The interactive confirmation prompt exits immediately on EOF; the prior implementation silently proceeded with deletion. This is exactly the failure mode SR-13 (chunk 3a) closes.

```bash
# CORRECT (interactive only)
sift purge --quarantine                  # interactive shell, prompts for "DELETE"
# OR (deliberate, fail-loud non-interactive)
sift purge --quarantine --yes            # caller has explicitly accepted
```

Without a TTY and without `--yes`, the command SHALL exit code 5 with `"refusing to purge non-interactively without --yes"` (chunk 3a SR-13).

**Anti-pattern 14: Using PR workflow instead of pushing direct to `main`.**

```bash
# WRONG (in this solo repo)
git checkout -b feature/add-foo
git push -u origin feature/add-foo
gh pr create --title "Add foo" --body "..."
```

`stschulznz/sift` is solo and main-direct, like the rest of this owner's repos. Feature branches add overhead with no review benefit and create divergence problems in the VFS workspace.

```bash
# CORRECT
git checkout main
git pull --rebase
# edit, test
git commit -am "Add foo"
git push origin main
```

Or for VFS-driven changes: push directly to `main` via the GitHub Contents API (`mcp_io_github_git_create_or_update_file`). The workspace's global Copilot instructions enforce this rule.

**Anti-pattern 15: Mutating `vendor/llm_router/` to fix Sift issues.**

```python
# WRONG: editing vendor/llm_router/adapters/github_copilot.py to log Sift run_id
log.info("sift run %s sending request", sift_run_id)
```

Vendored code SHALL NOT be modified (SR-11). Local edits make the next snapshot a merge problem and silently mask upstream fixes.

```python
# CORRECT: subclass or compose at the Sift boundary
class SiftCopilotAdapter:
    def __init__(self, inner: GitHubCopilotAdapter) -> None:
        self._inner = inner
    async def generate(self, request, *, sift_run_id: str):
        log.info("sift run %s sending request", sift_run_id)
        return await self._inner.generate(request)
```

All Sift-specific behaviour lives in `src/sift/`. The vendored snapshot is updated only by re-snapshotting and recording the new SHA in `VENDORED.md`.

---

## 15. Decision Log

| Decision | Alternatives considered | Rationale |
|----------|-------------------------|-----------|
| **Vendor the Solace LLM router for v1 instead of extracting it as a shared package** | (a) Wait for Solace to extract `llm_router` into its own PyPI package; (b) Fork the package and depend on the fork | The router is not yet extracted, and waiting blocks Sift v1 ship. Vendoring keeps Sift unblocked, lets us collect real-world friction data, and preserves the option to switch to a published package later (`VENDORED.md` records the snapshot SHA so the swap is mechanical). |
| **SQLite for state, not Postgres** | (a) Postgres; (b) Embedded LMDB; (c) Flat NDJSON file | SQLite ships with CPython, requires zero ops, supports WAL for concurrent reads with a single writer (chunk 2 6.10), and survives the v1 scale (1M-message inboxes are a few hundred MB of state). Postgres would force every user to run a database server for a CLI tool. |
| **`typer` for the CLI surface, not bare `click` or `argparse`** | (a) `click` directly; (b) `argparse`; (c) `fire` | `typer` builds on `click` but uses Python type hints; it composes naturally with the Pydantic-typed config models and produces good `--help` output without manual decoration. |
| **`apscheduler` as the in-process scheduler, with external cron as the documented alternative** | (a) Cron only; (b) systemd timers only; (c) Celery beat | The container deployment path needs an in-process scheduler (no cron in the container). `apscheduler` is dependency-light, has a coalesce-and-skip missed-fire policy that matches FR-30, and lets users on bare-metal hosts opt to use cron or systemd timers (chunk 3b 9.3, 10.3). |
| **`keyring` first, then env, then a locked-down secrets file** | (a) Env only; (b) HashiCorp Vault; (c) Cloud KMS | Aligns with the typical self-hoster: the OS keychain is the right default on a developer workstation; env vars work in containers; a 0600 file works in headless server installs. Vault / KMS are excessive for a single-user CLI. |
| **Microsoft Graph SDK NOT used; raw `httpx` instead** | (a) `msgraph-sdk-python`; (b) raw `requests` | The official SDK pulls a large dependency tree, has historically been async-unfriendly, and adds an abstraction layer that obscures throttle behaviour. Raw `httpx` keeps the throttle path observable (chunk 2 6.2) and the dependency surface narrow. |
| **`msal` for device-code OAuth** | (a) `authlib`; (b) hand-rolled device-code flow | `msal` is the Microsoft-maintained library that consumes the official metadata endpoints, handles token refresh, and is the clearest path to least-friction support if the device-code flow changes. |
| **Default model per provider: Copilot `gpt-4o`, Claude `claude-sonnet-4-5`, Azure OpenAI `gpt-4o`, Foundry `gpt-4o`** | (a) Provider-cheapest model per provider; (b) Provider-best model per provider; (c) `gpt-4o-mini` everywhere | `gpt-4o` and `claude-sonnet-4-5` hit the SC-3 macro F1 >= 0.80 floor on the labelled fixture during prototyping; cheaper variants did not. Per-provider defaults can be overridden in YAML. |
| **No hard-delete in v1** | (a) Allow `--hard-delete` flag with confirmation; (b) Allow hard-delete only for messages older than retention | Self-hoster mistakes are unrecoverable. A v1 that cannot delete cannot lose data. The quarantine folder + retention + `sift purge --quarantine` is sufficient and reversible up to the purge boundary. |
| **Reference-keeper as a mandatory override, not advisory** | (a) Advisory hint to the LLM; (b) Post-LLM optional second pass | The cost of a wrong "delete" on an invoice is far higher than the cost of a wrong "keep". Mandatory override (chunk 2 6.6) makes invoice protection a property of the system, not of the LLM's confidence. |
| **Sender-cache thresholds: N >= 6 prior LLM classifications, all matching, confidence >= 0.9** | (a) N >= 3; (b) majority-vote; (c) configurable with no default floor | At N=6 with unanimity and high confidence, the empirical false-positive rate on the labelled fixture is < 0.5% and the LLM-call savings exceed 60%. Lower thresholds did not pass SC-4. |
| **Backfill checkpoints at batch boundary, not per message** | (a) Per-message checkpoint; (b) Time-based checkpoint (every 60s) | Per-message checkpoints quadruple SQLite write volume and slow throughput by ~30% on prototype. Batch-boundary checkpoints recover < 1 batch of duplicate work in the worst case (a single batch is 50 messages, ~30 seconds of work). |
| **Pydantic v2 strict mode (`strict=True`, `extra="forbid"`)** | (a) Strict validation only; (b) Default Pydantic v2 (lenient coercion) | Coercion masks user-config errors (e.g., `"3"` becoming `3` silently). Strict mode forces the user to spot the typo at validate time, not at runtime against Graph. Aligns with Solace section 4. |
| **`uv` for packaging, dependency management, and lockfile** | (a) `pip-tools`; (b) `poetry`; (c) `pdm` | `uv` resolves and installs faster than the alternatives, is a single static binary, produces a deterministic lockfile, and runs cleanly in CI and in the Dockerfile. |
| **Docker as the primary deployment path; `pipx` + systemd as the secondary** | (a) Docker only; (b) `pipx` only; (c) Native install via `apt` / `dnf` packages | Most self-hosters already run a container engine; Docker is the lowest-friction install. Power users who prefer native install get `pipx` + systemd. Native packages are deferred to v1.x. |
| **Sender hashed (HMAC-SHA-256) in the audit log, not plain text** | (a) Plain sender; (b) SHA-256 of sender (no key); (c) Truncated sender | Plain sender is PII. Plain SHA-256 is rainbow-table-able for common addresses. HMAC with a per-installation random key (stored in the keychain) defeats rainbow tables and still allows dedup within an installation (SC-14, SR-04). |
| **Single-writer SQLite via lock + connection-pool readers** | (a) Multi-writer Postgres; (b) Per-process file DB; (c) In-memory + WAL flush | The Sift workload is one CLI process at a time (lock file enforces it). WAL gives concurrent reads even during a long-running write transaction. The lock + WAL combination meets the single-host, single-user profile. |
| **Allow-no-providers permitted only on `sift config validate`** | (a) Always reject zero providers; (b) Permit zero providers globally | A user who has misplaced a provider key still needs to run `sift config validate` to find out what is wrong. Every other subcommand requires at least one configured provider; otherwise exits 3 (chunk 3a SR-14). |
| **PyPI distribution name: `sift-mail`** | (a) `sift365`; (b) `siftbox`; (c) `siftmail` | The unprefixed `sift` is taken on PyPI. `sift-mail` is the most direct description of the tool's domain (Microsoft 365 mail) and reads well in `pipx install sift-mail`. |
| **Include `sift schedule` in the v1 CLI surface** (resolves chunk 3b OQ #1) | (a) Defer scheduler CLI to v1.x and have the container run a self-loop of `sift triage --run --foreground` | The container deployment path (chunk 3b 10.1) needs an in-container scheduler today. A self-loop wrapper would re-implement apscheduler poorly. Adding `sift schedule` at v1 keeps the container path thin and gives bare-metal users a third deployment option (apscheduler in a foreground process) alongside cron and systemd timers. |
| **Container long-running scheduler AND systemd timer are both supported** (resolves chunk 3b OQ #2) | (a) Externalise scheduling out of the container too; (b) Drop the systemd timer in favour of the in-container scheduler | The two paths serve different operators (container users vs. bare-metal Linux server users) and the cost of supporting both is one extra CLI subcommand and one systemd unit pair. Removing either path would force a re-platforming on a sizeable fraction of users. |
| **Purge events recorded in a dedicated `purge_log` SQLite table, not by audit-log scan** (resolves chunk 2 OQ #4) | (a) Scan rotated audit-log files for purge events | Audit logs rotate (FR-39) and may be archived off-host. A scan-based check fails open after archival. A dedicated table is robust to log rotation and supports the undo refusal check in O(1). |
| **`reply_to_self` enum value reserved but not implemented in v1** (resolves chunk 2 OQ #1) | (a) Implement a sixth pipeline stage; (b) Drop the enum value | Building the detector right requires real-data calibration that v1 does not have time for. Reserving the value lets v1.x add the stage without an enum migration. |

---

## 16. Glossary

| Term | Definition |
|------|------------|
| **Sift** | The CLI tool specified by this document; v1 PyPI distribution name `sift-mail`. |
| **Vendored router** | The Project Solace `llm_router` package copied verbatim into `vendor/llm_router/` and pinned by snapshot SHA in `VENDORED.md` (chunk 1 SR-11). Sift code never modifies it. |
| **Adapter** | A `BaseAdapter` subclass in the vendored router that talks to one provider (Copilot, Claude, Azure OpenAI, Foundry). |
| **Dispatch** | The vendored entry point `dispatch.generate(...)` that selects an adapter and returns a `GenerateResponse`; imported as `from sift.vendor.llm_router import dispatch, models` (chunk 3b 9.2). |
| **Pipeline** | The ordered classification stages applied to every message: rule -> reference_keeper -> sender_cache -> LLM -> low_confidence_review (chunk 1 FR-09). |
| **Rule layer** | Regex / keyword / attachment matchers that may classify a message before any LLM call (chunk 1 FR-15 .. FR-17). |
| **Reference-keeper** | A mandatory override (chunk 2 6.6) that protects invoices, receipts, tax documents, contracts, and statements from being routed to To-Be-Deleted. |
| **Sender cache** | A short-circuit that re-uses a stable per-sender classification when N >= 6 prior LLM classifications agree at confidence >= 0.9 (chunk 2 6.4). |
| **Run** | One invocation of `sift triage`, `sift backfill`, `sift undo`, or `sift purge`; identified by a UUID and recorded in the `runs` table (chunk 1 FR-12). |
| **Action** | One Graph mutation performed during a run (typically a move); recorded in the `action_log` table with enough metadata to reverse it (chunk 1 FR-10). |
| **Action log** | The append-only record of every action; used by `sift undo` to compute reverse moves. |
| **Audit log** | The structured NDJSON event stream recording everything the run did at metadata level (no message bodies, no credentials); rotated per FR-39. |
| **Sender hash** | An HMAC-SHA-256 of a sender address using a per-installation random key (stored in the keychain). Used in the audit log instead of plain addresses (SC-14, SR-04). |
| **Subject hash** | A short HMAC of the subject used for dedup in the audit log; never the subject text itself. |
| **NULL-not-zero** | The pricing-unknown contract inherited from Solace FR-46: when no pricing entry exists, `cost_usd` is SQL NULL, never 0.0 (chunk 1 SC-12). |
| **Pricing-unknown warning** | The structured event emitted when a `(provider, model)` pair has no pricing row and no override; surfaces in the per-run summary as `cost_usd=null` (Solace FR-47). |
| **Pricing override** | The YAML file referenced by `pricing_overrides_path` whose rows take precedence over the vendored router's built-in pricing table (chunk 2 5.2; chunk 3b 9.2). |
| **`agent_name`** | A field on `UsageRecord` set by Sift's `CostPersistenceAdapter` to `"sift"` so cross-agent cost reporting in shared deployments can attribute usage (chunk 1 FR-26). |
| **Device-code flow** | The OAuth flow used for delegated Microsoft Graph access; user opens a URL on a separate device and enters a short code (chunk 1 FR-01). |
| **Backfill** | The CLI mode that processes existing mail in a configured received-from / received-to / max-messages window (chunk 1 FR-27 .. FR-30). |
| **Triage (incremental)** | The default CLI mode that processes only mail received since the last successful run (chunk 1 NFR-02). |
| **Dry-run** | A run that classifies and logs every targeted message but performs no Graph mutations (chunk 1 FR-11, SC-15). |
| **Quarantine folder** | The Outlook folder configured for messages routed to "delete"; messages stay here until purged. |
| **Retention** | The minimum age before a message in the quarantine folder is eligible for `sift purge --quarantine` (chunk 2 4.2.5). |
| **Undo** | The reversal of every action in a single run, identified by `--run-id` (chunk 1 FR-31 .. FR-33). |
| **Purge** | The deletion of messages from the quarantine folder that have exceeded retention; performed only by `sift purge --quarantine` with explicit confirmation (chunk 1 FR-36; chunk 3a SR-13). |
| **Lock file** | `$XDG_DATA_HOME/sift/run.lock`; held by the running Sift process to prevent overlapping mutating runs (chunk 2 6.10). |
| **Missed-fire policy** | The scheduler behaviour when scheduled fires accumulate during downtime; v1 uses coalesce-and-skip (chunk 3b 9.3). |
| **Structured event** | A single JSON object emitted to the audit log per significant pipeline event; vocabulary defined in chunk 2 5.1.16. |
| **Exit code** | The numeric process exit value; v1 uses 0 (success), 1 (generic / lock contention), 3 (config or provider misconfiguration), 5 (refused interactive operation; chunk 3a SR-13), 6 (cost-tracker persistence failure; chunk 3a), 7 (concurrency / quota overload; chunk 3a SR-13). |
| **XDG path** | The configuration / cache / state directories from the XDG Base Directory specification: `$XDG_CONFIG_HOME` (default `~/.config`), `$XDG_CACHE_HOME` (`~/.cache`), `$XDG_DATA_HOME` (`~/.local/share`); Sift uses `<base>/sift/` under each. |
| **Reference compose file** | `docker/docker-compose.yml`, the canonical container deployment (chunk 3b 10.1). |
| **`gho_*` token** | A GitHub OAuth bearer token belonging to the GitHub Copilot adapter; long-lived; SR-03 forbids on-disk plaintext outside `~/.cache/sift/copilot-token.json` mode 0600. |
| **Live test** | A `pytest -m live` test that runs against real Microsoft Graph and real provider APIs; opt-in via `SIFT_LIVE=1`; nightly job is non-blocking (chunk 3b 11). |
| **Vendored test suite** | The tests shipped inside `vendor/llm_router/tests/`; run unmodified by Sift CI as the contract that the snapshot still works (chunk 1 SC-9, sub-issue 1). |
