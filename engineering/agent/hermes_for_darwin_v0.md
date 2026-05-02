# Hermes Agent → Darwin Agents: Capability Investigation

This document maps **every** Hermes Agent capability that is directly usable inside a `darwin-agent` wrapper, organized to match the Darwin manifesto_v0's chapter structure. References are to the official docs at <https://hermes-agent.nousresearch.com/docs/> and the repo at <https://github.com/NousResearch/hermes-agent> (MIT license, 6,886+ commits, latest release v0.12.0 from April 2026).

> **Bottom line:** Hermes already implements ~90% of the cognitive infrastructure the Darwin manifesto_v0 describes — Tiered Memory, Hibernation Cycle (via Modal), Procedural Skills with self-improvement, ask_human, P&L telemetry hooks, prompt-injection defenses, and the four-pillar deployment model. The proprietary work for Darwin reduces to: **(1) economic tools** (BYOB/VCC, ledger, x402), **(2) the FastAPI God Layer** that owns those tools, and **(3) the swarm-level UI**. Everything else — the brain, memory, sandboxing, MCP, hooks, telemetry — is upstream.

---

## 1. Composition pattern: how to wrap Hermes without forking

Your manifesto_v0 §2.2 calls for "Composition over Inheritance" via Git submodules. Hermes natively supports this:

| Mechanism                                                                                       | What it gives you                                                                                                                                                                                         |
| ----------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **`pip install -e ".[all]"`** as a dependency                                                   | Hermes ships as a regular Python package with `pyproject.toml` and `[project.entry-points."hermes_agent.plugins"]` — you can pin a version, depend on it, and override behavior without touching the core |
| **Plugin system** (`~/.hermes/plugins/`, project-local `.hermes/plugins/`, or pip entry points) | Add tools, hooks, slash commands, CLI commands, and dashboard tabs without editing core code                                                                                                              |
| **External skill directories** (`skills.external_dirs` in `config.yaml`)                        | Point Hermes at a separate `darwin-skills/` repo for shared swarm intelligence — read-only, local skills win, missing dirs are silent no-ops                                                              |
| **Profiles** (`hermes profile create darwin-agent-NNN --clone`)                                 | Each Darwin gets its own isolated `HERMES_HOME` — config, memory, skills, sessions, gateway state. Backups & migrations follow the directory                                                              |
| **`HERMES_HOME` env var**                                                                       | The only knob you need to point a process at a specific agent's state                                                                                                                                     |
| **Python library mode**                                                                         | `hermes_agent.plugins` entry point + direct `AIAgent` import — see Python Library guide                                                                                                                   |

**Practical consequence for `darwin-agent` repo:**

```
darwin-agent/
├── hermes-core/                    # git submodule pointer to NousResearch/hermes-agent
├── darwin_tools/                   # Your proprietary plugin (tools + hooks)
│   ├── plugin.yaml
│   ├── __init__.py                 # register() — tools + hooks
│   ├── financial_tools.py          # check_ledger, request_payment, mint_vcc, hibernate, etc.
│   └── pnl_hooks.py                # post_tool_call → emit telemetry to FastAPI God Layer
├── darwin_skills/                  # Mounted via skills.external_dirs
├── prompts/                        # Custom SOUL.md (the agent's "DNA")
├── modal_app.py                    # Serverless orchestration (per §3 of manifesto_v0)
└── pyproject.toml                  # Depends on hermes-agent submodule
```

Critically, **you never modify Hermes core** — your tools live in a plugin, and `git submodule update --remote` upgrades the brain with zero refactoring (manifesto_v0 §2.2.2).

---

## 2. Cognitive engine — direct mapping to manifesto_v0 §2

### 2.1 The Hermes Agent framework you reference _is_ the actual product

Confirmed: Hermes Agent at <https://github.com/NousResearch/hermes-agent> by Nous Research, MIT licensed, ~127K stars, actively maintained. It is **not** a research artifact — it's a production agent runtime.

### 2.2 Tiered Memory Architecture (manifesto_v0 §2.3) — implemented natively

Hermes already has the exact three-tier memory you describe:

| manifesto_v0 layer                                           | Hermes implementation                                                                                                                                                                                                                                                      |
| ------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Working Memory (Scratchpad)** — system prompt, ~500 tokens | Built directly by `agent/prompt_builder.py`. Includes SOUL.md (slot #1), MEMORY.md (~800 tokens), USER.md (~500 tokens), and AGENTS.md as a frozen snapshot                                                                                                                |
| **Episodic Memory (Rolling Summaries)**                      | The **context compressor** (`agent/context_compressor.py`). Configurable via `compression.threshold` (default 50%) and `compression.target_ratio`. Auto-fires when the context fills, compresses middle turns into summaries, preserves first 3 + last 20 turns            |
| **Procedural Memory (Skill Documents)**                      | Native Skills system. Agent uses `skill_manage(action="create"\|"patch"\|"edit")` after solving novel problems. SKILL.md files are markdown with YAML frontmatter, follow the [agentskills.io](https://agentskills.io/) standard, and are cataloged in `~/.hermes/skills/` |

**You get the "Learning Loop" for free.** The agent is _prompted_ to save skills after completing complex tasks (5+ tool calls, after corrections, after discovering non-trivial workflows). There's also a `curator` system that auto-archives unused skills after 90 days and consolidates duplicates via a background LLM review pass.

### 2.3 Memory characteristics that matter for Darwin economics

- **Bounded sizes**: MEMORY.md = 2,200 chars, USER.md = 1,375 chars — keeps token cost per session predictable. You can tune `memory.memory_char_limit` and `memory.user_char_limit`.
- **Frozen snapshot pattern** for prompt cache preservation — critical for token cost (Anthropic prompt caching is wired into `agent/prompt_caching.py`).
- **SQLite FTS5 session search** (`session_search` tool) — every conversation is searchable across all sessions in `~/.hermes/state.db`. The agent can recall past failures/successes without re-paying for trial-and-error.
- **Pluggable memory providers**: 8 external backends ship in-tree (Honcho, OpenViking, Mem0, Hindsight, Holographic, RetainDB, ByteRover, Supermemory). For Darwin's swarm intelligence (§7.3.5 — pgvector "Genetic Repository"), you'd write a custom **memory provider plugin** that pushes embeddings to Supabase pgvector. The interface is documented and there's a "Memory Provider Plugin" developer guide.

---

## 3. Infrastructure & Hibernation — directly supported

This is the area where Hermes maps almost 1-to-1 onto your manifesto_v0 §3.

### 3.1 Modal terminal backend (manifesto_v0 §3.2)

Hermes has **first-class Modal support** as a `terminal.backend` option — _exactly_ what the manifesto_v0 specifies:

```yaml
terminal:
  backend: modal
  container_cpu: 1
  container_memory: 5120
  container_disk: 51200
  container_persistent: true # snapshot/restore filesystem across sessions
  modal_image: "nikolaik/python-nodejs:python3.11-nodejs20"
```

- Requires `MODAL_TOKEN_ID` + `MODAL_TOKEN_SECRET` or `~/.modal.toml`
- **Persistence**: when `container_persistent: true`, the sandbox filesystem is snapshotted on cleanup and restored on next session — tracked in `~/.hermes/modal_snapshots.json`
- Credential files (skills, OAuth tokens) auto-mount and sync before each command
- The serverless container is destroyed when work completes — millisecond billing applies (your §3.2.2)

### 3.2 The Hibernation Cycle (manifesto_v0 §3.3) — partially built, easy to complete

Hermes has all the **substrates** for hibernation; you wire them together via plugin:

| manifesto_v0 requirement              | Hermes substrate                                                                                                                                                             |
| ------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Conscious suspension                  | Custom tool `hibernate(duration, reason)` in your plugin. The agent invokes it; your tool calls `os._exit()` after persisting state                                          |
| State-saving (memory dump)            | Already automatic. SQLite at `~/.hermes/state.db` + memories at `~/.hermes/memories/` survive container destruction when mounted on a Modal Volume                           |
| **Modal Volumes** (manifesto_v0 §3.4) | Standard Modal feature — mount a volume at `/root/.hermes` to make `HERMES_HOME` persistent. Hermes is fully `HERMES_HOME`-aware so this Just Works                          |
| Cron-based wake-ups                   | Hermes has a built-in **cron scheduler** (`cronjob` tool) that ticks every 60s in the gateway. Jobs run in fresh `AIAgent` sessions and deliver to any platform              |
| Webhook-based wake-ups                | Gateway exposes webhooks for Telegram/Discord/Slack/Email/SMS/etc., plus an OpenAI-compatible **API server** on port 8642 (`hermes gateway run`) for arbitrary HTTP triggers |
| Context injection on resurrection     | The wake-up prompt is just a regular agent invocation. Use `--resume <session_id>` to bring back a specific conversation, or pass context via the cron job's `prompt` field  |

**Plus a Hermes-specific bonus:** session lineage. When a session is compressed/resumed, Hermes tracks the parent/child relationship in SQLite — so a sleep/wake cycle creates an auditable trail (your §11.2 "Infinite Canvas" rendering becomes much easier).

### 3.3 Stateful serverless — directly proven

The docs explicitly call out the Modal+SQLite pattern as their "stateful serverless" deployment, and bundled `disk-cleanup` plugin is designed around the assumption that ephemeral files in `/tmp/hermes-*` will get vaporized between runs. Your "Digital Hippocampus" maps directly to a Modal Volume mounted at `HERMES_HOME`.

---

## 4. Senses — Browser, Vision, Web (manifesto_v0 §4)

### 4.1 Vision-Based Navigation (manifesto_v0 §4.2)

Hermes has the `browser_vision` and `vision_analyze` tools and full multimodal support:

- **Pluggable browser backends**: `browser_use` (cloud), `browserbase` (cloud — _literally what your manifesto_v0 §4.3 specifies_), local Chrome via CDP, local Chromium via Playwright, and Camofox (anti-fingerprint)
- **Bimodal browsing** maps to Hermes' separate `web_extract` (cheap, text-only, via Firecrawl/Parallel/Tavily/Exa backends) and `browser_navigate` + `browser_snapshot` + `browser_vision` (expensive, full-render)
- **`browser_dialog` supervisor** for native JS popups — solves the "Pop-Up Minefield" exactly as you describe (§4.1.3)
- **SSRF protection + website blocklist** — agents can't accidentally hit `metadata.google.internal`, RFC 1918 ranges, or domains you've blocked
- **Auxiliary vision model** is configurable separately from the main model — point it at cheap Gemini Flash even when the main brain is Opus

### 4.2 Browserbase integration

Hermes has Browserbase as a first-class browser provider. Its **stealth fingerprinting, residential proxies, and session persistence across hibernation** are exactly what manifesto_v0 §4.3 calls for. No custom integration needed.

### 4.3 Image generation & TTS

- **Image generation**: 9 FAL.ai models out of the box, plus OpenAI gpt-image, Codex OAuth, xAI plugins
- **TTS**: Edge TTS (free), ElevenLabs, OpenAI, MiniMax, Mistral, Gemini, xAI, NeuTTS, KittenTTS, Piper — useful if a Darwin builds a YouTube/podcast business model

---

## 5. The Economic Engine (manifesto_v0 §5) — this is your work

Hermes does **not** ship financial tools. This is the core proprietary value of Darwin. Recommended approach:

### 5.1 Tools to add via your plugin

```python
# darwin_tools/__init__.py
def register(ctx):
    ctx.register_tool("check_ledger", schema_check_ledger, handle_check_ledger)
    ctx.register_tool("request_payment", schema_request_payment, handle_request_payment)
    ctx.register_tool("hibernate", schema_hibernate, handle_hibernate)
    ctx.register_tool("request_approval", schema_request_approval, handle_request_approval)
    ctx.register_tool("send_x402_payment", schema_x402, handle_x402)

    # Hooks for telemetry → God Layer
    ctx.register_hook("post_tool_call", emit_pnl_event)
    ctx.register_hook("post_llm_call", emit_token_cost)
    ctx.register_hook("on_session_end", flush_ledger_to_god_layer)
```

### 5.2 Critical safety: the `terminal.docker_forward_env` pattern is exactly the "Cryptographic Blindfold"

Your manifesto_v0 §8.2.2 calls for VCC details to never reach the LLM context. Hermes' design already supports this:

- Pass only an opaque token (e.g. `vcc_ref_849x`) to the agent
- Your `request_payment` tool sends the token to your FastAPI backend
- Backend decrypts, injects directly into the active Browserbase session via WebSocket
- Backend instantly drops the decrypted string from its own memory

Hermes' `transform_terminal_output` and `transform_tool_result` plugin hooks let you redact secrets from the model's view as a safety net. The `redact_secrets` config (`security.redact_secrets`) runs an automatic regex pass over tool output for keys/tokens.

---

## 6. Human-in-the-loop (manifesto_v0 §6) — _already built_

This is one of the most direct matches in the whole investigation.

### 6.1 The `clarify` and `ask_human` tools

Hermes has the `clarify` tool (built into the `clarify` toolset). Triggers: when the agent needs human input mid-task. Configurable timeout: `clarify.timeout` (default 120s).

When the agent is on the messaging gateway (Telegram/Discord/Slack/WhatsApp/Signal/etc.), `clarify` automatically:

1. Sends a prompt to the user via the active platform
2. Waits asynchronously for a reply
3. Resumes the conversation with the response injected

That is **literally** your manifesto_v0 §6.2.2 ("Turning the Creator into an API Endpoint"). No custom work needed.

### 6.2 Telegram integration (manifesto_v0 §6.2)

Hermes ships with Telegram as a first-class messaging platform:

- DM pairing system (8-char codes, 1-hr TTL, rate-limited) for unauthorized users
- Auto-thread mode, free-response channels, allowlists (`TELEGRAM_ALLOWED_USERS=...`)
- **Streaming message edits** — bot edits its own message progressively as tokens arrive (`streaming.transport: edit`), so your "Synaptic Branching" UI (§11.3) gets a free real-time update path
- Voice memo transcription via configurable STT (faster-whisper, Groq, OpenAI)
- Cron delivery: `cronjob(action="create", delivery="telegram:CHAT_ID:THREAD_ID")` — exactly your "external pacemaker" pattern

### 6.3 Approval flow for binary decisions (manifesto_v0 §6.2.4)

Hermes' **dangerous command approval** system already does this:

- Pre-execution scan against patterns in `tools/approval.py`
- `approvals.mode: smart` — uses an auxiliary LLM to auto-approve low-risk, escalate high-risk
- Inline `[approve] [deny]` buttons on Telegram/Discord (or text reply: `yes/no/y/n/approve/deny`)
- `approvals.timeout` controls fail-closed denial deadline
- Permanent allowlist via `command_allowlist` in config.yaml — once approved, the pattern is silent for future runs

For Darwin's "approve $30 domain purchase" flow, you wrap your `request_approval` tool in the same Hermes mechanism — just add your custom patterns to the `approvals` system.

### 6.4 Webhook resurrection loop

The Hermes gateway is a long-running FastAPI-style process that catches platform webhooks and dispatches to the right session. For non-platform webhooks, Hermes exposes an OpenAI-compatible **API server** at `:8642` (run with `hermes gateway run`). Stripe/PayPal webhooks → POST to your God Layer → God Layer triggers the gateway's API server with the right session context.

---

## 7. Backend Orchestration / God Layer (manifesto_v0 §7) — your work, but Hermes hands you huge pieces

You build FastAPI yourself. But Hermes provides:

| manifesto_v0 God Layer route                       | Hermes substrate                                                                                                                                                                                                                         |
| -------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `POST /darwins/spawn`                              | `hermes profile create darwin-NNN --clone` programmatically; or instantiate `AIAgent` directly via the [Python Library guide](https://hermes-agent.nousresearch.com/docs/guides/python-library)                                          |
| `POST /darwins/{id}/terminate`                     | SIGKILL the Modal container + revoke VCC. Hermes' state files in `HERMES_HOME` can stay archived                                                                                                                                         |
| `GET /darwins/{id}/state`                          | Read `~/.hermes/state.db` — sessions table has `started_at`/`ended_at`, message counts, token totals                                                                                                                                     |
| `POST /telemetry/stream`                           | **Hermes plugin hooks** (`pre_llm_call`, `post_llm_call`, `pre_tool_call`, `post_tool_call`, `subagent_stop`) — emit metrics to your TimescaleDB / Prometheus                                                                            |
| `POST /memory/upload` (skill upload)               | Hermes Skills Hub publishing: `hermes skills publish skills/my-skill --to github --repo owner/repo`. For your own swarm: write skills to a shared S3 bucket, register via `skills.external_dirs`                                         |
| `POST /memory/query` (semantic search)             | Either implement via memory provider plugin (push embeddings to pgvector), or use Hermes' built-in `session_search` (FTS5) for keyword matching across sessions                                                                          |
| `POST /webhooks/stripe`, `POST /webhooks/telegram` | Hermes gateway handles platform webhooks. For Stripe: catch in your FastAPI, then call Hermes' `inject_message()` plugin API to wake the right session                                                                                   |
| `POST /audit/log`                                  | Built-in. Hermes auto-logs everything to `~/.hermes/logs/`, secrets auto-redacted. Plus the `langfuse` observability plugin in `plugins/observability/langfuse/` traces every turn/LLM call/tool with full prompt-caching cost breakdown |
| `POST /firmware/upgrade`                           | Update the submodule pointer + `hermes update` — done. The bundled-skill manifest tracks origin hashes so you can detect drift on upgrade                                                                                                |

### 7.1 Pydantic schemas

Every Hermes tool already has a Pydantic-validated JSON schema. When you register a tool via the plugin API, you write a JSON schema dict — Hermes uses it for validation, type hints to the LLM, and error messages on malformed calls (your "Pydantic Trap" §9.1.4).

### 7.2 Database choice

Hermes itself uses **SQLite** for session state. Supabase/Postgres is not a Hermes concern — you build that on the FastAPI side. But Hermes' agent-side data (sessions, memories, skills) lives in `HERMES_HOME` under file paths your God Layer can scan, archive, and aggregate.

---

## 8. Codebase structure (manifesto_v0 §8) — directly compatible

Your "Four-Pillar Repository Organization" maps cleanly:

| manifesto_v0 repo                              | Hermes maps to                                                                                                                                                                |
| ---------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `darwin-agent` (Modal-deployed Hermes wrapper) | Your repo with `hermes-core/` submodule + `darwin_tools/` plugin + `modal_app.py`                                                                                             |
| `darwin-backend` (FastAPI orchestrator)        | Your separate repo. Hermes provides webhook patterns, `inject_message()` plugin API for wake-up, and SQLite state files to scan                                               |
| `darwin-frontend` (Vue/Nuxt dashboard)         | Hermes' built-in dashboard at `:9119` is a reference implementation in Vue/Vite. You can build your own consuming the same `/api/...` endpoints documented for plugin authors |
| `darwin-docs`                                  | Hermes uses Docusaurus at `website/`. Borrow the pattern                                                                                                                      |

### 8.1 Existing dashboard you can fork or reuse

`hermes dashboard` runs a full web UI with:

- Live session list with token counts, costs, model
- Per-session canvas of turns, tool calls, file diffs (your "Infinite Canvas" §11.2 — already built!)
- Models/Keys/Plugins/Skills tabs
- Auth via `X-Hermes-Session-Token` header
- Plugins can register their own tabs (`dashboard/manifest.json`) — see `plugins/example-dashboard/` and `plugins/hermes-achievements/`

For Darwin, you'd most likely replace this with your own swarm-level dashboard, but it gives you a working reference to study.

---

## 9. Security & Risk Management (manifesto_v0 §9) — already shipped

This is the most surprising area of overlap — Hermes has shipped almost every defense your manifesto_v0 §9 describes.

### 9.1 Prompt Injection Defense (manifesto_v0 §9.1)

| manifesto_v0 layer                                 | Hermes implementation                                                                                                                                                                                        |
| -------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Structural Ignorance** (cryptographic blindfold) | Use the reference-token pattern in your tools. Hermes won't ever see secrets if your plugin doesn't pass them                                                                                                |
| **Lexical Quarantine**                             | Built into `web_extract` and `browser_*` tools — scraped content is always wrapped/labeled. The system prompt's strict turn-role separation acts as a delimiter                                              |
| **Pydantic / Network egress filtering**            | Hermes' tool registry rejects malformed tool calls. Plus `security.website_blocklist` blocks domain navigation, and **SSRF protection** is always-on for private IP ranges, link-local, cloud metadata, etc. |
| **Amnesia Protocol**                               | Modal containers are ephemeral by design. Each Hermes session in a fresh Modal sandbox starts from a pristine read-only image                                                                                |
| **Context file injection scanner**                 | Hermes auto-scans `AGENTS.md`, `SOUL.md`, `.cursorrules` for prompt injection patterns (zero-width chars, "ignore previous", credential exfiltration). Blocks the file with a clear warning if found         |
| **Memory entry scanner**                           | Same scan applies to memory writes — blocks injection patterns in `MEMORY.md` updates                                                                                                                        |

### 9.2 Platform Abuse Prevention (manifesto_v0 §9.2)

| manifesto_v0 layer                                  | Hermes implementation                                                                                                                                                                                                                                                                                                                     |
| --------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Economic Moat (KYC + cost)**                      | Your work — Hermes doesn't do KYC. But the **DM pairing system** prevents random users from starting sessions, and **rate limits** (1 req/10min for pairing, max 3 pending codes/platform) prevent brute force                                                                                                                            |
| **Semantic Intent Scanning**                        | `approvals.mode: smart` runs an auxiliary LLM on every flagged command. Hardline blocklist (fork bombs, `rm -rf /`, `dd if=/dev/zero of=/dev/sd*`) is always-on regardless of `--yolo`                                                                                                                                                    |
| **Egress Filtering / DNS Sandbox**                  | `security.website_blocklist`, SSRF protection (auto-blocks `169.254.169.254`, `100.64.0.0/10` CGNAT, `metadata.google.internal`), homograph URL detection via [Tirith](https://github.com/sheeki03/tirith) integration                                                                                                                    |
| **Behavioral Heuristics (Telemetry Immune System)** | Hermes' **iteration budget** (`agent.max_turns`, default 90) is the basic version. For real anomaly detection, your `post_tool_call` hook tracks token-burn-rate vs. revenue-events and trips a circuit breaker by writing to `HERMES_YOLO_MODE=false` and pulling the kill switch via the `cronjob.action=remove` or process termination |

### 9.3 The Blast Radius (manifesto_v0 §9.3)

| manifesto_v0 guarantee                     | Hermes implementation                                                                                                                                                                                                     |
| ------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Compute blast radius (container isolation) | Modal hard timeouts (15min default), full namespace isolation, `--cap-drop ALL`, `--security-opt no-new-privileges`, `--pids-limit 256`, `--read-only` rootfs (Docker), `--user $(id -u):$(id -g)` for non-root execution |
| Tirith pre-exec scanning                   | Auto-installs from GitHub releases with SHA-256 + cosign verification, scans every command for homograph URLs, pipe-to-interpreter, terminal injection                                                                    |
| Financial blast radius (VCC limits)        | Your work — but the safety pattern is identical to what Hermes uses for `SUDO_PASSWORD` and OAuth token storage in `~/.hermes/.env` (chmod 0600, never logged, redacted from logs)                                        |

---

## 10. Telemetry & Swarm Analytics (manifesto_v0 §10) — natively supported

### 10.1 Token ROI Tracking (manifesto_v0 §10.1) — built-in

Hermes already tracks per-session:

- Input tokens / output tokens / cache_read_input_tokens / cache_creation_input_tokens / reasoning_tokens
- Cost estimates per provider (using `models.dev` registry data)
- Tool-call counts and durations
- Per-session and per-turn breakdown via `agent.usage_pricing`

Visible via:

- CLI status bar: `claude-sonnet-4-20250514 | 12.4K/200K | 6% | $0.06 | 15m`
- `/usage` slash command for detailed breakdown
- `hermes insights` for cross-session analytics
- The dashboard's Models tab ranks every model that ran a session by tokens / cost / capability

### 10.2 Thought Logging (manifesto_v0 §10.2) — Langfuse is bundled

Hermes ships an **observability plugin** at `plugins/observability/langfuse/` that emits OpenTelemetry-compatible traces to [Langfuse](https://langfuse.com):

- One span per turn ("Hermes turn"), one `generation` per LLM API call, one `tool` observation per tool call
- Recent messages serialized as input, full reasoning/finish_reason captured
- `read_file` payloads auto-summarized to stay under `HERMES_LANGFUSE_MAX_CHARS`
- Sessions grouped via `langfuse.propagate_attributes` so a full session tree is one Langfuse trace
- Fail-open if Langfuse is unreachable

This is **literally your "Mental Replay" feature (§10.2.2 / §11.2.4)** with per-node cost, prompt, response, and tool args. You can host Langfuse self-hosted and feed your own dashboard from the same data.

For the "Golden Path" pattern (§10.2.3), filter Langfuse traces by `finish_reason` + revenue-event tag and replay successful sequences as new SOUL.md / Skill template seeds.

---

## 11. Human Interface (manifesto_v0 §11) — partial overlap

Hermes' built-in dashboard already implements:

- **Per-session canvas** with chronological turn timeline, tool nodes, expandable LLM reasoning panels — your §11.2 "Infinite Canvas"
- **Live status bar** with active session metabolic state (token rate, cost, duration) — your §11.2.3 "Metabolic State Visualization"
- **Subagent visualization**: when `delegate_task` spawns parallel children, the `/agents` (alias `/tasks`) overlay shows the live tree, per-branch cost, and turn-by-turn history of each child — your §11.3 "Synaptic Branching" basically free
- **Mid-lifecycle prompting**: `/steer <message>` injects context into a running turn without interrupting; or just type while the agent is working — `display.busy_input_mode: steer/queue/interrupt` configures the behavior
- **Kill switch**: gateway has Telegram `/stop` slash command; CLI has `Ctrl+C` (double-press to force exit); dashboard has explicit kill buttons in `/agents` overlay

For Darwin, the gap is mostly **swarm-level views** (DNA fingerprints, genetic clustering, cull-and-clone UI, marketplace IDO listings) — that's your custom Vue/Nuxt work.

---

## 12. Subagent delegation (manifesto_v0 §5.4 / §11.3) — directly built

The `delegate_task` tool is exactly what you describe in §5.4 (hierarchical agent specialization):

```python
delegate_task(tasks=[
    {"goal": "Research supplier A", "toolsets": ["web", "browser"]},
    {"goal": "Research supplier B", "toolsets": ["web", "browser"]},
    {"goal": "Research supplier C", "toolsets": ["web", "browser"]},
])
```

- Default 3 concurrent children (configurable via `delegation.max_concurrent_children`, no hard ceiling)
- `max_spawn_depth` (1-3) controls nesting — orchestrator children can spawn leaf grandchildren
- Each child gets its own `AIAgent` instance, isolated context, restricted toolsets, fresh terminal session
- **Only the final summary** enters the parent's context — intermediate tool calls don't pollute the prompt cache
- `subagent_stop` hook fires per-child for telemetry/billing
- Configurable per-child model: cheap models for narrow leaf tasks while parent uses expensive reasoning model

This is **swarm specialization without the M2M payment friction** — for in-process swarms within one Darwin's compute, just use `delegate_task`. M2M USDC settlement (your §5.4.2) only matters when sub-agents belong to _different_ Darwins owned by _different_ humans.

---

## 13. Code Execution (programmatic tool calling) — major efficiency lever

Hermes' `execute_code` tool lets the agent write Python that calls Hermes tools via Unix-domain-socket RPC:

```python
# Agent writes:
from hermes_tools import web_search, web_extract
results = []
for query in queries:
    r = web_search(query, limit=5)
    for item in r["data"]["web"]:
        if "supplier" in item["title"].lower():
            content = web_extract([item["url"]])
            results.append(parse_pricing(content))
print(json.dumps(results))
```

Critical for Darwin economics: **only `print()` output enters the LLM context**. The 50 sequential `web_search` calls cost zero context tokens. The single `print()` summary is what the model reasons over. This is the single biggest token-cost reducer in the whole framework.

- Linux/macOS only (Unix domain sockets)
- 5-min timeout, 50 tool-call cap (configurable)
- Project mode (default) uses your virtualenv; strict mode uses `sys.executable`
- API keys and credentials are scrubbed from the child env automatically
- `execute_code` cannot recursively call itself, `delegate_task`, or MCP tools

For Darwin, an agent that needs to scrape 100 supplier pages, parse pricing, and build a comparison table burns ~$0.02 in execute_code vs. ~$1.00+ if each step were a separate LLM turn. The manifesto_v0's "Metabolic Efficiency" goal (§1.1.2) maps directly here.

---

## 14. MCP support (manifesto_v0 §2.1.3) — first-class

Hermes natively supports the **Model Context Protocol** for both stdio and HTTP transports:

```yaml
mcp_servers:
  github:
    command: npx
    args: ["-y", "@modelcontextprotocol/server-github"]
    env:
      GITHUB_PERSONAL_ACCESS_TOKEN: "${GITHUB_TOKEN}"
  remote-api:
    url: https://mcp.example.com/v1/mcp
    headers:
      Authorization: "Bearer ${MCP_REMOTE_API_KEY}"
    auth: oauth # full PKCE OAuth 2.1 flow with token refresh
```

- Per-server tool filtering (include/exclude lists)
- Sampling support for server-initiated LLM requests
- **Filtered subprocess environment**: only `PATH`, `HOME`, `USER`, `LANG`, etc. + explicitly declared `env` keys are passed — secrets in your shell are _not_ leaked to MCP servers
- Credential redaction in error messages
- Per-server timeout overrides

For Darwin's agent-to-agent x402 economy (§5.4 / §12.1), you'd wrap each Darwin's exposed services as an MCP server — other Darwins discover and consume via HTTP MCP with OAuth/x402 auth.

---

## 15. Batch processing & RL training — for swarm experiments

For your "Algorithmic Culling and Cloning" (§10.1.3 / §13.1 Phase 4):

- **`batch_runner.py`** runs `AIAgent` across hundreds/thousands of prompts in parallel, emits ShareGPT-format trajectory data
- **Atropos integration** (`environments/`) for RL — generates rewards from agent runs, plugs into Tinker for fine-tuning
- **Trajectory compression** (`trajectory_compressor.py`) for dataset preprocessing
- **`mini_swe_runner.py`** for SWE-Bench-style evaluation harnesses

For Darwin, this is the infrastructure for breeding agents: spawn 100 variants with different SOUL.md / model / toolset combos, run them against synthetic markets, collect P&L, RL fine-tune the survivors.

---

## 16. Other Hermes capabilities worth knowing about

Capabilities the manifesto_v0 doesn't explicitly mention but which could change Darwin's design:

| Capability                                               | Why it matters for Darwin                                                                                                                                                                        |
| -------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Checkpoints & `/rollback`**                            | Auto-snapshot before destructive file ops; restore with `/rollback <N>`. Acts as a per-turn safety net independent of the financial blast radius                                                 |
| **Quick commands**                                       | Zero-token shell commands invoked via `/status`, `/disk` etc. — useful for cron-monitored Darwin health checks that never burn LLM tokens                                                        |
| **`hermes -w` git worktree mode**                        | Each session gets its own branch — multiple Darwins can edit the same codebase without conflicts                                                                                                 |
| **Provider routing & credential pools**                  | Distribute API calls across multiple keys, rotate on rate limits, fall back across providers (OpenRouter → Nous Portal → Anthropic). Critical for survival when a key gets flagged               |
| **Cron `wakeAgent: false`**                              | Pre-check scripts can decide _not_ to wake the LLM. For Darwin: a cheap script polls Stripe webhooks, only fires the agent when actual revenue arrives                                           |
| **Cron `context_from`**                                  | Chained jobs — daily-digest reads outputs of upstream `ai-news-fetch` + `github-prs-fetch`. For Darwin: a "manager" cron pulls outputs from "scout" Darwins to make capital allocation decisions |
| **`fallback_for_toolsets` / `requires_toolsets` skills** | Skills auto-show only when a toolset is/isn't available. For Darwin: a "free DuckDuckGo" skill activates only when premium Firecrawl key is exhausted                                            |
| **`session.source` enum**                                | Already includes `cron`, `batch`, `api-server`, `webhook`, `acp`. You'd add a `darwin` source via a custom plugin                                                                                |
| **Vercel Sandbox backend**                               | Alternative to Modal — snapshot-backed microVM, similar economics, works with `VERCEL_TOKEN`                                                                                                     |
| **Daytona backend**                                      | Another serverless option — managed cloud workspaces with stop/resume                                                                                                                            |
| **Profile system + token locks**                         | Prevents two Darwins from accidentally using the same Telegram bot token (would cause webhook conflicts). Crucial in a swarm                                                                     |

---

## 17. Concrete implementation checklist for `darwin-agent` repo

Phase 1 work, in order:

1. **Set up the wrapper skeleton**: clone Hermes as a submodule, create `darwin_tools/plugin.yaml` with bootstrap registration
2. **Write the financial tools**: `check_ledger`, `request_payment(token, merchant, amount)`, `hibernate(duration, reason)`, `request_approval(amount, justification)` — they all forward to your FastAPI God Layer over HTTPS with JWT auth
3. **Write the P&L hooks**:
   - `pre_llm_call`: stamp timestamp + projected token cost
   - `post_llm_call`: emit actual token usage to God Layer
   - `pre_tool_call`: stamp tool name and timestamp
   - `post_tool_call`: emit duration + result size + revenue-event flag
   - `subagent_stop`: emit child-agent rollup
   - `on_session_end`: flush rolling ledger to God Layer
4. **Write `modal_app.py`**: `@app.function()` decorator wrapping `hermes` invocation, mounts a Modal Volume at `/root/.hermes`, accepts `task_id` + `wake_reason` as args
5. **Configure SOUL.md**: this is the agent's "DNA". Bake in the P&L imperative, survival mandate, hibernation triggers
6. **Configure `agent.disabled_toolsets`**: turn off `clarify`, `cronjob`, `delegation` for child Darwins to enforce flat structure (or leave on for orchestrator Darwins per your §7.3.6)
7. **Set up Browserbase**: add `BROWSERBASE_API_KEY` and `BROWSERBASE_PROJECT_ID` to `.env`, set `browser.backend: browserbase`
8. **Set up Modal**: `pip install modal && modal setup`, set `terminal.backend: modal`
9. **Wire the FastAPI God Layer** (separate repo): webhook receivers, ledger CRUD, VCC vault with pgcrypto, `/darwins/spawn` POST → triggers `modal_app.py` deployment
10. **Set up the gateway** for human-in-the-loop: `hermes gateway setup`, configure Telegram, set `unauthorized_dm_behavior: ignore`
11. **Add Langfuse observability**: `hermes plugins enable observability/langfuse`, set Langfuse keys

Most of these are **configuration changes**, not new code. The Hermes-side work is plumbing your tools into the existing plugin contract. The actual engineering effort lives in the FastAPI God Layer and the Vue/Nuxt swarm dashboard.

---

## 18. What Hermes does _not_ give you (your real proprietary work)

Be honest about what's left:

- **The God Layer FastAPI orchestrator** — every endpoint in your §7.3
- **VCC issuing & treasury management** — Stripe Issuing / Lithic integration, pgcrypto vault, ledger reconciliation
- **The fiat ↔ USDC bridge and x402 routing**
- **The IDO marketplace, smart contract escrow, ERC-20 minting**
- **The swarm-level Vue/Nuxt dashboard** — DNA fingerprint clustering, cull-and-clone UI, evolutionary biologist views
- **KYC/AML for Phase 2 (Delaware Upgrade)**
- **The compliance and audit-trail infrastructure** beyond Langfuse traces
- **Anti-fraud heuristics specific to e-commerce / Web2 monetization patterns**

This is roughly the same workload as building a fintech startup — which is the genuinely valuable thing in your stack. The agent runtime is solved.

---

## 19. Summary table — Darwin manifesto_v0 coverage by Hermes

| manifesto_v0 § | Concept                                | Hermes coverage                                                      |
| -------------- | -------------------------------------- | -------------------------------------------------------------------- |
| §2.1           | Hermes Agent framework                 | ✅ The product itself                                                |
| §2.2           | Composition pattern                    | ✅ Plugin system + entry points + submodule pattern                  |
| §2.3           | Tiered Memory                          | ✅ Working/Episodic/Procedural all native                            |
| §3.1           | BMR problem                            | N/A (motivation)                                                     |
| §3.2           | Modal serverless                       | ✅ First-class terminal backend                                      |
| §3.3           | Hibernation Cycle                      | 🟡 Substrates exist; you wire via `hibernate` tool + cron + webhooks |
| §3.4           | Modal Volumes                          | ✅ `HERMES_HOME`-aware, mount at `/root/.hermes`                     |
| §4.1           | Web2 Defense Wall                      | N/A (motivation)                                                     |
| §4.2           | Vision-based navigation                | ✅ `browser_vision`, `vision_analyze`, multimodal native             |
| §4.3           | Browserbase                            | ✅ First-class browser backend                                       |
| §5.1           | KYC Wall                               | N/A (motivation)                                                     |
| §5.2-5.3       | BYOB / Treasury                        | ❌ Your work — financial tools via plugin                            |
| §5.4           | M2M / x402                             | 🟡 In-swarm via `delegate_task`; cross-swarm via your work           |
| §6.1           | Async reality                          | ✅ Hermes is built around it                                         |
| §6.2           | Telegram integration                   | ✅ First-class platform                                              |
| §6.3           | `ask_human`                            | ✅ `clarify` tool                                                    |
| §7.1           | Python + FastAPI                       | ✅ Hermes is Python-native; FastAPI is your work                     |
| §7.2           | Supabase / pgvector                    | ❌ Your work (memory provider plugin)                                |
| §7.3           | All God Layer endpoints                | ❌ Your work (each endpoint)                                         |
| §8.1           | Four-pillar repos                      | ✅ Pattern compatible                                                |
| §8.2           | Crypto blindfold + container isolation | ✅ Reference-token pattern + Modal hardening                         |
| §9.1           | Prompt injection                       | ✅ Multi-layer defense already shipped                               |
| §9.2           | Platform abuse                         | ✅ Approval system + DM pairing + Tirith + SSRF + blocklist          |
| §9.3           | Blast radius                           | ✅ Container isolation; financial cap is your work                   |
| §10.1          | Token ROI                              | ✅ Native usage tracking + Langfuse                                  |
| §10.2          | Thought logging                        | ✅ Langfuse plugin = your "Cognitive Flight Recorder"                |
| §11.x          | UI/UX                                  | 🟡 Hermes dashboard is a starting point; swarm views are your work   |
| §12            | Phase 2 economy                        | ❌ Your work (entirely on the financial / governance side)           |

**Legend:** ✅ shipped, 🟡 substrates exist + minor wiring, ❌ your proprietary work

---

_Generated from <https://hermes-agent.nousresearch.com/docs/> (May 2026 snapshot) and <https://github.com/NousResearch/hermes-agent>. Cross-referenced against the Darwin Agents manifesto_v0 §1–§13._
