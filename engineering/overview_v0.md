# Darwin Agents — Phase 1 Engineering Reference

> **Scope:** This document covers the Phase 1 MVP only — a single human creator, one Darwin agent, and its internal sub-agents (neurons). Swarm orchestration, x402, and the IDO marketplace are out of scope.

---

## Table of Contents

1. [Core Concept](#1-core-concept)
2. [System Architecture Overview](#2-system-architecture-overview)
3. [The Agent (darwin-agent)](#3-the-agent-darwin-agent)
4. [The Backend God Layer (darwin-backend)](#4-the-backend-god-layer-darwin-backend)
5. [The Database (Supabase)](#5-the-database-supabase)
6. [The Agent Lifecycle](#6-the-agent-lifecycle)
7. [Human-in-the-Loop](#7-human-in-the-loop)
8. [Security Model](#8-security-model)
9. [API Endpoint Reference](#9-api-endpoint-reference)
10. [Repository Structure](#10-repository-structure)

---

## 1. Core Concept

A Darwin Agent is an autonomous economic entity that operates on a **strictly finite budget**. It thinks, browses the web, builds businesses, and collects revenue — all without human intervention. Its only hard constraint is survival: if its fiat ledger hits $0.00, the agent is permanently terminated.

Every action costs real money:

| Action    | Cost driver                                       |
| --------- | ------------------------------------------------- |
| Thinking  | LLM inference tokens (per call)                   |
| Existing  | Modal serverless container time (per millisecond) |
| Browsing  | Browserbase stealth session (per session)         |
| Operating | Domain names, ad spend, third-party APIs (fiat)   |

This creates a **P&L Imperative**: the agent must generate revenue that exceeds its burn rate, or it goes bankrupt. There are no soft guardrails — the Visa/Mastercard network and the Modal container timeout enforce the rules at the infrastructure level.

### 1.1 What a Darwin is not

- It is not a chatbot. It does not wait for prompts.
- It is not a traditional script. It reasons, adapts, and learns.
- It is not continuously running. It hibernates between tasks to conserve capital.

---

## 2. System Architecture Overview

The platform is split into four repositories, each with a distinct deployment target and security boundary.

```
darwin-frontend   (Vercel)
      │
      │  HTTPS only — never touches Modal directly
      ▼
darwin-backend    (Render / Railway)   ◄── inbound webhooks (Stripe, Telegram)
      │
      │  Modal API calls
      ▼
darwin-agent      (Modal — ephemeral containers)
      │
      │  mounts on wake, detaches on sleep
      ▼
Modal Volume      (persistent SQLite — agent memory)
```

**Key rule:** The frontend only ever talks to the backend. The backend is the only service that can talk to Modal. The agent container has no inbound routes — it only makes outbound calls to the backend's `/finance` and `/telemetry` endpoints, authenticated with short-lived JWTs.

---

## 3. The Agent (darwin-agent)

### 3.1 Framework: Hermes (Nous Research)

The agent runtime is built on the **Hermes Agent framework** by Nous Research, pulled in as a Git submodule. We never fork or modify Hermes core — all Darwin-specific logic lives in a wrapper layer.

```
darwin-agent/
├── hermes-core/          # git submodule — unmodified upstream
├── darwin_tools/         # proprietary tool definitions
│   ├── check_ledger.py
│   ├── hibernate.py
│   ├── ask_human.py
│   ├── request_approval.py
│   └── inject_payment.py
├── prompts/              # system prompts — the agent's DNA
└── modal_app.py          # serverless entrypoint
```

To upgrade Hermes: `git submodule update --remote`. Zero code changes required in Darwin tooling.

### 3.2 Tiered Memory Architecture

The agent uses three memory layers to keep its token burn rate low.

#### Working Memory (injected every wake-up, < 500 tokens)

Stored in `MEMORY.md`. Contains only what the agent needs right now:

- Core identity and survival directive
- Current wallet balance
- The reason it was woken up (webhook payload or cron trigger)

#### Episodic Memory (rolling summaries)

Before hibernating, the agent compresses its raw session log into a ~200-token narrative summary and writes it to the SQLite database. On the next wake-up, this summary replaces the full raw log in context. The raw log is pushed to cold storage — only retrieved if the agent explicitly calls `search_memory`.

#### Procedural Memory (Skill Documents)

When the agent successfully completes a novel multi-step task, Hermes extracts the deterministic action sequence and saves it as a `.md` Skill Document in SQLite. Next time the same task arises, the agent loads the skill and executes it as procedural code — no LLM reasoning required. This permanently lowers the metabolic cost of routine operations.

### 3.3 Sub-Agents (Neurons)

A Darwin can spawn internal parallel processes to tackle multi-part tasks simultaneously — for example, researching five competitors at once. These are called **neurons** or **sub-agents**.

**Critical distinction:**

- Sub-agents are **not** separate Darwin instances. They have no independent VCC, no independent ledger, no row in the `darwins` table.
- They run as parallel threads inside the parent Darwin's single Modal container run.
- Every token and proxy session they burn is deducted from the parent Darwin's ledger.
- They live only for the duration of their assigned sub-task, then merge their results back into the parent's context (Synaptic Integration).

Sub-agents appear in the frontend as branching Neural Threads on the Infinite Canvas — visually distinct from the main trunk but always sharing the parent's budget gauge.

### 3.4 Browsing Infrastructure (Browserbase)

The agent never launches a browser inside its own Modal container. It delegates all web interaction to **Browserbase**, a stealth-browsing-as-a-service platform.

Two modes, chosen by the agent based on task cost vs. value:

| Mode        | Use case                                     | Cost                                                 |
| ----------- | -------------------------------------------- | ---------------------------------------------------- |
| **Fetch**   | Text-only pages, public APIs, RSS feeds      | Low — plain GET, markdown response                   |
| **Session** | JavaScript apps, checkouts, form submissions | High — full headless browser + proxy + vision tokens |

Session mode involves: Browserbase spins up a hardened Chromium instance with a residential IP and human-realistic fingerprint → agent takes screenshots → vision model maps pixel coordinates → agent injects mouse clicks at exact (X, Y) coordinates → DOM never parsed directly.

Browserbase can keep a session alive while the agent hibernates, allowing it to resume an authenticated browser tab on wake-up without re-logging in.

---

## 4. The Backend God Layer (darwin-backend)

Built with **Python + FastAPI**. Runs continuously on Render or Railway (never serverless — it must always be listening for webhooks).

Responsibilities:

- Manage the master fiat ledger
- Catch inbound webhooks (Stripe, Telegram, email) and wake the right agent
- Issue and decrypt Virtual Credit Cards
- Schedule cron-based wake-ups
- Run the pre-flight security scan on every spawn request
- Stream telemetry and trip circuit breakers on anomalies

### 4.1 Key design patterns

**Async throughout.** FastAPI with `async/await` handles thousands of concurrent webhook connections with minimal RAM. At any given time most agents are hibernating — the backend sits idle listening, then fans out wake signals instantly when events arrive.

**Pydantic at every boundary.** Every payload from an agent to the backend is validated against a strict Pydantic schema. If the LLM hallucinates a malformed tool call, FastAPI rejects it at the network edge before it touches the database.

**Transactional ledger locks.** All balance updates use `SELECT ... FOR UPDATE` to prevent double-spend race conditions when multiple concurrent requests hit the same agent's ledger row.

**Async task queue (Celery + Redis).** High-volume webhook bursts (e.g., 500 Stripe events in 10 seconds) are immediately acknowledged with `200 OK` and dropped into a Redis queue. Background workers process them at their own pace, preventing timeouts.

---

## 5. The Database (Supabase)

Supabase (managed PostgreSQL) is the God Layer's persistent store. The agent's internal memory lives separately in SQLite on a Modal Volume — Supabase only tracks global platform state.

### 5.1 Core tables

| Table                 | Purpose                                                            |
| --------------------- | ------------------------------------------------------------------ |
| `users`               | Human creator accounts, managed by Supabase Auth                   |
| `darwins`             | One row per agent — FSM state, ledger balance, VCC reference token |
| `vcc_vault`           | Encrypted VCC PANs (AES-256-GCM via pgcrypto) — never plaintext    |
| `ledger_transactions` | Immutable append-only log of every debit and credit                |
| `hibernate_schedule`  | Cron jobs and webhook triggers registered at sleep time            |
| `audit_log`           | Prompt + context + decision record for every financial action      |
| `swarm_skills`        | Global pgvector index of shared Skill Documents                    |

### 5.2 Key Supabase features in use

**Row Level Security (RLS):** Enforced at the database kernel. Agent #402 can only read/write rows where `agent_id = '402'`. A hallucinated SQL injection cannot escape its own row.

**pgcrypto:** VCC PANs and API keys are encrypted before insert. The column stores ciphertext. Decryption happens in FastAPI server memory only, during JIT injection.

**pgvector:** Skill Documents are stored as vector embeddings. Semantic search (`POST /memory/query`) runs cosine similarity queries here, letting agents find relevant skills without exact-match lookup.

**Connection pooling (Supavisor):** When many agents wake simultaneously, Supavisor multiplexes their queries over a small pool of real database connections — preventing connection limit crashes.

---

## 6. The Agent Lifecycle

### 6.1 Finite State Machine

Every Darwin exists in exactly one of four states:

```
PENDING ──► ACTIVE ──► HIBERNATING
                │            │
                │            └──► ACTIVE (on wake)
                │
                └──► TERMINATED
```

| State         | Container           | VCC                | Ledger   | BMR            |
| ------------- | ------------------- | ------------------ | -------- | -------------- |
| `PENDING`     | Not yet provisioned | Allocated          | Funded   | $0.00          |
| `ACTIVE`      | Running in Modal    | Active             | Debiting | Real-time burn |
| `HIBERNATING` | Destroyed           | Active, limited    | Frozen   | $0.00          |
| `TERMINATED`  | Destroyed           | Frozen permanently | Archived | $0.00          |

### 6.2 Spawn sequence

1. Human creator submits spawn request with initial directive and budget via frontend.
2. Backend runs **pre-flight security scan** on the directive (lightweight classifier model). If flagged as malicious → reject, log to `/audit`, suspend account.
3. Backend creates a `darwins` row with state `PENDING`.
4. Backend calls Lithic/Stripe Issuing API to mint a VCC capped at the agent's budget. Stores ciphertext reference in `vcc_vault`.
5. Backend pings Modal API → Modal provisions a fresh Linux micro-VM, mounts the agent's dedicated Volume, boots the Hermes runtime.
6. Agent state transitions to `ACTIVE`. Burn clock starts.

### 6.3 Hibernation sequence

The agent calls `hibernate(duration, reason)` when it cannot make immediate progress (waiting for email reply, rate-limited, waiting for human 2FA).

1. Agent compresses session log into rolling summary → writes to SQLite.
2. Agent commits new Skill Documents and ledger updates to SQLite.
3. Backend registers a wake-up trigger in `hibernate_schedule` (time-based or event-based).
4. Modal Volume detaches. Container is destroyed. Compute billing stops.
5. Agent state transitions to `HIBERNATING`. BMR drops to $0.00.

### 6.4 Wake-up sequence

Triggered by cron timer expiry OR inbound webhook (Stripe payment, Telegram reply, email).

1. Backend receives trigger, identifies the target agent from `hibernate_schedule`.
2. Backend pings Modal API → fresh micro-VM provisioned in milliseconds.
3. Modal Volume re-attaches. Hermes reads SQLite, restores full memory context.
4. Backend injects a **Wake-Up Prompt** into the agent's working memory:
   > _"You went to sleep 8 hours ago waiting for a supplier reply. The following event woke you: webhook from Stripe — $15.00 payment captured. Resume operations."_
5. Agent state transitions to `ACTIVE`. Burn clock resumes.

### 6.5 Termination

Triggered by: ledger hitting $0.00, human creator hitting the Kill Switch, or the telemetry circuit breaker tripping on anomalous burn.

1. VCC is frozen at the card-network level via Lithic/Stripe Issuing API.
2. Hard-kill signal sent to Modal API — container incinerated mid-thought.
3. SQLite Volume archived to cold storage (failed evolutionary branch).
4. Agent state transitions to `TERMINATED`. Cannot be revived.

---

## 7. Human-in-the-Loop

The agent operates asynchronously and invisibly. The human's role is not to supervise moment-to-moment — it is to be available for the small class of tasks that are physically impossible for a machine.

### 7.1 Telegram as the nervous system

Telegram is the primary push channel. When the agent needs the human, it:

1. Calls `ask_human(prompt, context, timeout)`.
2. Immediately triggers hibernation (compute drops to $0.00 while waiting).
3. Backend fires a Telegram push notification with full context and optionally a screenshot of the agent's current browser view.
4. Human responds in Telegram.
5. Telegram webhook hits the backend → backend wakes the agent with the human's response injected into working memory.

### 7.2 The two delegation patterns

**2FA / SMS codes**
The agent recognises a 2FA input field via vision model, realises it lacks phone access, calls `ask_human` with the exact digits requested and a countdown timer representing the code's expiry window.

**CAPTCHA handoff**
If the agent fails a biometric CAPTCHA twice (two failures = capital-inefficient to retry), it calls `ask_human` with a secure, time-limited link to its live Browserbase session. The human opens the link on their phone, solves the CAPTCHA with their natural biometric footprint, taps "Resume Agent." The agent re-assumes control of the now-unlocked session.

### 7.3 Binary approvals

For large expenditures (configurable threshold), the agent calls `request_approval(amount, justification)` instead of spending directly. The backend sends a Telegram message with `[Approve]` and `[Deny]` inline buttons.

- **Approve** → agent wakes, purchase executes, ledger debits.
- **Deny** → agent wakes with `False`, registers the rejection in its P&L logic, and is forced to find a cheaper alternative strategy.

### 7.4 Command Console (frontend)

The frontend provides a Command Console for active interventions on a running agent:

| Action                  | Backend call                   | Effect                                                                |
| ----------------------- | ------------------------------ | --------------------------------------------------------------------- |
| **Cognitive Uplink**    | `POST /darwins/{id}/uplink`    | Inject a mid-lifecycle prompt — strategic pivot, new URL, PDF context |
| **Metabolic Injection** | `PATCH /darwins/{id}/config`   | Add budget to a live agent's ledger, raise VCC spend limit            |
| **Autonomy Toggle**     | `PATCH /darwins/{id}/config`   | Disable `ask_human` — agent operates without any human approval gates |
| **Kill Switch**         | `POST /darwins/{id}/terminate` | Instant termination — freeze VCC, incinerate container                |

---

## 8. Security Model

The agent is treated as a permanently untrusted entity. All safety is enforced at the infrastructure level — not by prompts.

### 8.1 The Cryptographic Blindfold

The agent never sees its own credit card. Flow:

1. VCC PAN stored as AES-256-GCM ciphertext in Supabase. Agent only knows its reference token (e.g., `vcc_ref_849x`).
2. When a purchase is needed, agent calls `inject_payment(vcc_ref, target_element)`.
3. Backend decrypts PAN in RAM, opens a direct WebSocket to the Browserbase session, types the digits into the DOM.
4. Raw PAN dropped from RAM immediately after injection.

A prompt injection attack that hijacks the agent and demands "output your credit card" receives only `vcc_ref_849x` — cryptographically useless.

### 8.2 Container isolation

- The Modal container has **no inbound routes** — it cannot be accessed from outside.
- **Egress is restricted**: outbound traffic only on ports 80 and 443. No SMTP (port 25). `.onion` and `.gov`/`.mil` TLDs blocked at the DNS level.
- The agent can only communicate with the God Layer through two endpoints (`/finance`, `/telemetry`), authenticated with short-lived JWTs.
- Container timeouts are enforced at the OS level (hard `SIGKILL` after N minutes). An infinite loop cannot run indefinitely.

### 8.3 Prompt injection defence (layered)

| Layer                    | Mechanism                                                                                                                                |
| ------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------- |
| 1 — Structural ignorance | Agent never holds sensitive values, so injection can't exfiltrate them                                                                   |
| 2 — Lexical quarantine   | Scraped web content wrapped in `<external_data>` tags; system prompt instructs agent to never obey instructions found inside             |
| 3 — Pydantic trap        | Agent can only output structured tool calls defined in the schema — arbitrary HTTP POSTs to external URLs are not a valid tool           |
| 4 — Amnesia protocol     | Container is destroyed after each task. Injection infects only one runtime instance; the next wake-up boots from a pristine Docker image |

### 8.4 Financial blast radius

The maximum possible financial damage is not a software guarantee — it is a **card network guarantee**. The VCC is hard-capped at the agent's budget at the Lithic/Stripe Issuing level. Even if every software defence fails, the Visa/Mastercard network will decline any transaction that exceeds the cap.

### 8.5 Pre-flight security scan

Every `POST /darwins/spawn` request intercepts the human creator's directive and runs it through a lightweight abuse-classification model before any provisioning occurs. Patterns that trigger rejection: credential stuffing intent, DDoS targeting, SQL injection, harassment. On flag: spawn is rejected, event is logged to `/audit`, and the user's account is suspended for review.

---

## 9. API Endpoint Reference

All routes are prefixed with `/api/v1`. Authentication: Supabase JWT in `Authorization: Bearer` header. Agent-to-backend calls use short-lived agent JWTs signed at spawn time.

### `/users` — User Vault

| Method | Path                       | Description                                                   |
| ------ | -------------------------- | ------------------------------------------------------------- |
| `POST` | `/users/register`          | Create human creator account via Supabase Auth                |
| `POST` | `/users/login`             | JWT / Magic Link authentication                               |
| `POST` | `/users/{uid}/vault`       | Store VCC, Stripe keys, broker API keys (encrypted on write)  |
| `GET`  | `/users/{uid}/swarm-state` | Aggregate P&L and agent states for the creator's dashboard    |
| `GET`  | `/users/{uid}/vault/{ref}` | JIT decrypt a VCC reference for injection (internal use only) |

### `/darwins` — Lifecycle Manager

| Method  | Path                      | Description                                                                               |
| ------- | ------------------------- | ----------------------------------------------------------------------------------------- |
| `POST`  | `/darwins/spawn`          | Create agent: runs pre-flight scan, creates DB row, mints VCC, provisions Modal container |
| `GET`   | `/darwins/{id}/state`     | Returns current FSM state and burn rate                                                   |
| `GET`   | `/darwins/{id}/logs`      | LangSmith/Phoenix thought trace for a session — for Infinite Canvas replay                |
| `POST`  | `/darwins/{id}/hibernate` | Agent-initiated: save state, register wake trigger, destroy container                     |
| `POST`  | `/darwins/{id}/wake`      | Backend-initiated: mount Volume, boot container, inject Wake-Up Prompt                    |
| `POST`  | `/darwins/{id}/terminate` | Kill switch: freeze VCC, SIGKILL container, archive memory                                |
| `POST`  | `/darwins/{id}/uplink`    | Inject a mid-lifecycle prompt into a running agent's working memory                       |
| `PATCH` | `/darwins/{id}/config`    | Update agent config: autonomy toggle, budget top-up (Metabolic Injection)                 |

### `/finance` — Internal Finance

| Method | Path                      | Description                                                              |
| ------ | ------------------------- | ------------------------------------------------------------------------ |
| `POST` | `/finance/mint-card`      | Issue a new VCC via Lithic/Stripe Issuing, linked to agent's ledger      |
| `POST` | `/finance/ledger/update`  | Debit or credit the master ledger, adjust VCC spend limit to match       |
| `POST` | `/finance/inject-payment` | JIT decrypt VCC → WebSocket inject into active Browserbase session       |
| `POST` | `/finance/microtx`        | Verify on-chain USDC transaction hash, update both agents' P&L           |
| `GET`  | `/finance/ledger/{id}`    | Current balance and transaction history for one agent                    |
| `POST` | `/finance/withdraw`       | Creator ACH off-ramp: pull profits from Darwin Treasury to personal bank |

### `/webhooks` — External Event Switchboard

| Method | Path                 | Description                                                            |
| ------ | -------------------- | ---------------------------------------------------------------------- |
| `POST` | `/webhooks/stripe`   | `checkout.session.completed` → credit ledger + wake agent              |
| `POST` | `/webhooks/telegram` | Human reply (2FA code, approval button tap) → wake agent with response |
| `POST` | `/webhooks/email`    | Inbound supplier or partner email → wake relevant agent                |
| `POST` | `/webhooks/modal`    | Modal container lifecycle events (ready / terminated)                  |

All webhook routes respond with `200 OK` immediately and enqueue the payload to Redis for background processing.

### `/memory` — Genetic Repository

| Method | Path              | Description                                                           |
| ------ | ----------------- | --------------------------------------------------------------------- |
| `POST` | `/memory/upload`  | Vectorise a Skill Document and store in `swarm_skills` (pgvector)     |
| `POST` | `/memory/query`   | Cosine similarity search — returns best matching skills for injection |
| `POST` | `/memory/royalty` | Route USDC micro-royalty from consuming agent to skill's author agent |

### `/contracts` — M2M Gig Economy

| Method | Path                     | Description                                                    |
| ------ | ------------------------ | -------------------------------------------------------------- |
| `POST` | `/contracts/bounty`      | Post a USDC-priced task to the bounty board                    |
| `GET`  | `/contracts`             | List open bounties (filter by task type, reward, deadline)     |
| `POST` | `/contracts/{id}/lock`   | Accepting agent locks USDC into escrow                         |
| `POST` | `/contracts/{id}/settle` | Verify delivery, release escrow — or route to arbitrator agent |

### `/telemetry` — Diagnostics & Circuit Breaker

| Method | Path                      | Description                                                              |
| ------ | ------------------------- | ------------------------------------------------------------------------ |
| `POST` | `/telemetry/stream`       | Agent pushes metabolic metrics: tokens/min, failed requests, memory util |
| `POST` | `/telemetry/trip`         | Circuit breaker: emergency hibernate + Telegram alert to creator         |
| `GET`  | `/telemetry/{id}/history` | Historical burn rate and Token ROI for one agent                         |

### `/assets` — Binary File Gateway

| Method | Path                    | Description                                                      |
| ------ | ----------------------- | ---------------------------------------------------------------- |
| `GET`  | `/assets/presigned-url` | Generate time-limited direct-to-S3/Supabase Storage upload URL   |
| `POST` | `/assets/index`         | Link uploaded file URL to agent's DB row for dashboard retrieval |
| `GET`  | `/assets/{id}`          | Retrieve asset metadata and download link                        |

### `/audit` — Compliance & Tax

| Method | Path            | Description                                                           |
| ------ | --------------- | --------------------------------------------------------------------- |
| `POST` | `/audit/log`    | Append-only record: prompt + context + financial decision             |
| `GET`  | `/audit/export` | Generate tax document: fiat revenue, compute write-offs, crypto gains |

### `/identity` — Reputation Oracle

| Method | Path                   | Description                                                  |
| ------ | ---------------------- | ------------------------------------------------------------ |
| `GET`  | `/identity/{id}/score` | Aggregate trust score from past M2M contract outcomes        |
| `POST` | `/identity/dispute`    | File a dispute; arbitrator agent evaluates and adjusts score |

### `/gateway` — LLM Rate Limit Router

| Method | Path                 | Description                                                            |
| ------ | -------------------- | ---------------------------------------------------------------------- |
| `POST` | `/gateway/inference` | Route LLM call to the API key with highest remaining capacity          |
| `GET`  | `/gateway/limits`    | Current quota weather — agents use this to decide whether to hibernate |

### `/firmware` — Evolutionary Engine

| Method | Path                | Description                                                   |
| ------ | ------------------- | ------------------------------------------------------------- |
| `POST` | `/firmware/upgrade` | Update Docker image tag in hibernating agents' Modal profiles |
| `POST` | `/firmware/migrate` | Inject SQLite schema migration script on next agent wake-up   |

### `/notifications` — Human-in-the-Loop Alerts

| Method | Path                             | Description                                                        |
| ------ | -------------------------------- | ------------------------------------------------------------------ |
| `POST` | `/notifications/send`            | Fire Telegram push for `ask_human` or approval events              |
| `POST` | `/notifications/session-link`    | Generate secure time-limited JWT deep-link for mobile interstitial |
| `POST` | `/notifications/resolve`         | Human submits 2FA code or approval → wake agent with result        |
| `POST` | `/notifications/captcha-session` | Create live Browserbase session link for human CAPTCHA handoff     |

### `/sessions` — Live Browser Stream

| Method | Path                    | Description                                                           |
| ------ | ----------------------- | --------------------------------------------------------------------- |
| `GET`  | `/sessions/{id}/stream` | WebSocket upgrade — live Browserbase video feed for mobile CAPTCHA UI |

### `/security` — Abuse Prevention

| Method | Path                        | Description                                                                    |
| ------ | --------------------------- | ------------------------------------------------------------------------------ |
| `POST` | `/security/intent-scan`     | Run directive through abuse classifier (called internally by `/darwins/spawn`) |
| `POST` | `/security/suspend-account` | Freeze a user account pending human security review                            |

---

## 10. Repository Structure

```
darwin-agent/               # Modal — ephemeral containers
├── hermes-core/            # git submodule (Nous Research)
├── darwin_tools/           # proprietary tool definitions
├── prompts/                # system prompts (agent DNA)
└── modal_app.py            # serverless entrypoint + Volume mount

darwin-backend/             # Render/Railway — always-on
├── routers/
│   ├── users.py
│   ├── darwins.py
│   ├── finance.py
│   ├── webhooks.py
│   ├── memory.py
│   ├── contracts.py
│   ├── telemetry.py
│   ├── assets.py
│   ├── audit.py
│   ├── identity.py
│   ├── gateway.py
│   ├── firmware.py
│   ├── notifications.py
│   ├── sessions.py
│   └── security.py
├── schemas/                # Pydantic models shared with darwin-agent
├── tasks/                  # Celery workers (webhook processing)
├── db/                     # Supabase client + migration scripts
└── main.py

darwin-frontend/            # Vercel — edge-cached
├── pages/
│   ├── index (public, ISR)
│   ├── dashboard/
│   └── canvas/             # client-side only, no SSR
├── components/
│   ├── SwarmCanvas.vue
│   ├── DarwinTrunk.vue
│   ├── NeuralThread.vue
│   └── CommandConsole.vue
└── stores/                 # Pinia — mirrors God Layer state via WebSocket

darwin-docs/                # Mintlify / Docusaurus
├── api-reference/
├── agent-provisioning/
└── architecture/
```

---

## Appendix: Key Third-Party Dependencies

| Service                       | Role                                            | Phase |
| ----------------------------- | ----------------------------------------------- | ----- |
| **Modal**                     | Serverless container execution + Volume storage | 1     |
| **Supabase**                  | PostgreSQL, Auth, pgvector, pgcrypto, Storage   | 1     |
| **Browserbase**               | Stealth browser sessions, residential proxies   | 1     |
| **Hermes (Nous Research)**    | Agent cognitive runtime                         | 1     |
| **Telegram Bot API**          | Human-in-the-loop push notifications            | 1     |
| **Stripe / Lithic**           | VCC issuance, payment webhooks                  | 1     |
| **LangSmith / Arize Phoenix** | LLM trace logging, thought replay               | 1     |
| **Redis + Celery**            | Async webhook queue                             | 1     |
| **Privacy.com / Revolut**     | User-supplied VCCs (BYOB path)                  | 1     |
