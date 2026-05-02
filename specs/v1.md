# Sift v1 Specification, Chunk 3b of 4: Integration Specifications, Deployment, Acceptance Criteria

This chunk writes Section 9 (Integration Specifications), Section 10 (Deployment), and Section 11 (Acceptance Criteria / Test Plan). It builds on identifiers established in `specs/v1-chunk1.md` (FR-NN, NFR-NN, SR-NN), `specs/v1-chunk2.md` (data models, CLI, configuration, component specifications), and `specs/v1-chunk3a.md` (deepened Section 7 Security Requirements with the new SR-13 and SR-14, plus the canonical exit codes 5, 6, 7).

Canonical exit codes used throughout this chunk (from chunk 3a, with chunk 1 / chunk 2 back-port agreed):

| Code | Meaning |
|-----:|---------|
| 0 | Success |
| 1 | User error or configuration validation error |
| 2 | Transient infrastructure error (Graph 5xx after retries, SQLite I/O, network unreachable) |
| 3 | Authentication or security mode failure at startup (FR-23, SR-01, SR-02, SR-03, SR-08, SR-14) |
| 4 | Throttled and could not complete on a single message (FR-04, 8 consecutive 429s) |
| 5 | Graceful no-TTY purge decline (FR-36) |
| 6 | Run ceiling breached (SR-13) |
| 7 | Keychain unavailable at startup (SR-01) |
| 64 | EX_USAGE (unknown subcommand, FR-34) |

---

## 9. Integration Specifications

This section defines every external interface Sift integrates with: the Microsoft Graph mail surface, the vendored Project Solace LLM router, the host scheduling subsystem, and the secrets backends. Each integration is anchored to identifiers from chunks 1, 2, and 3a.

### 9.1 Microsoft Graph

Sift v1 integrates with one Microsoft Graph endpoint family: the user-delegated mail surface (`/me/messages`, `/me/mailFolders`). All Graph traffic flows through the `GraphClient` component (chunk 2 section 5.3.2 and section 6.2). This section deepens the integration contract.

**Property table.**

| Property | Value |
|---|---|
| Auth | Delegated OAuth 2.0 device-code flow per FR-01; tokens stored per SR-01 |
| Scopes | `Mail.ReadWrite`, `MailboxSettings.Read`, `offline_access`, `User.Read` (SR-06 minimum set) |
| Endpoint base | `https://graph.microsoft.com/v1.0` (configurable for Azure sovereign clouds via `graph.base_url`) |
| Beta endpoint | NOT used in v1; SHALL be rejected by configuration validator |
| Rate limit policy | Throttle-aware: `Retry-After` always wins over computed backoff; per-host concurrency cap; hard 600-second cap on any single backoff window; abort on 8 consecutive 429s on the same message |
| Paging | `@odata.nextLink` cursor; `$top` defaults to 50, ceiling 999 (FR-03); paging state checkpointed at each page boundary (FR-06) |
| Selected fields (`$select`) | `id`, `parentFolderId`, `from`, `toRecipients`, `ccRecipients`, `subject`, `bodyPreview`, `hasAttachments`, `receivedDateTime`, `isRead`, `internetMessageHeaders` (filtered to `List-Unsubscribe`, `X-Mailer`, `Auto-Submitted`); never `body.content` to keep payloads small and out of the audit path (SR-05) |
| Attachment lookup | Performed only when `hasAttachments=true` AND a downstream stage (reference-keeper FR-16) requires content type; lookup uses `$select=name,contentType,size`; never downloads `contentBytes` |
| Move call | `POST /me/messages/{id}/move` with body `{"destinationId": "<folder_id>"}`; one HTTP call per message; batched only via the parallel concurrency pool, never via Graph `$batch` in v1 (open question carried from chunk 1) |
| Folder lookup | `GET /me/mailFolders/{id}/childFolders?$top=999` walked under the configured `folders.parent_path` (default `Triage`); folder IDs cached in the `folder_taxonomy` SQLite table; cache invalidated when a 404 on a known folder ID is observed |
| Folder creation | `POST /me/mailFolders/{parent_id}/childFolders` with `{"displayName": "<label>"}` for any label in `folders.label_to_folder_name` not present at startup (FR-08) |
| Default concurrency | 4 in-flight requests (`graph.concurrency`, FR-05) |
| Concurrency hard cap | 16; configuration values above 16 SHALL fail validation with exit code 1 (SR-13) |
| User-Agent | `sift/<version> (+https://github.com/stschulznz/sift)` on every request |
| Connect timeout | 10 seconds |
| Read timeout | `graph.request_timeout_seconds` (default 60) |

**Throttle-aware client.** The retry strategy SHALL be deterministic and inspectable from the structured log:

1. On `HTTP 429` or `HTTP 503`, the client SHALL inspect the `Retry-After` header.
2. When `Retry-After` is present and parseable (delta-seconds or HTTP-date), the wait SHALL be exactly that value plus a uniformly distributed jitter in `[0, 1.0)` seconds. The header value SHALL win over any computed backoff. A WARNING SHALL be emitted: `event=graph_retry_after_honored`, `wait_seconds`, `attempt`, `message_id` (when applicable).
3. When `Retry-After` is absent, the client SHALL apply exponential backoff with jitter at the schedule `1s, 2s, 4s, 8s, 16s, 32s` and SHALL apply a uniformly distributed jitter in `[0, current_step)`.
4. The total cumulative wait for a single Graph request SHALL be hard-capped at 600 seconds. When the next backoff would push cumulative wait past 600 seconds, the client SHALL stop retrying that request and surface a `GRAPH_RETRY_BUDGET_EXHAUSTED` error to the caller.
5. The per-message attempt counter SHALL be hard-capped at `max_attempts=8`. On the 9th attempt for the same `message_id`, Sift SHALL abort the run with exit code 4 (FR-04) and emit `event=graph_message_retry_exhausted` at ERROR with `message_id`, `attempts=8`, `last_status`, `last_retry_after`. The current run checkpoint (FR-06) SHALL be written before exit so the next `--resume` does not re-attempt the failing message without operator action.
6. Per-host concurrency SHALL be enforced by an `asyncio.Semaphore` sized to `graph.concurrency`. On `HTTP 429`, the semaphore SHALL transition to a temporary half-open state: only one request to the host SHALL be permitted to fly until a non-429 response is observed, after which the semaphore returns to its configured size.
7. `HTTP 5xx` responses other than 503 SHALL use the same exponential schedule as 4xx-throttled responses but SHALL NOT trigger the half-open state. Cumulative cap and per-message attempt cap apply identically.
8. Network-level errors (DNS, TCP reset, TLS handshake failure) SHALL count toward `max_attempts`. Persistent failure across all attempts SHALL exit with code 2 (transient infrastructure).

Cross-references: FR-04, FR-05, FR-06, NFR-07, SR-13.

### 9.2 Vendored LLM router

Sift v1 vendors the Project Solace router into `vendor/llm_router/` (chunk 1 Section 2 architecture, `VENDORED.md`). The vendored code runs unmodified at its public API surface (chunk 1 success criterion 9). This section defines how Sift wires into it.

**Import path.** Sift code SHALL import the router exclusively through:

```python
from sift.vendor.llm_router import dispatch, models
```

`sift.vendor.llm_router` SHALL be a thin re-export shim that imports from `vendor.llm_router.<original_module>`. No Sift module SHALL import from the `vendor.llm_router` package directly; this gives Sift a single seam for adding router instrumentation without modifying vendored files.

**Configuration injection at startup.** During Sift's startup (after configuration load, before any LLM dispatch), the Sift bootstrap SHALL inject three Sift-side adapters into the router:

1. `CostPersistenceAdapter`: implements the router's `UsageSink` protocol; on every `UsageRecord` emitted by the router, calls `StateStore.record_usage(usage_record)` (chunk 2 section 6.9). One `UsageRecord` per generation SHALL be persisted to the `usage_log` SQLite table. The adapter SHALL set `agent_name = "sift"` on every record before persistence (chunk 1 FR-26).
2. `PricingOverrideLoader`: when `pricing_overrides_path` is set in YAML (chunk 2 section 5.2), loads the YAML file at startup and merges its rows into the router's pricing table; rows in the override file take precedence over the router's built-in pricing.
3. `CopilotAuthCachePath`: passes `$XDG_CONFIG_HOME/sift/copilot-auth.json` (default `~/.config/sift/copilot-auth.json`) to the GitHub Copilot adapter as its writable cache location (SR-03). The adapter's read-only XDG auto-discovery (SR-03) is unchanged.

Injection SHALL be idempotent (re-running bootstrap in the same process SHALL replace, not stack, prior adapters). The router SHALL NOT be mutated globally; injection happens against a per-process router instance returned by `models.build_router(config)`.

**Per-provider credential plumbing via `LLMConfig`.** The Sift configuration loader SHALL construct an `LLMConfig` (chunk 2 section 4.2.3) and SHALL pass it to `models.build_router`. The router constructs each adapter with its own `ProviderConfig` slice and SHALL set per-adapter environment variables at adapter construction, never globally:

| Provider | Environment variables consumed |
|---|---|
| GitHub Copilot | None (XDG auto-discovery per SR-03 plus the Sift-managed `copilot-auth.json`) |
| Anthropic Claude | `ANTHROPIC_API_KEY` |
| Azure OpenAI | `AZURE_OPENAI_API_KEY`, `AZURE_OPENAI_ENDPOINT` (variable names configurable via `llm.azure_openai.api_key_env` and `endpoint_env`); deployment name resolved from `llm.azure_openai.deployment_map[<logical_model>]` |
| Microsoft Foundry | `FOUNDRY_TENANT_ID`, `FOUNDRY_CLIENT_ID`, `FOUNDRY_CLIENT_SECRET`, `FOUNDRY_ENDPOINT` (variable names configurable via the `*_env` fields on `MicrosoftFoundryProviderConfig`) |

Each adapter SHALL only read its own variables (SR-14). A shared "get any provider key" helper SHALL NOT exist.

**Health.** Sift SHALL call `router.health_check()`:

1. At startup, after adapter construction, before binding any subcommand that performs LLM dispatch (`triage`, `backfill`, `llm test`, `llm models`).
2. On every invocation of `sift llm models` (chunk 2 section 5.1.14).

The health check SHALL run against every constructed adapter in parallel, bounded by `llm.health_check_deadline_seconds` (default 4.0, Solace FR-51). Each adapter's result SHALL be one of `healthy`, `unhealthy`, `not_configured`. When ALL FOUR adapters return `not_configured` or `unhealthy`, Sift SHALL exit with code 3 and a remediation message that names the unhealthy state of each provider. The override `--allow-no-providers` SHALL be accepted ONLY by `sift config validate`; it SHALL be rejected by every other subcommand with exit code 1.

When the active provider per `llm.provider` is `unhealthy` or `not_configured`, Sift SHALL exit with code 3 regardless of whether other providers are healthy (no silent fallback in v1, NFR-08, SR-14).

**Cost capture.** Every dispatched generation SHALL yield exactly one `UsageRecord`, written to the `usage_log` SQLite table. The record SHALL contain at minimum: `usage_id` (UUID4), `run_id`, `provider`, `model`, `prompt_tokens`, `completion_tokens`, `cost_usd` (or `NULL` per Solace FR-46 and chunk 1 FR-26), `agent_name="sift"`, `created_at` (UTC ISO-8601). When the active model has no pricing entry in the router's pricing table (and no `pricing_overrides_path` row), the record SHALL persist with `cost_usd=NULL`; the router SHALL emit one `pricing_unknown` warning per `(provider, model)` per process to avoid log floods. Sift SHALL NOT coerce `NULL` to `0.00` at any layer; `sift status --json` SHALL render unknown costs as JSON `null`.

Cross-references: chunk 1 FR-21, FR-23, FR-26; chunk 1 success criteria 9 and 12; SR-02, SR-03, SR-04, SR-14; chunk 2 section 6.9 (cost tracker).

### 9.3 Scheduling

Sift v1 ships an in-process scheduler (`apscheduler`) plus runbook coverage for external cron and systemd timers (Section 10.3). All paths share the same lock-file primitive and the same missed-fire policy.

**Scheduler engine.** `apscheduler.schedulers.asyncio.AsyncIOScheduler` runs cron-style jobs derived from `SiftConfig.schedule` (a list of cron expressions, each binding to `sift triage --run` with optional argument overrides). The in-process scheduler is invoked by `sift schedule` (added to the v1 CLI surface; back-port note carried from chunk 2). When `sift schedule` is not used, external cron or `sift-triage.timer` (Section 10.3) call `sift triage --run` directly; the lock file ensures all three paths are mutually exclusive on a single host.

**Lock file.** `$XDG_DATA_HOME/sift/sift.lock` (default `~/.local/share/sift/sift.lock`), file mode 0600 (SR-08). Acquired with `fcntl.flock(fd, LOCK_EX | LOCK_NB)` on POSIX and `msvcrt.locking()` on Windows. Released on normal exit AND on `SIGINT` / `SIGTERM` via an installed signal handler.

Lock-file content (one JSON object, written under the lock):

```json
{
  "pid": 12345,
  "host": "sift-host-01",
  "started_at": "2026-04-30T13:22:05.142Z",
  "command": "sift triage --run --config /etc/sift/config.yaml",
  "run_id": "5b28e1ac-9f7e-4f2d-b8d9-6f0d5c54a7cc"
}
```

**Stale-lock detection.** On contention, the candidate process SHALL inspect the lock file. When `host` matches the local hostname AND the recorded `pid` is not running, the lock SHALL be reclaimed automatically with a WARNING `event=stale_lock_reclaimed`. When `host` does not match (e.g., shared filesystem), the candidate SHALL NOT reclaim and SHALL exit with code 1 and a message naming the recorded host and PID. When `host` matches and the recorded `pid` IS running, the candidate SHALL exit with code 1 and a message naming the holder.

**Missed-fire policy.** `apscheduler` SHALL be configured with `coalesce=True` and `misfire_grace_time=0`. When the host is asleep, paused, or otherwise unable to fire scheduled triage jobs for some interval, missed firings SHALL be discarded. Sift SHALL NEVER run multiple back-to-back triage jobs to "catch up" on missed runs. The next scheduled firing SHALL run normally; the operator can run `sift triage --since <timestamp>` manually if a catch-up is desired.

**Backfills.** Backfills SHALL be operator-initiated only (`sift backfill ...` from a shell or an explicit one-shot job). Backfills SHALL NEVER be scheduled in v1. The configuration loader SHALL reject any `schedule` entry whose argv contains `backfill` and SHALL exit with code 1.

Cross-references: FR-29, FR-30, FR-34, FR-35, NFR-12; SR-08; chunk 2 section 6.10.

### 9.4 Secrets backends

Sift uses one logical secret store with three real backends (selected at runtime by the host environment) and one explicit fallback. The Microsoft Graph token (FR-01, SR-01) and provider API keys (SR-02) flow through the same store; the GitHub Copilot cache (SR-03) is a separate file managed by the Copilot adapter.

**Backend table.**

| Backend | Selection | Library | Notes |
|---|---|---|---|
| macOS Keychain | Default on macOS when a graphical session is present | `keyring` (`keyring.backends.macOS.Keyring`) | Stored under service `sift`, account `<key_name>` |
| Windows Credential Manager | Default on Windows | `keyring` (`keyring.backends.Windows.WinVaultKeyring`) | Stored under target `sift:<key_name>` |
| Linux Secret Service | Default on Linux desktops with a running DBus session and a Secret Service provider (GNOME Keyring, KWallet) | `keyring` (`keyring.backends.SecretService.Keyring`) | Schema `sift.<key_name>` |
| File fallback | Selected when `secret_backend: "file"` in YAML, OR auto-selected on headless Linux when no Secret Service is reachable AND `secret_backend: "auto"` | Plain dotenv format | `~/.config/sift/secrets.env`, file mode 0600, parent 0700, both verified on every read (SR-01, SR-02, SR-08) |

Backend selection at startup follows this fixed order:

1. If `secret_backend` is set explicitly in YAML, use that backend; do not probe.
2. If `secret_backend == "auto"` (default), call `keyring.get_keyring()`. When the result is one of the three platform keyrings AND `keyring.get_password("sift", "_probe")` succeeds, use the keyring backend.
3. When the keyring probe fails (`KeyringError` or backend reports `keyring.errors.NoKeyringError`), fall through to the file backend at `~/.config/sift/secrets.env`.
4. When neither the keyring probe nor a writable 0600 file path is available, exit with code 7 (CONFIG_KEYCHAIN_UNAVAILABLE per chunk 3a) and a remediation message naming the keyring backend errors and the file-system error. Sift SHALL NOT silently downgrade to an insecure storage path (SR-01).

**Migration path between backends.** `sift auth migrate --from <backend> --to <backend>` SHALL be available in v1 to copy stored credentials between backends. The command SHALL read every Sift-managed key from the source backend, write each to the target backend, and (after successful write) delete from the source ONLY when `--delete-source` is provided. The default is non-destructive copy. Migration SHALL refuse to run when the source and target backends are identical.

**Redaction rules in `sift config show`.** The configured backend identifier SHALL be displayed (e.g. `secret_backend: keyring (Windows Credential Manager)`). Stored secret VALUES SHALL NEVER be displayed; per-key presence SHALL be displayed as `present` or `missing`. The `--reveal` flag SHALL NOT exist in v1. When invoked with `--show-paths`, the file fallback path SHALL be displayed only when the file backend is active. The structured logger SHALL apply the SR-04 redactor to every key value before any log line is emitted.

Cross-references: FR-01, FR-37; SR-01, SR-02, SR-03, SR-04, SR-08, SR-14; chunk 3a Section 7.

---

## 10. Deployment

This section defines the three v1 distribution paths (Docker / Podman as primary, pipx for individual users, systemd timers for Linux servers), every environment variable Sift reads, and the canonical filesystem layout. All three paths share the same configuration model (chunk 2 Section 4.2 and Section 5.2) and the same secrets backends (Section 9.4).

### 10.1 Docker / Podman (primary)

Sift v1 ships an OCI image as the primary distribution channel. The image runs identically under Docker and Podman.

**`Dockerfile` requirements.**

- Base image: `python:3.13-slim` (pinned by digest in the released image; floating tag in the repository `Dockerfile` for ergonomic local builds).
- Dedicated non-root user: `sift` (UID 10001, GID 10001), home directory `/home/sift`.
- Working directory: `/app`.
- Copy `vendor/llm_router/` BEFORE `pip install -e .` so the install discovers the vendored package in its declared path (chunk 1 Section 2 architecture).
- Install with `pip install --no-cache-dir -e .[deploy]`. The `deploy` extra includes `apscheduler`, `keyring`, `cryptography`, the four LLM provider HTTP clients, and `gunicorn` is NOT installed (no HTTP surface in v1).
- `ENTRYPOINT ["sift"]` and `CMD ["status", "--json"]`. Operators override `CMD` to `["schedule"]` for the long-running scheduler container or to `["triage", "--run"]` for one-shot containers.
- `HEALTHCHECK --interval=60s --timeout=10s --start-period=30s --retries=3 CMD sift status --json | jq -e '.last_run.status != "failed"'`. The healthcheck SHALL NOT make outbound network calls. It reads only the SQLite state store. Containers without `jq` installed SHALL fail their build; the `Dockerfile` SHALL `apt-get install -y --no-install-recommends jq`.
- Multi-arch build: `docker buildx build --platform linux/amd64,linux/arm64`. The release pipeline pushes both arches under the same tag.
- Image labels per OCI annotations: `org.opencontainers.image.title`, `image.source`, `image.version`, `image.licenses=Apache-2.0`.
- File modes preserved (`COPY --chown=sift:sift`, then `chmod 0644` on YAML, `chmod 0600` on any seeded secret files at image-time).

**`docker-compose.yml` reference.**

```yaml
services:
  sift:
    image: ghcr.io/stschulznz/sift:0.1.0
    container_name: sift
    restart: unless-stopped
    user: "10001:10001"
    command: ["schedule"]
    environment:
      - SIFT_CONFIG=/config/config.yaml
      - SIFT_LOG_FORMAT=json
      - TZ=UTC
    env_file:
      - ./sift.env
    volumes:
      - ./config/config.yaml:/config/config.yaml:ro
      - sift-data:/home/sift/.local/share/sift
      - sift-cfg:/home/sift/.config/sift
      - sift-state:/home/sift/.local/state/sift
    healthcheck:
      test: ["CMD-SHELL", "sift status --json | jq -e '.last_run.status != \"failed\"'"]
      interval: 60s
      timeout: 10s
      start_period: 30s
      retries: 3

volumes:
  sift-data:
  sift-cfg:
  sift-state:
```

The compose file SHALL declare named volumes (not host bind mounts) for the SQLite database, the secrets / token cache, and the structured log directory. The YAML config is bind-mounted read-only. The `sift.env` file holds provider environment variables (per Section 10.4) and SHALL be at host file mode 0600.

**Podman parity.** The same image and the same compose file run under `podman compose up` without modification. The `user: "10001:10001"` mapping SHALL work under rootless Podman; volume ownership SHALL be handled with `:U` mount flags when bind mounts are used (`./config/config.yaml:/config/config.yaml:ro,U`). Documentation in `docs/operations.md` SHALL provide the exact podman command lines.

**No outbound healthcheck.** The healthcheck SHALL NOT call Microsoft Graph, SHALL NOT call any LLM provider, and SHALL NOT consume LLM tokens. It SHALL be safe to fire every 60 seconds for the container's lifetime (NFR-09).

### 10.2 pipx

`pipx` is the supported path for individual users on a workstation who prefer not to run a container.

- Install: `pipx install sift`.
- The release artefact is published to PyPI as `sift-mail` (since `sift` is taken on PyPI; the CLI command remains `sift`); install command in user docs is `pipx install sift-mail` and the CLI invocation is `sift`.
- First-run flow: `sift auth login` performs the device-code flow (FR-01) and writes the Graph token to the configured backend (Section 9.4).
- Cache and config locations follow XDG: `$XDG_CONFIG_HOME/sift/` (default `~/.config/sift/`), `$XDG_DATA_HOME/sift/` (default `~/.local/share/sift/`), `$XDG_STATE_HOME/sift/` (default `~/.local/state/sift/`).
- Upgrade: `pipx upgrade sift-mail`. The vendored router travels with the package (chunk 1 Section 2); upgrades SHALL NOT require a separate router install.

### 10.3 systemd unit (Linux servers)

For Linux servers that prefer a systemd-managed deployment over a container, Sift ships a unit file plus a paired timer.

**`sift.service` (oneshot, invoked by the timer).**

```ini
[Unit]
Description=Sift Microsoft 365 inbox triage (oneshot)
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
User=sift
Group=sift
WorkingDirectory=/var/lib/sift
ExecStart=/usr/bin/sift triage --run --config /etc/sift/config.yaml

# Hardening
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=/var/lib/sift /var/log/sift
NoNewPrivileges=true
PrivateTmp=true
PrivateDevices=true
ProtectKernelTunables=true
ProtectKernelModules=true
ProtectControlGroups=true
RestrictAddressFamilies=AF_INET AF_INET6 AF_UNIX
LockPersonality=true
MemoryDenyWriteExecute=true
RestrictNamespaces=true
SystemCallArchitectures=native
SystemCallFilter=@system-service
SystemCallFilter=~@privileged @resources

# Environment
EnvironmentFile=/etc/sift/sift.env
Environment=SIFT_LOG_FORMAT=json
Environment=TZ=UTC
StandardOutput=journal
StandardError=journal
```

**`sift-triage.timer` (paired timer).**

```ini
[Unit]
Description=Run Sift triage on a schedule
Requires=sift.service

[Timer]
OnCalendar=*:0/15
RandomizedDelaySec=30s
Persistent=false
Unit=sift.service
AccuracySec=10s

[Install]
WantedBy=timers.target
```

The timer SHALL pair with the oneshot service so that overlapping invocations are impossible at the systemd level (the unit refuses to start while a prior instance is active). `Persistent=false` SHALL be used so that missed firings during downtime are discarded, matching the Section 9.3 missed-fire policy. The Sift lock file (Section 9.3) provides the cross-distribution backstop.

**Dedicated user.** The package install SHALL create a system user `sift` (no shell, no home-directory shell access; `useradd --system --no-create-home --shell /usr/sbin/nologin sift`). Configuration directory `/etc/sift/` SHALL be owned by `root:sift` mode `0750`; data directory `/var/lib/sift/` SHALL be owned by `sift:sift` mode `0750`; log directory `/var/log/sift/` SHALL be owned by `sift:sift` mode `0750`. The systemd unit SHALL refuse to start when any of these modes are wider than declared.

**`ReadWritePaths` discipline.** `ReadWritePaths` SHALL be limited to `/var/lib/sift` and `/var/log/sift`. Any deployment that needs to write elsewhere (e.g., a non-default `state_db_path`) SHALL extend `ReadWritePaths` via a drop-in unit; the packaged unit SHALL NOT relax the default.

### 10.4 Environment variables

Every environment variable Sift reads, its default, and the configuration field it overrides.

**Sift-native variables.**

| Variable | Default | Overrides |
|---|---|---|
| `SIFT_CONFIG` | `~/.config/sift/config.yaml` then `./sift.yaml` | Config file path; CLI `--config` wins over this |
| `SIFT_LOG_FORMAT` | `json` | Top-level `log_format` (`json` or `text`); JSON SHALL be the default in container and systemd deployments |
| `SIFT_LOG_LEVEL` | `INFO` | Top-level `log_level` (one of `DEBUG`, `INFO`, `WARNING`, `ERROR`) |
| `SIFT_DATA_DIR` | `$XDG_DATA_HOME/sift` then `~/.local/share/sift` | Base for `state_db_path`, lock file, audit log when those are not absolute |
| `SIFT_CONFIG_DIR` | `$XDG_CONFIG_HOME/sift` then `~/.config/sift` | Base for `secrets.env`, `copilot-auth.json` |
| `SIFT_STATE_DIR` | `$XDG_STATE_HOME/sift` then `~/.local/state/sift` | Base for the structured log file when `log_to_file=true` |
| `SIFT_DRY_RUN` | unset | When set to `1`, every mutating subcommand SHALL behave as if `--dry-run` were passed (FR-11); the run summary SHALL note that `SIFT_DRY_RUN` was active |
| `SIFT_ALLOW_NO_PROVIDERS` | unset | When set to `1`, equivalent to `--allow-no-providers`; only honoured by `sift config validate` (Section 9.2) |

**Microsoft Graph.**

| Variable | Default | Overrides |
|---|---|---|
| `SIFT_GRAPH_TENANT_ID` | `common` | `graph.tenant_id` |
| `SIFT_GRAPH_CLIENT_ID` | unset (REQUIRED) | `graph.client_id`; the configuration loader SHALL exit code 1 when neither YAML nor env supplies a value |
| `SIFT_GRAPH_BASE_URL` | `https://graph.microsoft.com/v1.0` | `graph.base_url` (intended for sovereign clouds) |
| `SIFT_GRAPH_CONCURRENCY` | `4` | `graph.concurrency` (FR-05); SHALL be rejected with exit code 1 when > 16 (SR-13) |

**Per-provider variables (consumed by the vendored router via Section 9.2 plumbing).**

| Provider | Variable | Notes |
|---|---|---|
| GitHub Copilot | None (XDG auto-discovery; SR-03) | The Sift-managed cache lives at `$XDG_CONFIG_HOME/sift/copilot-auth.json` mode 0600 |
| Anthropic Claude | `ANTHROPIC_API_KEY` | Required when `llm.provider == anthropic` |
| Azure OpenAI | `AZURE_OPENAI_API_KEY`, `AZURE_OPENAI_ENDPOINT` | Variable names configurable via `llm.azure_openai.api_key_env` / `endpoint_env`; deployment names live in YAML `llm.azure_openai.deployment_map` (chunk 2 section 4.2.3) |
| Microsoft Foundry | `FOUNDRY_TENANT_ID`, `FOUNDRY_CLIENT_ID`, `FOUNDRY_CLIENT_SECRET`, `FOUNDRY_ENDPOINT` | Variable names configurable via the `*_env` fields on `MicrosoftFoundryProviderConfig` |

YAML SHALL NOT carry secret values for any provider (SR-02). The configuration loader SHALL reject any YAML document that includes a key matching `*_api_key`, `*_secret`, `*_token`, or `*_password` at any nesting depth (chunk 3a SR-02).

**Container-only.**

| Variable | Default | Notes |
|---|---|---|
| `TZ` | `UTC` | Container time zone; Sift's structured timestamps are always UTC regardless |
| `PYTHONUNBUFFERED` | `1` (set in image) | Ensures journal / docker logs emit lines as written |

### 10.5 Filesystem layout

Canonical paths under XDG resolution. Container, pipx, and systemd deployments share the same layout (the systemd unit overrides via `SIFT_DATA_DIR` etc. to use `/var/lib/sift` and `/var/log/sift`).

| Path | Purpose | Mode |
|---|---|---|
| `$XDG_CONFIG_HOME/sift/config.yaml` | YAML config (chunk 2 Section 5.2) | 0644 |
| `$XDG_CONFIG_HOME/sift/secrets.env` | Fallback secrets store (Section 9.4) | 0600 |
| `$XDG_CONFIG_HOME/sift/copilot-auth.json` | GitHub Copilot `gho_*` cache (SR-03) | 0600 |
| `$XDG_DATA_HOME/sift/sift.db` | SQLite state store (FR-06, NFR-06, SR-08) | 0600 |
| `$XDG_DATA_HOME/sift/sift.lock` | Run lock file (Section 9.3) | 0600 |
| `$XDG_STATE_HOME/sift/sift.log` | Structured JSON log, rotated 10 files of 10 MB each | 0600 |
| `$XDG_DATA_HOME/sift/audit.log` | Audit log (FR-38, FR-39); separate rotation 5 files of 50 MB each | 0600 |
| `$XDG_CONFIG_HOME/sift/` (parent) | Config directory | 0700 |
| `$XDG_DATA_HOME/sift/` (parent) | Data directory | 0700 |
| `$XDG_STATE_HOME/sift/` (parent) | State directory | 0700 |

Sift SHALL refuse to start (exit code 3) when any of the above files exist at a wider mode than the declared mode, or when any of the parent directories exist at a wider mode than 0700 (chunk 3a FILE_MODE_TOO_WIDE). The container image SHALL ship with these directories pre-created at the correct modes.

Cross-references: chunk 1 Section 2 (architecture), FR-37, NFR-12; SR-01, SR-02, SR-03, SR-08; chunk 2 section 5.2; chunk 3a Section 7.

---

## 11. Acceptance Criteria (Test Plan)

This section enumerates the executable acceptance tests for v1. Each test is written in Given / When / Then form with concrete fixture values. Tests are grouped by component. Coverage SHALL satisfy NFR-10 (overall test coverage targets) and SHALL include every test mandated in the chunk 3b brief.

The full test suite SHALL be executed in CI on every PR. Tests that touch live external services (Graph, LLM providers) SHALL be marked `@pytest.mark.live` and SHALL be skipped in default CI runs; a nightly job SHALL run the live suite against a dedicated test tenant and a dedicated provider account.

### 11.1 Microsoft Graph throttle compliance

**AC-GRAPH-01: `Retry-After` honoured exactly.**
- Given a Graph fixture that returns `HTTP 429` with `Retry-After: 5` on the first call to `POST /me/messages/{id}/move` and `HTTP 200` on the second.
- When `GraphClient.move_message(message_id, folder_id)` is invoked.
- Then the second call SHALL fire after a wall-clock delay of at least 5.0 seconds and at most 6.0 seconds (5s plus the upper bound of the `[0, 1.0)` jitter), AND a structured event `event=graph_retry_after_honored` with `wait_seconds` between 5.0 and 6.0 SHALL be present in the captured log stream.

**AC-GRAPH-02: Exponential backoff when no `Retry-After`.**
- Given a Graph fixture that returns `HTTP 429` without `Retry-After` on the first three calls and `HTTP 200` on the fourth.
- When `GraphClient.move_message` is invoked.
- Then the recorded inter-call delays SHALL fall in `[1.0, 2.0)`, `[2.0, 4.0)`, `[4.0, 8.0)` seconds respectively (per the schedule plus the per-step jitter).

**AC-GRAPH-03: 8 consecutive 429s on one message exits with code 4.**
- Given a Graph fixture that returns `HTTP 429` (no `Retry-After`) for every call to `POST /me/messages/abc/move`.
- When `sift triage --run --message-id abc` is executed.
- Then the process SHALL exit with code 4 within `1+2+4+8+16+32+64+128 = 255` seconds plus jitter overhead, AND a structured event `event=graph_message_retry_exhausted` with `message_id="abc"` and `attempts=8` SHALL be present, AND the run checkpoint SHALL be written before exit.

**AC-GRAPH-04: 600-second cumulative cap.**
- Given a Graph fixture that returns `HTTP 503` with `Retry-After: 700` on a `GET /me/messages` call.
- When the client attempts the request.
- Then the client SHALL emit `event=graph_retry_budget_exhausted` and SHALL surface a `GRAPH_RETRY_BUDGET_EXHAUSTED` error to the caller, AND total wall-clock wait SHALL NOT exceed 601 seconds.

**AC-GRAPH-05: Concurrency cap enforced.**
- Given `graph.concurrency=4` and 100 queued move operations.
- When the client dispatches them in parallel.
- Then at no observed instant SHALL more than 4 in-flight HTTP requests to the Graph host be open (verified via the test transport's per-call instrumentation).

**AC-GRAPH-06: Half-open semaphore on 429.**
- Given `graph.concurrency=4` and a fixture that returns `HTTP 429` on the next request.
- When the response arrives.
- Then the next dispatched request SHALL be the only in-flight request to the host until a non-429 response is observed, after which concurrency SHALL return to 4.

### 11.2 LLM providers (smoke, missing-key, unreachable)

For each of the four v1 providers (GitHub Copilot, Anthropic Claude, Azure OpenAI, Microsoft Foundry), the same triplet of tests SHALL pass. Tests are tagged by provider; pseudonyms below.

**AC-LLM-{PROVIDER}-SMOKE: configured + reachable.**
- Given valid credentials for `{PROVIDER}` and network reachability to its endpoint.
- When `sift llm test --provider {PROVIDER}` is executed.
- Then the command SHALL exit with code 0 AND `router.health_check()` SHALL return `healthy` for `{PROVIDER}` AND a `UsageRecord` SHALL be persisted with `provider="{PROVIDER}"`, `agent_name="sift"`, and a non-null token count.

**AC-LLM-{PROVIDER}-MISSING: missing key returns PROVIDER_NOT_CONFIGURED.**
- Given an environment with all `{PROVIDER}` environment variables unset.
- When `router.health_check()` is invoked.
- Then `{PROVIDER}` SHALL be reported as `not_configured` AND a single INFO log line SHALL name the missing environment variable(s) (by name, never value, per SR-02).

**AC-LLM-{PROVIDER}-UNREACHABLE: unreachable endpoint returns PROVIDER_UNAVAILABLE.**
- Given valid credentials but a network fixture that returns connection refused for the `{PROVIDER}` endpoint.
- When `router.health_check()` is invoked.
- Then `{PROVIDER}` SHALL be reported as `unhealthy` AND a single WARNING log line SHALL include the endpoint host and the network-level error class.

**AC-LLM-ALL-FAIL-EXIT-3: all four unreachable at startup.**
- Given a configuration that nominates all four providers but where every provider returns `unhealthy` from `health_check()`.
- When `sift triage --run` is executed without `--allow-no-providers`.
- Then the process SHALL exit with code 3 AND the stderr message SHALL name the unhealthy state of each provider.

**AC-LLM-ALLOW-NO-PROVIDERS-SCOPE.**
- Given the same all-unhealthy state.
- When `sift triage --run --allow-no-providers` is executed.
- Then the process SHALL exit with code 1 (the override SHALL be rejected by `triage`).
- And: `sift config validate --allow-no-providers` SHALL exit 0 in the same environment (the override SHALL be accepted by `config validate` only).

### 11.3 Pipeline ordering and rule layer

**AC-PIPE-01: Sender on deny-list routes to To-Be-Deleted regardless of LLM verdict.**
- Given a rule layer with `deny_list: ["@spam.example"]` and an inbox message from `bot@spam.example`.
- And a fixture LLM that, if asked, would return classification `Action` with confidence `0.99`.
- When `sift triage --run` processes the message.
- Then the recorded classification SHALL have `target="To-Be-Deleted"` and `source="rule"` (FR-09 stage 1), AND the LLM SHALL NOT be invoked (verified by zero `UsageRecord` rows for that `message_id`).

**AC-PIPE-02: PDF attachment + subject "invoice" lands in Reference even when LLM proposed To-Be-Deleted.**
- Given a message with `hasAttachments=true`, attachment `contentType="application/pdf"`, and subject `"April invoice from Acme"`.
- And a fixture LLM that returns classification `To-Be-Deleted` with confidence `0.95`.
- When `sift triage --run` processes the message.
- Then the recorded classification SHALL have `target="Reference"` and `source="reference_keeper"` (FR-09 stage 2), AND a structured event `event=reference_keeper_override` with `original_llm_target="To-Be-Deleted"` SHALL be emitted.

### 11.4 Sender cache shortcut

**AC-CACHE-01: 7th from same sender skips LLM after 6 consecutive Newsletter classifications at >= 0.9.**
- Given a sender `news@example.org` with six prior Sift records all classified `Newsletters` at confidence >= 0.9 (FR-18, sender cache write).
- When the 7th message from `news@example.org` is processed.
- Then the recorded classification SHALL have `source="sender_cache"`, `target="Newsletters"`, `confidence=0.99` (the capped sender-cache confidence per chunk 2 section 4.2.7), AND zero `UsageRecord` rows SHALL be added for that `message_id` (FR-19).

**AC-CACHE-02: Sender cache invalidation resets the consecutive counter.**
- Given the AC-CACHE-01 state plus the 7th message having been moved by sender-cache.
- When `sift undo --run-id <run_of_7th>` is executed.
- Then the sender cache row for `news@example.org` SHALL show `consecutive_count` reset by one (i.e., reverted to the count prior to the 7th message), AND a subsequent 7th message SHALL re-invoke the LLM (FR-20).

### 11.5 Backfill resumability

**AC-BACKFILL-01: Kill mid-batch then `--resume RUN_ID` reprocesses at most one batch.**
- Given a backfill `RUN_ID` of 1000 messages with `batch_size=200` (chunk 2 section 4.2.8).
- And the process is killed (`SIGKILL`) midway through batch 3 (after 450 messages have been moved, partway through the 5th batch's classifications).
- When `sift backfill --resume RUN_ID` is executed.
- Then the resumed run SHALL re-classify at most 200 messages (one batch worth, the partially-completed batch) AND SHALL NOT re-classify messages from batches 1 and 2 (verified by zero new `UsageRecord` rows for those `message_id`s) AND SHALL complete the remaining 800-or-so messages.

**AC-BACKFILL-02: No message classified twice across resume.**
- Given the same setup as AC-BACKFILL-01.
- When the resumed run completes.
- Then for every `message_id` in the run, the count of `UsageRecord` rows SHALL be 1 (or 0 for messages handled by rule, reference-keeper, or sender-cache shortcuts), with at most 200 messages potentially having 2 records (the in-flight batch at kill time).

**AC-BACKFILL-03: No message skipped across resume.**
- Given the same setup.
- When the resumed run completes.
- Then for every `message_id` in the original 1000, exactly one `processing_record` row SHALL exist with a non-null `target_folder_id` AND every Graph move call SHALL have either succeeded or be present in the failure log.

### 11.6 Undo

**AC-UNDO-01: Every move recorded in run R is reverted.**
- Given a completed `RUN_ID=R` with 50 successful moves recorded in `processing_record`.
- When `sift undo --run-id R` is executed.
- Then for each of the 50 messages, a Graph move call SHALL be issued from `new_folder_id` back to `previous_folder_id` (verified by Graph mock call recording) AND the run summary SHALL report `reverted=50, skipped=0, failed=0`.

**AC-UNDO-02: Idempotent on second invocation.**
- Given AC-UNDO-01 has completed.
- When `sift undo --run-id R` is executed a second time.
- Then the command SHALL exit with code 0 AND the run summary SHALL report `reverted=0, skipped=50, failed=0` (every message is already in its original folder; nothing to revert) AND zero new Graph move calls SHALL be issued.

**AC-UNDO-03: Manually moved message is reported and skipped, not errored.**
- Given a completed `RUN_ID=R` containing one message `msg-X` that the user has since moved manually to a folder OTHER THAN the Sift target.
- When `sift undo --run-id R` is executed.
- Then `msg-X` SHALL be reported as `skipped` with reason `"manually_moved"` AND the process SHALL exit with code 0 (or code 5 only if other operations failed, NOT for a skipped message alone).

### 11.7 Quarantine retention and purge

**AC-PURGE-01: `--dry-run` lists messages older than `retention_days`.**
- Given a quarantine folder with 3 messages aged 5 days, 35 days, and 100 days, and `retention.quarantine_retention_days=30`.
- When `sift purge --quarantine --dry-run` is executed.
- Then the output SHALL list the 35-day and 100-day messages by `message_id` and `received_at` AND the 5-day message SHALL NOT be listed AND zero Graph move calls SHALL be issued AND zero Graph DELETE calls SHALL be issued.

**AC-PURGE-02: Real purge requires interactive confirmation.**
- Given the same quarantine state.
- And a TTY is attached AND `--yes` is NOT supplied.
- When `sift purge --quarantine` is executed and the prompt `Permanently remove 2 messages from quarantine? [y/N]` is answered `y`.
- Then exactly 2 messages SHALL be removed via `POST /me/messages/{id}/move` to the deleted-items folder (FR-36 specifies move-to-DeletedItems, NOT hard delete) AND the audit log SHALL contain one entry per removal naming the operator's response.

**AC-PURGE-03: No-TTY without `--yes` exits cleanly with code 5.**
- Given the same quarantine state.
- And no TTY is attached (e.g., container `docker exec` or systemd) AND `--yes` is NOT supplied.
- When `sift purge --quarantine` is executed.
- Then the process SHALL exit with code 5 (chunk 3a graceful no-TTY purge decline) AND zero Graph move or DELETE calls SHALL be issued AND a structured event `event=purge_declined_no_tty` SHALL be emitted.

### 11.8 Hard-delete safety

**AC-HARDDELETE-01: Static check prohibits `DELETE /me/messages/`.**
- Given the Sift source tree (excluding `vendor/` and `tests/fixtures/`).
- When the static check `ripgrep '"DELETE.*messages|client\.delete\("/me/messages|graph_delete'` is executed.
- Then zero matches SHALL be returned. CI SHALL fail the build on any match.

**AC-HARDDELETE-02: `sift triage` issues zero DELETE calls in integration.**
- Given a Graph mock that records every HTTP method and path it receives.
- When `sift triage --run` processes a 100-message fixture inbox containing senders matched by deny-list, by reference-keeper PDFs, by sender-cache, and by LLM verdicts including `To-Be-Deleted`.
- Then the count of HTTP requests with method `DELETE` against any path matching `/me/messages` SHALL be exactly 0.

**AC-HARDDELETE-03: `sift purge` issues zero DELETE calls (move-to-DeletedItems only).**
- Given AC-PURGE-02's confirmed-purge scenario.
- When the purge runs against the recording Graph mock.
- Then the count of HTTP requests with method `DELETE` against any path matching `/me/messages` SHALL be exactly 0.

### 11.9 Cost tracking and usage records

**AC-COST-01: NULL-not-zero when pricing is unknown.**
- Given an active model `gpt-experimental-x` that has no row in the router's pricing table and no `pricing_overrides_path` row.
- When `sift triage --run` makes one generation against that model.
- Then the persisted `UsageRecord` SHALL have `cost_usd IS NULL` (verified by `SELECT cost_usd FROM usage_log WHERE usage_id = ?`) AND exactly one WARNING log line `event=pricing_unknown` with `provider` and `model` SHALL be emitted in the process; subsequent generations against the same `(provider, model)` in the same process SHALL NOT emit a duplicate warning.

**AC-COST-02: `sift status --json` renders unknown cost as JSON `null`.**
- Given the AC-COST-01 state.
- When `sift status --json --run-id <RUN>` is executed.
- Then the JSON output SHALL contain `"cost_usd": null` for the unknown-pricing record AND SHALL NOT contain `"cost_usd": 0` or `"cost_usd": 0.0`.

**AC-COST-03: `agent_name="sift"` on every record.**
- Given any completed Sift run with at least one LLM generation.
- When the test queries `SELECT DISTINCT agent_name FROM usage_log WHERE run_id = ?`.
- Then the result SHALL be exactly `[("sift",)]` with no other values.

**AC-COST-04: One UsageRecord per generation.**
- Given a triage run that processes 50 messages where 20 reach the LLM stage.
- When the run completes.
- Then the count of `usage_log` rows for the run SHALL equal 20 (one per generation) AND no row SHALL have a `NULL` `usage_id`, `provider`, or `model`.

### 11.10 Configuration validation

**AC-CFG-01: YAML with unknown provider name fails with a clear Pydantic error.**
- Given a YAML file with `llm.provider: "openai-but-not-azure"`.
- When `sift config validate --config <path>` is executed.
- Then the process SHALL exit with code 1 AND stderr SHALL contain a Pydantic validation error naming `llm.provider` and listing the four valid values (`github_copilot`, `anthropic`, `azure_openai`, `microsoft_foundry`).

**AC-CFG-02: YAML containing API key fails with SR-02 error.**
- Given a YAML file containing `llm.claude.api_key: "sk-ant-..."`.
- When `sift config validate` is executed.
- Then the process SHALL exit with code 1 AND stderr SHALL contain the message `"API keys SHALL NOT be present in YAML"` and a remediation hint pointing to the `ANTHROPIC_API_KEY` environment variable.

**AC-CFG-03: `graph.concurrency=20` rejected.**
- Given a YAML file with `graph.concurrency: 20`.
- When `sift config validate` is executed.
- Then the process SHALL exit with code 1 AND stderr SHALL name the SR-13 ceiling of 16.

**AC-CFG-04: Unknown nested key rejected.**
- Given a YAML file with `llm.claud.api_base: "https://typo.example"` (typo in `claude`).
- When `sift config validate` is executed.
- Then the process SHALL exit with code 1 due to `extra="forbid"` on `LLMConfig` (chunk 2 section 4.2.3).

### 11.11 Audit log content

**AC-AUDIT-01: Required fields present.**
- Given a completed run with at least one moved message.
- When the test reads the most recent audit log entry for that run.
- Then the entry SHALL contain (as JSON keys): `run_id`, `message_id`, `previous_folder`, `new_folder`, `classification.source`, `classification.confidence`, `decided_at`.

**AC-AUDIT-02: Forbidden fields absent.**
- Given the same audit entry.
- When the entry is parsed.
- Then it SHALL NOT contain `subject` AND SHALL NOT contain `body` AND SHALL NOT contain a plaintext `sender_address` field; it MAY contain `sender_hash` (SHA-256 over the lower-cased sender address).

**AC-AUDIT-03: Audit log scanner finds zero credentials in a full backfill.**
- Given a completed 1000-message backfill against a fixture inbox.
- When `gitleaks --no-git --source <audit_log_path>` is executed.
- Then the scanner SHALL report zero matches.

**AC-AUDIT-04: Audit log rotation.**
- Given an audit log file at `audit_max_bytes=52428800` (50 MiB).
- When the next entry would push the file past 50 MiB.
- Then the current file SHALL be rotated to `audit.log.1` AND a fresh `audit.log` SHALL be created at mode 0600 AND no prior entries SHALL be overwritten in-place.

### 11.12 Scheduling and locking

**AC-SCHED-01: Lock contention exits with code 1 on a same-host live PID.**
- Given a `sift triage --run` already holding the lock with a live PID.
- When a second `sift triage --run` is launched on the same host.
- Then the second invocation SHALL exit with code 1 within 2 seconds AND stderr SHALL name the holding PID and `started_at`.

**AC-SCHED-02: Stale-lock auto-reclaim on dead local PID.**
- Given a lock file naming a PID that is not running on the local host AND `host` matching the local hostname.
- When `sift triage --run` is launched.
- Then the lock SHALL be reclaimed AND a WARNING `event=stale_lock_reclaimed` with the dead `pid` SHALL be emitted AND the run SHALL proceed normally.

**AC-SCHED-03: Lock not reclaimed across hosts.**
- Given a lock file whose `host` does not match the local hostname.
- When `sift triage --run` is launched.
- Then the process SHALL exit with code 1 AND the lock file SHALL NOT be modified.

**AC-SCHED-04: Missed firings discarded, no catch-up.**
- Given an `apscheduler` cron that should have fired 4 times during a host pause window.
- When the host resumes.
- Then exactly 1 triage run SHALL fire at the next scheduled time AND zero "catch-up" runs SHALL be queued.

### 11.13 Deployment smoke

**AC-DEPLOY-01: Container healthcheck passes after first successful run.**
- Given the Sift container started with the reference compose file and a state DB whose `last_run.status` is `success`.
- When the Docker healthcheck command is invoked.
- Then it SHALL exit with code 0 AND SHALL NOT make any outbound network call (verified by attaching `tcpdump` filtered on the container's network namespace and asserting zero outbound packets during the check).

**AC-DEPLOY-02: systemd unit refuses overlap.**
- Given the `sift.service` oneshot is currently active.
- When `systemctl start sift.service` is invoked again.
- Then systemd SHALL refuse to start the second invocation AND the journal SHALL contain the rejection message.

**AC-DEPLOY-03: pipx install + first-run wizard.**
- Given a clean user account with `pipx` installed.
- When `pipx install sift-mail` then `sift auth login --device-code` is executed and the device-code flow is completed.
- Then `sift status` SHALL exit with code 0 AND the secrets backend SHALL contain a Graph access token AND `~/.config/sift/copilot-auth.json` SHALL exist at mode 0600 (or be absent when the active provider is not `github_copilot`).

### 11.14 Test data and fixtures

The following fixture artefacts SHALL ship in `tests/fixtures/`:

- `inbox-100-mixed.json`: 100 Graph-shaped message records covering newsletters, notifications, invoices (PDF attachments), calendar invites, deny-list senders, and conversational threads.
- `inbox-1000-backfill.json`: 1000-message synthesis used by AC-BACKFILL-* tests.
- `labelled-200.json`: the labelled test set referenced by chunk 1 success criterion 3 (macro F1 >= 0.80 on each provider).
- `labelled-50-invoices.json`: the false-positive test set referenced by chunk 1 success criterion 4 (< 1% misclassification as `To-Be-Deleted`).
- `graph-mock-recorder/`: an in-memory Graph mock that records every HTTP method and path; used by AC-HARDDELETE-* and AC-DEPLOY-01.
- `pricing-overrides-test.yaml`: a pricing overrides file used by AC-COST-01 and AC-COST-02.
- `failing-keyring-fixture.py`: a `keyring` backend fixture that always raises, used by chunk 3a SR-01 verification and Section 9.4 fallback tests.

Test data SHALL contain only synthetic addresses (`@example.com`, `@example.org`, `@spam.example`) and synthetic content; real inbox content SHALL NEVER appear in fixtures.

---

### Open questions for chunk 3b

1. **Scheduler CLI in v1.** Section 9.3 references a `sift schedule` subcommand, but chunk 2's CLI surface (FR-34) and section 5.1 do not enumerate it (the chunk 2 scheduler component note explicitly says `sift schedule` is NOT in v1 and that v1 relies on external cron). The Section 10.3 systemd timer covers Linux server users, and Section 10.1 documents a long-running `command: ["schedule"]` for the container. Confirm whether `sift schedule` SHALL be added to v1's CLI surface (back-port to chunk 2) or whether the container long-running command SHALL invoke `sift triage --run --foreground` on a self-contained loop instead. Recommend adding `sift schedule` to v1 because the container path needs it and the systemd timer path does not.
2. **Container long-running scheduler vs systemd timer.** The Section 10.1 compose file uses `command: ["schedule"]` (in-process apscheduler), while Section 10.3 deliberately externalises scheduling to systemd. Confirm whether this dual approach is acceptable, or whether the container path SHALL also externalise (e.g., to a sidecar cron container) for parity.
3. **PyPI package name.** Section 10.2 uses `sift-mail` for the PyPI distribution because `sift` is taken. Confirm whether to register the alternate name now or to use a different name (e.g., `sift365`, `siftmail`, `siftbox`) before v0.1.0 ships.
4. **Live-suite tenant.** Section 11 marks live tests `@pytest.mark.live`. Confirm whether a dedicated Microsoft 365 test tenant and four dedicated provider accounts will be available for the nightly job, or whether the live suite SHALL be operator-run only against the maintainer's personal tenant.
