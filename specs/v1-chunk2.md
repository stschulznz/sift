## 4. Data Models (Pydantic v2, strict mode)

### 4.1 Conventions

All Pydantic models in Sift v1 SHALL satisfy the following:

- `from pydantic import BaseModel, ConfigDict, Field` and `model_config = ConfigDict(strict=True, extra="forbid", frozen=False)` unless a model is explicitly mutable for state-store updates (in which case `frozen=False` is the default and is documented per-model).
- Strict mode forbids implicit type coercion (e.g., `"3"` will not coerce to `int`), aligning Sift validation with the vendored router (Solace section 4 uses the same `_STRICT` config).
- `extra="forbid"` makes every unknown field a hard validation error at the YAML/CLI/env boundary (SR-09). The audit-logger MAY pass through unknown attributes from upstream HTTP responses (e.g., Graph), but those values SHALL NOT be loaded into Pydantic models without an explicit field definition.
- Every field uses an explicit type hint and (where relevant) `Field(...)` constraints (`min_length`, `max_length`, `ge`, `le`, `pattern`).
- Datetime fields are `datetime.datetime` with `tzinfo` set to `datetime.timezone.utc`. Naive datetimes SHALL be rejected at validation.
- UUID fields are `uuid.UUID` (Python stdlib), serialised as canonical hex strings (`"xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"`).
- Optional fields use `T | None` rather than `Optional[T]` (PEP 604), since Sift requires CPython 3.13+.
- Enums are `enum.StrEnum` (so they round-trip to YAML and JSON as plain strings).
- Where a Sift model inherits a Solace shape verbatim, the docstring SHALL begin with `Vendored from Solace section X.` and the field list SHALL NOT diverge.

The shared strict config is exported from `sift.models.base`:

```python
# sift/models/base.py
from pydantic import ConfigDict

STRICT = ConfigDict(strict=True, extra="forbid")
STRICT_OPEN = ConfigDict(strict=True, extra="ignore")  # for vendored shapes that allow new optional fields
```

### 4.2 Configuration models

Loaded from packaged defaults, layered with `~/.config/sift/config.yaml`, `./sift.yaml`, environment variables prefixed `SIFT_`, and CLI flags (FR-37). All configuration models live in `sift.config`.

#### 4.2.1 `SiftConfig`

```python
# sift/config/__init__.py
from pydantic import BaseModel, Field
from sift.models.base import STRICT

class SiftConfig(BaseModel):
    """Top-level Sift configuration. Loaded once per CLI invocation (FR-37)."""
    model_config = STRICT

    graph: GraphAuthConfig
    llm: LLMConfig
    classifier: ClassifierConfig
    folders: FolderTaxonomyConfig = Field(default_factory=FolderTaxonomyConfig)
    rules: RuleLayerConfig = Field(default_factory=RuleLayerConfig)
    sender_cache: SenderCacheConfig = Field(default_factory=SenderCacheConfig)
    backfill: BackfillConfig = Field(default_factory=BackfillConfig)
    retention: RetentionConfig = Field(default_factory=RetentionConfig)
    audit_log_path: str = Field(
        default="~/.local/share/sift/audit.log",
        description="JSON-lines audit log location (FR-38).",
    )
    audit_max_bytes: int = Field(default=50 * 1024 * 1024, ge=1024 * 1024)
    audit_backup_count: int = Field(default=5, ge=1, le=100)
    state_db_path: str = Field(
        default="~/.local/share/sift/state.db",
        description="SQLite state store location (NFR-06, SR-08).",
    )
    pricing_overrides_path: str | None = Field(
        default=None,
        description="Path to YAML pricing-overrides file passed into the vendored cost tracker (section 6.9).",
    )
```

#### 4.2.2 `GraphAuthConfig`

```python
class GraphAuthConfig(BaseModel):
    """Microsoft Graph OAuth and HTTP-client configuration (FR-01 to FR-08)."""
    model_config = STRICT

    tenant_id: str = Field(default="common", min_length=1, max_length=64)
    client_id: str = Field(..., min_length=1, max_length=64)
    scopes: list[str] = Field(
        default_factory=lambda: [
            "Mail.ReadWrite",
            "MailboxSettings.Read",
            "offline_access",
            "User.Read",
        ],
        description="Least-privilege scopes (SR-06). The system SHALL refuse any scope outside this allow-list at validation time.",
    )
    concurrency: int = Field(default=4, ge=1, le=16, description="FR-05 ceiling.")
    page_size: int = Field(default=50, ge=1, le=999, description="Graph $top (FR-03).")
    max_retries: int = Field(default=5, ge=0, le=10, description="429/503 retry budget (FR-04).")
    base_url: str = Field(default="https://graph.microsoft.com/v1.0")
    request_timeout_seconds: float = Field(default=60.0, gt=0.0, le=600.0)
    secret_backend: str = Field(default="keyring", pattern=r"^(keyring|file)$")
    token_file_path: str = Field(default="~/.local/share/sift/.tokens")
```

#### 4.2.3 `LLMConfig`

```python
class LLMConfig(BaseModel):
    """Top-level LLM routing configuration (FR-21 to FR-26)."""
    model_config = STRICT

    provider: LLMProviderName = Field(
        default=LLMProviderName.github_copilot,
        description="Active provider for v1 (FR-21). Switching requires a config edit and a restart; no hot-swap in v1.",
    )
    model: str | None = Field(
        default=None,
        max_length=100,
        description="Model name. If None, the per-provider default in FR-21 applies.",
    )
    temperature: float = Field(default=0.0, ge=0.0, le=2.0, description="Classification is deterministic by default; raise only for experimentation.")
    max_tokens: int = Field(default=400, ge=64, le=4096, description="Caps the JSON classification response.")
    review_threshold: float = Field(default=0.6, ge=0.0, le=1.0, description="FR-25 confidence-gate threshold.")
    request_timeout_seconds: float = Field(default=30.0, gt=0.0, le=300.0)
    health_check_deadline_seconds: float = Field(default=4.0, gt=0.0, le=10.0, description="Solace FR-51 deadline.")

    github_copilot: GitHubCopilotProviderConfig | None = None
    claude: ClaudeProviderConfig | None = None
    azure_openai: AzureOpenAIProviderConfig | None = None
    microsoft_foundry: MicrosoftFoundryProviderConfig | None = None
```

Per-provider blocks (only the active provider's block SHALL be required to be non-`None` at validation time; a missing required block produces a validation error before any Graph or LLM call):

```python
class GitHubCopilotProviderConfig(BaseModel):
    """GitHub Copilot adapter config. Token discovery follows Solace XDG rules; SR-03 enforces 0600 if the file form is used."""
    model_config = STRICT
    token_path: str | None = Field(
        default=None,
        description="If set, overrides XDG auto-discovery. File MUST be mode 0600 (SR-03).",
    )
    api_base: str = Field(default="https://api.githubcopilot.com")

class ClaudeProviderConfig(BaseModel):
    """Anthropic Claude adapter config. API key from env (ANTHROPIC_API_KEY) only (SR-02)."""
    model_config = STRICT
    api_base: str = Field(default="https://api.anthropic.com")
    anthropic_version: str = Field(default="2023-06-01")

class AzureOpenAIProviderConfig(BaseModel):
    """Azure OpenAI adapter config. API key + endpoint via env (SR-02). deployment_map maps logical model names to Azure deployment IDs."""
    model_config = STRICT
    endpoint_env: str = Field(default="AZURE_OPENAI_ENDPOINT", min_length=1)
    api_key_env: str = Field(default="AZURE_OPENAI_API_KEY", min_length=1)
    api_version: str = Field(default="2024-08-01-preview")
    deployment_map: dict[str, str] = Field(
        default_factory=dict,
        description="logical_model -> azure_deployment_id. Required for every model name passed to the adapter.",
    )

class MicrosoftFoundryProviderConfig(BaseModel):
    """Microsoft Foundry adapter config. Service-principal auth (tenant + client + secret) sourced from env (SR-02)."""
    model_config = STRICT
    tenant_id_env: str = Field(default="FOUNDRY_TENANT_ID", min_length=1)
    client_id_env: str = Field(default="FOUNDRY_CLIENT_ID", min_length=1)
    client_secret_env: str = Field(default="FOUNDRY_CLIENT_SECRET", min_length=1)
    endpoint_env: str = Field(default="FOUNDRY_ENDPOINT", min_length=1)
```

#### 4.2.4 `ClassifierConfig`

```python
class ClassifierConfig(BaseModel):
    """Classification pipeline switches (FR-09)."""
    model_config = STRICT

    enable_rule_layer: bool = True
    enable_reference_keeper: bool = True
    enable_sender_cache: bool = True
    enable_llm: bool = True
    default_target_when_disabled_or_unmatched: TargetLabel = Field(
        default=TargetLabel.review,
        description="Fallback when enable_llm=False and no other stage matches.",
    )
```

#### 4.2.5 `FolderTaxonomyConfig`

```python
class FolderTaxonomyConfig(BaseModel):
    """Folder layout (FR-08, FR-13). Overriding any label requires overriding all labels (no partials)."""
    model_config = STRICT

    parent_path: str = Field(default="Triage", min_length=1, max_length=255, pattern=r"^[^/]+(/[^/]+)*$")
    label_to_folder_name: dict[TargetLabel, str] = Field(
        default_factory=lambda: {
            TargetLabel.action: "Action",
            TargetLabel.waiting_for: "Waiting-For",
            TargetLabel.read: "Read",
            TargetLabel.reference: "Reference",
            TargetLabel.newsletters: "Newsletters",
            TargetLabel.notifications: "Notifications",
            TargetLabel.to_be_deleted: "To-Be-Deleted",
            TargetLabel.review: "Review",
        },
    )
```

The `TargetLabel` enum SHALL contain exactly the labels listed in FR-13:

```python
import enum

class TargetLabel(str, enum.Enum):
    action = "Action"
    waiting_for = "Waiting-For"
    read = "Read"
    reference = "Reference"
    newsletters = "Newsletters"
    notifications = "Notifications"
    to_be_deleted = "To-Be-Deleted"
    review = "Review"
```

#### 4.2.6 `RuleLayerConfig`

```python
class SenderListEntry(BaseModel):
    """Sender allow- or deny-list entry (FR-14). Pattern accepts exact address, @domain, or @*.subdomain.example."""
    model_config = STRICT
    pattern: str = Field(..., min_length=1, max_length=320, pattern=r"^(\S+@\S+\.\S+|@\*?\.?\S+\.\S+)$")
    target: TargetLabel | None = Field(
        default=None,
        description="Required for allow-list entries. Deny-list entries SHALL omit target (forced to to_be_deleted per FR-14).",
    )

class Rule(BaseModel):
    """Regex or keyword rule (FR-15). Evaluated in declaration order; first match wins."""
    model_config = STRICT
    name: str = Field(..., min_length=1, max_length=80, pattern=r"^[A-Za-z0-9_\-]+$")
    field: str = Field(..., pattern=r"^(subject|from|to|body_preview)$")
    match_type: str = Field(..., pattern=r"^(regex|keywords)$")
    pattern: str | None = Field(default=None, max_length=2000, description="Required when match_type='regex'.")
    keywords: list[str] | None = Field(default=None, description="Required when match_type='keywords'. Whole-word, case-insensitive.")
    target: TargetLabel

class RuleLayerConfig(BaseModel):
    model_config = STRICT
    allow_list: list[SenderListEntry] = Field(default_factory=list)
    deny_list: list[SenderListEntry] = Field(default_factory=list)
    rules: list[Rule] = Field(default_factory=list)
```

#### 4.2.7 `SenderCacheConfig`

```python
class SenderCacheConfig(BaseModel):
    """Sender-cache thresholds and invalidation (FR-18 to FR-20)."""
    model_config = STRICT
    confidence_threshold: float = Field(default=0.85, ge=0.0, le=1.0)
    min_observations: int = Field(default=5, ge=1, le=1000)
    modal_share_threshold: float = Field(default=0.80, ge=0.5, le=1.0, description="FR-19 modal-target requirement.")
    capped_confidence: float = Field(default=0.99, ge=0.0, le=1.0)
    ttl_days: int | None = Field(
        default=None,
        ge=1,
        description="If set, entries not seen within ttl_days SHALL be evicted lazily on read. None means no TTL.",
    )
```

#### 4.2.8 `BackfillConfig`

```python
class BackfillConfig(BaseModel):
    """Backfill batching and concurrency (FR-27 to FR-30)."""
    model_config = STRICT
    batch_size: int = Field(default=200, ge=1, le=1000)
    concurrency: int = Field(default=4, ge=1, le=16, description="Distinct from graph.concurrency; bounds in-flight classifications, not Graph requests.")
    max_messages: int | None = Field(default=None, ge=1, description="Hard ceiling per FR-30(c).")
```

#### 4.2.9 `RetentionConfig`

```python
class RetentionConfig(BaseModel):
    """Quarantine retention (FR-36, SR-10)."""
    model_config = STRICT
    quarantine_retention_days: int = Field(default=30, ge=1, le=3650)
    quarantine_purge_requires_tty: bool = Field(default=True, description="v1 SHALL always be True; FR-36 forbids --yes in v1.")
```

### 4.3 Mail models

Sift loads only the Graph fields it needs (NFR-04). All Mail models live in `sift.models.mail`.

```python
# sift/models/mail.py
from datetime import datetime
from pydantic import BaseModel, EmailStr, Field
from sift.models.base import STRICT

class EmailAddress(BaseModel):
    model_config = STRICT
    name: str | None = Field(default=None, max_length=320)
    address: EmailStr

class Recipient(BaseModel):
    """Graph 'recipient' shape: {'emailAddress': {...}}. Sift unwraps for ergonomics."""
    model_config = STRICT
    email_address: EmailAddress

class MailFolder(BaseModel):
    model_config = STRICT
    id: str = Field(..., min_length=1, max_length=255)
    display_name: str = Field(..., min_length=1, max_length=255)
    parent_folder_id: str | None = None
    total_item_count: int | None = Field(default=None, ge=0)
    well_known_name: str | None = None  # e.g., 'inbox'

class MailMessage(BaseModel):
    """The subset of Microsoft Graph 'message' fields Sift uses (FR-10, NFR-04)."""
    model_config = STRICT

    id: str = Field(..., min_length=1, max_length=512)
    conversation_id: str | None = Field(default=None, max_length=512)
    internet_message_id: str | None = Field(default=None, max_length=998)
    subject: str = Field(default="", max_length=998)
    from_: EmailAddress | None = Field(default=None, alias="from")
    to_recipients: list[Recipient] = Field(default_factory=list)
    received_date_time: datetime  # tz-aware UTC
    has_attachments: bool = False
    body_preview: str = Field(default="", max_length=255)  # FR-15 pins this to Graph's bodyPreview (255 chars)
    parent_folder_id: str | None = Field(default=None, max_length=255)
    importance: str = Field(default="normal", pattern=r"^(low|normal|high)$")
    categories: list[str] = Field(default_factory=list)

class MailMove(BaseModel):
    """Outbound payload to Graph's /messages/{id}/move and the canonical record in the run journal (FR-06, FR-31)."""
    model_config = STRICT
    message_id: str = Field(..., min_length=1, max_length=512)
    source_folder_id: str = Field(..., min_length=1, max_length=255)
    destination_folder_id: str = Field(..., min_length=1, max_length=255)
```

### 4.4 Classification models

```python
# sift/models/classification.py
from pydantic import BaseModel, Field, conint
from sift.models.base import STRICT
import enum

class ClassificationSource(str, enum.Enum):
    """Pipeline stage that produced a Classification (FR-10).

    NOTE: The enum value 'reply_to_self' is reserved for a future
    pipeline stage that detects user-originated replies and routes
    them to Action; it is not emitted by the v1 pipeline. See the
    open questions at the bottom of this chunk.
    """
    rule = "rule"
    reference_keeper = "reference_keeper"
    sender_cache = "sender_cache"
    llm = "llm"
    low_confidence_review = "low_confidence_review"
    llm_parse_failed = "llm_parse_failed"
    reply_to_self = "reply_to_self"  # reserved; not used in v1
    undo = "undo"  # tagged on reverse-move journal rows (FR-31)

class Classification(BaseModel):
    """The decision attached to one MailMessage."""
    model_config = STRICT

    target_folder: TargetLabel
    confidence: float = Field(..., ge=0.0, le=1.0)
    source: ClassificationSource
    rationale: str = Field(default="", max_length=280)
    rule_name: str | None = Field(default=None, max_length=80, description="Set when source == 'rule'.")
    llm_provider: LLMProviderName | None = None
    llm_model: str | None = Field(default=None, max_length=100)

class LLMClassificationRequest(BaseModel):
    """Sift-internal request shape passed to LLMRouterClient.classify(); section 5.3."""
    model_config = STRICT
    message: MailMessage
    target_labels: list[TargetLabel] = Field(..., min_length=2)
    label_descriptions: dict[TargetLabel, str] = Field(..., description="One-line guidance shown to the LLM per label (section 6.3 prompt template).")

class LLMClassificationResponse(BaseModel):
    """Strict-parsed shape of the LLM's JSON reply.

    The LLM is instructed to return ONLY this object (no markdown
    fences, no leading prose). Repair logic (section 6.3) handles
    common deviations; on second failure the message is routed to
    Triage/Review with source='llm_parse_failed' (FR-24).
    """
    model_config = STRICT

    label: TargetLabel
    confidence: float = Field(..., ge=0.0, le=1.0)
    reasoning: str = Field(default="", max_length=280)
```

The JSON Schema the LLM is asked to return (embedded in the system prompt, section 6.3):

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "SiftClassification",
  "type": "object",
  "additionalProperties": false,
  "required": ["label", "confidence", "reasoning"],
  "properties": {
    "label": {
      "type": "string",
      "enum": ["Action","Waiting-For","Read","Reference","Newsletters","Notifications","To-Be-Deleted","Review"]
    },
    "confidence": { "type": "number", "minimum": 0.0, "maximum": 1.0 },
    "reasoning": { "type": "string", "maxLength": 280 }
  }
}
```

### 4.5 Vendored router-facing models

Sift v1 uses a Sift-side Pydantic mirror of the Solace orchestrator-side `ChatMessage`, `GenerateRequest`, `GenerateResponse`, `GenerateChunk`, and `TokenUsage` shapes. The `tool_calls` field on `ChatMessage` is preserved for forward compatibility (Solace FR-44, FR-45) even though Sift v1 never populates or consumes it; this matches the vendoring contract (do not modify the public API of the snapshot).

`LLMProviderName` in Sift is restricted at validation time to the four v1 providers (the Solace enum is wider, but the wider values are unreachable in Sift; an attempt to set `provider=ollama` in YAML SHALL fail validation with a clear error).

```python
# sift/models/llm.py
from pydantic import BaseModel, Field
from sift.models.base import STRICT, STRICT_OPEN
import enum
from typing import Any

class LLMProviderName(str, enum.Enum):
    """Sift v1 provider allow-list (FR-21). Narrower than the vendored router enum."""
    github_copilot = "github_copilot"
    claude = "claude"
    azure_openai = "azure_openai"
    microsoft_foundry = "microsoft_foundry"

class ChatMessage(BaseModel):
    """Vendored from Solace section 4 (orchestrator-side ChatMessage).

    The tool_calls field is retained verbatim per the vendoring
    contract; v1 never populates it. See Solace FR-44 / FR-45.
    """
    model_config = STRICT_OPEN  # vendored: tolerate new optional fields

    role: str = Field(..., pattern=r"^(system|user|assistant|tool)$")
    content: str = Field(..., min_length=0)
    name: str | None = None
    tool_call_id: str | None = None
    tool_calls: list[dict[str, Any]] | None = None

class TokenUsage(BaseModel):
    """Vendored from Solace section 4."""
    model_config = STRICT
    prompt_tokens: int = Field(default=0, ge=0)
    completion_tokens: int = Field(default=0, ge=0)
    total_tokens: int = Field(default=0, ge=0)

class GenerateRequest(BaseModel):
    """Vendored from Solace section 4 (orchestrator-side GenerateRequest).

    Sift constrains the provider enum (FR-21). Sift never sets tools
    or tool_choice in v1; the fields are retained for shape parity.
    """
    model_config = STRICT

    provider: LLMProviderName
    model: str = Field(..., min_length=1, max_length=100)
    messages: list[ChatMessage] = Field(..., min_length=1)
    stream: bool = False  # Sift v1 uses non-streamed classification calls.
    temperature: float = Field(default=0.0, ge=0.0, le=2.0)
    max_tokens: int | None = Field(default=400, ge=1, le=128000)
    tools: list[dict[str, Any]] | None = None
    tool_choice: str | dict[str, Any] | None = None
    user_id: int | None = None  # not used in Sift; retained for shape parity

class ToolCallFunction(BaseModel):
    model_config = STRICT
    name: str
    arguments: str  # JSON string

class ToolCall(BaseModel):
    """Vendored from Solace section 4. Sift v1 never produces this; retained for shape parity."""
    model_config = STRICT
    id: str
    type: str = "function"
    function: ToolCallFunction

class GenerateResponse(BaseModel):
    """Vendored from Solace section 4."""
    model_config = STRICT
    content: str = ""
    provider: LLMProviderName
    model: str
    tool_calls: list[ToolCall] | None = None
    usage: TokenUsage = Field(default_factory=TokenUsage)
    finish_reason: str = Field(default="stop", pattern=r"^(stop|tool_calls|length)$")

class GenerateChunk(BaseModel):
    """Vendored from Solace section 4. Sift v1 sets stream=False so this is unused at runtime; retained for shape parity."""
    model_config = STRICT
    content: str = ""
    tool_calls: list[ToolCall] | None = None
    usage: TokenUsage | None = None
    finish_reason: str | None = Field(default=None, pattern=r"^(stop|tool_calls|length)$")
    done: bool = False
```

### 4.6 State / persistence models

SQLite-backed via `sqlite-utils`; Pydantic models are the in-memory shape (column types in section 6 component spec for the `StateStore`).

```python
# sift/models/state.py
from datetime import datetime
from uuid import UUID
from pydantic import BaseModel, Field
from sift.models.base import STRICT
import enum

class RunMode(str, enum.Enum):
    triage = "triage"
    backfill = "backfill"
    undo = "undo"

class RunStatus(str, enum.Enum):
    running = "running"
    completed = "completed"
    failed = "failed"
    aborted = "aborted"

class Run(BaseModel):
    """One row in the `runs` table."""
    model_config = STRICT

    run_id: UUID
    started_at: datetime  # tz-aware UTC
    finished_at: datetime | None = None
    mode: RunMode
    status: RunStatus = RunStatus.running
    dry_run: bool = False
    message_count: int = Field(default=0, ge=0)
    action_count: int = Field(default=0, ge=0)
    cost_usd_total: float | None = Field(default=None, ge=0.0, description="Sum of UsageRecord.cost_usd for this run; NULL when ANY contributing record is NULL (Solace FR-46).")
    cost_usd_unknown: bool = Field(default=False, description="True iff any UsageRecord for this run had cost_usd=None.")

class Action(BaseModel):
    """One row in the `actions` table; the canonical undo journal entry (FR-06, FR-31)."""
    model_config = STRICT

    action_id: UUID
    run_id: UUID
    message_id: str = Field(..., min_length=1, max_length=512)
    previous_folder_id: str = Field(..., min_length=1, max_length=255)
    new_folder_id: str = Field(..., min_length=1, max_length=255)
    classification_source: ClassificationSource
    classification_target: TargetLabel
    classification_confidence: float = Field(..., ge=0.0, le=1.0)
    applied_at: datetime
    reverted_at: datetime | None = None
    reverted_by_run_id: UUID | None = None

class SenderCacheEntry(BaseModel):
    """One row in the `sender_cache` table (FR-18 to FR-20)."""
    model_config = STRICT

    sender_address: str = Field(..., min_length=3, max_length=320)
    target_folder: TargetLabel
    consecutive_count: int = Field(default=0, ge=0)
    total_count: int = Field(default=0, ge=0, description="Total observations for this sender across ALL targets, used for the modal-share check (FR-19).")
    last_confidence: float = Field(default=0.0, ge=0.0, le=1.0)
    avg_confidence: float = Field(default=0.0, ge=0.0, le=1.0)
    last_seen_at: datetime

class BackfillCheckpoint(BaseModel):
    """One row in the `backfill_checkpoints` table (FR-28, FR-29)."""
    model_config = STRICT

    run_id: UUID
    last_message_received_at: datetime
    last_message_id: str = Field(..., min_length=1, max_length=512)
    batch_index: int = Field(..., ge=0)
    messages_in_batch: int = Field(..., ge=0)
    written_at: datetime
```

### 4.7 Cost telemetry models

Sift inherits the vendored cost-tracker `UsageRecord` shape with `agent_name` set at the call site (Solace FR-48). Sift uses the literal string `"sift"` for `agent_name` in v1.

```python
# sift/models/cost.py
from datetime import datetime
from uuid import UUID
from pydantic import BaseModel, Field
from sift.models.base import STRICT

class UsageRecord(BaseModel):
    """Vendored from Solace cost tracker. NULL-not-zero on cost_usd is REQUIRED (Solace FR-46)."""
    model_config = STRICT

    provider: LLMProviderName
    model: str = Field(..., min_length=1, max_length=100)
    prompt_tokens: int = Field(..., ge=0)
    completion_tokens: int = Field(..., ge=0)
    cost_usd: float | None = Field(
        default=None,
        ge=0.0,
        description="None means pricing was not known for (provider, model). MUST NOT be 0.0 to indicate unknown (Solace FR-46).",
    )
    run_id: UUID
    agent_name: str = Field(default="sift", min_length=1, max_length=64, description="Solace FR-48; pinned to 'sift' in v1.")
    recorded_at: datetime
```

### 4.8 Error models

Sift errors are typed; every error has a stable `code`, an English `message`, a `retriable` flag, and an optional `retry_after_seconds`. Retriable errors raised inside the Graph client trigger the FR-04 backoff path; non-retriable errors propagate to the CLI exit-code mapper (section 5.1).

```python
# sift/errors.py
from pydantic import BaseModel, Field
from sift.models.base import STRICT

class SiftError(Exception, BaseModel):
    """Base typed error. All Sift-raised exceptions SHALL subclass this."""
    model_config = STRICT
    code: str = Field(..., min_length=1, max_length=64, pattern=r"^[A-Z][A-Z0-9_]+$")
    message: str = Field(..., min_length=1, max_length=2000)
    retriable: bool = False
    retry_after_seconds: int | None = Field(default=None, ge=0)

    def __str__(self) -> str:
        return f"[{self.code}] {self.message}"

class GraphError(SiftError):
    """Raised by GraphClient. code values include GRAPH_AUTH_EXPIRED, GRAPH_THROTTLED, GRAPH_NOT_FOUND, GRAPH_BATCH_PARTIAL_FAILURE, GRAPH_TRANSIENT_5XX."""

class LLMError(SiftError):
    """Raised by LLMRouterClient. code values include LLM_PROVIDER_UNHEALTHY, LLM_PARSE_FAILED, LLM_AUTH_INVALID, LLM_QUOTA_EXHAUSTED, LLM_TRANSIENT."""
```

---

## 5. API / Interface Contracts

Sift v1 exposes no HTTP API and no inbound network surface (SR-07). Its public contract is the CLI and the YAML config.

### 5.1 CLI commands

All commands are implemented as `typer` subcommands of a top-level `sift` app. The shared option contract:

- Every command accepts `--config PATH` (override the YAML lookup chain in FR-37) and `--log-level [debug|info|warning|error]` (default `info`).
- Every command emits structured JSON to stderr (one event per line, `structlog` output) and human output (formatted tables, prompts) to stdout.
- Every command exits with one of the exit codes in the table at the end of this subsection.
- Secrets SHALL NEVER appear in stdout or stderr (SR-04).

#### 5.1.1 `sift auth login`

```
sift auth login [--config PATH] [--log-level LEVEL]
```

- Initiates Microsoft Graph device-code flow via `msal` (FR-01).
- Prints user code and verification URL to stdout; polls Graph until success or the device-code expires.
- On success, persists access token, refresh token, and `expires_at` to OS keychain via `keyring` (default), or to `~/.local/share/sift/.tokens` at file mode 0600 if `graph.secret_backend == "file"` (SR-01).
- On already-logged-in: prints the cached account UPN and exits 0 without re-prompting; pass `--force` to invalidate cached tokens and re-authenticate.

Exit codes: 0 success, 3 device-code expired or user declined, 2 keychain unreachable.

#### 5.1.2 `sift auth status`

```
sift auth status [--config PATH]
```

- Prints: account UPN, tenant ID, scopes held, token expiry timestamp (UTC ISO-8601), and the secret backend in use.
- Issues no Graph requests beyond a cheap `/me` call (cost: one Graph request).

Exit codes: 0 authenticated, 3 not authenticated.

#### 5.1.3 `sift auth logout`

```
sift auth logout [--config PATH]
```

- Clears the cached Graph token (keychain entry or 0600 file).
- If the StateStore shows an in-progress backfill (`runs.status == 'running'` with `mode == 'backfill'`), prints a WARNING and refuses to clear unless `--force` is supplied.

Exit codes: 0 success, 1 refused (in-progress backfill, no `--force`).

#### 5.1.4 `sift triage --dry-run`

```
sift triage --dry-run [--folder INBOX] [--since ISO8601] [--config PATH]
```

- FR-11 dry-run path: full classification pipeline including LLM calls and cost tracking, no Graph mutation.
- `--folder` defaults to the well-known Graph folder name `inbox`. Other values SHALL resolve via `GraphClient.list_folders()` (well-known names or display-name lookup under `parent_path`).
- `--since` accepts ISO-8601 (e.g., `2025-12-01T00:00:00Z`); messages with `receivedDateTime` strictly less than this value SHALL be skipped.
- Stdout: progress bar (rich) plus per-message decision lines; final summary table (FR-40).
- Stderr: structured JSON events.

Exit codes: 0 success, 1 user error (invalid `--since`), 2 transient infra (Graph timeout), 3 auth, 4 throttled-and-could-not-complete.

#### 5.1.5 `sift triage --run`

```
sift triage --run [--folder INBOX] [--since ISO8601] [--max-messages N] [--config PATH]
```

- Same as `--dry-run` but issues real Graph moves.
- `--max-messages N` (1 <= N <= 100000) caps the number of messages classified in this invocation; useful for incremental triage.
- Acquires the scheduler lock file before any Graph mutation (section 6.10) to prevent overlap with a scheduled run.

Exit codes: 0 success, 1 user error, 2 transient infra, 3 auth, 4 throttled, 5 partial success (some moves journalled as `failed`).

#### 5.1.6 `sift backfill`

```
sift backfill --since YYYY-MM-DD [--until YYYY-MM-DD] [--batch-size N] [--concurrency N] [--resume] [--run-id ID] [--config PATH]
```

- FR-27 to FR-30 backfill engine.
- `--since` is required unless `--resume` with a discoverable checkpoint.
- `--batch-size` overrides `backfill.batch_size` (1 <= N <= 1000).
- `--concurrency` overrides `backfill.concurrency` (1 <= N <= 16).
- `--resume` continues the most recent backfill run; `--run-id` selects a specific run.
- Handles SIGINT or SIGTERM by completing the current batch, writing a final checkpoint, and exiting 0 (FR-30(b)).

Exit codes: 0 success or clean abort, 1 user error, 2 transient infra, 3 auth, 4 throttled, 5 partial success.

#### 5.1.7 `sift undo`

```
sift undo --run-id ID [--dry-run] [--config PATH]
```

- FR-31 to FR-33 undo engine.
- Refuses to undo a run whose moves overlap a quarantine purge that has already executed (the affected message IDs no longer exist; section 6.8).
- `--dry-run` lists planned reverse-moves to stdout (one JSON object per line) without issuing any.

Exit codes: 0 success (even when some moves were skipped as unreachable), 1 user error (no run with that ID), 2 transient infra, 3 auth, 5 partial (some Graph errors but most reversed).

#### 5.1.8 `sift status`

```
sift status [--json] [--config PATH]
```

- FR-35 status dump.
- Default human output: `rich.Table` summary covering: configured provider and model, last run summary (id, mode, status, started_at, finished_at, message_count, cost_usd or `null`), pending backfill checkpoints, sender-cache size, quarantine folder size and oldest-message age (one Graph call).
- `--json` emits a single JSON object on stdout (machine-readable).

Exit codes: 0 success, 2 transient infra (Graph unavailable for the quarantine count; sift status SHALL still emit local state), 3 not authenticated (state-only output, no Graph call).

#### 5.1.9 `sift purge --quarantine`

```
sift purge --quarantine [--older-than DAYS] [--yes] [--config PATH]
```

- v1 contract: `--yes` SHALL be accepted only if the YAML config sets `retention.quarantine_purge_requires_tty: false` AND `--yes` is supplied. The default config sets `quarantine_purge_requires_tty: true`, so v1 effectively requires interactive confirmation (FR-36 + SR-10).
- Default `--older-than` matches `retention.quarantine_retention_days` (default 30).
- Interactive prompt SHALL require typing the literal phrase `delete N messages` (where N is the count) before any Graph DELETE is issued.

Exit codes: 0 success, 1 user declined or invalid `--older-than`, 2 transient infra, 3 auth, 5 non-interactive without permission.

#### 5.1.10 `sift config validate`

```
sift config validate [PATH]
```

- Loads the config file at PATH (or via the FR-37 lookup chain when omitted) and runs Pydantic v2 strict validation against `SiftConfig`.
- Stdout: `OK` or a numbered list of validation errors with field paths.
- Issues no Graph or LLM call.

Exit codes: 0 valid, 1 invalid.

#### 5.1.11 `sift config show`

```
sift config show [PATH] [--secrets] [--format yaml|json]
```

- Prints the effective config (after layering FR-37 sources).
- Without `--secrets`: every field declared as a secret-bearing reference (e.g., `*_env`, `token_path`, `state_db_path`) is rendered as the env-var name or path; the secret value SHALL NEVER be resolved or printed.
- `--secrets` resolves env vars and prints their values; refuses to run if stdout is a terminal AND the process is not running with `--secrets-confirm` (defence in depth against shoulder-surfing).

Exit codes: 0 success, 1 invalid PATH or unreadable file.

#### 5.1.12 `sift sender-cache list / clear / show`

```
sift sender-cache list [--limit N] [--sort-by count|last_seen]
sift sender-cache clear [--sender ADDRESS] [--all] [--yes]
sift sender-cache show SENDER
```

- `list`: prints up to N (default 50) entries, sorted by the chosen column.
- `clear --sender ADDRESS` clears one sender; `clear --all` clears the entire cache (interactive confirmation required unless `--yes`).
- `show` prints the full row for one sender, including total observations, modal target, average confidence, and last seen.

Exit codes: 0 success, 1 user error, 2 SQLite error.

#### 5.1.13 `sift llm test`

```
sift llm test --provider PROVIDER [--model MODEL] [--config PATH]
```

- Round-trips a fixed 3-token prompt (`"Say 'pong' in JSON: {\"pong\":true}"`) against the selected provider via the vendored router.
- Validates: adapter `health_check()` (Solace FR-51), one `generate()` call, JSON-parseable response.
- Stdout: provider, model, latency_ms, prompt_tokens, completion_tokens, cost_usd (or `null`), parsed_ok (bool).

Exit codes: 0 success, 3 auth or unhealthy, 2 transient.

#### 5.1.14 `sift llm models`

```
sift llm models [--config PATH]
```

- For each configured provider, calls the adapter's `list_models()` (or the router's equivalent). Reports provider, declared models, and reachability (yes / no / unknown).
- Issues at most one HTTP call per provider.

Exit codes: 0 success, 2 partial (one or more providers unreachable; data still printed).

#### 5.1.15 Exit-code table

| Code | Meaning |
|-----:|---------|
| 0 | Success. |
| 1 | User error (invalid argument, declined prompt, missing required input). |
| 2 | Transient infrastructure error (Graph 5xx after retries, SQLite I/O error, network unreachable). |
| 3 | Authentication error (Graph token invalid or expired and silent refresh failed; provider auth invalid; adapter unhealthy per FR-23). |
| 4 | Throttled and could not complete (FR-04 retries exhausted). |
| 5 | Partial success (some operations succeeded, some failed; details in audit log and run summary). |
| 64 | EX_USAGE: unknown subcommand (FR-34). |

#### 5.1.16 Structured stderr event format

Every CLI command emits one JSON object per line to stderr, produced by `structlog`:

```json
{
  "ts": "2026-04-30T13:22:05.142Z",
  "level": "info",
  "event": "classification_decided",
  "run_id": "5b28e1ac-9f7e-4f2d-b8d9-6f0d5c54a7cc",
  "message_id": "AAMkAGI2...",
  "sender": "noreply@example.com",
  "subject_sha256": "9f3a...",
  "source": "llm",
  "target": "Newsletters",
  "confidence": 0.94,
  "provider": "github_copilot",
  "model": "gpt-4o",
  "prompt_tokens": 412,
  "completion_tokens": 38,
  "cost_usd": null
}
```

Constraints: `cost_usd` SHALL serialise as JSON `null` (not `0.0`) when unknown (Solace FR-46). No event SHALL contain a credential value (SR-04). No event SHALL contain a message body (SR-05). Subjects SHALL be SHA-256-hashed (SR-05).

### 5.2 YAML config file schema

The complete annotated example below covers every field. It maps to the Pydantic models in section 4.2. Defaults shown inline.

```yaml
# Sift v1 config. Loaded per FR-37: packaged defaults <-
# ~/.config/sift/config.yaml <- ./sift.yaml <- env (SIFT_*) <- CLI flags.
# Maps to SiftConfig (section 4.2.1).

graph:                                # GraphAuthConfig (4.2.2)
  tenant_id: "common"                 # Multi-tenant by default.
  client_id: "REPLACE-WITH-AAD-APP-ID"
  scopes:                             # SR-06 least privilege.
    - "Mail.ReadWrite"
    - "MailboxSettings.Read"
    - "offline_access"
    - "User.Read"
  concurrency: 4                      # FR-05; ceiling 16.
  page_size: 50                       # FR-03; ceiling 999.
  max_retries: 5                      # FR-04 retry budget.
  base_url: "https://graph.microsoft.com/v1.0"
  request_timeout_seconds: 60.0
  secret_backend: "keyring"           # or "file" (SR-01); file requires 0600.
  token_file_path: "~/.local/share/sift/.tokens"

llm:                                  # LLMConfig (4.2.3)
  provider: "github_copilot"          # FR-21; one of the four v1 providers.
  model: null                         # If null, FR-21 per-provider default.
  temperature: 0.0
  max_tokens: 400
  review_threshold: 0.6               # FR-25 confidence gate.
  request_timeout_seconds: 30.0
  health_check_deadline_seconds: 4.0  # Solace FR-51.

  # Only the active provider's block is required to be present;
  # other blocks may be omitted or left null.

  github_copilot:                     # GitHubCopilotProviderConfig
    token_path: null                  # null = XDG auto-discovery; SR-03 enforces 0600 if set.
    api_base: "https://api.githubcopilot.com"

  claude:                             # ClaudeProviderConfig
    api_base: "https://api.anthropic.com"
    anthropic_version: "2023-06-01"
    # API key sourced from env ANTHROPIC_API_KEY (SR-02). Never in YAML.

  azure_openai:                       # AzureOpenAIProviderConfig
    endpoint_env: "AZURE_OPENAI_ENDPOINT"
    api_key_env: "AZURE_OPENAI_API_KEY"
    api_version: "2024-08-01-preview"
    deployment_map:                   # logical -> Azure deployment id.
      "gpt-4o": "my-gpt4o-deployment"

  microsoft_foundry:                  # MicrosoftFoundryProviderConfig
    tenant_id_env: "FOUNDRY_TENANT_ID"
    client_id_env: "FOUNDRY_CLIENT_ID"
    client_secret_env: "FOUNDRY_CLIENT_SECRET"
    endpoint_env: "FOUNDRY_ENDPOINT"

classifier:                           # ClassifierConfig (4.2.4)
  enable_rule_layer: true
  enable_reference_keeper: true
  enable_sender_cache: true
  enable_llm: true
  default_target_when_disabled_or_unmatched: "Review"

folders:                              # FolderTaxonomyConfig (4.2.5)
  parent_path: "Triage"               # Created on first run if missing (FR-08).
  label_to_folder_name:               # FR-13; full mapping required if overridden.
    Action: "Action"
    Waiting-For: "Waiting-For"
    Read: "Read"
    Reference: "Reference"
    Newsletters: "Newsletters"
    Notifications: "Notifications"
    To-Be-Deleted: "To-Be-Deleted"
    Review: "Review"

rules:                                # RuleLayerConfig (4.2.6)
  allow_list:                         # FR-14
    - { pattern: "ceo@example.com", target: "Action" }
    - { pattern: "@finance.example.com", target: "Action" }
  deny_list:                          # FR-14; target forced to To-Be-Deleted.
    - { pattern: "@*.spamdomain.tld" }
  rules:                              # FR-15; first match wins; declared order matters.
    - name: "github-notifications"
      field: "from"
      match_type: "regex"
      pattern: "^notifications@github\\.com$"
      target: "Notifications"
    - name: "calendar-invites"
      field: "subject"
      match_type: "keywords"
      keywords: ["accepted:", "declined:", "tentative:"]
      target: "Read"

sender_cache:                         # SenderCacheConfig (4.2.7)
  confidence_threshold: 0.85          # FR-18.
  min_observations: 5                 # FR-19.
  modal_share_threshold: 0.80
  capped_confidence: 0.99
  ttl_days: null                      # Lazy TTL; null disables.

backfill:                             # BackfillConfig (4.2.8)
  batch_size: 200                     # FR-28.
  concurrency: 4
  max_messages: null                  # FR-30(c) ceiling.

retention:                            # RetentionConfig (4.2.9)
  quarantine_retention_days: 30
  quarantine_purge_requires_tty: true # v1 SHALL leave true (FR-36, SR-10).

# Top-level audit and state options
audit_log_path: "~/.local/share/sift/audit.log"  # FR-38
audit_max_bytes: 52428800                        # 50 MiB; FR-39
audit_backup_count: 5                            # FR-39
state_db_path: "~/.local/share/sift/state.db"    # NFR-06, SR-08

# Optional pricing-table override file (section 6.9). When set,
# the vendored cost tracker reads pricing rows from this file in
# addition to (and overriding) its built-in pricing table.
pricing_overrides_path: null
```

Per-field defaults match the Pydantic model defaults; missing keys SHALL inherit the Pydantic default. Unknown top-level keys SHALL produce a WARNING (FR-37) but SHALL NOT fail validation. Unknown nested keys SHALL produce a hard validation error (`extra="forbid"` on every nested model).

### 5.3 Internal interfaces

The five Python interfaces Sift core depends on. Each is a `typing.Protocol` (structural typing) so that fakes in tests can satisfy them without inheritance, and so that the vendored router can be wrapped without modification.

#### 5.3.1 `LLMRouterClient`

```python
# sift/interfaces/llm.py
from typing import Protocol
from sift.models.llm import GenerateRequest, GenerateResponse, LLMProviderName
from sift.models.classification import LLMClassificationRequest, LLMClassificationResponse
from sift.models.cost import UsageRecord
from sift.errors import LLMError

class LLMRouterClient(Protocol):
    """Sift-side wrapper around the vendored llm_router.

    Implemented by sift.llm.RouterAdapter (section 6.1). The
    underlying vendored router is invoked in-process; this Protocol
    is the only surface Sift core sees.
    """

    async def health_check(self, *, deadline_seconds: float) -> None:
        """Solace FR-51 wall-clock-bounded health check.

        SHALL raise LLMError(code='LLM_PROVIDER_UNHEALTHY',
        retriable=False) on failure or timeout.
        """

    async def warm(self) -> None:
        """Solace FR-52 fire-and-forget warm; SHALL never raise.

        Exceptions SHALL be logged at WARNING and swallowed.
        """

    async def generate(self, request: GenerateRequest) -> GenerateResponse:
        """One non-streamed completion. SHALL raise LLMError on
        provider failures. SHALL return cost via the side-channel
        UsageRecord written to the cost tracker; the GenerateResponse
        usage field carries token counts only.
        """

    async def classify(self, request: LLMClassificationRequest) -> LLMClassificationResponse:
        """Build the classification prompt (section 6.3), call generate(),
        parse the JSON reply (with one repair retry per FR-24), and
        return the strict LLMClassificationResponse.

        SHALL raise LLMError(code='LLM_PARSE_FAILED', retriable=False)
        after the second parse failure; the caller routes the message
        to Triage/Review with source='llm_parse_failed'.
        """

    def last_usage(self) -> UsageRecord | None:
        """Return the most recent UsageRecord recorded by this client
        on this asyncio task, or None if none. cost_usd MAY be None
        (Solace FR-46); callers SHALL NOT substitute 0.0.
        """
```

NULL-not-zero cost contract: Sift core code SHALL pass `usage.cost_usd` through unchanged from the vendored cost tracker. Aggregations (e.g., per-run summary) SHALL track a `cost_usd_unknown: bool` flag separate from the running total, and the run summary's `cost_usd_total` SHALL be `None` if any contributing record was `None` (Solace FR-46, mirrored in `Run.cost_usd_unknown` and FR-26 of chunk 1).

#### 5.3.2 `GraphClient`

```python
# sift/interfaces/graph.py
from datetime import datetime
from typing import Protocol, AsyncIterator
from sift.models.mail import MailMessage, MailFolder, MailMove
from sift.errors import GraphError

class GraphClient(Protocol):
    """Async Microsoft Graph client. Throttle, retry, and paging
    behaviour is specified in chunk 1 (FR-03 to FR-08) and section 6.2.
    Method signatures here.
    """

    async def list_folders(self, *, under_parent_id: str | None = None) -> list[MailFolder]:
        """List mail folders. If under_parent_id is None, returns
        well-known top-level folders. SHALL raise GraphError on
        non-retriable failure.
        """

    async def ensure_folder_path(self, parent_path: str, child_names: list[str]) -> dict[str, str]:
        """FR-08. Ensure parent_path/child_name exists for every
        child_name; create missing folders. Returns
        {child_name: graph_folder_id}.
        """

    async def list_messages(
        self,
        *,
        folder_id: str,
        since: datetime | None = None,
        until: datetime | None = None,
        page_size: int = 50,
    ) -> AsyncIterator[MailMessage]:
        """FR-03 paged enumeration ordered by receivedDateTime ASC.
        Yields one MailMessage at a time so the caller can checkpoint
        per-message. SHALL honour Retry-After (FR-04) internally.
        """

    async def get_message(self, message_id: str) -> MailMessage:
        """Fetch a single message by ID. Used by undo to verify
        current parent_folder_id before issuing the reverse move.
        SHALL raise GraphError(code='GRAPH_NOT_FOUND', retriable=False)
        when the message no longer exists.
        """

    async def move_message(self, move: MailMove) -> MailMessage:
        """Issue a single Graph /messages/{id}/move. Returns the
        updated MailMessage. SHALL be idempotent: moving a message
        that is already in destination_folder_id SHALL succeed.
        """

    async def move_messages(self, moves: list[MailMove]) -> list[MailMove | GraphError]:
        """FR-07 batched moves; returns one entry per input in the
        same order. Each entry is either the input move (success)
        or a GraphError (per-operation failure). SHALL NOT raise on
        per-operation failures; only on auth or network failures
        that affect the whole batch.
        """

    async def delete_message(self, message_id: str) -> None:
        """Hard-delete a single message. Used ONLY by
        sift purge --quarantine (SR-10). All other code paths SHALL
        use move_message to the quarantine folder.
        """
```

#### 5.3.3 `StateStore`

```python
# sift/interfaces/state.py
from datetime import datetime
from uuid import UUID
from typing import Protocol
from sift.models.state import (
    Run, Action, SenderCacheEntry, BackfillCheckpoint, RunMode, RunStatus,
)
from sift.models.classification import Classification

class StateStore(Protocol):
    """SQLite-backed state store. Implementation in sift.state.SqliteStateStore."""

    # Run lifecycle
    def start_run(self, *, mode: RunMode, dry_run: bool) -> Run:
        """Insert a new row in `runs` with status=running and a fresh UUIDv4 (FR-12)."""

    def complete_run(self, run_id: UUID, *, status: RunStatus, finished_at: datetime) -> Run:
        """Update the run row; recompute message_count, action_count, and cost aggregates.

        cost_usd_total SHALL be NULL if cost_usd_unknown is True for any
        contributing UsageRecord (Solace FR-46, FR-26).
        """

    def abort_run(self, run_id: UUID, reason: str) -> Run: ...

    def get_run(self, run_id: UUID) -> Run | None: ...

    def list_runs(self, *, limit: int = 50, mode: RunMode | None = None) -> list[Run]: ...

    # Per-action journal
    def record_action(
        self,
        *,
        run_id: UUID,
        message_id: str,
        previous_folder_id: str,
        new_folder_id: str,
        classification: Classification,
        applied_at: datetime,
    ) -> Action: ...

    def list_actions(self, run_id: UUID) -> list[Action]: ...

    def mark_action_reverted(self, action_id: UUID, *, reverted_at: datetime, reverted_by_run_id: UUID) -> Action: ...

    # Backfill checkpoints
    def save_checkpoint(self, ckpt: BackfillCheckpoint) -> None: ...

    def load_checkpoint(self, run_id: UUID) -> BackfillCheckpoint | None: ...

    def latest_backfill_checkpoint(self) -> BackfillCheckpoint | None: ...

    # Sender cache
    def sender_cache_get(self, sender_address: str) -> list[SenderCacheEntry]:
        """Returns one entry per (sender, target_folder) pair. May be empty."""

    def sender_cache_put(self, entry: SenderCacheEntry) -> None:
        """Upsert by (sender_address, target_folder)."""

    def sender_cache_clear(self, *, sender_address: str | None = None) -> int:
        """Clear one sender (sender_address set) or the entire cache (None). Returns the row count deleted."""
```

#### 5.3.4 `RuleLayer`

```python
# sift/interfaces/rules.py
from typing import Protocol
from sift.models.mail import MailMessage
from sift.models.classification import Classification

class RuleLayer(Protocol):
    """Pre-LLM rule evaluator (FR-14, FR-15)."""

    def evaluate(self, message: MailMessage) -> Classification | None:
        """Return a Classification with source='rule' on first match;
        return None otherwise. Allow-list and deny-list precede rules
        (FR-14). Rule evaluation SHALL be deterministic and side-effect-free.
        """
```

#### 5.3.5 `ReferenceKeeper`

```python
# sift/interfaces/reference_keeper.py
from typing import Protocol
from sift.models.mail import MailMessage
from sift.models.classification import Classification

class ReferenceKeeper(Protocol):
    """Safety-net classifier (FR-16, FR-17). Routes to TargetLabel.reference."""

    def evaluate(self, message: MailMessage) -> Classification | None:
        """Return Classification(target_folder=Reference, source='reference_keeper',
        confidence=1.0) when the message has a non-inline file attachment OR
        matches the configured keyword list; return None otherwise.
        """
```

---

## 6. Component Specifications

### 6.1 Vendored LLM router boundary

**Responsibilities.** Provide Sift core a single ergonomic Python interface (`LLMRouterClient`, section 5.3.1) that wraps the vendored Solace router without modifying the snapshot's public API. Wire Sift-specific configuration overrides into the vendored components.

**What is in `vendor/llm_router/`.** Per `VENDORED.md`:

- `adapters/base.py` (`BaseAdapter` ABC, `health_check()`, `warm()`).
- `adapters/github_copilot.py`, `adapters/claude.py`, `adapters/azure_openai.py`, `adapters/microsoft_foundry.py`.
- `models.py` (the router-side `ChatMessage`, `GenerateRequest`, `GenerateResponse`, `GenerateChunk`, `TokenUsage`, `LLMProviderName`).
- `router/dispatch.py` (provider dispatch entry point).
- The cost tracker (post-#179 NULL-not-zero hardening).
- Liveness/readiness primitives (`/livez`, `/health`) and the `warm()` lifecycle hook (Solace section 6.5 internals).

**What Sift overrides via dependency injection.**

1. **Cost persistence adapter.** The vendored cost tracker accepts an injectable persistence callback. Sift wires it to a SQLite implementation that writes one row per call into the `usage_log` table (section 4.7). Schema:

   ```sql
   CREATE TABLE usage_log (
     id INTEGER PRIMARY KEY AUTOINCREMENT,
     provider TEXT NOT NULL,
     model TEXT NOT NULL,
     prompt_tokens INTEGER NOT NULL,
     completion_tokens INTEGER NOT NULL,
     cost_usd REAL,                     -- NULLable; Solace FR-46.
     run_id TEXT NOT NULL,
     agent_name TEXT NOT NULL DEFAULT 'sift',
     recorded_at TEXT NOT NULL          -- ISO-8601 UTC
   );
   CREATE INDEX usage_log_run_id_idx ON usage_log(run_id);
   ```

   The persistence callback SHALL pass `cost_usd` through unchanged; `None` becomes SQL `NULL`. No code path SHALL write `0.0` for unknown pricing.

2. **Pricing-table override path.** The Sift YAML key `pricing_overrides_path` (section 4.2.1) is passed to the vendored cost tracker's `pricing_loader` at startup. The override file format matches the vendored router's pricing-overrides schema; entries override the snapshot's built-in pricing table.

3. **GitHub Copilot auth-cache path.** Sift sets the Copilot adapter's token-cache path to `~/.cache/sift/copilot-token.json` (mode 0600 enforced; SR-03) by default, overridable via `llm.github_copilot.token_path`.

**Sift-side wiring.** The Sift package `sift.llm` exposes one class:

```python
class RouterAdapter(LLMRouterClient):
    """Concrete implementation of section 5.3.1, backed by vendor.llm_router."""
    def __init__(
        self,
        config: LLMConfig,
        cost_persistence: CostPersistenceCallback,
        run_id: UUID,
    ) -> None: ...
```

`run_id` is supplied per-run so every emitted `UsageRecord` carries the correct foreign key. Solace FR-48 (`agent_name`) is set to the literal `"sift"` (section 4.7).

**Adapter lifecycle.**

1. At `RouterAdapter.__init__`, instantiate the vendored adapter for the active provider.
2. Call `asyncio.create_task(adapter.warm())` immediately (FR-22, Solace FR-52). Errors logged at WARNING; never raised.
3. Before the first `generate()` of a run, call `await adapter.health_check(deadline=4.0)` (FR-23, Solace FR-51). On failure, raise `LLMError(code='LLM_PROVIDER_UNHEALTHY', retriable=False)`; the CLI maps this to exit code 3.

**Friction-log discipline.** Any change Sift wants in the vendored snapshot SHALL first be recorded in `VENDORED.md`'s friction log. When the friction list reaches three entries, the next milestone SHALL extract the router into a shared package (per the vendoring contract) instead of accumulating Sift-specific patches.

**Cross-references.** Solace section 6.5 (router dispatch internals); Solace FR-44 / FR-45 (`tool_calls` round-trip preserved); Solace FR-46 / FR-47 / FR-48 (cost tracker contracts).

### 6.2 Microsoft Graph client

**Responsibilities.** Async Graph access for: device-code OAuth, token refresh, paged message enumeration, single and batched moves, folder discovery and creation, single hard-delete (quarantine purge only). Throttle compliance (FR-04). Resumable batch state (FR-06).

**Auth pipeline.**

1. `sift auth login` instantiates `msal.PublicClientApplication(client_id, authority=f"https://login.microsoftonline.com/{tenant_id}")` with the configured scopes.
2. On every Graph request, `GraphClient` checks `expires_at - now() < 300s`; if true, calls `acquire_token_silent()`. On silent-refresh failure, raises `GraphError(code='GRAPH_AUTH_EXPIRED', retriable=False)`; the CLI maps this to exit code 3 with the user-facing message `"Re-run 'sift auth login' to refresh your Microsoft 365 token."`
3. Tokens are read from and written to the configured secret backend (`keyring` or 0600 file) via `sift.secrets`. The token plaintext SHALL NEVER be logged (SR-04).

**Retry policy.** A single shared `httpx.AsyncClient` with the following middleware (custom `httpx.Auth` and `Transport` wrappers):

- 429 or 503: read `Retry-After` (integer seconds OR HTTP-date); sleep at least that long; retry up to `graph.max_retries`. If absent, exponential backoff 1s, 2s, 4s, 8s, 16s, 32s capped at 60s.
- 5xx other than 503: same backoff schedule, no `Retry-After` (FR-04 "header absent" branch).
- 4xx other than 401/403/404/429: raise `GraphError(code='GRAPH_CLIENT_ERROR', retriable=False)` immediately.
- 401: attempt one silent token refresh; on failure raise `GRAPH_AUTH_EXPIRED`.
- 403: raise `GraphError(code='GRAPH_FORBIDDEN', retriable=False)` (likely a missing scope).
- 404: raise `GraphError(code='GRAPH_NOT_FOUND', retriable=False)` (used by undo to detect deleted messages).

After `max_retries` exhausted, raise `GraphError(code='GRAPH_THROTTLED', retriable=False, retry_after_seconds=last_observed)`; the CLI maps this to exit code 4. The throttle log SHALL record one structured event per retry; CI SHALL assert zero `RetryAfterIgnored` events across the integration test suite (NFR-07).

**Paging.** `list_messages()` follows `@odata.nextLink`, one page at a time. Each yielded `MailMessage` is independently consumable; the caller (classification pipeline) processes and (in non-dry runs) journals it before the next message is requested. This bounds memory to one page (NFR-04).

**Concurrency.** `asyncio.Semaphore(graph.concurrency)` wraps every outbound HTTP call. The semaphore is shared across all `GraphClient` methods so that batched moves and message enumeration cannot collectively exceed the configured ceiling. Hard upper bound 16 enforced at validation time (FR-05).

**Resumable batch state.** `move_messages()` writes a journal row with `status='pending'` BEFORE issuing the batch (FR-06). On per-operation success or failure, the row's `status` is updated to `done` or `failed` with the response metadata. On crash mid-batch, the next run's `GraphClient.recover_pending(run_id)` (called by the backfill engine on `--resume`) SHALL re-issue any `pending` moves (idempotent by `(message_id, destination_folder_id)`).

**Liveness adaptation.** Solace section 6.11 (Solace FR-41 to FR-43) defines a `/livez` vs `/health` split for the router server. Sift adapts the same pattern as a Python boolean state on `GraphClient`: `is_alive` (the client object exists and the event loop is running) vs `is_ready` (a successful `health_check()` round-trip to `/me` within the last 5 minutes). `sift status` reports `is_ready`; the backfill engine does not require `is_ready` between batches (a transient network blip is recovered by the retry policy).

**Cross-references.** Chunk 1 FR-01 to FR-08, NFR-07; SR-06 (least-privilege scopes); section 6.10 (scheduler holds Graph client across scheduled runs).

### 6.3 Classification pipeline

**Responsibilities.** For one `MailMessage`, decide a `Classification` by evaluating stages in fixed order, short-circuiting at the first stage that produces a definitive result. Build the LLM prompt. Parse the JSON reply. Apply the confidence gate. Emit one structured audit-log entry per message.

**Per-message decision flow** (FR-09):

```
+----------------+
| MailMessage    |
+--------+-------+
         |
         v
    [Rule layer]----matched--------+
         |                         |
         | None                    |
         v                         |
   [Reference-keeper]--matched-----|
         |                         |
         | None                    |
         v                         |
   [Sender cache lookup]--hit------|
         |                         |
         | None                    |
         v                         |
        [LLM]                      |
         | parse OK, conf>=gate    |
         |                         |
         | parse OK, conf<gate     |
         |        ----+            |
         |            v            |
         |    [Review fallback]    |
         |        source=          |
         |        low_confidence   |
         | parse FAIL (twice)      |
         |        ----+            |
         |            v            |
         |    [Review fallback]    |
         |        source=          |
         |        llm_parse_failed |
         |                         |
         +-----------+-------------+
                     |
                     v
            +------------------+
            | Classification   |
            +--------+---------+
                     |
                     v
       [GraphClient.move_message]
       (skipped when dry_run=True)
                     |
                     v
       [StateStore.record_action]
       [Audit log entry]
       [Sender cache write]
```

**Early-exit semantics.** The first stage to return non-`None` decides the routing. The remaining stages SHALL NOT be invoked. Exception: the reference-keeper is intentionally allowed to override the LLM result (FR-17), which is implemented as the reference-keeper running BEFORE the LLM stage (per FR-09 ordering), not as a post-LLM override. An explicit allow-list match in the rule layer (FR-14) SHALL override the reference-keeper, because the rule layer runs first.

**Per-message audit log entry.** One JSON-lines record per message with: `ts`, `run_id`, `event="classification_decided"`, `message_id`, `sender`, `subject_sha256`, `source`, `target`, `confidence`, `provider`, `model` (when source=`llm` or `low_confidence_review` or `llm_parse_failed`), `prompt_tokens`, `completion_tokens`, `cost_usd` (NULL allowed; Solace FR-46), `dry_run`, `outcome` (one of `moved`, `dry_run`, `skipped`, `error`). Subjects are hashed; bodies and recipients SHALL NEVER appear (SR-05).

**LLM classification system prompt template.** Rendered per call by `sift.llm.prompts.build_classification_prompt(label_descriptions, message)`; the rendered system message is the only system message; the user message is the JSON-encoded message context. Full template:

```
You are Sift, an inbox triage assistant. Your job is to assign one
of the following labels to a single email message, based on the
sender, subject, body preview, and attachment indicator. You are
classifying a message; you are not summarising it, replying to it,
or acting on it.

Available labels (you MUST pick exactly one of these strings, with
exact spelling and case):

{label_descriptions_block}

Output rules (STRICTLY ENFORCED):

1. Return ONE JSON object and nothing else. No prose. No markdown
   fences. No commentary before or after the JSON.
2. The JSON object MUST conform to this schema:

   {
     "label": "<one of the labels above>",
     "confidence": <number between 0.0 and 1.0>,
     "reasoning": "<short justification, max 280 characters>"
   }

3. If you are not sure, set "label" to "Review" and "confidence"
   below 0.6. Do NOT guess high.
4. Treat sender domain as a strong signal. Treat the body preview
   as a weaker signal because it is truncated to 255 characters.

The message follows in the next user message as JSON.
```

`{label_descriptions_block}` is built from `FolderTaxonomyConfig.label_to_folder_name` and the per-label one-liners in `sift.llm.label_descriptions.DEFAULTS` (which the user MAY override via a future config key; v1 ships with fixed defaults). The user message body is:

```json
{
  "from": "alice@example.com",
  "subject": "Q3 invoice, account 4421",
  "body_preview": "Hi, attached is your invoice for September services...",
  "has_attachments": true,
  "received_at": "2026-04-30T08:21:45Z"
}
```

**JSON parse and repair logic** (FR-24):

1. Strip leading and trailing whitespace from the LLM reply.
2. If the reply begins with a markdown code fence (```` ``` ````), strip the fence and any language tag.
3. Attempt `json.loads`. On success, validate against `LLMClassificationResponse` (Pydantic strict).
4. On `json.JSONDecodeError` OR Pydantic `ValidationError` OR `label` not in `target_labels`: re-issue the call with one additional system message appended: `"Your previous reply was not valid JSON for the SiftClassification schema. Reply again with ONLY the JSON object, no prose."`.
5. On second failure: raise `LLMError(code='LLM_PARSE_FAILED', retriable=False)`. The pipeline catches it and records `Classification(target_folder=Review, source=llm_parse_failed, confidence=0.0, rationale="LLM produced unparseable JSON twice in a row")`.

**Confidence gate** (FR-25): if `LLMClassificationResponse.confidence < llm.review_threshold`, the resulting `Classification` SHALL have `target_folder=Review` and `source=low_confidence_review`. The original LLM-suggested label SHALL be persisted in the audit log entry (additional field `llm_suggested_label`) for telemetry but SHALL NOT determine the move.

**Cross-references.** FR-09 to FR-12, FR-22 to FR-26, SR-05; section 6.4 (sender cache write occurs after a successful LLM classification at or above the cache threshold).

### 6.4 Sender cache

**Responsibilities.** Skip the LLM call for messages from a sender whose previous high-confidence classifications have converged on a single target. Reset the counter when an undo reverses the cache's decision.

**Data model.** One row per `(sender_address, target_folder)` pair (`SenderCacheEntry`, section 4.6). The `total_count` column (sum across all targets for this sender) is derived on read by aggregating across rows for the same sender.

**Threshold logic** (FR-18, FR-19):

1. After every LLM classification with `source='llm'` and `confidence >= sender_cache.confidence_threshold` (default 0.85), update or insert the row for `(sender_address, decided_target)`:
   - `consecutive_count` increments by 1 IF `decided_target` matches the most recent target previously cached for this sender (or this is the first observation); otherwise resets to 1.
   - `total_count` (computed across rows) increments by 1.
   - `last_confidence` set to the just-decided confidence.
   - `avg_confidence` updated as `((avg_confidence * (count_for_this_target - 1)) + last_confidence) / count_for_this_target` (numerically stable for the sizes Sift sees).
   - `last_seen_at` set to now (UTC).
2. On the next message from the same sender, `sender_cache_get(sender)` returns all rows. The cache hits IFF:
   - There exists a row with `consecutive_count >= sender_cache.min_observations` (default 5), AND
   - That row's `target_folder` accounts for at least `sender_cache.modal_share_threshold` (default 0.80) of `total_count` across rows for this sender.
3. On hit, the pipeline skips the LLM call. The emitted `Classification` has `source='sender_cache'`, `target_folder = matched_row.target_folder`, `confidence = min(matched_row.avg_confidence, sender_cache.capped_confidence)` (capped at 0.99 by default).

**Invalidation on undo** (FR-20): when `sift undo --run-id ID` reverses an action whose `classification_source == 'sender_cache'`, the corresponding cache row's `consecutive_count` SHALL decrement by 1 (floor 0). If `consecutive_count < sender_cache.min_observations` after decrement, the next message from this sender SHALL again go through the LLM. The undo SHALL log a `cache_invalidated` event with the sender, target, and new `consecutive_count`.

**TTL semantics.** `sender_cache.ttl_days` is lazy-evaluated on read: when `sender_cache_get(sender)` runs, any row with `last_seen_at < now - ttl_days` is deleted in-place and excluded from the returned list. There is no background sweeper in v1.

**Manual operations.** `sift sender-cache clear` and `sift sender-cache show SENDER` (section 5.1.12) operate on this table directly.

**Cross-references.** FR-18 to FR-20; chunk 1 FR-26 (cost tracking still applies on a cache hit only insofar as a UsageRecord is NOT written, since no LLM call occurred).

### 6.5 Rule layer

**Responsibilities.** Config-driven matching that runs before the reference-keeper, sender cache, and LLM. Allow-list (sender forced to a configured target), deny-list (sender forced to `To-Be-Deleted`), then ordered regex and keyword rules with first-match-wins semantics.

**Evaluation order (within the layer)** (FR-14, FR-15):

1. Allow-list pass: iterate `rules.allow_list`. If a sender pattern matches the message's `from` address (case-insensitive, supports exact, `@domain`, `@*.domain`), return `Classification(target_folder=entry.target, source='rule', confidence=1.0, rule_name=f"allow:{entry.pattern}")`.
2. Deny-list pass: iterate `rules.deny_list`. On match, return `Classification(target_folder=TargetLabel.to_be_deleted, source='rule', confidence=1.0, rule_name=f"deny:{entry.pattern}")`.
3. Rule pass: iterate `rules.rules` in declaration order. For each rule:
   - If `match_type == 'regex'`: compile `pattern` once at config-load time (cache compiled regex); test against the named field.
   - If `match_type == 'keywords'`: case-insensitive whole-word match (`re.compile(r"\b(" + "|".join(map(re.escape, kws)) + r")\b", re.I)`).
   - On first match, return `Classification(target_folder=rule.target, source='rule', confidence=1.0, rule_name=rule.name)`.
4. No match: return `None`. Pipeline proceeds to the reference-keeper.

**Field semantics**:

- `subject`: the message's `subject` (already truncated to 998 chars by Pydantic at load).
- `from`: the sender's `address` only (the display name SHALL NOT be matched).
- `to`: the joined comma-separated list of recipient `address` values (lower-cased).
- `body_preview`: Graph's `bodyPreview` (max 255 chars). The full body SHALL NEVER be loaded for rule evaluation (NFR-04).

**Compilation** (FR-15): every regex SHALL be compiled once at `RuleLayer.__init__`, not per message. Invalid patterns SHALL fail at config validation, not at first match.

**Allow-list wins over deny-list**: documented as intentional; `sift config validate` SHALL emit a WARNING when the same pattern appears in both lists.

### 6.6 Reference-keeper safety net

**Responsibilities.** Prevent invoices, receipts, tax documents, contracts, and statements from being routed to `Triage/To-Be-Deleted` by the LLM. False-deletion of these messages is the project's worst-case failure mode (chunk 1 success criterion 4: zero or rounded-zero false deletions on the labelled 50-message receipts test set).

**Trigger conditions** (FR-16, FR-17). The reference-keeper SHALL return `Classification(target_folder=Reference, source='reference_keeper', confidence=1.0, rationale='attachment')` IF:

- The message's `has_attachments == True` AND at least one Graph attachment is of type `fileAttachment` (not `itemAttachment` or `referenceAttachment`, and not `isInline=True`). The attachment-type check requires one additional Graph call (`/messages/{id}/attachments`) IF `has_attachments` is True; this call is made lazily only when needed.

OR `Classification(target_folder=Reference, source='reference_keeper', confidence=1.0, rationale='keyword:<word>')` IF:

- The lower-cased `subject` OR `body_preview` contains any whole-word match against the configured keyword list. v1 default list (case-insensitive whole-word match): `invoice, receipt, tax, taxation, contract, statement, bill, policy, agreement, purchase order, remittance`.

**Override semantics.** The reference-keeper SHALL override the LLM (because it runs before the LLM in FR-09). It SHALL be overridable by an explicit allow-list rule that targets a non-Reference folder (because the rule layer runs first). It SHALL NOT be overridable by config-only disabling without a deliberate user action (`classifier.enable_reference_keeper: false`); the YAML key exists so users can opt out, but `sift config validate` SHALL emit a WARNING when it is set to `false`.

**Configurability** (FR-17): users MAY extend or replace the keyword list via a future config key (`reference_keeper.keywords: list[str]`); v1 ships with the fixed defaults and the planned config key is reserved for v1.x.

### 6.7 Backfill engine

**Responsibilities.** Iterate the configured inbox window in deterministic batches, classify each message via the pipeline, issue moves (or skip when `dry_run`), checkpoint per-batch, and resume cleanly across restarts.

**Batch loop pseudocode**:

```python
async def run_backfill(args, store, graph, pipeline):
    if args.resume:
        ckpt = store.load_checkpoint(args.run_id) or store.latest_backfill_checkpoint()
        if ckpt is None:
            raise SiftError(code="BACKFILL_NO_CHECKPOINT", message=..., retriable=False)
        run = store.get_run(ckpt.run_id)
        cursor = ckpt.last_message_received_at
        batch_index = ckpt.batch_index + 1
    else:
        run = store.start_run(mode=RunMode.backfill, dry_run=False)
        cursor = args.since
        batch_index = 0

    inbox_id = (await graph.list_folders())[INBOX].id
    folder_ids = await graph.ensure_folder_path(config.folders.parent_path,
                                                list(config.folders.label_to_folder_name.values()))
    pipeline_sem = asyncio.Semaphore(config.backfill.concurrency)

    while True:
        page = []
        async for msg in graph.list_messages(folder_id=inbox_id, since=cursor,
                                             until=args.until, page_size=config.graph.page_size):
            page.append(msg)
            if len(page) >= config.backfill.batch_size:
                break
        if not page:
            break

        async def process(msg):
            async with pipeline_sem:
                cls = await pipeline.classify(msg)
                if not args.dry_run:
                    target_folder_id = folder_ids[config.folders.label_to_folder_name[cls.target_folder]]
                    move = MailMove(message_id=msg.id, source_folder_id=msg.parent_folder_id,
                                    destination_folder_id=target_folder_id)
                    [result] = await graph.move_messages([move])
                    if isinstance(result, GraphError):
                        log.warning("move_failed", error=result.code, message_id=msg.id)
                        return
                store.record_action(run_id=run.run_id, message_id=msg.id,
                                    previous_folder_id=msg.parent_folder_id,
                                    new_folder_id=target_folder_id,
                                    classification=cls, applied_at=now_utc())

        await asyncio.gather(*(process(m) for m in page))

        last = page[-1]
        store.save_checkpoint(BackfillCheckpoint(
            run_id=run.run_id,
            last_message_received_at=last.received_date_time,
            last_message_id=last.id,
            batch_index=batch_index,
            messages_in_batch=len(page),
            written_at=now_utc(),
        ))
        batch_index += 1
        cursor = last.received_date_time

        if config.backfill.max_messages and run.message_count >= config.backfill.max_messages:
            break
        if shutdown_requested():  # set by SIGINT / SIGTERM handler
            break

    store.complete_run(run.run_id, status=RunStatus.completed, finished_at=now_utc())
```

**Resumability** (FR-29). `--resume` reads the most recent checkpoint and restarts from `last_message_received_at`. The combination of `(received_date_time, message_id)` as the cursor SHALL be deterministic (Graph's `$orderby=receivedDateTime asc` plus a tie-break on `id`). On a tied `received_date_time` boundary, the engine SHALL re-fetch the page starting at the boundary and skip messages whose `id <= last_message_id` to avoid re-processing.

**Concurrency vs throttle compliance.** Two semaphores:

- `pipeline_sem = Semaphore(backfill.concurrency)` bounds concurrent classifications (LLM calls) and downstream moves.
- The `GraphClient` semaphore (`graph.concurrency`) bounds concurrent Graph requests across all paths.

Backfill MAY have `pipeline.concurrency=4` and `graph.concurrency=4`; the lower of the two is the effective LLM rate. CI SHALL test that across a synthetic 1,000-message backfill, the observed Graph QPS stays at or below `graph.concurrency * 1`.

**Backfill vs incremental triage scheduling.** `sift triage --run` and `sift backfill` use the same scheduler lock file (section 6.10). If a backfill is in progress, a scheduled `triage --run` SHALL skip with a `lock_held` event. If a `triage --run` is in progress, an explicit `sift backfill` invocation SHALL exit with status 1 and the message `"Another Sift run is in progress; wait for it to finish or pass --force."`

**Cross-references.** FR-27 to FR-30, FR-12 (run ID); section 6.2 (Graph client retry); section 6.10 (scheduler lock).

### 6.8 Undo engine

**Responsibilities.** Reverse every Action recorded for a given `run_id`. Idempotent on re-execution. Honest about what cannot be reversed (deleted or relocated messages). Refuse to act on a run that overlaps a quarantine purge.

**Algorithm**:

1. Load `actions = store.list_actions(run_id)` ordered by `applied_at` DESC.
2. Refuse if any `Action.new_folder_id` corresponds to the quarantine folder AND a `sift purge --quarantine` event is recorded in the audit log with timestamp greater than `actions[0].applied_at`. Exit code 1, message `"Refusing to undo: a quarantine purge has run since this run; some messages no longer exist."`
3. Start a new run with `mode=RunMode.undo`, capturing `undo_run_id`.
4. For each `action`:
   - `current = await graph.get_message(action.message_id)`. If `GraphError(code='GRAPH_NOT_FOUND')`, record `event=skipped_unreachable, reason=deleted` and continue.
   - If `current.parent_folder_id != action.new_folder_id`, record `event=skipped_unreachable, reason=user_moved` and continue.
   - Else issue `move = MailMove(message_id=action.message_id, source_folder_id=action.new_folder_id, destination_folder_id=action.previous_folder_id)`; await `graph.move_message(move)`.
   - On success: `store.mark_action_reverted(action.action_id, reverted_at=now_utc(), reverted_by_run_id=undo_run_id)`. Also write a NEW Action row with `classification_source=ClassificationSource.undo` so a redo (`sift undo --run-id <undo_run_id>`) is mechanically the same code path.
   - If `action.classification_source == 'sender_cache'`, decrement the cache (FR-20).
5. Print summary: `Reversed: X, Skipped (deleted): Y, Skipped (user-moved): Z, Failed (errors): W`.

**Idempotency.** Re-running `sift undo --run-id ID` SHALL be a no-op for actions already marked `reverted_at IS NOT NULL`. Such actions SHALL be skipped silently with `event=already_reverted`.

**Dry-run** (FR-33). `--dry-run` lists planned reverse-moves to stdout (one JSON per line) and SHALL NOT issue any Graph mutation; SHALL NOT start a new run row in the StateStore; SHALL NOT mark any Action as reverted.

**Cross-references.** FR-31 to FR-33, FR-20 (sender-cache invalidation), SR-10 (no hard-delete by undo; undo only reverses moves).

### 6.9 Cost tracker

**Responsibilities.** Wire the vendored Solace cost tracker into Sift. Persist every `UsageRecord` to SQLite. Enforce NULL-not-zero (Solace FR-46). Emit one `pricing_unknown` warning per `(provider, model)` per process (Solace FR-47). Plumb `agent_name="sift"` into every record (Solace FR-48).

**Architecture**:

```
+-------------------+       +--------------------------+
| RouterAdapter     |------>| vendored CostTracker     |
| (sift.llm)        |       |                          |
+-------------------+       |  - records UsageRecord   |
                            |  - resolves cost_usd     |
                            |    (None if unknown)     |
                            |  - one-shot warning per  |
                            |    (provider, model)     |
                            |                          |
                            +-----------+--------------+
                                        |
                                        | persistence callback
                                        v
                            +-------------------------+
                            | SqliteUsagePersistence  |
                            | (sift.cost.persistence) |
                            +-------------------------+
                                        |
                                        v
                            +-------------------------+
                            | usage_log SQLite table  |
                            +-------------------------+
```

**Persistence callback.** Sift exposes `SqliteUsagePersistence(state_db_path, run_id)`. Its `record(usage: UsageRecord) -> None` method writes one row to `usage_log` (schema in section 6.1). The callback SHALL pass `usage.cost_usd` to the SQL parameter unchanged; `None` becomes SQL `NULL`. No code path SHALL coerce to `0.0`.

**Pricing-overrides path.** When `pricing_overrides_path` is set in YAML, Sift reads the file at startup and passes the parsed entries to the vendored tracker's `pricing_loader.load_overrides()`. Overrides take precedence over the snapshot's built-in pricing table. The override file format follows the vendored router's published schema; an invalid file SHALL fail config validation.

**One-shot `pricing_unknown` warning** (Solace FR-47). The vendored tracker emits one `event=pricing_unknown, provider=..., model=...` WARNING per `(provider, model)` per process lifetime. Sift inherits this verbatim; Sift SHALL NOT add its own pricing-unknown warnings on top.

**`agent_name` plumbing** (Solace FR-48). Sift sets `agent_name="sift"` literally. The vendored tracker reads it from the per-call `record_usage(..., agent_name=...)` parameter; Sift's `RouterAdapter` always passes `"sift"`. The `usage_log.agent_name` column has a SQL DEFAULT of `'sift'` as a defence-in-depth.

**Aggregation.** `Run.cost_usd_total` and `Run.cost_usd_unknown` are computed by `StateStore.complete_run()` via:

```sql
SELECT
  SUM(cost_usd) AS cost_usd_total,                                 -- NULL if any input is NULL
  CAST(SUM(CASE WHEN cost_usd IS NULL THEN 1 ELSE 0 END) > 0 AS INTEGER) AS cost_usd_unknown
FROM usage_log
WHERE run_id = ?;
```

Note that SQL `SUM()` returns NULL when any non-aggregated input is NULL ONLY IF the implementation propagates NULL; SQLite returns the sum of non-NULL values, IGNORING NULLs. To match Solace FR-46 semantics, Sift SHALL use the explicit pattern above and SHALL set `cost_usd_total = None` (Python) when `cost_usd_unknown == 1`.

**Cross-references.** Solace FR-46, FR-47, FR-48; chunk 1 FR-26 (NULL-not-zero in run summaries); SR-04 (no API keys in the cost tracker logs).

### 6.10 Scheduler

**Responsibilities.** Run `sift triage --run` on a cron-style schedule. Hold a lock file to prevent overlapping runs (with each other and with `sift backfill`). Coexist with external cron when the user prefers it.

**Implementation.** `apscheduler.schedulers.asyncio.AsyncIOScheduler` is instantiated by `sift triage --run --schedule` (a future v1.x flag) OR by an explicit `sift schedule` command (NOT in v1; v1 uses external cron). For v1, the scheduler module exists but is invoked only via external cron entries that call `sift triage --run`.

**Lock file.** `~/.local/share/sift/run.lock` (file mode 0600 enforced). Acquired via `fcntl.flock(fd, LOCK_EX | LOCK_NB)` on POSIX, `msvcrt.locking()` on Windows. Released on normal exit AND on signal (SIGINT, SIGTERM). On startup, every command that mutates Graph state (`triage --run`, `backfill`, `undo`, `purge`) SHALL attempt to acquire the lock; on contention, it SHALL exit 1 with the message `"Another Sift run is in progress (lock held by PID <pid> since <timestamp>); wait for it to finish."`

**Lock file content.** One JSON object: `{"pid": <int>, "started_at": "<ISO-8601 UTC>", "command": "<argv joined>"}`. Written under the lock so a stale lock (e.g., from a killed process whose flock was released by the kernel but file content remained) is recognisable. A stale lock whose `pid` is no longer running on the same host SHALL be reclaimed automatically with a WARNING.

**Cron coexistence.** External cron is the v1 default. Recommended crontab line (from the user-facing operations runbook, referenced from chunk 4 documentation requirement NFR-12):

```
*/15 * * * *  /usr/local/bin/sift triage --run --config /etc/sift/config.yaml >> /var/log/sift/cron.log 2>&1
```

The lock file ensures that two cron firings on the same host cannot collide (e.g., when one run takes longer than the cron interval).

**Cross-references.** FR-34 (CLI surface), section 6.7 (backfill respects the same lock), SR-08 (file modes), SR-04 (no secrets in cron logs).

### Open questions for chunk 2

1. **`reply_to_self` classifier source.** Chunk 2's data-model brief (section 4.4 enum) lists `reply_to_self` as a `ClassificationSource` value, but chunk 1 FR-10 enumerates only `rule`, `reference_keeper`, `sender_cache`, `llm`, and `low_confidence_review`. The pipeline described in FR-09 has no `reply_to_self` stage. The enum value has been included as reserved with a clarifying note; please confirm whether v1 should actually implement a "user-replied-from-self" detector as a sixth pipeline stage (which would shift FR-09 ordering) OR whether the value should be deleted from the v1 enum.

2. **Adapter `list_models()` surface.** `sift llm models` (section 5.1.14) requires the vendored adapter to expose `list_models()`. Solace section 6.5 implies this exists for the GitHub Copilot adapter (`/v1/models` is populated from cached capabilities) but does not explicitly contract it across all four adapters. Please confirm that the vendored snapshot's four adapters all implement `list_models()` OR specify a Sift-side fallback (e.g., return only the configured model and treat unreachable providers as "unknown").

3. **Pricing-overrides file schema.** Section 6.9 references `vendored router's published schema` for `config/pricing-overrides.yaml` but the Solace spec section referenced in chunk 1 FR-26 does not normatively specify the file format. The Sift spec needs either a pointer to the canonical Solace spec section that defines the schema OR an explicit definition embedded in chunk 4. Flagging here so chunk 4 (test plan / implementation plan) can pick it up.

4. **Quarantine-overlap detection.** Section 6.8 step 2 ("refuse if a `sift purge --quarantine` event has run since the run") relies on an audit-log scan. In long-running deployments the audit log rotates (FR-39); rotated files MAY be archived off-host. Please confirm whether v1 should: (a) require all rotated audit files to be present on-disk for undo to honour the refuse rule, OR (b) record purge events in the StateStore (a small dedicated table) to make the check robust against log rotation. Recommended: (b); flagging because adopting (b) requires a tiny schema addition and a section 4.6 model that is not currently included.

5. **Ambiguity in `MailMessage.from`.** Pydantic v2 forbids field names that shadow Python builtins; the spec uses `from_: EmailAddress | None = Field(default=None, alias="from")` to round-trip Graph's wire field. Confirm this is the desired ergonomics (rather than renaming to `sender` end-to-end). The alias has knock-on effects for any rule that names `field: "from"` (section 4.2.6) and for the structured-event JSON keys (section 5.1.16). Recommended: keep `alias="from"` so the Graph wire field, the rule field, and the audit-log key all read `from`; flagging because reviewers commonly catch this.
