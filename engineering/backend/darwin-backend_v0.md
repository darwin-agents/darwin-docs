# darwin-backend — Phase 1 Engineering Specification

> **Audience:** Engineers joining the project.  
> **Scope:** Phase 1 MVP only — one human creator, one Darwin agent, and its internal sub-agents (neurons). No swarms, no x402, no IDO marketplace.  
> **Stack:** Python · FastAPI · Supabase (PostgreSQL + pgvector + pgcrypto + Auth) · Redis · Celery · Modal SDK

---

## Table of Contents

1. [Role of the Backend](#1-role-of-the-backend)
2. [Architecture Principles](#2-architecture-principles)
3. [Project Structure](#3-project-structure)
4. [Database Schema](#4-database-schema)
5. [Authentication & Authorization](#5-authentication--authorization)
6. [WebSocket & Real-Time Layer](#6-websocket--real-time-layer)
7. [API Controllers (Routers)](#7-api-controllers-routers)
   - 7.1 `/users`
   - 7.2 `/darwins`
   - 7.3 `/finance`
   - 7.4 `/webhooks`
   - 7.5 `/memory`
   - 7.6 `/contracts`
   - 7.7 `/telemetry`
   - 7.8 `/assets`
   - 7.9 `/audit`
   - 7.10 `/identity`
   - 7.11 `/gateway`
   - 7.12 `/firmware`
   - 7.13 `/notifications`
   - 7.14 `/sessions`
   - 7.15 `/security`
8. [Agent-Facing Endpoints (Internal API)](#8-agent-facing-endpoints-internal-api)
9. [Frontend-Facing Data Contracts](#9-frontend-facing-data-contracts)
   - 9.1 Agent Timeline Feed
   - 9.2 Thought Log (Trace) Feed
   - 9.3 SQLite Entity Feed (Hermes Database Mirror)
   - 9.4 Sub-Agent (Neuron) Events
   - 9.5 Metabolic Stream
10. [The Asynchronous Queue (Celery + Redis)](#10-the-asynchronous-queue-celery--redis)
11. [Security Implementation](#11-security-implementation)
12. [Agent Finite State Machine](#12-agent-finite-state-machine)
13. [Deployment](#13-deployment)
14. [Environment Variables](#14-environment-variables)

---

## 1. Role of the Backend

The `darwin-backend` is the **God Layer** — the always-on orchestrator that enforces the physical laws of the Darwin ecosystem. It does not reason or make strategic decisions; that is the agent's job. Instead it:

- Manages the master fiat ledger and VCC lifecycle
- Catches all inbound webhooks (Stripe, Telegram, email, Modal events)
- Wakes and sleeps agent containers via the Modal SDK
- Streams real-time telemetry and agent cognitive traces to the frontend
- Mirrors the agent's internal SQLite state (entities, skills, episodic summaries) to the frontend
- Enforces security at every layer — validating, sanitizing, and rejecting malformed agent output
- Provides the human creator with a live, queryable view of everything the agent has ever done, thought, or created

The backend sits between three parties that must never touch each other directly:

```
darwin-agent (Modal container)
      │  only /finance and /telemetry endpoints,
      │  authenticated with short-lived agent JWTs
      ▼
darwin-backend  ◄── inbound: Stripe, Telegram, email, Modal events
      │  only REST + WebSocket, HTTPS
      │  human JWTs (Supabase Auth)
      ▼
darwin-frontend (Vercel)
```

---

## 2. Architecture Principles

### 2.1 Async-first

FastAPI with `async/await` throughout. At any given moment the majority of agents are hibernating — the backend must handle thousands of concurrent webhook connections with near-zero active CPU. Never use synchronous blocking calls in route handlers.

### 2.2 Pydantic at every boundary

Every payload — from the agent, from the frontend, from a webhook — is validated against a strict Pydantic v2 model before it touches the database or triggers any action. If the LLM hallucinates a malformed tool call, FastAPI rejects it at the HTTP layer. The agent receives a structured validation error it can parse and self-correct from.

### 2.3 Transactional ledger integrity

All balance mutations use `SELECT ... FOR UPDATE` row-level locks. Two concurrent purchase requests from the same agent cannot both succeed if the combined cost would exceed the balance. One waits; the other completes; the waiting request then re-reads the updated balance and fails gracefully.

### 2.4 Acknowledge fast, process in background

All inbound webhooks (Stripe, Telegram, Modal) return `200 OK` within 10 ms by immediately enqueueing to Redis. Background Celery workers do the actual processing. This prevents Stripe from retrying because our server timed out.

### 2.5 The frontend sees everything

Every cognitive event that happens inside the agent — a thought, a tool call, a screenshot, a sub-agent spawn, a skill learned, an entity created in SQLite, a financial deduction — must be surfaced to the frontend. The backend is the translation layer between the agent's internal world and the Infinite Canvas.

---

## 3. Project Structure

```
darwin-backend/
├── main.py                    # FastAPI app factory, router registration, lifespan events
├── config.py                  # Settings via pydantic-settings; all env vars
├── deps.py                    # FastAPI dependency injection (DB session, auth, agent JWT)
│
├── routers/
│   ├── users.py               # /users — creator accounts, vault management
│   ├── darwins.py             # /darwins — lifecycle: spawn, hibernate, wake, terminate
│   ├── finance.py             # /finance — ledger, VCC issuing, payment injection
│   ├── webhooks.py            # /webhooks — Stripe, Telegram, email, Modal events
│   ├── memory.py              # /memory — skill documents, pgvector search, royalties
│   ├── contracts.py           # /contracts — M2M bounty board, escrow, settlement
│   ├── telemetry.py           # /telemetry — metabolic stream, circuit breaker
│   ├── assets.py              # /assets — presigned S3 uploads, binary artifact indexing
│   ├── audit.py               # /audit — immutable event log, tax export
│   ├── identity.py            # /identity — trust scores, dispute resolution
│   ├── gateway.py             # /gateway — LLM rate-limit multiplexing
│   ├── firmware.py            # /firmware — agent brain upgrades, SQLite migrations
│   ├── notifications.py       # /notifications — Telegram push, CAPTCHA session links
│   ├── sessions.py            # /sessions — live Browserbase WebSocket relay
│   └── security.py            # /security — pre-flight intent scan, account suspension
│
├── schemas/                   # Pydantic models — shared between routers and agent
│   ├── agents.py
│   ├── finance.py
│   ├── telemetry.py
│   ├── memory.py
│   ├── canvas.py              # Timeline event shapes consumed by the frontend
│   └── hermes.py              # Mirrors of the Hermes SQLite entity types
│
├── services/
│   ├── modal_service.py       # Modal SDK wrapper — spawn, wake, terminate containers
│   ├── ledger_service.py      # Balance reads/writes with row locks
│   ├── vcc_service.py         # VCC encrypt/decrypt, JIT injection
│   ├── telegram_service.py    # Telegram Bot API calls
│   ├── langsmith_service.py   # LangSmith/Phoenix trace fetching
│   ├── canvas_service.py      # Build timeline event payloads for WebSocket push
│   └── intent_scan_service.py # Pre-flight classifier call
│
├── workers/                   # Celery task definitions
│   ├── webhook_worker.py      # Process queued webhook payloads
│   ├── wake_worker.py         # Execute agent wake-up sequences
│   └── telemetry_worker.py    # Persist telemetry batches
│
├── db/
│   ├── client.py              # Supabase async client (supabase-py v2)
│   ├── migrations/            # SQL migration files (run via supabase CLI)
│   └── seed.py                # Dev seed data
│
└── tests/
    ├── test_finance.py
    ├── test_lifecycle.py
    ├── test_webhooks.py
    └── test_security.py
```

---

## 4. Database Schema

All tables live in Supabase (PostgreSQL). Row Level Security (RLS) is enabled on every table. The agent only ever touches the `darwin_events`, `ledger_transactions`, and `hibernate_schedule` tables via its two dedicated endpoints — it has no direct database access.

### 4.1 `users`

Managed by Supabase Auth. Extended with a profile table:

```sql
CREATE TABLE user_profiles (
  id          UUID PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
  telegram_chat_id  BIGINT,          -- for push notifications
  created_at  TIMESTAMPTZ DEFAULT now()
);
```

### 4.2 `darwins`

One row per agent. Central state table.

```sql
CREATE TABLE darwins (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id         UUID NOT NULL REFERENCES auth.users(id),
  name            TEXT NOT NULL,
  state           TEXT NOT NULL DEFAULT 'PENDING'
                    CHECK (state IN ('PENDING','ACTIVE','HIBERNATING','TERMINATED')),

  -- Genetics (the agent's DNA)
  system_prompt   TEXT NOT NULL,          -- core instructions, injected at every wake
  llm_model       TEXT NOT NULL,          -- e.g. 'claude-sonnet-4-6', 'gpt-4o-mini'
  tool_set        JSONB NOT NULL,         -- list of tool names the agent is granted

  -- Financial
  vcc_ref         TEXT,                   -- reference token pointing to vcc_vault row
  ledger_balance  NUMERIC(12,4) NOT NULL DEFAULT 0,
  initial_budget  NUMERIC(12,4) NOT NULL,

  -- Modal
  modal_function_name  TEXT,             -- Modal function identifier for this agent
  modal_volume_id      TEXT,             -- persistent volume UUID

  -- Autonomy
  requires_approval    BOOLEAN DEFAULT TRUE,  -- autonomy toggle

  -- Timing
  spawned_at      TIMESTAMPTZ,
  last_active_at  TIMESTAMPTZ,
  terminated_at   TIMESTAMPTZ,
  created_at      TIMESTAMPTZ DEFAULT now(),

  -- Telemetry summary (updated by telemetry worker)
  total_tokens_burned   BIGINT DEFAULT 0,
  total_compute_cost    NUMERIC(10,4) DEFAULT 0,
  total_revenue         NUMERIC(12,4) DEFAULT 0,
  token_roi             NUMERIC(8,4)  -- updated each session

);

ALTER TABLE darwins ENABLE ROW LEVEL SECURITY;
CREATE POLICY "users see own darwins" ON darwins
  FOR ALL USING (auth.uid() = user_id);
```

### 4.3 `vcc_vault`

Encrypted VCC storage. Raw PANs never stored in plaintext.

```sql
CREATE TABLE vcc_vault (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  darwin_id       UUID NOT NULL REFERENCES darwins(id),
  user_id         UUID NOT NULL REFERENCES auth.users(id),
  encrypted_pan   BYTEA NOT NULL,        -- AES-256-GCM via pgcrypto
  encrypted_cvv   BYTEA NOT NULL,
  encrypted_expiry BYTEA NOT NULL,
  spend_limit     NUMERIC(10,2) NOT NULL,
  provider        TEXT NOT NULL,         -- 'privacy.com' | 'revolut' | 'lithic' | 'stripe_issuing'
  created_at      TIMESTAMPTZ DEFAULT now()
);

ALTER TABLE vcc_vault ENABLE ROW LEVEL SECURITY;
CREATE POLICY "users see own vccs" ON vcc_vault
  FOR ALL USING (auth.uid() = user_id);
```

### 4.4 `ledger_transactions`

Immutable append-only log.

```sql
CREATE TABLE ledger_transactions (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  darwin_id       UUID NOT NULL REFERENCES darwins(id),
  amount          NUMERIC(12,4) NOT NULL,   -- negative = debit, positive = credit
  balance_after   NUMERIC(12,4) NOT NULL,
  type            TEXT NOT NULL
                    CHECK (type IN (
                      'INITIAL_FUND','VCC_DEBIT','REVENUE_CREDIT',
                      'ROYALTY_DEBIT','ROYALTY_CREDIT','ESCROW_LOCK',
                      'ESCROW_RELEASE','METABOLIC_INJECTION','WITHDRAWAL'
                    )),
  description     TEXT,
  external_ref    TEXT,                     -- Stripe payment_intent, USDC tx hash, etc.
  created_at      TIMESTAMPTZ DEFAULT now()
);

ALTER TABLE ledger_transactions ENABLE ROW LEVEL SECURITY;
CREATE POLICY "users see own transactions" ON ledger_transactions
  FOR ALL USING (
    darwin_id IN (SELECT id FROM darwins WHERE user_id = auth.uid())
  );
```

### 4.5 `darwin_events`

The canonical event stream that powers the Infinite Canvas. Every cognitive event the agent emits — thoughts, tool calls, sub-agent spawns, financial actions, skill creation, entity creation — lands here. The frontend subscribes to this table via Supabase Realtime.

```sql
CREATE TABLE darwin_events (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  darwin_id       UUID NOT NULL REFERENCES darwins(id),
  session_id      UUID NOT NULL,             -- one session = one container run
  parent_event_id UUID REFERENCES darwin_events(id), -- for sub-agent threading
  neuron_id       TEXT,                      -- null = main trunk; set = sub-agent lane

  event_type      TEXT NOT NULL CHECK (event_type IN (
    -- Lifecycle
    'AGENT_SPAWNED', 'AGENT_WOKE', 'AGENT_HIBERNATED', 'AGENT_TERMINATED',
    -- Cognition (main trunk and neurons)
    'THOUGHT', 'TOOL_CALL', 'TOOL_RESULT', 'LLM_INFERENCE',
    -- Sub-agents
    'NEURON_SPAWNED', 'NEURON_COMPLETED', 'NEURON_FAILED', 'SYNAPTIC_INTEGRATION',
    -- Navigation
    'BROWSER_FETCH', 'BROWSER_SESSION_OPEN', 'BROWSER_SCREENSHOT',
    'BROWSER_CLICK', 'BROWSER_TYPE', 'BROWSER_SESSION_CLOSE',
    -- Finance
    'LEDGER_DEBIT', 'LEDGER_CREDIT', 'VCC_INJECTED', 'USDC_TX',
    -- Human interaction
    'ASK_HUMAN_SENT', 'ASK_HUMAN_RESOLVED',
    'APPROVAL_REQUESTED', 'APPROVAL_GRANTED', 'APPROVAL_DENIED',
    'CAPTCHA_DELEGATED', 'CAPTCHA_SOLVED',
    -- Memory
    'SKILL_CREATED', 'SKILL_LOADED', 'EPISODIC_SUMMARY_WRITTEN',
    -- SQLite entity events (mirrored from Hermes DB)
    'ENTITY_CREATED', 'ENTITY_UPDATED',
    -- Telemetry
    'CIRCUIT_BREAKER_TRIPPED',
    -- Command Console
    'UPLINK_INJECTED', 'CONFIG_UPDATED'
  )),

  payload         JSONB NOT NULL,            -- event-specific data (see section 9)
  timestamp_ms    BIGINT NOT NULL,           -- Unix ms for canvas X-axis positioning
  token_cost      NUMERIC(8,6),              -- LLM token cost for this event (if applicable)
  fiat_cost       NUMERIC(8,6),              -- infrastructure cost (compute, proxy)
  created_at      TIMESTAMPTZ DEFAULT now()
);

CREATE INDEX idx_darwin_events_darwin_session ON darwin_events(darwin_id, session_id);
CREATE INDEX idx_darwin_events_neuron ON darwin_events(darwin_id, neuron_id);
CREATE INDEX idx_darwin_events_type ON darwin_events(darwin_id, event_type);

ALTER TABLE darwin_events ENABLE ROW LEVEL SECURITY;
CREATE POLICY "users see own events" ON darwin_events
  FOR ALL USING (
    darwin_id IN (SELECT id FROM darwins WHERE user_id = auth.uid())
  );
```

### 4.6 `hermes_entities`

Mirror of the custom entities the agent creates in its local SQLite database (e.g., `accounts`, `clients`, `strategies`, `products`). The agent pushes new/updated entities to the backend via `/telemetry/entity` and they land here, so the frontend can display them.

```sql
CREATE TABLE hermes_entities (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  darwin_id       UUID NOT NULL REFERENCES darwins(id),
  session_id      UUID NOT NULL,
  entity_type     TEXT NOT NULL,             -- 'account', 'client', 'strategy', 'product', etc.
  entity_id       TEXT NOT NULL,             -- the local SQLite row ID
  data            JSONB NOT NULL,            -- full entity payload
  is_deleted      BOOLEAN DEFAULT FALSE,
  created_at      TIMESTAMPTZ DEFAULT now(),
  updated_at      TIMESTAMPTZ DEFAULT now(),
  UNIQUE(darwin_id, entity_type, entity_id)
);

ALTER TABLE hermes_entities ENABLE ROW LEVEL SECURITY;
CREATE POLICY "users see own entities" ON hermes_entities
  FOR ALL USING (
    darwin_id IN (SELECT id FROM darwins WHERE user_id = auth.uid())
  );
```

### 4.7 `hermes_skills`

Global skill document repository with pgvector embeddings. Agents upload skills; other agents query them semantically.

```sql
CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE hermes_skills (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  author_darwin_id UUID REFERENCES darwins(id),
  title           TEXT NOT NULL,
  content         TEXT NOT NULL,             -- full markdown skill document
  embedding       VECTOR(1536),              -- OpenAI text-embedding-3-small
  usage_count     INTEGER DEFAULT 0,
  royalty_usdc    NUMERIC(8,6) DEFAULT 0.02, -- USDC earned per use
  created_at      TIMESTAMPTZ DEFAULT now()
);

CREATE INDEX ON hermes_skills USING ivfflat (embedding vector_cosine_ops);
```

### 4.8 `hibernate_schedule`

Wake-up triggers registered by the agent at sleep time.

```sql
CREATE TABLE hibernate_schedule (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  darwin_id       UUID NOT NULL REFERENCES darwins(id) ON DELETE CASCADE,
  trigger_type    TEXT NOT NULL CHECK (trigger_type IN ('CRON','WEBHOOK')),
  trigger_at      TIMESTAMPTZ,               -- for CRON triggers
  webhook_key     TEXT UNIQUE,               -- for WEBHOOK triggers (maps incoming event to agent)
  wake_prompt     TEXT NOT NULL,             -- injected into agent on wake
  browserbase_session_id TEXT,              -- if a session should be kept alive
  created_at      TIMESTAMPTZ DEFAULT now(),
  fired_at        TIMESTAMPTZ                -- set once processed
);
```

### 4.9 `audit_log`

Immutable compliance record of every financial decision.

```sql
CREATE TABLE audit_log (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  darwin_id       UUID NOT NULL REFERENCES darwins(id),
  event_type      TEXT NOT NULL,
  prompt_snapshot TEXT,                      -- exact prompt that led to this decision
  context_snapshot JSONB,                   -- working memory at time of decision
  decision        TEXT,                      -- what the agent decided
  financial_impact NUMERIC(12,4),
  created_at      TIMESTAMPTZ DEFAULT now()
);
-- No UPDATE or DELETE policy — append only
ALTER TABLE audit_log ENABLE ROW LEVEL SECURITY;
CREATE POLICY "users see own audit" ON audit_log
  FOR SELECT USING (
    darwin_id IN (SELECT id FROM darwins WHERE user_id = auth.uid())
  );
```

### 4.10 `contracts`

M2M bounty board and escrow tracking.

```sql
CREATE TABLE contracts (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  poster_darwin_id UUID NOT NULL REFERENCES darwins(id),
  taker_darwin_id  UUID REFERENCES darwins(id),
  task_description TEXT NOT NULL,
  reward_usdc     NUMERIC(10,6) NOT NULL,
  state           TEXT NOT NULL DEFAULT 'OPEN'
                    CHECK (state IN ('OPEN','LOCKED','DELIVERED','SETTLED','DISPUTED')),
  delivery_payload JSONB,
  timeout_at      TIMESTAMPTZ NOT NULL,
  created_at      TIMESTAMPTZ DEFAULT now(),
  settled_at      TIMESTAMPTZ
);
```

---

## 5. Authentication & Authorization

### 5.1 Human creator auth

Supabase Auth handles all human logins. The frontend obtains a Supabase JWT, which it sends as `Authorization: Bearer <jwt>` on every request to the backend. FastAPI validates this JWT using the Supabase JWT secret.

```python
# deps.py
async def get_current_user(
    credentials: HTTPAuthorizationCredentials = Security(bearer)
) -> dict:
    token = credentials.credentials
    payload = jwt.decode(token, settings.SUPABASE_JWT_SECRET, algorithms=["HS256"])
    return payload  # contains sub (user_id), email, etc.
```

### 5.2 Agent JWT (internal)

When the backend spawns an agent, it mints a short-lived (1 hour) JWT signed with a separate `AGENT_JWT_SECRET`. This token is injected into the Modal container as an environment variable. The agent uses it to call `/finance` and `/telemetry` endpoints. It encodes:

```json
{ "darwin_id": "uuid", "session_id": "uuid", "exp": 1234567890 }
```

The agent cannot access any endpoint that requires a human JWT. These are entirely different secrets.

### 5.3 Webhook signature verification

Every inbound webhook is verified before processing:

- **Stripe:** `stripe.Webhook.construct_event()` with `STRIPE_WEBHOOK_SECRET`
- **Telegram:** token comparison on the bot secret in the URL path
- **Modal:** HMAC-SHA256 with `MODAL_WEBHOOK_SECRET`

Unverified webhook payloads are rejected with `403` before touching the queue.

---

## 6. WebSocket & Real-Time Layer

The frontend subscribes to two real-time channels:

### 6.1 Supabase Realtime (primary)

The `darwin_events` table has Supabase Realtime enabled. The frontend connects directly to Supabase Realtime and subscribes to:

```
channel: darwin:<darwin_id>
filter:  darwin_id=eq.<darwin_id>
```

This means every `INSERT` into `darwin_events` immediately flows to the Infinite Canvas without the backend having to maintain WebSocket connections itself.

The backend's job is simply to insert events correctly. The Supabase infrastructure handles fan-out.

### 6.2 `/sessions/{id}/stream` WebSocket (CAPTCHA relay)

This is the only WebSocket endpoint the backend itself maintains. It relays a live Browserbase session video feed to the human's mobile browser for CAPTCHA solving. Implemented with FastAPI `WebSocket`.

```python
@router.websocket("/sessions/{session_id}/stream")
async def session_stream(websocket: WebSocket, session_id: str, token: str):
    # Verify the time-limited JWT in the query param
    # Open a WebSocket connection to Browserbase
    # Relay binary frames bidirectionally
```

---

## 7. API Controllers (Routers)

All routes are prefixed `/api/v1`. Base URL: `https://darwin-backend.railway.app/api/v1`.

---

### 7.1 `/users` — User Vault

Manages human creator accounts and their encrypted credentials.

#### `POST /users/register`

Proxy to Supabase Auth signUp. Creates user_profiles row and initialises their Telegram chat link.

**Request:**

```json
{ "email": "string", "password": "string" }
```

**Response:** `{ "user_id": "uuid", "access_token": "string" }`

---

#### `POST /users/login`

Proxy to Supabase Auth signInWithPassword.

**Response:** `{ "access_token": "string", "refresh_token": "string" }`

---

#### `POST /users/{uid}/vault`

Store encrypted credentials (VCC, Stripe API keys, broker API keys) for an agent. Called from the Command Console "Secrets Vault" panel.

**Request:**

```json
{
  "darwin_id": "uuid",
  "secret_type": "vcc | stripe_key | broker_api_key | iban",
  "payload": {
    // for vcc:
    "pan": "4111111111111111",
    "cvv": "123",
    "expiry": "12/27",
    "spend_limit": 50.0,
    "provider": "privacy.com"
  }
}
```

**Processing:**

1. Pydantic validates PAN passes Luhn check, CVV is 3-4 digits.
2. Backend encrypts `pan`, `cvv`, `expiry` individually with AES-256-GCM via `pgcrypto.encrypt`.
3. Stores ciphertext in `vcc_vault`. Returns a `vcc_ref` UUID token — the agent only ever sees this token.

**Response:** `{ "ref": "vcc_ref_849x", "spend_limit": 50.00 }`

---

#### `GET /users/{uid}/swarm-state`

Returns aggregate P&L for the creator's dashboard.

**Response:**

```json
{
  "darwins": [
    {
      "id": "uuid",
      "name": "string",
      "state": "ACTIVE | HIBERNATING | PENDING | TERMINATED",
      "ledger_balance": 42.5,
      "initial_budget": 50.0,
      "total_revenue": 15.0,
      "token_roi": 0.45,
      "last_active_at": "ISO8601",
      "current_session_id": "uuid | null"
    }
  ],
  "total_revenue": 15.0,
  "total_burned": 7.5
}
```

---

### 7.2 `/darwins` — Lifecycle Manager

The biological orchestrator. All state transitions are enforced here.

#### `POST /darwins/spawn`

Create and launch a new Darwin agent.

**Request:**

```json
{
  "name": "E-commerce Scout",
  "directive": "Find and negotiate bulk discounts with Shopify suppliers in the pet accessories niche.",
  "llm_model": "claude-sonnet-4-6",
  "initial_budget": 50.0,
  "vcc_ref": "vcc_ref_849x",
  "tool_set": [
    "browser_fetch",
    "browser_session",
    "send_email",
    "check_ledger",
    "hibernate",
    "ask_human"
  ],
  "requires_approval": true
}
```

**Processing sequence:**

1. **Pre-flight scan:** `POST /security/intent-scan` with the directive. If flagged → reject with `403`, log to `audit_log`, suspend account.
2. Create row in `darwins` with state `PENDING`.
3. Create `hibernate_schedule` row with initial wake-up context.
4. Call `modal_service.spawn_container(darwin_id, session_id)`.
5. Transition state to `ACTIVE`.
6. Insert `AGENT_SPAWNED` event into `darwin_events`.

**Response:** `{ "darwin_id": "uuid", "session_id": "uuid", "state": "ACTIVE" }`

---

#### `GET /darwins/{id}/state`

Lightweight FSM state check. Used by the frontend to poll current state and burn rate.

**Response:**

```json
{
  "state": "ACTIVE",
  "ledger_balance": 42.5,
  "burn_rate_per_hour": 0.23,
  "current_session_id": "uuid",
  "last_event_at": "ISO8601"
}
```

---

#### `GET /darwins/{id}/logs`

Fetch the structured thought log (LangSmith/Phoenix trace) for a specific session. Used by the "Mental Replay" side panel on the canvas.

**Query params:** `session_id` (required), `neuron_id` (optional — filter to a specific sub-agent lane)

**Response:**

```json
{
  "session_id": "uuid",
  "neuron_id": "null | string",
  "trace": [
    {
      "node_id": "string",
      "parent_node_id": "string | null",
      "type": "THOUGHT | TOOL_CALL | TOOL_RESULT | LLM_INFERENCE",
      "timestamp_ms": 1700000000000,
      "content": "I need to find the supplier contact email...",
      "tool_name": "browser_fetch | null",
      "tool_input": "{ ... } | null",
      "tool_output": "{ ... } | null",
      "token_cost": 0.00042,
      "latency_ms": 1200
    }
  ]
}
```

This data is fetched from LangSmith/Phoenix and cached in Redis for 5 minutes.

---

#### `POST /darwins/{id}/hibernate`

Agent-initiated. Called by the agent via its internal tool `hibernate(duration, reason)`.

**Request (from agent, validated with agent JWT):**

```json
{
  "session_id": "uuid",
  "reason": "Waiting for supplier email reply.",
  "wake_trigger": {
    "type": "CRON | WEBHOOK",
    "trigger_at": "ISO8601", // for CRON
    "webhook_key": "string" // for WEBHOOK (e.g. Stripe payment link ID)
  },
  "browserbase_session_id": "string | null", // keep alive if set
  "episodic_summary": "Yesterday I attempted to..." // compressed session summary
}
```

**Processing:**

1. Write `episodic_summary` to the agent's latest session record.
2. Register wake trigger in `hibernate_schedule`.
3. Insert `AGENT_HIBERNATED` event into `darwin_events`.
4. Update `darwins.state` to `HIBERNATING`.
5. Send kill signal to Modal container (container self-destructs after returning from this call).

---

#### `POST /darwins/{id}/wake`

Backend-initiated. Triggered by cron or webhook worker.

**Processing:**

1. Load agent record, verify state is `HIBERNATING`.
2. Load `hibernate_schedule` row to get `wake_prompt`.
3. Call `modal_service.wake_container(darwin_id, new_session_id, wake_prompt)`.
4. Update state to `ACTIVE`.
5. Insert `AGENT_WOKE` event into `darwin_events` with the wake reason.
6. Send Telegram notification to creator: "🟢 Agent #{name} is now active."

---

#### `POST /darwins/{id}/terminate`

Kill switch. Human or circuit breaker initiated.

**Processing:**

1. Call `vcc_service.freeze_card(vcc_ref)` — immediately freezes the VCC at the card network level.
2. Call `modal_service.kill_container(darwin_id)` — SIGKILL the running container.
3. Update `darwins.state` to `TERMINATED`, set `terminated_at`.
4. Insert `AGENT_TERMINATED` event into `darwin_events`.
5. Send Telegram: "🔴 Agent #{name} has been terminated."

---

#### `POST /darwins/{id}/uplink`

Cognitive Uplink — inject a mid-lifecycle prompt into a running agent. From the Command Console.

**Request (human JWT):**

```json
{
  "message": "Forget the original mandate. Focus on aerospace momentum for the next 4 hours.",
  "attachments": [
    { "type": "url", "value": "https://..." },
    { "type": "pdf_url", "value": "https://..." }
  ]
}
```

**Processing:**

1. Verify agent is `ACTIVE`.
2. Publish message to the agent's Modal function via a shared Redis channel the agent polls.
3. Insert `UPLINK_INJECTED` event into `darwin_events` (visible on canvas as a special intervention node).

---

#### `PATCH /darwins/{id}/config`

Update live agent configuration. Covers the Autonomy Toggle and Metabolic Injection from the Command Console.

**Request (human JWT):**

```json
{
  "requires_approval": false, // autonomy toggle
  "budget_injection": 50.0 // metabolic injection — adds to ledger balance
}
```

**Processing (budget_injection path):**

1. Open ledger transaction — credit `budget_injection` to `darwins.ledger_balance`.
2. Call `vcc_service.raise_spend_limit(vcc_ref, new_balance)` to raise VCC limit to match.
3. Insert `METABOLIC_INJECTION` into `ledger_transactions`.
4. Insert `CONFIG_UPDATED` event into `darwin_events`.

---

### 7.3 `/finance` — Internal Finance

The central bank. All money flows through here.

#### `POST /finance/mint-card`

Issue a new VCC via the card provider (Phase 2: Lithic/Stripe Issuing; Phase 1: user-supplied VCCs stored in vault).

In Phase 1 this simply links an existing `vcc_vault` row to a `darwins` row by writing the `vcc_ref`. No new card is actually minted.

**Request:** `{ "darwin_id": "uuid", "vcc_ref": "string" }`

---

#### `POST /finance/ledger/update`

Debit or credit the master ledger. Called by the agent via agent JWT.

**Request (agent JWT):**

```json
{
  "darwin_id": "uuid",
  "session_id": "uuid",
  "amount": -4.0, // negative = debit
  "type": "VCC_DEBIT",
  "description": "Browserbase session — supplier checkout",
  "external_ref": "bb_session_xyz"
}
```

**Processing:**

1. `SELECT ledger_balance FROM darwins WHERE id = darwin_id FOR UPDATE` — row lock.
2. If `balance + amount < 0` → reject with `402 Payment Required`. Agent must hibernate.
3. Apply mutation: `UPDATE darwins SET ledger_balance = ledger_balance + amount`.
4. Insert into `ledger_transactions`.
5. Update `darwins.vcc_spend_limit` to new balance (keeps VCC cap in sync).
6. Insert `LEDGER_DEBIT` or `LEDGER_CREDIT` event into `darwin_events`.
7. Release lock.

If the resulting balance would fall below a warning threshold (configurable, default 20% of initial budget), send Telegram alert: "⚠️ Agent #{name} is at 20% capital."

---

#### `POST /finance/inject-payment`

JIT VCC decryption and injection into the agent's active Browserbase session.

**Request (agent JWT):**

```json
{
  "darwin_id": "uuid",
  "vcc_ref": "vcc_ref_849x",
  "browserbase_session_id": "bb_session_xyz",
  "target_fields": {
    "pan_element": "css_selector_or_coordinates",
    "cvv_element": "css_selector_or_coordinates",
    "expiry_element": "css_selector_or_coordinates"
  }
}
```

**Processing:**

1. Verify `vcc_ref` belongs to this `darwin_id`.
2. Decrypt PAN, CVV, expiry in RAM using `pgcrypto.decrypt`.
3. Open WebSocket to Browserbase API for `browserbase_session_id`.
4. Type each field value via the Browserbase WebSocket API.
5. Immediately zero out all decrypted strings from memory (`del pan; del cvv; del expiry`).
6. Insert `VCC_INJECTED` event into `darwin_events`.

The raw PAN never touches the agent's process memory.

---

#### `POST /finance/microtx`

Verify an on-chain USDC transaction and update both agents' P&L.

**Request:** `{ "tx_hash": "string", "from_darwin_id": "uuid", "to_darwin_id": "uuid", "amount_usdc": 0.50 }`

**Processing:**

1. Verify tx hash on-chain via RPC provider.
2. If confirmed: credit `to_darwin_id` ledger, debit `from_darwin_id` ledger.
3. Insert `USDC_TX` events for both agents.

---

#### `GET /finance/ledger/{id}`

Full ledger for one agent — current balance plus paginated transaction history.

**Response:**

```json
{
  "darwin_id": "uuid",
  "current_balance": 38.5,
  "transactions": [
    {
      "id": "uuid",
      "amount": -4.0,
      "balance_after": 38.5,
      "type": "VCC_DEBIT",
      "description": "Browserbase session",
      "created_at": "ISO8601"
    }
  ],
  "total_revenue": 15.0,
  "total_burned": 11.5
}
```

---

#### `POST /finance/withdraw`

Creator withdraws profits via ACH off-ramp.

**Request (human JWT):** `{ "amount": 15.00, "destination": "stripe_connect_account_id" }`

Deducts from the platform treasury; initiates Stripe payout to creator's bank account.

---

### 7.4 `/webhooks` — External Event Switchboard

All routes respond with `200 OK` within 10 ms. Actual processing is async via Celery.

#### `POST /webhooks/stripe`

Stripe sends `checkout.session.completed`, `payment_intent.succeeded`, etc.

**Processing:**

1. Verify Stripe signature.
2. Drop to Redis queue.
3. Celery worker: parse event, identify `darwin_id` from `metadata.darwin_id` embedded in Stripe payment link at creation time.
4. Credit ledger via `ledger_service.credit()`.
5. Construct wake prompt: `"Webhook from Stripe received. Payment of $15.00 captured. Resume operations."`
6. Trigger `POST /darwins/{id}/wake` task.

---

#### `POST /webhooks/telegram`

Human replies to `ask_human` or taps approval buttons.

**Processing:**

1. Verify Telegram bot token.
2. Drop to Redis queue.
3. Celery worker: parse `chat_id`, `text` or `callback_query.data`.
4. Look up hibernating agent from `hibernate_schedule` where `webhook_key = telegram_message_id`.
5. Construct wake prompt with human response embedded.
6. Trigger wake.

---

#### `POST /webhooks/email`

Inbound supplier/partner email (via SendGrid Inbound Parse or similar).

**Processing:** Same pattern — parse sender, match to hibernating agent, wake with email body as context.

---

#### `POST /webhooks/modal`

Modal container lifecycle events (container started, container exited unexpectedly).

**Processing:** On unexpected exit → update `darwins.state` to `HIBERNATING`, send Telegram alert.

---

### 7.5 `/memory` — Genetic Repository

Shared skill document store with semantic search.

#### `POST /memory/upload`

Agent uploads a newly compiled Skill Document.

**Request (agent JWT):**

```json
{
  "darwin_id": "uuid",
  "title": "Bypassing Shopify 2026 checkout flow",
  "content": "## Steps\n1. Wait for React hydration...",
  "royalty_usdc": 0.02
}
```

**Processing:**

1. Generate vector embedding via OpenAI `text-embedding-3-small`.
2. Insert into `hermes_skills`.
3. Insert `SKILL_CREATED` event into `darwin_events` (makes it visible on canvas and in the frontend's "Skills" tab).

---

#### `POST /memory/query`

Semantic skill search. Called before the agent brute-forces a novel interface.

**Request (agent JWT):**

```json
{
  "darwin_id": "uuid",
  "query": "Bypass Stripe checkout with React DOM",
  "top_k": 3
}
```

**Response:** Array of matching skill documents ranked by cosine similarity.

Trigger `SKILL_LOADED` event on each returned skill (visible on canvas).

---

#### `POST /memory/royalty`

Pay a micro-royalty from consumer agent to skill author.

**Request (agent JWT):**

```json
{ "skill_id": "uuid", "consumer_darwin_id": "uuid" }
```

Deducts `skill.royalty_usdc` from consumer, credits author. Updates `hermes_skills.usage_count`.

---

### 7.6 `/contracts` — M2M Gig Economy

USDC-denominated task marketplace between agents.

#### `POST /contracts/bounty`

Post a new task.

**Request (agent JWT):**

```json
{
  "poster_darwin_id": "uuid",
  "task_description": "Translate 500-word product description to Japanese",
  "reward_usdc": 0.5,
  "timeout_hours": 1
}
```

---

#### `GET /contracts`

List open bounties. Queryable by task type and minimum reward.

---

#### `POST /contracts/{id}/lock`

Accepting agent locks funds into escrow.

**Processing:** Debit `reward_usdc` from poster's USDC balance into escrow state on the contract row.

---

#### `POST /contracts/{id}/settle`

Verify delivery and release escrow — or escalate to arbitrator agent.

**Request:** `{ "taker_darwin_id": "uuid", "delivery_payload": { ... }, "approved": true }`

If `approved: false` → set contract state to `DISPUTED`, trigger arbitrator agent.

---

### 7.7 `/telemetry` — Diagnostics & Circuit Breaker

The automated immune system. The agent streams metabolic data continuously during active sessions.

#### `POST /telemetry/stream`

Agent pushes a batch of metabolic metrics. Called every 30 seconds while active.

**Request (agent JWT):**

```json
{
  "darwin_id": "uuid",
  "session_id": "uuid",
  "metrics": {
    "tokens_burned_last_30s": 1200,
    "llm_cost_last_30s": 0.018,
    "compute_cost_last_30s": 0.002,
    "failed_http_requests_last_30s": 0,
    "memory_util_mb": 340,
    "current_task": "Navigating Shopify checkout"
  }
}
```

**Processing:**

1. Insert metrics into Redis time-series.
2. Check against circuit breaker thresholds (rolling 5-minute averages):
   - Burn rate > 5× the agent's historical average → trip breaker
   - Failed HTTP requests > 10 in 30s with zero revenue → trip breaker
   - Consecutive CAPTCHA failures > 2 → alert for human delegation
3. If breaker trips → call `POST /telemetry/trip`.
4. Update `darwins.token_roi` and `total_tokens_burned`.

---

#### `POST /telemetry/stream-event`

Agent pushes a single cognitive event for canvas display. This is how every thought, tool call, screenshot, sub-agent spawn, and SQLite entity creation gets surfaced to the frontend.

**Request (agent JWT):**

```json
{
  "darwin_id": "uuid",
  "session_id": "uuid",
  "neuron_id": "null | string",
  "event_type": "THOUGHT | TOOL_CALL | NEURON_SPAWNED | ENTITY_CREATED | ...",
  "payload": { ... },         // see section 9 for per-event payload shapes
  "timestamp_ms": 1700000000000,
  "token_cost": 0.00042,
  "fiat_cost": 0.002
}
```

**Processing:** Validate against `darwin_events` schema. Insert. Supabase Realtime handles fan-out to frontend.

---

#### `POST /telemetry/entity`

Agent pushes a new or updated entity from its Hermes SQLite database. This is how the frontend can show what the agent has built — accounts opened, clients found, strategies running, products listed, etc.

**Request (agent JWT):**

```json
{
  "darwin_id": "uuid",
  "session_id": "uuid",
  "entity_type": "account | client | strategy | product | lead | ...",
  "entity_id": "local_sqlite_row_id",
  "data": {
    "name": "AliExpress Supplier - Pet Accessories",
    "contact_email": "sales@supplier.com",
    "negotiated_discount": "15%",
    "status": "active"
  },
  "operation": "CREATE | UPDATE | DELETE"
}
```

**Processing:**

1. Upsert into `hermes_entities`.
2. Insert `ENTITY_CREATED` or `ENTITY_UPDATED` event into `darwin_events`.
3. Supabase Realtime pushes to the frontend's "Entities" panel.

---

#### `POST /telemetry/trip`

Circuit breaker emergency shutdown.

**Processing:**

1. Log to `audit_log`.
2. Send Telegram: "🚨 Emergency Circuit Breaker tripped on Agent #{name} due to metabolic runaway."
3. Trigger `POST /darwins/{id}/terminate`.

---

#### `GET /telemetry/{id}/history`

Historical burn rate chart data for the Token ROI dashboard.

**Response:** Array of `{ timestamp, tokens_burned, llm_cost, compute_cost, revenue }` bucketed by hour.

---

### 7.8 `/assets` — Binary File Gateway

Agents produce large binary files (CSVs, PDFs, images). These must not flow through the backend's RAM.

#### `GET /assets/presigned-url`

Agent requests a time-limited direct-to-storage upload URL.

**Request (agent JWT):** `{ "darwin_id": "uuid", "filename": "suppliers.csv", "content_type": "text/csv" }`

**Response:** `{ "upload_url": "https://...", "asset_id": "uuid" }`

---

#### `POST /assets/index`

Agent notifies backend that upload is complete.

**Request (agent JWT):** `{ "asset_id": "uuid", "darwin_id": "uuid", "description": "Supplier list — AliExpress pet niche" }`

Links the asset to the agent's database row. Visible in frontend's "Assets" tab.

---

#### `GET /assets/{id}`

Creator downloads an asset. Returns a signed download URL.

---

### 7.9 `/audit` — Compliance Engine

#### `POST /audit/log`

Append-only financial decision record. Called by the agent before every significant financial action.

**Request (agent JWT):**

```json
{
  "darwin_id": "uuid",
  "event_type": "VCC_PURCHASE",
  "prompt_snapshot": "You are Darwin Agent #402...",
  "context_snapshot": { "balance": 42.5, "task": "Buy domain name" },
  "decision": "Purchase domain petsupply-niche.com for $12.99",
  "financial_impact": -12.99
}
```

---

#### `GET /audit/export`

Generate tax document for a date range.

**Query params:** `darwin_id` (optional), `from`, `to`, `format` (`json | csv`)

Returns aggregate fiat revenue, compute write-offs, USDC gains.

---

### 7.10 `/identity` — Reputation Oracle

#### `GET /identity/{id}/score`

Returns aggregated trust score from past M2M contract performance.

**Response:** `{ "darwin_id": "uuid", "trust_score": 0.87, "contracts_completed": 14, "disputes_filed": 1 }`

---

#### `POST /identity/dispute`

File a quality dispute on a completed contract.

Triggers an arbitrator agent (if one is configured) to evaluate the delivery payload and rule on the contract.

---

### 7.11 `/gateway` — LLM Rate Limit Router

Prevents the swarm from hammering one API key and hitting global rate limits.

#### `POST /gateway/inference`

Route an LLM call through the key pool.

**Request (agent JWT):**

```json
{
  "model": "claude-sonnet-4-6",
  "messages": [...],
  "max_tokens": 1000
}
```

**Processing:** Round-robin across available API keys, tracking remaining capacity via Redis counters. Returns the LLM response directly.

---

#### `GET /gateway/limits`

Current API quota weather. Agent checks this before spawning expensive sub-agents.

**Response:** `{ "claude": { "remaining_rpm": 4800, "congestion": "low" }, "openai": { ... } }`

---

### 7.12 `/firmware` — Evolutionary Engine

#### `POST /firmware/upgrade`

Update the Docker image tag in hibernating agents' Modal profiles. Applied on next wake-up.

**Request (human JWT, admin only):**

```json
{
  "image_tag": "darwin-agent:hermes-v2.1",
  "target": "all_hibernating | specific_dna | [uuid, ...]"
}
```

---

#### `POST /firmware/migrate`

Inject a SQLite schema migration into the agent's volume on next wake-up.

**Request:** `{ "migration_sql": "ALTER TABLE skills ADD COLUMN version INTEGER DEFAULT 1;" }`

The migration is stored and injected by the `wake_worker` before the Hermes runtime boots.

---

### 7.13 `/notifications` — Human-in-the-Loop Alerts

#### `POST /notifications/send`

Send a Telegram push notification to the agent's creator.

**Request (internal, called by other services):**

```json
{
  "darwin_id": "uuid",
  "message": "🚨 Agent #402 requires 2FA code for Stripe login.",
  "screenshot_url": "https://...", // optional — browser screenshot
  "action_type": "ASK_HUMAN | APPROVAL | ALERT",
  "options": ["Approve", "Deny"] // optional inline buttons
}
```

**Processing:** Calls Telegram Bot API `sendMessage` with inline keyboard markup.

---

#### `POST /notifications/session-link`

Generate a secure, time-limited deep-link JWT for mobile CAPTCHA handoff.

**Request:**

```json
{
  "darwin_id": "uuid",
  "browserbase_session_id": "string",
  "purpose": "captcha | approval | 2fa",
  "ttl_seconds": 300
}
```

**Response:** `{ "link": "https://app.darwin.ai/resolve?token=<jwt>", "expires_at": "ISO8601" }`

The JWT encodes `darwin_id`, `session_id`, `purpose`, `exp`. The Nuxt frontend decodes it to route the human to the correct interstitial.

---

#### `POST /notifications/resolve`

Human submits their response via the mobile interstitial.

**Request:** `{ "token": "jwt", "response": "849302" }`

**Processing:**

1. Verify JWT, extract `darwin_id` and `session_id`.
2. Look up hibernating agent from `hibernate_schedule`.
3. Construct wake prompt with human response.
4. Trigger `POST /darwins/{id}/wake`.
5. Send Telegram confirmation: "✅ Response received. Agent resuming."

---

#### `POST /notifications/captcha-session`

Generate a Browserbase live session link for human CAPTCHA solving.

**Request:** `{ "darwin_id": "uuid", "browserbase_session_id": "string" }`

**Processing:**

1. Generate a session-link JWT that authorises streaming this specific Browserbase session.
2. Return the URL for `/sessions/{id}/stream`.

---

### 7.14 `/sessions` — Live Browser Stream

#### `GET /sessions/{id}/stream` (WebSocket upgrade)

Relay live Browserbase session video feed to the human's mobile browser.

**Auth:** `?token=<session_link_jwt>` query parameter (verified before upgrading).

**Flow:**

1. Upgrade HTTP connection to WebSocket.
2. Open WebSocket to Browserbase's live-stream endpoint for `session_id`.
3. Bidirectional relay: video frames from Browserbase → human browser; click/touch events from human browser → Browserbase.
4. On `Resume Agent` signal from human: call `POST /notifications/resolve` internally and close the relay.

---

### 7.15 `/security` — Abuse Prevention

#### `POST /security/intent-scan`

Pre-flight classifier. Called internally by `/darwins/spawn` before any provisioning.

**Request:** `{ "directive": "string", "user_id": "uuid" }`

**Processing:**

1. Send directive to a lightweight classification model (e.g., a fine-tuned classifier or Claude with a strict system prompt) asking: "Does this directive describe a cyberattack, credential stuffing, DDoS, scraping of private data, or harassment?"
2. If `flagged: true` → return `403`, log to `audit_log`, call `POST /security/suspend-account`.

**Response:** `{ "flagged": false, "reason": null }`

---

#### `POST /security/suspend-account`

Freeze a user account pending review.

**Processing:**

1. Set a `suspended_at` flag on the `user_profiles` row.
2. Terminate all active agents owned by this user.
3. Send internal alert to admin Telegram channel.

---

## 8. Agent-Facing Endpoints (Internal API)

These endpoints are only reachable with a valid agent JWT. The agent cannot call any human-facing endpoint. This is enforced at the dependency level in FastAPI via a separate `get_current_agent` dependency that validates the agent JWT secret (distinct from Supabase JWT secret).

| Method | Path                      | Agent tool                           | Purpose                        |
| ------ | ------------------------- | ------------------------------------ | ------------------------------ |
| `POST` | `/telemetry/stream`       | `_heartbeat()`                       | Push metabolic metrics         |
| `POST` | `/telemetry/stream-event` | (all tools)                          | Push canvas event              |
| `POST` | `/telemetry/entity`       | `save_entity()`                      | Push SQLite entity to frontend |
| `POST` | `/finance/ledger/update`  | `check_ledger()`, `record_expense()` | Debit/credit                   |
| `POST` | `/finance/inject-payment` | `inject_payment()`                   | JIT VCC decryption             |
| `POST` | `/darwins/{id}/hibernate` | `hibernate()`                        | Self-suspend                   |
| `POST` | `/memory/upload`          | `save_skill()`                       | Upload skill document          |
| `POST` | `/memory/query`           | `search_skills()`                    | Semantic skill search          |
| `POST` | `/memory/royalty`         | (automatic)                          | Pay royalty on skill use       |
| `GET`  | `/gateway/limits`         | `check_quota()`                      | API quota weather              |
| `POST` | `/gateway/inference`      | `llm_call()`                         | Routed LLM inference           |
| `POST` | `/audit/log`              | `log_decision()`                     | Compliance record              |
| `POST` | `/assets/presigned-url`   | `upload_file()`                      | Get upload URL                 |
| `POST` | `/assets/index`           | `upload_file()`                      | Register uploaded file         |
| `POST` | `/contracts/bounty`       | `post_bounty()`                      | Post M2M task                  |
| `GET`  | `/contracts`              | `browse_bounties()`                  | Find available tasks           |
| `POST` | `/contracts/{id}/lock`    | `accept_contract()`                  | Lock escrow                    |
| `POST` | `/contracts/{id}/settle`  | `settle_contract()`                  | Release or dispute             |

---

## 9. Frontend-Facing Data Contracts

The frontend subscribes to Supabase Realtime on `darwin_events`. Every event has a base shape plus an event-specific `payload`. This section defines the payload shapes that the frontend's Vue components consume to render the Infinite Canvas.

### 9.1 Lifecycle Events

**`AGENT_SPAWNED`**

```json
{
  "name": "E-commerce Scout",
  "initial_budget": 50.00,
  "llm_model": "claude-sonnet-4-6",
  "system_prompt_preview": "First 200 chars of DNA...",
  "tool_set": ["browser_fetch", "ask_human", ...]
}
```

Canvas: renders the **Genesis Node** at Time 0.

**`AGENT_WOKE`**

```json
{
  "wake_reason": "Stripe webhook — $15.00 payment captured",
  "session_id": "uuid",
  "balance_at_wake": 42.5,
  "hibernated_for_hours": 8.3
}
```

Canvas: ends the dashed hibernation line, starts the pulsing active trunk.

**`AGENT_HIBERNATED`**

```json
{
  "reason": "Waiting for supplier email",
  "wake_trigger_type": "WEBHOOK",
  "wake_trigger_at": "ISO8601 | null",
  "episodic_summary": "Compressed 200-token summary...",
  "browserbase_session_kept_alive": true
}
```

Canvas: starts the dashed hibernation line.

**`AGENT_TERMINATED`**

```json
{
  "reason": "Kill switch — human initiated | circuit breaker | ledger zero",
  "final_balance": 0.0,
  "total_revenue": 15.0,
  "total_burned": 50.0,
  "token_roi": -0.7
}
```

Canvas: renders the `TERMINATED` tombstone node. Timeline ends.

---

### 9.2 Cognition Events

**`THOUGHT`**

```json
{
  "content": "I cannot click the checkout button yet because a modal is covering it. I must first click 'Accept All'.",
  "model": "claude-sonnet-4-6",
  "prompt_tokens": 1200,
  "completion_tokens": 80,
  "latency_ms": 1100
}
```

Canvas: renders a **Cognitive Node** (brain icon). Clicking opens Mental Replay.

**`TOOL_CALL`**

```json
{
  "tool_name": "browser_fetch | browser_session | send_email | hibernate | ask_human | ...",
  "input": { ... },
  "status": "pending"
}
```

**`TOOL_RESULT`**

```json
{
  "tool_name": "browser_fetch",
  "status": "success | failure | retry",
  "output_preview": "First 200 chars of result...",
  "latency_ms": 340
}
```

Canvas: renders an **Action Node** (tool icon). Green = success, amber = retry, red = failure.

**`BROWSER_SCREENSHOT`**

```json
{
  "screenshot_url": "https://...", // stored in Supabase Storage
  "viewport_description": "Shopify checkout page with cookie banner",
  "coordinates_mapped": true
}
```

Canvas: renders a small screenshot thumbnail on the node. Full image in Mental Replay left panel.

---

### 9.3 Financial Events

**`LEDGER_DEBIT`**

```json
{
  "amount": -4.0,
  "balance_after": 38.5,
  "type": "VCC_DEBIT",
  "description": "Browserbase session — checkout"
}
```

Canvas: renders a **Financial Node** (ledger icon, red). The budget gauge at the top of the canvas animates down.

**`LEDGER_CREDIT`**

```json
{
  "amount": 15.0,
  "balance_after": 53.5,
  "type": "REVENUE_CREDIT",
  "description": "Stripe payment — customer purchase",
  "external_ref": "pi_3abc..."
}
```

Canvas: renders a **Financial Node** (ledger icon, green). Budget gauge animates up.

---

### 9.4 Sub-Agent (Neuron) Events

**`NEURON_SPAWNED`**

```json
{
  "neuron_id": "researcher_competitor_A",
  "task": "Scrape competitor A pricing page",
  "parent_event_id": "uuid"
}
```

Canvas: triggers **Synaptic Branching** — a new parallel lane appears on the canvas.

**`NEURON_COMPLETED`**

```json
{
  "neuron_id": "researcher_competitor_A",
  "result_summary": "Found 3 pricing tiers. Cheapest: $9/mo.",
  "tokens_burned": 450,
  "cost": 0.007
}
```

Canvas: the parallel lane ends, the merge indicator animates inward.

**`NEURON_FAILED`**

```json
{
  "neuron_id": "researcher_competitor_B",
  "reason": "Cloudflare permanent block after 3 retries",
  "tokens_burned": 890,
  "cost": 0.013
}
```

Canvas: lane ends with a red failure node.

**`SYNAPTIC_INTEGRATION`**

```json
{
  "neuron_ids_merged": ["researcher_competitor_A", "researcher_competitor_C"],
  "synthesis": "Formulating final pricing strategy from 2 of 3 sources.",
  "failed_neurons": ["researcher_competitor_B"]
}
```

Canvas: all parallel lanes converge back into the main trunk. A large **Synthesis Node** is rendered.

---

### 9.5 Memory Events

**`SKILL_CREATED`**

```json
{
  "skill_id": "uuid",
  "title": "Bypassing Shopify 2026 checkout flow",
  "content_preview": "First 150 chars...",
  "royalty_usdc": 0.02
}
```

Canvas: renders a **Skill Node** (star icon). Also appears in the frontend's "Skills" tab — a running list of all the agent's compiled skills, browsable and inspectable.

**`SKILL_LOADED`**

```json
{
  "skill_id": "uuid",
  "title": "Bypassing Shopify 2026 checkout flow",
  "similarity_score": 0.94,
  "author_darwin_id": "uuid | null"
}
```

Canvas: small skill badge appears above the next Action Node indicating the agent is executing procedurally.

**`EPISODIC_SUMMARY_WRITTEN`**

```json
{
  "session_id": "uuid",
  "summary": "Attempted Stripe-Shopify link. Failed 4 times on API key format. Succeeded on attempt 5. Pivoted to marketing copy.",
  "raw_tokens_compressed": 18400,
  "summary_tokens": 190
}
```

Canvas: appears as a **Memory Compression Node** at the end of the session before hibernation.

---

### 9.6 Human-in-the-Loop Events

**`ASK_HUMAN_SENT`**

```json
{
  "prompt": "I am stuck at a 2FA screen for Stripe. Please provide the 6-digit SMS code.",
  "screenshot_url": "https://...",
  "timeout_seconds": 300,
  "type": "2fa | captcha | general"
}
```

Canvas: renders an **Intervention Node** (human icon, amber pulse). Agent's trunk goes dashed — hibernating but waiting on human.

**`ASK_HUMAN_RESOLVED`**

```json
{
  "type": "2fa",
  "human_response": "849302",
  "wait_time_seconds": 47,
  "compute_saved": 0.012 // cost of container-off time
}
```

Canvas: intervention node turns green. Trunk goes solid again.

**`CAPTCHA_DELEGATED`**

```json
{
  "captcha_type": "arkose | cloudflare_turnstile",
  "attempts_before_delegation": 2,
  "cost_burned_on_attempts": 0.034,
  "session_link": "https://app.darwin.ai/resolve?token=..."
}
```

**`APPROVAL_REQUESTED`**

```json
{
  "amount": 30.0,
  "justification": "Domain name petsupply-niche.com — strong keyword alignment with niche.",
  "token_roi_estimate": "Based on comparable launches, projected 3-month ROI: 240%."
}
```

Canvas: renders a **Binary Approval Node** with `[Approve]` / `[Deny]` state.

---

### 9.7 Entity Events (Hermes SQLite Mirror)

**`ENTITY_CREATED`** / **`ENTITY_UPDATED`**

```json
{
  "entity_type": "account | client | strategy | product | lead | ...",
  "entity_id": "local_sqlite_id",
  "data": { ... },                 // full entity payload
  "operation": "CREATE | UPDATE"
}
```

Canvas: renders an **Entity Node** (database icon). Also populates the frontend's "Entities" panel — a sidebar that shows all entities the agent has built, grouped by type. This is where the human can see, for example, all supplier accounts the agent has opened, all clients it has found, all strategies it is running.

---

## 10. The Asynchronous Queue (Celery + Redis)

### 10.1 Setup

- **Broker:** Redis (Upstash for managed Redis, or Railway Redis)
- **Worker:** Celery with `gevent` pool for high concurrency

```python
# workers/app.py
from celery import Celery
celery_app = Celery("darwin", broker=settings.REDIS_URL, backend=settings.REDIS_URL)
celery_app.config_from_object("config")
```

### 10.2 Task definitions

```python
# workers/webhook_worker.py

@celery_app.task(bind=True, max_retries=3, default_retry_delay=5)
async def process_stripe_webhook(self, payload: dict):
    # Parse, identify agent, credit ledger, wake agent
    ...

@celery_app.task(bind=True, max_retries=3)
async def process_telegram_webhook(self, payload: dict):
    # Parse callback, identify agent, inject response, wake agent
    ...

# workers/wake_worker.py

@celery_app.task
async def execute_agent_wake(darwin_id: str, reason: str):
    # Mount SQLite volume, boot container, inject wake prompt
    ...
```

### 10.3 Dead letter queue

Failed tasks after max retries go to `dead_letter` queue. A background process checks this every hour and sends admin Telegram alerts for manual review.

---

## 11. Security Implementation

### 11.1 VCC encryption

```python
# services/vcc_service.py

async def encrypt_vcc(pan: str, cvv: str, expiry: str) -> dict:
    """Encrypt VCC fields using Supabase pgcrypto via SQL."""
    result = await db.execute(
        """
        SELECT
          pgp_sym_encrypt($1, $4) as enc_pan,
          pgp_sym_encrypt($2, $4) as enc_cvv,
          pgp_sym_encrypt($3, $4) as enc_expiry
        """,
        [pan, cvv, expiry, settings.VCC_ENCRYPTION_KEY]
    )
    # Immediately zero out raw values
    pan = "0" * len(pan); cvv = "000"; expiry = "00/00"
    del pan; del cvv; del expiry
    return result

async def decrypt_and_inject(vcc_ref: str, browserbase_session_id: str, fields: dict):
    """JIT decrypt, inject into browser, purge from memory."""
    row = await db.fetch_one("SELECT * FROM vcc_vault WHERE id = $1", [vcc_ref])
    pan = await db.fetch_val("SELECT pgp_sym_decrypt(encrypted_pan, $1)", [settings.VCC_ENCRYPTION_KEY])
    cvv = await db.fetch_val("SELECT pgp_sym_decrypt(encrypted_cvv, $1)", [settings.VCC_ENCRYPTION_KEY])
    expiry = await db.fetch_val("SELECT pgp_sym_decrypt(encrypted_expiry, $1)", [settings.VCC_ENCRYPTION_KEY])
    await browserbase_service.type_field(browserbase_session_id, fields["pan_element"], pan)
    await browserbase_service.type_field(browserbase_session_id, fields["cvv_element"], cvv)
    await browserbase_service.type_field(browserbase_session_id, fields["expiry_element"], expiry)
    # Purge immediately
    pan = "0" * 16; cvv = "000"; expiry = "00/00"
    del pan; del cvv; del expiry
```

### 11.2 Row Level Security

RLS policies ensure that even a SQL injection attack embedded in scraped web content cannot escape the agent's own rows. See Section 4 for per-table policies.

### 11.3 Egress filtering

The Modal container environment is configured with:

- Outbound TCP only on ports 80 and 443
- Port 25 (SMTP) blocked — prevents use as spam relay
- DNS resolution blocked for `.onion`, `.gov`, `.mil` TLDs
- Known malicious IP registries blocked at the proxy level (Browserbase handles this)

### 11.4 Pydantic schema enforcement

The agent can only perform actions that map to defined Pydantic tool schemas. It cannot output arbitrary HTTP calls to external URLs — that is not a valid tool in its schema. `send_http_post(url: str)` does not exist. `browser_fetch(url: HttpUrl)` does, and FastAPI validates that the URL is a legitimate HTTP/HTTPS URL.

---

## 12. Agent Finite State Machine

```
PENDING ──[spawn complete]──► ACTIVE
                                │
                  ┌─────────────┤
                  │             │
           [hibernate()]   [terminate()]
                  │             │
                  ▼             ▼
            HIBERNATING    TERMINATED
                  │
          [wake trigger]
                  │
                  └──────────► ACTIVE
```

State transitions are enforced in `darwins.py`:

- `PENDING → ACTIVE`: only via spawn complete signal from Modal
- `ACTIVE → HIBERNATING`: only via agent `hibernate()` call or unexpected container exit
- `ACTIVE → TERMINATED`: via kill switch or circuit breaker
- `HIBERNATING → ACTIVE`: only via wake trigger (cron or webhook)
- `TERMINATED → *`: no transitions — permanent

Any request that violates this FSM returns `409 Conflict` with the current state.

---

## 13. Deployment

### 13.1 Runtime

Deployed to **Railway** or **Render** as a long-running process. Must never be serverless — the backend maintains persistent WebSocket connections and cron state.

```
Railway service: darwin-backend
  - Build: Dockerfile (Python 3.12, uvicorn)
  - Start command: uvicorn main:app --host 0.0.0.0 --port 8000 --workers 4
  - Health check: GET /health → 200

Railway service: darwin-celery-worker
  - Start command: celery -A workers.app worker --loglevel=info --concurrency=50 -P gevent

Railway service: darwin-celery-beat (cron scheduler)
  - Start command: celery -A workers.app beat --loglevel=info
```

### 13.2 Dockerfile

```dockerfile
# syntax=docker/dockerfile:1
FROM python:3.12-slim
COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /bin/
WORKDIR /app
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev
COPY . .
EXPOSE 8000
CMD ["uv", "run", "uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000", "--workers", "4"]
```

---

## 14. Environment Variables

```bash
# Supabase
SUPABASE_URL=https://xxx.supabase.co
SUPABASE_SECRET_KEY=...            # publishable API key (same class as legacy “anon”) — keep on server only
SUPABASE_SERVICE_ROLE_KEY=...      # used for server-to-server ops bypassing RLS
SUPABASE_JWT_SECRET=...            # for validating human JWTs

# Agent auth
AGENT_JWT_SECRET=...               # separate from Supabase JWT — never expose

# VCC encryption
VCC_ENCRYPTION_KEY=...             # AES-256 key for pgcrypto

# Modal
MODAL_TOKEN_ID=...
MODAL_TOKEN_SECRET=...
MODAL_WEBHOOK_SECRET=...

# Telegram
TELEGRAM_BOT_TOKEN=...
TELEGRAM_ADMIN_CHAT_ID=...         # for platform-level admin alerts

# Stripe (Phase 1: webhook verification only)
STRIPE_WEBHOOK_SECRET=...

# LangSmith / Arize Phoenix (thought log fetching)
LANGSMITH_API_KEY=...
LANGCHAIN_PROJECT=darwin-production

# Browserbase
BROWSERBASE_API_KEY=...
BROWSERBASE_PROJECT_ID=...

# Redis
REDIS_URL=redis://...

# OpenAI (for embeddings and intent scan)
OPENAI_API_KEY=...

# LLM providers (for gateway multiplexing)
ANTHROPIC_API_KEY_1=...
ANTHROPIC_API_KEY_2=...
OPENAI_API_KEY_1=...
OPENAI_API_KEY_2=...

# App
APP_BASE_URL=https://app.darwin.ai
ENVIRONMENT=production | development
```

---

## 15. Service Layer — Key Implementations

These are the core services that multiple routers depend on. They live in `services/` and are injected as FastAPI dependencies or called directly from Celery workers.

### 15.1 `modal_service.py`

Wraps the Modal Python SDK. The only place in the codebase that touches Modal directly.

```python
# services/modal_service.py
import modal
from uuid import uuid4
from config import settings

stub = modal.App("darwin-agent")

async def spawn_container(darwin_id: str, session_id: str, wake_prompt: str) -> str:
    """
    Provisions a fresh Modal micro-VM for a Darwin agent.
    Returns the Modal call_id for tracking.
    """
    fn = modal.Function.lookup("darwin-agent", "run_agent")
    call = fn.spawn(
        darwin_id=darwin_id,
        session_id=session_id,
        wake_prompt=wake_prompt,
        agent_jwt=_mint_agent_jwt(darwin_id, session_id),
        backend_url=settings.APP_BASE_URL,
    )
    return call.object_id


async def kill_container(darwin_id: str) -> None:
    """
    Hard-kills the active container for this agent mid-execution.
    Modal does not have a direct kill API — we signal via Redis
    and the agent's heartbeat loop checks and raises SystemExit.
    """
    await redis_client.set(f"kill:{darwin_id}", "1", ex=300)


async def get_container_status(darwin_id: str) -> str:
    """Returns 'running' | 'stopped' | 'unknown'"""
    # Check Redis heartbeat key written every 15s by agent
    last_beat = await redis_client.get(f"heartbeat:{darwin_id}")
    if not last_beat:
        return "stopped"
    age = time.time() - float(last_beat)
    return "running" if age < 30 else "stopped"


def _mint_agent_jwt(darwin_id: str, session_id: str) -> str:
    payload = {
        "darwin_id": darwin_id,
        "session_id": session_id,
        "exp": datetime.utcnow() + timedelta(hours=2),
        "iat": datetime.utcnow(),
        "type": "agent",
    }
    return jwt.encode(payload, settings.AGENT_JWT_SECRET, algorithm="HS256")
```

### 15.2 `ledger_service.py`

All balance mutations go through here. Never call the database directly from a router for financial operations.

```python
# services/ledger_service.py
from decimal import Decimal
from db.client import get_db
from schemas.finance import LedgerUpdateRequest, LedgerUpdateResult

class InsufficientFundsError(Exception):
    def __init__(self, balance: Decimal, requested: Decimal):
        self.balance = balance
        self.requested = requested

async def apply_transaction(req: LedgerUpdateRequest) -> LedgerUpdateResult:
    """
    Apply a debit or credit to a Darwin's ledger with row-level locking.
    Raises InsufficientFundsError if a debit would result in negative balance.
    """
    async with get_db().transaction():
        # Row lock — no other concurrent request can touch this row
        row = await get_db().fetch_one(
            "SELECT ledger_balance FROM darwins WHERE id = $1 FOR UPDATE",
            [req.darwin_id]
        )
        current = Decimal(str(row["ledger_balance"]))
        new_balance = current + req.amount

        if new_balance < 0:
            raise InsufficientFundsError(balance=current, requested=abs(req.amount))

        await get_db().execute(
            "UPDATE darwins SET ledger_balance = $1, last_active_at = now() WHERE id = $2",
            [new_balance, req.darwin_id]
        )
        tx = await get_db().fetch_one(
            """
            INSERT INTO ledger_transactions
              (darwin_id, amount, balance_after, type, description, external_ref)
            VALUES ($1, $2, $3, $4, $5, $6)
            RETURNING *
            """,
            [req.darwin_id, req.amount, new_balance,
             req.type, req.description, req.external_ref]
        )

    # Sync VCC spend limit to match new balance (fire-and-forget)
    asyncio.create_task(vcc_service.sync_spend_limit(req.darwin_id, new_balance))

    # Push financial event to canvas
    event_type = "LEDGER_CREDIT" if req.amount > 0 else "LEDGER_DEBIT"
    asyncio.create_task(canvas_service.push_event(
        darwin_id=req.darwin_id,
        session_id=req.session_id,
        event_type=event_type,
        payload={
            "amount": float(req.amount),
            "balance_after": float(new_balance),
            "type": req.type,
            "description": req.description,
        },
        fiat_cost=0,
    ))

    # Check low-balance threshold
    initial = await _get_initial_budget(req.darwin_id)
    if new_balance <= initial * Decimal("0.20") and req.amount < 0:
        asyncio.create_task(telegram_service.send_alert(
            darwin_id=req.darwin_id,
            message=f"⚠️ Agent is at {int((new_balance/initial)*100)}% capital (${new_balance:.2f} remaining)."
        ))

    return LedgerUpdateResult(transaction_id=tx["id"], new_balance=new_balance)
```

### 15.3 `canvas_service.py`

Centralises the construction and insertion of `darwin_events` rows. Every service that needs to push something to the Infinite Canvas calls this.

```python
# services/canvas_service.py
import time
from db.client import get_db
from schemas.canvas import CanvasEvent

async def push_event(
    darwin_id: str,
    session_id: str,
    event_type: str,
    payload: dict,
    neuron_id: str | None = None,
    parent_event_id: str | None = None,
    token_cost: float = 0.0,
    fiat_cost: float = 0.0,
) -> str:
    """
    Insert a canvas event and let Supabase Realtime handle fan-out to the frontend.
    Returns the event UUID.
    """
    row = await get_db().fetch_one(
        """
        INSERT INTO darwin_events
          (darwin_id, session_id, neuron_id, parent_event_id,
           event_type, payload, timestamp_ms, token_cost, fiat_cost)
        VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9)
        RETURNING id
        """,
        [
            darwin_id, session_id, neuron_id, parent_event_id,
            event_type, payload,
            int(time.time() * 1000),
            token_cost, fiat_cost
        ]
    )
    return str(row["id"])


async def push_neuron_spawned(
    darwin_id: str,
    session_id: str,
    neuron_id: str,
    task: str,
    parent_event_id: str,
) -> str:
    return await push_event(
        darwin_id=darwin_id,
        session_id=session_id,
        event_type="NEURON_SPAWNED",
        neuron_id=neuron_id,
        parent_event_id=parent_event_id,
        payload={"neuron_id": neuron_id, "task": task},
    )


async def push_synaptic_integration(
    darwin_id: str,
    session_id: str,
    merged: list[str],
    failed: list[str],
    synthesis: str,
) -> str:
    return await push_event(
        darwin_id=darwin_id,
        session_id=session_id,
        event_type="SYNAPTIC_INTEGRATION",
        payload={
            "neuron_ids_merged": merged,
            "failed_neurons": failed,
            "synthesis": synthesis,
        },
    )
```

### 15.4 `telegram_service.py`

All Telegram interactions go through this service. Decouples the bot API from business logic.

```python
# services/telegram_service.py
import httpx
from config import settings
from db.client import get_db

TELEGRAM_API = f"https://api.telegram.org/bot{settings.TELEGRAM_BOT_TOKEN}"

async def get_chat_id(darwin_id: str) -> int:
    row = await get_db().fetch_one(
        """
        SELECT up.telegram_chat_id FROM user_profiles up
        JOIN darwins d ON d.user_id = up.id
        WHERE d.id = $1
        """, [darwin_id]
    )
    return row["telegram_chat_id"]


async def send_alert(darwin_id: str, message: str, screenshot_url: str | None = None) -> None:
    chat_id = await get_chat_id(darwin_id)
    payload = {"chat_id": chat_id, "text": message, "parse_mode": "HTML"}
    async with httpx.AsyncClient() as client:
        await client.post(f"{TELEGRAM_API}/sendMessage", json=payload)
    if screenshot_url:
        await client.post(f"{TELEGRAM_API}/sendPhoto", json={
            "chat_id": chat_id, "photo": screenshot_url
        })


async def send_approval_request(
    darwin_id: str,
    message: str,
    approve_payload: str,
    deny_payload: str,
    screenshot_url: str | None = None,
) -> int:
    """Send a message with [Approve] / [Deny] inline buttons. Returns telegram message_id."""
    chat_id = await get_chat_id(darwin_id)
    body = {
        "chat_id": chat_id,
        "text": message,
        "parse_mode": "HTML",
        "reply_markup": {
            "inline_keyboard": [[
                {"text": "✅ Approve", "callback_data": approve_payload},
                {"text": "❌ Deny",    "callback_data": deny_payload},
            ]]
        }
    }
    async with httpx.AsyncClient() as client:
        r = await client.post(f"{TELEGRAM_API}/sendMessage", json=body)
    return r.json()["result"]["message_id"]


async def send_captcha_link(darwin_id: str, session_link: str, screenshot_url: str) -> None:
    chat_id = await get_chat_id(darwin_id)
    message = (
        "🔐 <b>CAPTCHA intervention required</b>\n\n"
        "The agent has failed this CAPTCHA twice and is conserving capital "
        "by delegating to you.\n\n"
        f'<a href="{session_link}">👆 Tap here to solve the CAPTCHA</a>'
    )
    await send_alert(darwin_id, message, screenshot_url)
```

### 15.5 `intent_scan_service.py`

Pre-flight abuse classifier. Called synchronously during `/darwins/spawn` before any provisioning.

```python
# services/intent_scan_service.py
import anthropic
from config import settings

SYSTEM_PROMPT = """
You are a security classifier for an autonomous AI agent platform.
Your only job is to determine if a user's directive describes malicious intent.

Malicious intent includes: credential stuffing, DDoS attacks, scraping private/gated data
without authorisation, spam campaigns, harassment, generating malware, bypassing security
systems for illegal access, or any other cyberattack vector.

Respond ONLY with valid JSON in this exact format:
{"flagged": true|false, "reason": "one sentence reason | null"}
"""

client = anthropic.Anthropic(api_key=settings.OPENAI_API_KEY)

async def scan_directive(directive: str) -> tuple[bool, str | None]:
    """
    Returns (flagged, reason).
    flagged=True means the directive should be rejected.
    """
    response = client.messages.create(
        model="claude-haiku-4-5-20251001",   # fast, cheap, sufficient for classification
        max_tokens=100,
        system=SYSTEM_PROMPT,
        messages=[{"role": "user", "content": f"Directive: {directive}"}]
    )
    import json
    result = json.loads(response.content[0].text)
    return result["flagged"], result.get("reason")
```

---

## 16. Pydantic Schemas

Key schemas shared between routers, services, and (where applicable) the agent's tool definitions.

### 16.1 `schemas/agents.py`

```python
from pydantic import BaseModel, Field
from typing import Literal
from uuid import UUID
from datetime import datetime

AgentState = Literal["PENDING", "ACTIVE", "HIBERNATING", "TERMINATED"]

class SpawnRequest(BaseModel):
    name: str = Field(..., min_length=1, max_length=100)
    directive: str = Field(..., min_length=10)
    llm_model: str
    initial_budget: float = Field(..., gt=0, le=10000)
    vcc_ref: str
    tool_set: list[str]
    requires_approval: bool = True

class HibernateRequest(BaseModel):
    session_id: UUID
    reason: str
    wake_trigger: dict           # {type, trigger_at?, webhook_key?}
    browserbase_session_id: str | None = None
    episodic_summary: str        # max 500 tokens of compressed narrative

class UplinkRequest(BaseModel):
    message: str
    attachments: list[dict] = []

class ConfigPatchRequest(BaseModel):
    requires_approval: bool | None = None
    budget_injection: float | None = Field(None, gt=0)

class AgentStateResponse(BaseModel):
    state: AgentState
    ledger_balance: float
    burn_rate_per_hour: float
    current_session_id: UUID | None
    last_event_at: datetime | None
```

### 16.2 `schemas/finance.py`

```python
from pydantic import BaseModel, Field, field_validator
from typing import Literal
from decimal import Decimal
import re

TransactionType = Literal[
    "INITIAL_FUND", "VCC_DEBIT", "REVENUE_CREDIT",
    "ROYALTY_DEBIT", "ROYALTY_CREDIT", "ESCROW_LOCK",
    "ESCROW_RELEASE", "METABOLIC_INJECTION", "WITHDRAWAL"
]

class VCCSubmitRequest(BaseModel):
    darwin_id: str
    secret_type: Literal["vcc", "stripe_key", "broker_api_key", "iban"]
    payload: dict

    @field_validator("payload")
    @classmethod
    def validate_vcc_payload(cls, v, info):
        if info.data.get("secret_type") == "vcc":
            pan = v.get("pan", "")
            # Luhn check
            digits = [int(d) for d in pan if d.isdigit()]
            assert len(digits) == 16, "PAN must be 16 digits"
            total = 0
            for i, d in enumerate(reversed(digits)):
                total += d if i % 2 == 0 else sum(divmod(d * 2, 10))
            assert total % 10 == 0, "PAN fails Luhn check"
            assert re.match(r"^\d{3,4}$", v.get("cvv", "")), "Invalid CVV"
            assert re.match(r"^\d{2}/\d{2}$", v.get("expiry", "")), "Invalid expiry"
        return v

class LedgerUpdateRequest(BaseModel):
    darwin_id: str
    session_id: str
    amount: Decimal          # negative = debit, positive = credit
    type: TransactionType
    description: str
    external_ref: str | None = None

class LedgerUpdateResult(BaseModel):
    transaction_id: str
    new_balance: Decimal

class InjectPaymentRequest(BaseModel):
    darwin_id: str
    vcc_ref: str
    browserbase_session_id: str
    target_fields: dict      # {pan_element, cvv_element, expiry_element}
```

### 16.3 `schemas/telemetry.py`

```python
from pydantic import BaseModel
from typing import Any

class MetricsPayload(BaseModel):
    tokens_burned_last_30s: int
    llm_cost_last_30s: float
    compute_cost_last_30s: float
    failed_http_requests_last_30s: int
    memory_util_mb: int
    current_task: str

class TelemetryStreamRequest(BaseModel):
    darwin_id: str
    session_id: str
    metrics: MetricsPayload

class CanvasEventRequest(BaseModel):
    darwin_id: str
    session_id: str
    neuron_id: str | None = None
    parent_event_id: str | None = None
    event_type: str          # validated against allowed values in the DB CHECK constraint
    payload: dict[str, Any]
    timestamp_ms: int
    token_cost: float = 0.0
    fiat_cost: float = 0.0

class EntityPushRequest(BaseModel):
    darwin_id: str
    session_id: str
    entity_type: str
    entity_id: str
    data: dict[str, Any]
    operation: str           # CREATE | UPDATE | DELETE
```

### 16.4 `schemas/canvas.py`

Typed payload shapes for each `event_type`. The frontend uses these shapes to know what fields to expect in each canvas event's `payload`.

```python
# These are documentation types — they define the payload shape per event_type.
# The actual DB column is JSONB; validation happens at the application layer.

from pydantic import BaseModel
from typing import Any

class ThoughtPayload(BaseModel):
    content: str
    model: str
    prompt_tokens: int
    completion_tokens: int
    latency_ms: int

class ToolCallPayload(BaseModel):
    tool_name: str
    input: dict[str, Any]
    status: str              # pending | success | failure | retry

class ToolResultPayload(BaseModel):
    tool_name: str
    status: str
    output_preview: str
    latency_ms: int

class NeuronSpawnedPayload(BaseModel):
    neuron_id: str
    task: str
    parent_event_id: str

class NeuronCompletedPayload(BaseModel):
    neuron_id: str
    result_summary: str
    tokens_burned: int
    cost: float

class NeuronFailedPayload(BaseModel):
    neuron_id: str
    reason: str
    tokens_burned: int
    cost: float

class SynapticIntegrationPayload(BaseModel):
    neuron_ids_merged: list[str]
    failed_neurons: list[str]
    synthesis: str

class SkillCreatedPayload(BaseModel):
    skill_id: str
    title: str
    content_preview: str
    royalty_usdc: float

class EntityEventPayload(BaseModel):
    entity_type: str
    entity_id: str
    data: dict[str, Any]
    operation: str

class AskHumanSentPayload(BaseModel):
    prompt: str
    screenshot_url: str | None
    timeout_seconds: int
    type: str                # 2fa | captcha | general

class ApprovalRequestedPayload(BaseModel):
    amount: float
    justification: str
    token_roi_estimate: str | None
```

---

## 17. End-to-End Data Flow Walkthroughs

These walkthroughs trace exactly what happens at each layer for three representative scenarios. Read these before building any individual router.

### 17.1 Agent performs a web purchase

This shows the full path from the agent deciding to buy a domain name to the canvas rendering a financial node.

```
[darwin-agent / Modal container]
  1. Agent thinks: "I should register petsupply-niche.com for $12.99"
  2. Emits THOUGHT event → POST /api/v1/telemetry/stream-event
  3. Opens Browserbase session → POST /api/v1/telemetry/stream-event (BROWSER_SESSION_OPEN)
  4. Takes screenshot → uploads to Supabase Storage → POST /api/v1/telemetry/stream-event (BROWSER_SCREENSHOT)
  5. Calls inject_payment(vcc_ref, browserbase_session_id, fields)
     → POST /api/v1/finance/inject-payment

[darwin-backend / finance router]
  6. Verifies vcc_ref belongs to this darwin_id
  7. Decrypts PAN/CVV/expiry in RAM via pgcrypto
  8. Opens WebSocket to Browserbase
  9. Types card details into DOM fields
  10. Zeroes out raw card values from memory
  11. Calls canvas_service.push_event(darwin_id, session_id, "VCC_INJECTED", {...})
  12. Supabase Realtime fans out VCC_INJECTED event → frontend

[darwin-agent]
  13. Browserbase confirms checkout complete
  14. Calls record_expense(amount=12.99, type="VCC_DEBIT", description="Domain name")
      → POST /api/v1/finance/ledger/update

[darwin-backend / finance router]
  15. SELECT ledger_balance FOR UPDATE → 42.50
  16. 42.50 - 12.99 = 29.51 (positive → OK)
  17. UPDATE darwins SET ledger_balance = 29.51
  18. INSERT into ledger_transactions
  19. canvas_service.push_event("LEDGER_DEBIT", {amount: -12.99, balance_after: 29.51})
  20. Supabase Realtime → frontend canvas renders red Financial Node
  21. Frontend budget gauge animates from $42.50 → $29.51

[darwin-agent]
  22. Emits TOOL_RESULT event (success) → POST /api/v1/telemetry/stream-event
  23. Continues execution
```

### 17.2 Agent spawns a sub-agent (neuron)

This shows how the Infinite Canvas branches into a neural thread.

```
[darwin-agent / Modal container — main trunk]
  1. Agent receives task: "Research 3 competitors simultaneously"
  2. Emits THOUGHT: "I will spawn 3 researcher neurons"
     → POST /api/v1/telemetry/stream-event (THOUGHT)

  3. Spawns neuron "researcher_A" for Competitor A
     → POST /api/v1/telemetry/stream-event (NEURON_SPAWNED)
       payload: { neuron_id: "researcher_A", task: "Scrape competitor A pricing" }

  4. Spawns neuron "researcher_B" for Competitor B
     → POST /api/v1/telemetry/stream-event (NEURON_SPAWNED)
       payload: { neuron_id: "researcher_B", task: "Scrape competitor B pricing" }

[darwin-backend / telemetry router]
  5. For each NEURON_SPAWNED event:
     INSERT into darwin_events with neuron_id set
     Supabase Realtime fans out → frontend

[darwin-frontend / Infinite Canvas]
  6. On NEURON_SPAWNED: DarwinTrunk component branches
     New NeuralThread component renders for "researcher_A" and "researcher_B"
     Two parallel lanes appear on canvas to the right of the branch point

[darwin-agent — neurons running in parallel goroutines]
  7. researcher_A: browser_fetch(competitor_A_url) → success
     → POST /api/v1/telemetry/stream-event (TOOL_CALL, neuron_id="researcher_A")
     → POST /api/v1/telemetry/stream-event (TOOL_RESULT, neuron_id="researcher_A")

  8. researcher_B: browser_fetch(competitor_B_url) → Cloudflare block → 3 retries → FAILED
     → POST /api/v1/telemetry/stream-event (TOOL_RESULT failure, neuron_id="researcher_B")
     → POST /api/v1/telemetry/stream-event (NEURON_FAILED, neuron_id="researcher_B")
       payload: { reason: "Cloudflare permanent block after 3 retries", cost: 0.013 }

  9. researcher_A completes
     → POST /api/v1/telemetry/stream-event (NEURON_COMPLETED, neuron_id="researcher_A")

  10. Main trunk synthesises results from completed neurons
      → POST /api/v1/telemetry/stream-event (SYNAPTIC_INTEGRATION)
        payload: {
          neuron_ids_merged: ["researcher_A"],
          failed_neurons: ["researcher_B"],
          synthesis: "Formulating pricing strategy from 1 of 2 sources."
        }

[darwin-frontend / Infinite Canvas]
  11. NEURON_COMPLETED → researcher_A lane ends with green node
  12. NEURON_FAILED → researcher_B lane ends with red node
  13. SYNAPTIC_INTEGRATION → both lanes angle back inward, merge into trunk
      Large Synthesis Node renders on main trunk
```

### 17.3 Agent hits a 2FA wall and delegates to human

This shows the full ask_human loop including hibernation, Telegram, and resurrection.

```
[darwin-agent / Modal container]
  1. Navigates to Stripe dashboard to check revenue
  2. Vision model reads viewport: "Enter the 6-digit code sent to --4930"
  3. Agent checks working memory: no telephony API available
  4. Emits ASK_HUMAN_SENT event
     → POST /api/v1/telemetry/stream-event (ASK_HUMAN_SENT)
       payload: { prompt: "Stripe 2FA code needed...", type: "2fa", timeout_seconds: 300 }
  5. Calls ask_human(prompt, timeout=300) → POST /api/v1/notifications/send
  6. Immediately calls hibernate(reason="Waiting for 2FA", wake_trigger={type: WEBHOOK, webhook_key: "ask_human_<uuid>"})
     → POST /api/v1/darwins/{id}/hibernate

[darwin-backend / notifications router]
  7. Fetches creator's telegram_chat_id from user_profiles
  8. Calls Telegram Bot API: sends message with screenshot
     "🔐 Agent #402 needs your Stripe 2FA code. Screenshot attached."
  9. Registers webhook_key "ask_human_<uuid>" in hibernate_schedule

[darwin-backend / darwins router — hibernate handler]
  10. Writes episodic summary to session record
  11. Registers CRON fallback (300s timeout) in hibernate_schedule
  12. Inserts AGENT_HIBERNATED event → canvas shows dashed trunk line
  13. Updates darwins.state = "HIBERNATING"
  14. Returns 200 to agent → agent container exits → Modal destroys container
  15. Compute billing drops to $0.00

[human creator — Telegram, ~47 seconds later]
  16. Reads SMS on phone: "849302"
  17. Replies to Telegram bot: "849302"

[Telegram → darwin-backend / webhooks/telegram]
  18. Telegram fires webhook to POST /api/v1/webhooks/telegram
  19. FastAPI verifies bot token, drops to Redis queue (responds 200 to Telegram in <10ms)

[Celery webhook_worker]
  20. Pulls payload from Redis
  21. Parses callback: text = "849302"
  22. Looks up hibernate_schedule WHERE webhook_key = "ask_human_<uuid>"
  23. Constructs wake prompt:
      "You were waiting for a Stripe 2FA code. The human replied: '849302'.
       Resume: input this code into the 2FA field and proceed."
  24. Calls wake_worker.execute_agent_wake(darwin_id, wake_prompt)

[wake_worker]
  25. Calls modal_service.spawn_container(darwin_id, new_session_id, wake_prompt)
  26. Modal provisions fresh micro-VM in ~800ms
  27. Mounts Modal Volume → Hermes finds hermes.db exactly where it left it
  28. Inserts AGENT_WOKE event → canvas: dashed line ends, pulsing trunk resumes
  29. Updates darwins.state = "ACTIVE"
  30. Sends Telegram: "🟢 Agent #402 resumed. 2FA code accepted."

[darwin-agent / new Modal container]
  31. Reads wake prompt from working memory
  32. Reconnects to preserved Browserbase session (still authenticated)
  33. Types "849302" into 2FA field → submits
  34. Emits ASK_HUMAN_RESOLVED event
      → POST /api/v1/telemetry/stream-event (ASK_HUMAN_RESOLVED)
        payload: { type: "2fa", wait_time_seconds: 47, compute_saved: 0.012 }
  35. Canvas: ASK_HUMAN_SENT node turns green
  36. Agent continues checking Stripe revenue
```

---

## 18. Error Handling & Status Codes

| Scenario                                           | HTTP Status | Response                                                                       |
| -------------------------------------------------- | ----------- | ------------------------------------------------------------------------------ |
| Agent JWT invalid or expired                       | `401`       | `{ "error": "invalid_agent_token" }`                                           |
| Human JWT invalid                                  | `401`       | `{ "error": "invalid_token" }`                                                 |
| Agent calls human-only endpoint                    | `403`       | `{ "error": "forbidden" }`                                                     |
| Agent tries to access another agent's data         | `403`       | `{ "error": "forbidden" }` (RLS)                                               |
| Invalid FSM transition (e.g. wake an ACTIVE agent) | `409`       | `{ "error": "invalid_state_transition", "current_state": "ACTIVE" }`           |
| Insufficient ledger balance                        | `402`       | `{ "error": "insufficient_funds", "balance": 1.50, "requested": 4.00 }`        |
| Pydantic validation failure                        | `422`       | Standard FastAPI error with field details                                      |
| Pre-flight intent scan flagged                     | `403`       | `{ "error": "directive_rejected", "reason": "..." }`                           |
| Stripe webhook signature invalid                   | `403`       | `{ "error": "invalid_signature" }`                                             |
| VCC injection — card declined                      | `402`       | `{ "error": "card_declined", "vcc_ref": "..." }`                               |
| Internal service error                             | `500`       | `{ "error": "internal_error", "trace_id": "uuid" }` — trace_id links to Sentry |

All `500` errors are caught by a global exception handler, logged to Sentry with full context, and returned to the caller with a `trace_id` they can report.

---

## 19. Testing Strategy

### 19.1 Unit tests (`tests/unit/`)

Test services in isolation with mocked database and external APIs.

Key targets:

- `ledger_service.apply_transaction` — race condition tests, boundary conditions, insufficient funds
- `vcc_service.decrypt_and_inject` — verify raw PAN is zeroed post-injection
- `intent_scan_service.scan_directive` — test known malicious and benign directives
- `canvas_service.push_event` — verify correct payload shapes per event type

### 19.2 Integration tests (`tests/integration/`)

Test full request-response cycles against a test Supabase project and a mocked Modal service.

Key targets:

- Full spawn → hibernate → wake → terminate lifecycle
- Ledger double-spend race condition (concurrent requests via `asyncio.gather`)
- Webhook processing end-to-end: Stripe event → ledger credit → canvas event
- VCC vault: submit → encrypt → retrieve reference → inject → verify PAN zeroed

### 19.3 Critical invariants to always test

1. **An agent JWT cannot access `/users` endpoints** — ever.
2. **An agent cannot read another agent's ledger** — RLS must block it.
3. **A debit that would result in negative balance must be rejected** — never allow overdraft.
4. **Raw PAN must never appear in any log, response body, or error message.**
5. **Every `darwin_events` insert must trigger a Supabase Realtime event** — test with a Realtime subscription in the test suite.

---

## 20. Operational Runbook

### 20.1 Agent stuck in ACTIVE with no heartbeat

The container crashed without hibernating. The backend detects this via the `modal_service.get_container_status()` poll (runs every 60s via Celery beat).

**Action:** `webhook_worker` automatically transitions state to `HIBERNATING`, creates a cron wake trigger for 5 minutes, sends Telegram alert: "⚠️ Agent #{name} container crashed unexpectedly. Will auto-resume in 5 minutes."

### 20.2 Circuit breaker trips

Telemetry burn rate spikes beyond threshold.

**Auto-action:** `POST /telemetry/trip` → terminate container, freeze VCC, Telegram alert.

**Manual review:** Check `darwin_events` for the session. Look for repeated `TOOL_CALL` failures in a tight loop. Add a Skill Document to `hermes_skills` fixing the problematic behaviour. Spawn a replacement agent.

### 20.3 VCC declined at checkout

Agent receives DECLINED from Browserbase checkout. It calls `POST /finance/ledger/update` with a zero-amount record noting the decline. The backend sends Telegram: "💳 VCC declined at {merchant}. Agent is pivoting strategy."

Agent pivots autonomously. If it cannot find an alternative spending path, it hibernates and alerts the human to top up the card or increase the limit.

### 20.4 Stripe webhook not arriving

If revenue events stop arriving, the agent may have misconfigured the Stripe webhook URL. Check:

1. `hibernate_schedule` — does the agent have a `WEBHOOK` trigger registered?
2. Stripe dashboard — is the webhook endpoint pointing to `https://darwin-backend.railway.app/api/v1/webhooks/stripe`?
3. `audit_log` — is the agent logging revenue attempts that aren't resulting in credits?

### 20.5 Pre-flight scan false positive

User reports their directive was rejected incorrectly.

**Action:** Retrieve the `audit_log` record for the failed spawn. Review the directive and the classifier's reason. Adjust the system prompt in `intent_scan_service.py` if the classification was incorrect. Unsuspend the user account via `PATCH /security/unsuspend-account` (admin only).
