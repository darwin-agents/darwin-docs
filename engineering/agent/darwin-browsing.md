# Darwin Agent Browser Backends

## Browserbase vs Browser-use — Current State (May 2026)

---

## TL;DR

Both Browserbase and Browser-use are **native Hermes backends** — one env var each, same browser tools, no wrapper code needed. The real tradeoff is stealth rate vs. session persistence reliability. Neither is a clear winner today; the optimal setup uses both.

---

## What Hermes Actually Supports

Hermes has six browser backends out of the box:

| Backend              | Type                      | Config                                           |
| -------------------- | ------------------------- | ------------------------------------------------ |
| Browserbase          | Cloud (managed)           | `BROWSERBASE_API_KEY` + `BROWSERBASE_PROJECT_ID` |
| Browser-use          | Cloud (managed)           | `BROWSER_USE_API_KEY`                            |
| Firecrawl            | Cloud (scraping-focused)  | `FIRECRAWL_API_KEY`                              |
| Camofox              | Local (Firefox + stealth) | `CAMOFOX_URL`                                    |
| Local Chrome via CDP | Local                     | `/browser connect`                               |
| agent-browser CLI    | Local (Chromium)          | auto-installed                                   |

When both Browserbase and Browser-use keys are set, **Browserbase takes priority** by default. You can override this per-profile in `~/.hermes/config.yaml`.

---

## The Core Tradeoff

### Stealth rate

Browser-use Cloud runs a custom Chromium fork with C++-level stealth patches and a purpose-built `bu-ultra` model that completes tasks in fewer steps. On the Browser-use Stealth Benchmark (71 sites — Cloudflare, Akamai, PerimeterX, Datadome), the numbers are stark:

| Provider          | Stealth success rate |
| ----------------- | -------------------- |
| Browser-use Cloud | ~81%                 |
| Browserbase       | ~42%                 |

For Darwin agents, a blocked session bleeds P&L without generating value — the exact death spiral the manifesto describes. The stealth gap is real and operationally significant.

### Session persistence across hibernation

This is where Browserbase still has the edge — but it's a narrowing one.

**Browserbase:** has a dedicated keep-alive API. Darwin's `lifecycle.py` calls it explicitly at `POST /finance/keep-browser-session` before the Modal container dies. On wake-up, `browser_navigate(existing_session_id=session_id)` resumes the exact authenticated tab — cookies, 2FA state, mid-checkout position — regardless of how long the agent slept.

**Browser-use:** persistence in Hermes currently relies on:

- `BROWSER_PERSIST_BETWEEN_TURNS=1` — prevents per-turn cleanup
- `BROWSER_INACTIVITY_TIMEOUT` set to a large value (e.g. `2592000` for 30 days) as a workaround

There are active open bugs (Hermes issue #11020) where per-turn cleanup kills persistent sessions, and the `BROWSER_INACTIVITY_TIMEOUT=0` sentinel does the opposite of what's documented. The workarounds exist, but they're fragile relative to an explicit API call.

**Camofox** (local self-hosted) has the most robust persistence model — `browser.camofox.managed_persistence: true` sends a deterministic `userId` to the Firefox server, which reuses the same profile across full agent restarts. No timeout games. But it requires running your own Camofox server and has no residential proxy pool.

---

## Darwin-Specific Requirements Checklist

| Requirement                                     | Browserbase     | Browser-use           | Camofox                 |
| ----------------------------------------------- | --------------- | --------------------- | ----------------------- |
| Session keep-alive across Modal container death | ✅ explicit API | ⚠️ workaround (buggy) | ✅ robust (self-hosted) |
| Screenshots for VLM vision pipeline             | ✅              | ✅                    | ✅                      |
| Residential proxy included                      | ✅              | ✅ 195 countries      | ❌                      |
| CAPTCHA solving                                 | ✅              | ✅                    | ❌                      |
| Zero Modal container weight (offloaded)         | ✅              | ✅                    | ❌ (sidecar)            |
| Cloudflare bypass                               | ⚠️ 42%          | ✅ 81%                | ⚠️ variable             |
| Native Hermes backend (no wrapper)              | ✅              | ✅                    | ✅                      |
| VCC inject via `browserbase_service`            | ✅              | ❌ needs rewrite      | ❌                      |
| `/sessions/{id}/stream` WebSocket relay         | ✅ built-in     | ❌ needs rewrite      | ❌                      |
| Production stability                            | ✅ GA + SOC 2   | ✅ GA + SOC 2         | ⚠️ self-managed         |

---

## The `darwin-backend` coupling problem

Two backend services are currently hardcoded to Browserbase:

**1. VCC injection** (`darwin-backend`):

```python
await browserbase_service.type_field(browserbase_session_id, fields["pan_element"], pan)
await browserbase_service.type_field(browserbase_session_id, fields["cvv_element"], cvv)
await browserbase_service.type_field(browserbase_session_id, fields["expiry_element"], expiry)
```

**2. Live CAPTCHA stream** (`/sessions/{id}/stream` WebSocket):

```python
# Opens WebSocket to Browserbase's live-stream endpoint
# Bidirectional relay: video frames → human browser; touch events → Browserbase
```

Switching to Browser-use as primary would require abstracting both of these behind a `BrowserBackend` interface. Not complex, but not zero work — estimate 1–2 days of backend refactoring.

---

## Recommended Architecture

Rather than choosing one, use a layered strategy that maps each backend to the task type it wins at:

### Layer 1 — Lightpanda (inside Modal, Fetch mode)

For open/text-heavy pages, public APIs, RSS feeds. Runs inside the Modal container itself.

- 11× faster than Chrome, 9× less RAM (24MB vs 207MB peak)
- CDP-compatible — drop-in for Puppeteer/Playwright scripts
- No screenshots, no stealth — strictly Fetch mode only
- AGPL-3.0, currently beta

### Layer 2 — Browserbase (primary Session mode)

For all authenticated flows, VCC injection, CAPTCHA handoff, and any session that needs to survive hibernation.

- Keep as primary until the Browser-use persistence bugs are resolved
- Required for `/sessions/{id}/stream` WebSocket relay (Telegram CAPTCHA handoff)
- Required for the current `browserbase_service` VCC injection flow

### Layer 3 — Browser-use Cloud (stealth fallback)

For targets that actively block Browserbase (aggressive Cloudflare, Datadome, PerimeterX sites).

- Agent detects a Browserbase block → retries the task via Browser-use
- Already a valid Hermes backend — switching is one config key
- Implement as a tool-level retry, not a full reroute

```yaml
# ~/.hermes/config.yaml (Darwin agent profile)
browser:
  cloud_provider: browserbase # primary
  # BROWSER_USE_API_KEY set in .env as stealth fallback
  auto_local_for_private_urls: true # Lightpanda/local for LAN/localhost
```

---

## Migration Path to Browser-use as Primary

When Hermes fixes issue #11020 and Browser-use ships explicit session persistence (tracked), the migration is:

1. Abstract `browserbase_service` into `browser_backend_service` with Browserbase and Browser-use implementations
2. Abstract `/sessions/{id}/stream` to proxy either provider's stream format
3. Flip `cloud_provider: browser_use` in the Darwin agent config
4. Update `POST /finance/keep-browser-session` to call Browser-use's session API instead of Browserbase's

Estimated effort: 2–3 days. The Hermes tool layer requires zero changes.

---

## Open Issues to Watch

| Issue                                           | Where         | Impact                                 |
| ----------------------------------------------- | ------------- | -------------------------------------- |
| Per-turn cleanup kills persistent sessions      | Hermes #11020 | Browser-use persistence on hibernation |
| `BROWSER_INACTIVITY_TIMEOUT=0` inverts behavior | Hermes #11020 | Aggressive session reaping             |
| `.env.example` ships aggressive 2-min timeout   | Hermes #11020 | Default breaks persistent workflows    |

---

## Decision Summary

> **Keep Browserbase as primary Session-mode backend now.** Use Browser-use as a stealth fallback for high-friction targets. Revisit primary switch once Hermes #11020 is resolved and Browser-use ships an explicit session persistence API comparable to Browserbase's keep-alive endpoint.

The stealth argument for Browser-use is strong. The persistence argument for Browserbase is currently stronger for Darwin's hibernation-heavy workload. Both will be true simultaneously once the bug is fixed.
