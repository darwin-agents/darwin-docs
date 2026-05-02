# darwin-frontend — Phase 1 Engineering Specification

> **Audience:** Engineers joining the project.  
> **Scope:** Phase 1 MVP — one human creator, one Darwin agent, its neurons, and the Command Console. No swarms, no IDO marketplace.  
> **Stack:** Vue 3 · Nuxt 3 · Pinia · Supabase Realtime · TailwindCSS · TypeScript

---

## Table of Contents

1. [Role of the Frontend](#1-role-of-the-frontend)
2. [Technology Stack & Rationale](#2-technology-stack--rationale)
3. [Repository Structure](#3-repository-structure)
4. [Routing Architecture](#4-routing-architecture)
5. [Supabase Realtime — The Data Pipeline](#5-supabase-realtime--the-data-pipeline)
6. [Pinia Stores](#6-pinia-stores)
   - 6.1 `useAgentStore`
   - 6.2 `useCanvasStore`
   - 6.3 `useNotificationStore`
   - 6.4 `useAuthStore`
7. [Page Components](#7-page-components)
   - 7.1 Landing / Login (`/`)
   - 7.2 Dashboard (`/dashboard`)
   - 7.3 Infinite Canvas (`/canvas/[id]`)
   - 7.4 Spawn Agent (`/spawn`)
   - 7.5 Mobile Interstitial (`/resolve`)
8. [The Infinite Canvas](#8-the-infinite-canvas)
   - 8.1 Layout & Coordinate System
   - 8.2 Node Types & Visual Language
   - 8.3 Neural Thread Branching
   - 8.4 Metabolic State Visualization
   - 8.5 Canvas Virtualization
   - 8.6 Mental Replay Slide-Over
9. [Component Library](#9-component-library)
   - 9.1 `<CanvasViewport />`
   - 9.2 `<DarwinTrunk />`
   - 9.3 `<NeuralThread />`
   - 9.4 `<CanvasNode />`
   - 9.5 `<BudgetGauge />`
   - 9.6 `<HeartbeatBar />`
   - 9.7 `<MentalReplayPanel />`
   - 9.8 `<CommandConsole />`
   - 9.9 `<SecretsVault />`
   - 9.10 `<EntityPanel />`
   - 9.11 `<SkillsPanel />`
   - 9.12 `<LedgerPanel />`
   - 9.13 `<InterventionNode />`
   - 9.14 `<MobileInterstitial />`
   - 9.15 `<SpawnForm />`
10. [API Client Layer](#10-api-client-layer)
11. [TypeScript Types](#11-typescript-types)
12. [Design System](#12-design-system)
13. [Deployment](#13-deployment)
14. [Environment Variables](#14-environment-variables)
15. [Testing Strategy](#15-testing-strategy)

---

## 1. Role of the Frontend

The frontend has two distinct purposes that must never be conflated:

**1. The Infinite Canvas** — a real-time, zoomable, horizontally-scrolling visualization of every decision, thought, tool call, sub-agent spawn, financial event, and human intervention that has ever happened in a Darwin agent's life. Time runs left to right. The human creator watches this as a live feed while the agent is active, and audits it historically after sessions end.

**2. The Command Console** — an overlay panel that lets the human creator intervene: inject mid-lifecycle prompts, add budget, toggle autonomy, manage credentials, and hit the kill switch.

The frontend is purely a **display and control surface**. It never talks to Modal directly. It never touches the agent. It only ever talks to `darwin-backend` over HTTPS REST, and receives real-time event data from Supabase Realtime (which is a WebSocket connection to the database, not to the backend).

The golden rule: **the frontend must display everything the agent does, with zero polling**. All data arrives via Supabase Realtime push. The canvas updates the instant the agent emits an event.

---

## 2. Technology Stack & Rationale

| Layer | Choice | Why |
|---|---|---|
| Framework | **Nuxt 3** | Hybrid rendering — ISR for public pages, pure CSR for the canvas |
| UI library | **Vue 3 Composition API** | Proxy-based reactivity updates only the changed DOM node, not the whole canvas |
| State | **Pinia** | Single store per domain; Vue devtools integration; typed stores |
| Styling | **TailwindCSS** | Utility-first; no runtime CSS-in-JS overhead on a 60fps canvas |
| Real-time | **Supabase Realtime** | Direct WebSocket to the `darwin_events` table; no backend intermediary |
| Canvas rendering | **Native SVG + CSS transforms** | No canvas library dependency; precise control over node layout and animation |
| HTTP client | **$fetch (Nuxt built-in)** | SSR-compatible; auto-handles cookies and CSRF |
| Type safety | **TypeScript** throughout | Every event payload type mirrors the backend Pydantic schemas |
| Icons | **Lucide Vue** | Consistent icon set; tree-shakeable |
| Auth | **Supabase Auth (Vue SDK)** | Magic links and OAuth; session persisted in cookies for SSR |

**Why not React / Next.js?**  
The canvas renders thousands of nodes with live data mutations every few seconds. Vue 3's fine-grained proxy reactivity tracks exactly which `ref()` changed and surgically updates only the corresponding DOM node. React's VDOM diffing re-evaluates the whole component subtree on every state change — at canvas scale this drops to <30fps without heavy memoization that becomes a maintenance burden.

**Why native SVG, not a canvas library (Konva, Pixi, etc.)?**  
Nodes need to be real DOM elements — they receive click events, contain interactive Vue components (approval buttons, screenshot thumbnails), and must be accessible. A `<canvas>` bitmap element cannot host Vue components. The performance ceiling of SVG is sufficient for a single-agent timeline.

---

## 3. Repository Structure

```
darwin-frontend/
├── nuxt.config.ts                 # Nuxt config — route rules, modules, build
├── tailwind.config.ts
├── tsconfig.json
│
├── app.vue                        # Root layout — auth guard + global error boundary
│
├── pages/
│   ├── index.vue                  # Public landing page (ISR)
│   ├── login.vue                  # Auth page (ISR)
│   ├── dashboard.vue              # Agent list + P&L overview (CSR)
│   ├── spawn.vue                  # New agent form (CSR)
│   ├── canvas/
│   │   └── [id].vue               # Infinite Canvas for agent [id] (CSR, no SSR)
│   └── resolve.vue                # Mobile interstitial — 2FA / CAPTCHA / approval (CSR)
│
├── components/
│   ├── canvas/
│   │   ├── CanvasViewport.vue     # Pan/zoom container; virtual scroll engine
│   │   ├── DarwinTrunk.vue        # Main agent timeline track
│   │   ├── NeuralThread.vue       # Sub-agent parallel lane
│   │   ├── CanvasNode.vue         # Generic node renderer — dispatches by event_type
│   │   ├── nodes/
│   │   │   ├── GenesisNode.vue    # AGENT_SPAWNED
│   │   │   ├── ThoughtNode.vue    # THOUGHT
│   │   │   ├── ToolNode.vue       # TOOL_CALL / TOOL_RESULT
│   │   │   ├── FinancialNode.vue  # LEDGER_DEBIT / LEDGER_CREDIT
│   │   │   ├── SkillNode.vue      # SKILL_CREATED / SKILL_LOADED
│   │   │   ├── EntityNode.vue     # ENTITY_CREATED / ENTITY_UPDATED
│   │   │   ├── MemoryNode.vue     # EPISODIC_SUMMARY_WRITTEN
│   │   │   ├── InterventionNode.vue # ASK_HUMAN_SENT / APPROVAL_REQUESTED
│   │   │   ├── HibernateNode.vue  # AGENT_HIBERNATED
│   │   │   ├── WakeNode.vue       # AGENT_WOKE
│   │   │   ├── SynapticNode.vue   # SYNAPTIC_INTEGRATION
│   │   │   └── TerminatedNode.vue # AGENT_TERMINATED
│   │   ├── BudgetGauge.vue        # Animated budget bar — top of canvas
│   │   ├── HeartbeatBar.vue       # Real-time burn rate sparkline
│   │   └── MentalReplayPanel.vue  # Slide-over: screenshot + thought trace
│   │
│   ├── console/
│   │   ├── CommandConsole.vue     # Right-panel overlay — all interventions
│   │   ├── CognitiveUplink.vue    # Mid-lifecycle prompt injection
│   │   ├── SecretsVault.vue       # VCC + API key management
│   │   ├── MetabolicInjection.vue # Budget top-up
│   │   ├── AutonomyToggle.vue     # Enable/disable ask_human gate
│   │   └── KillSwitch.vue         # Emergency termination
│   │
│   ├── panels/
│   │   ├── EntityPanel.vue        # Right sidebar: all agent-created entities
│   │   ├── SkillsPanel.vue        # Right sidebar: compiled skill documents
│   │   ├── LedgerPanel.vue        # Right sidebar: transaction history
│   │   └── AssetsPanel.vue        # Right sidebar: uploaded files
│   │
│   ├── mobile/
│   │   └── MobileInterstitial.vue # Full-screen: 2FA input / CAPTCHA relay / approval
│   │
│   ├── spawn/
│   │   └── SpawnForm.vue          # Multi-step agent creation form
│   │
│   └── ui/                        # Shared atoms
│       ├── DarwinButton.vue
│       ├── DarwinBadge.vue
│       ├── DarwinCard.vue
│       ├── DarwinModal.vue
│       └── DarwinToast.vue
│
├── stores/
│   ├── agent.ts                   # useAgentStore — FSM state, balance, config
│   ├── canvas.ts                  # useCanvasStore — ordered event list, neuron lanes
│   ├── notification.ts            # useNotificationStore — pending interventions
│   └── auth.ts                    # useAuthStore — Supabase session
│
├── composables/
│   ├── useRealtimeCanvas.ts       # Supabase Realtime subscription + event routing
│   ├── useCanvasLayout.ts         # X-position calculation from timestamp_ms
│   ├── useVirtualScroll.ts        # Viewport-based node mount/unmount
│   ├── useApprovalFlow.ts         # Approve/deny handler → PATCH /darwins/{id}/config
│   └── useInterventionJWT.ts      # Decode resolve?token= for mobile interstitial
│
├── types/
│   ├── events.ts                  # All darwin_events payload shapes (mirrors backend schemas)
│   ├── agent.ts                   # Agent, Session, DarwinState types
│   ├── finance.ts                 # LedgerTransaction, VCC types
│   └── canvas.ts                  # CanvasNode, NeuronLane, CanvasLayout types
│
├── utils/
│   ├── formatCost.ts              # "$0.042" formatting
│   ├── formatDuration.ts          # "8h 22m" from seconds
│   ├── eventColor.ts              # Node color by event_type
│   └── tokenRoi.ts                # Token ROI calculation
│
└── middleware/
    └── auth.ts                    # Global auth guard — redirect to /login if no session
```

---

## 4. Routing Architecture

Nuxt 3's route rules define the rendering strategy per route.

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  routeRules: {
    // Public pages — pre-rendered at build time, revalidated every 60s
    '/':          { isr: 60 },
    '/login':     { isr: 60 },

    // Authenticated app — pure client-side, no SSR
    // SSR would require the server to maintain Supabase Realtime WebSocket connections
    // and hold canvas state server-side, which defeats the point
    '/dashboard': { ssr: false },
    '/spawn':     { ssr: false },
    '/canvas/**': { ssr: false },
    '/resolve':   { ssr: false },
  },

  modules: [
    '@nuxtjs/supabase',
    '@pinia/nuxt',
    '@nuxtjs/tailwindcss',
  ],

  supabase: {
    url: process.env.SUPABASE_URL,
    key: process.env.SUPABASE_ANON_KEY,
    redirectOptions: {
      login: '/login',
      callback: '/dashboard',
      exclude: ['/', '/login'],
    },
  },
})
```

---

## 5. Supabase Realtime — The Data Pipeline

This is the most critical architectural piece. Every canvas update arrives through this pipeline — not through polling, not through a backend WebSocket, but directly from the Supabase database via a Realtime subscription.

```
darwin-agent (Modal) 
  → POST /telemetry/stream-event 
  → darwin-backend inserts into darwin_events table
  → Supabase Realtime broadcasts to all subscribers
  → Frontend canvas updates in < 200ms
```

### `composables/useRealtimeCanvas.ts`

```typescript
import { useSupabaseClient } from '#imports'
import { useCanvasStore } from '~/stores/canvas'
import { useAgentStore } from '~/stores/agent'
import { useNotificationStore } from '~/stores/notification'
import type { DarwinEvent } from '~/types/events'

export function useRealtimeCanvas(darwinId: string) {
  const supabase = useSupabaseClient()
  const canvas = useCanvasStore()
  const agent = useAgentStore()
  const notifications = useNotificationStore()

  let channel: ReturnType<typeof supabase.channel> | null = null

  function subscribe() {
    channel = supabase
      .channel(`darwin:${darwinId}`)
      .on(
        'postgres_changes',
        {
          event: 'INSERT',
          schema: 'public',
          table: 'darwin_events',
          filter: `darwin_id=eq.${darwinId}`,
        },
        (payload) => {
          const event = payload.new as DarwinEvent
          routeEvent(event)
        }
      )
      .subscribe()
  }

  function routeEvent(event: DarwinEvent) {
    // 1. All events go to the canvas store (for timeline rendering)
    canvas.appendEvent(event)

    // 2. Lifecycle events also update agent store
    if (['AGENT_SPAWNED', 'AGENT_WOKE', 'AGENT_HIBERNATED', 'AGENT_TERMINATED'].includes(event.event_type)) {
      agent.handleLifecycleEvent(event)
    }

    // 3. Financial events update the live budget gauge
    if (['LEDGER_DEBIT', 'LEDGER_CREDIT'].includes(event.event_type)) {
      agent.updateBalance(event.payload.balance_after)
    }

    // 4. Intervention events create notification banners
    if (['ASK_HUMAN_SENT', 'APPROVAL_REQUESTED', 'CAPTCHA_DELEGATED'].includes(event.event_type)) {
      notifications.add(event)
    }

    // 5. Intervention resolved — clear the pending notification
    if (['ASK_HUMAN_RESOLVED', 'APPROVAL_GRANTED', 'APPROVAL_DENIED'].includes(event.event_type)) {
      notifications.resolve(event)
    }
  }

  function unsubscribe() {
    if (channel) supabase.removeChannel(channel)
  }

  return { subscribe, unsubscribe }
}
```

### Loading historical events on mount

When the canvas page first loads, it fetches all past events from the REST API before subscribing to Realtime. This ensures the historical timeline is present before live events start streaming in.

```typescript
// In pages/canvas/[id].vue — onMounted
const { data } = await useFetch(`/api/v1/darwins/${darwinId}/events`, {
  headers: { Authorization: `Bearer ${session.value?.access_token}` },
  query: { limit: 2000, order: 'timestamp_ms.asc' },
})
canvas.loadHistoricalEvents(data.value.events)
// Then subscribe to real-time
const { subscribe } = useRealtimeCanvas(darwinId)
subscribe()
```

---

## 6. Pinia Stores

### 6.1 `useAgentStore` — `stores/agent.ts`

The source of truth for the agent's current FSM state, live balance, and configuration. Components read from here; they never fetch directly.

```typescript
import { defineStore } from 'pinia'
import type { AgentState, DarwinAgent } from '~/types/agent'
import type { DarwinEvent } from '~/types/events'

export const useAgentStore = defineStore('agent', () => {
  // State
  const agent = ref<DarwinAgent | null>(null)
  const state = ref<AgentState>('PENDING')
  const balance = ref(0)
  const burnRatePerHour = ref(0)
  const sessionId = ref<string | null>(null)
  const isAutonomyEnabled = ref(false)
  const lastEventAt = ref<string | null>(null)

  // Derived
  const pctRemaining = computed(() =>
    agent.value ? (balance.value / agent.value.initial_budget) * 100 : 0
  )
  const isActive = computed(() => state.value === 'ACTIVE')
  const isHibernating = computed(() => state.value === 'HIBERNATING')
  const isTerminated = computed(() => state.value === 'TERMINATED')

  // Actions
  function handleLifecycleEvent(event: DarwinEvent) {
    lastEventAt.value = new Date().toISOString()
    switch (event.event_type) {
      case 'AGENT_SPAWNED':
        state.value = 'ACTIVE'
        sessionId.value = event.session_id
        balance.value = event.payload.initial_budget
        break
      case 'AGENT_WOKE':
        state.value = 'ACTIVE'
        sessionId.value = event.session_id
        balance.value = event.payload.balance_at_wake
        break
      case 'AGENT_HIBERNATED':
        state.value = 'HIBERNATING'
        sessionId.value = null
        break
      case 'AGENT_TERMINATED':
        state.value = 'TERMINATED'
        sessionId.value = null
        balance.value = event.payload.final_balance
        break
    }
  }

  function updateBalance(newBalance: number) {
    balance.value = newBalance
  }

  async function fetchAgent(darwinId: string) {
    const session = useSupabaseSession()
    const data = await $fetch(`/api/v1/darwins/${darwinId}/state`, {
      headers: { Authorization: `Bearer ${session.value?.access_token}` },
    })
    state.value = data.state
    balance.value = data.ledger_balance
    burnRatePerHour.value = data.burn_rate_per_hour
    sessionId.value = data.current_session_id
  }

  async function postUplink(darwinId: string, message: string, attachments: object[] = []) {
    const session = useSupabaseSession()
    await $fetch(`/api/v1/darwins/${darwinId}/uplink`, {
      method: 'POST',
      headers: { Authorization: `Bearer ${session.value?.access_token}` },
      body: { message, attachments },
    })
  }

  async function patchConfig(darwinId: string, patch: { requires_approval?: boolean; budget_injection?: number }) {
    const session = useSupabaseSession()
    await $fetch(`/api/v1/darwins/${darwinId}/config`, {
      method: 'PATCH',
      headers: { Authorization: `Bearer ${session.value?.access_token}` },
      body: patch,
    })
    if (patch.requires_approval !== undefined) {
      isAutonomyEnabled.value = !patch.requires_approval
    }
    if (patch.budget_injection) {
      balance.value += patch.budget_injection
    }
  }

  async function terminate(darwinId: string) {
    const session = useSupabaseSession()
    await $fetch(`/api/v1/darwins/${darwinId}/terminate`, {
      method: 'POST',
      headers: { Authorization: `Bearer ${session.value?.access_token}` },
    })
    state.value = 'TERMINATED'
  }

  return {
    agent, state, balance, burnRatePerHour, sessionId,
    isAutonomyEnabled, lastEventAt, pctRemaining,
    isActive, isHibernating, isTerminated,
    handleLifecycleEvent, updateBalance, fetchAgent,
    postUplink, patchConfig, terminate,
  }
})
```

### 6.2 `useCanvasStore` — `stores/canvas.ts`

Manages the ordered list of canvas events and the neuron lane map. This is the data model the Infinite Canvas renders from.

```typescript
import { defineStore } from 'pinia'
import type { DarwinEvent } from '~/types/events'
import type { NeuronLane } from '~/types/canvas'

export const useCanvasStore = defineStore('canvas', () => {
  // All events, ordered by timestamp_ms ascending
  const events = ref<DarwinEvent[]>([])

  // Active neuron lanes: neuron_id → { events, status }
  const neuronLanes = ref<Map<string, NeuronLane>>(new Map())

  // Currently selected event (for Mental Replay)
  const selectedEventId = ref<string | null>(null)

  // Canvas viewport state
  const scrollX = ref(0)          // horizontal scroll position (px)
  const zoomLevel = ref(1)         // 1 = default; 0.5 = zoomed out; 2 = zoomed in

  // Derived: events for the main trunk (no neuron_id)
  const trunkEvents = computed(() =>
    events.value.filter(e => !e.neuron_id)
  )

  // Derived: max timestamp_ms (right edge of the canvas)
  const maxTimestamp = computed(() =>
    events.value.reduce((max, e) => Math.max(max, e.timestamp_ms), 0)
  )

  function appendEvent(event: DarwinEvent) {
    // Deduplicate — Supabase Realtime can deliver duplicates on reconnect
    if (events.value.some(e => e.id === event.id)) return
    events.value.push(event)

    // Route neuron events to their lane
    if (event.neuron_id) {
      if (!neuronLanes.value.has(event.neuron_id)) {
        neuronLanes.value.set(event.neuron_id, {
          neuron_id: event.neuron_id,
          status: 'active',
          events: [],
          spawn_timestamp_ms: event.timestamp_ms,
          end_timestamp_ms: null,
        })
      }
      const lane = neuronLanes.value.get(event.neuron_id)!
      lane.events.push(event)
      if (['NEURON_COMPLETED', 'NEURON_FAILED'].includes(event.event_type)) {
        lane.status = event.event_type === 'NEURON_COMPLETED' ? 'completed' : 'failed'
        lane.end_timestamp_ms = event.timestamp_ms
      }
    }
  }

  function loadHistoricalEvents(historicalEvents: DarwinEvent[]) {
    events.value = []
    neuronLanes.value.clear()
    historicalEvents.forEach(appendEvent)
  }

  function selectEvent(eventId: string | null) {
    selectedEventId.value = eventId
  }

  function setScroll(x: number) {
    scrollX.value = x
  }

  function setZoom(level: number) {
    zoomLevel.value = Math.max(0.25, Math.min(4, level))
  }

  return {
    events, neuronLanes, selectedEventId,
    scrollX, zoomLevel,
    trunkEvents, maxTimestamp,
    appendEvent, loadHistoricalEvents,
    selectEvent, setScroll, setZoom,
  }
})
```

### 6.3 `useNotificationStore` — `stores/notification.ts`

Manages pending human interventions — the amber banners that appear when the agent needs input.

```typescript
import { defineStore } from 'pinia'
import type { DarwinEvent } from '~/types/events'

export interface PendingIntervention {
  id: string                   // event ID
  event_type: string
  payload: Record<string, unknown>
  received_at: string
  resolved: boolean
}

export const useNotificationStore = defineStore('notification', () => {
  const interventions = ref<PendingIntervention[]>([])

  const pending = computed(() => interventions.value.filter(i => !i.resolved))

  function add(event: DarwinEvent) {
    interventions.value.push({
      id: event.id,
      event_type: event.event_type,
      payload: event.payload,
      received_at: new Date().toISOString(),
      resolved: false,
    })
  }

  function resolve(event: DarwinEvent) {
    // Match by correlating the resolution event back to the trigger
    // ASK_HUMAN_RESOLVED resolves ASK_HUMAN_SENT, APPROVAL_GRANTED resolves APPROVAL_REQUESTED
    const pendingMap: Record<string, string> = {
      'ASK_HUMAN_RESOLVED': 'ASK_HUMAN_SENT',
      'CAPTCHA_SOLVED': 'CAPTCHA_DELEGATED',
      'APPROVAL_GRANTED': 'APPROVAL_REQUESTED',
      'APPROVAL_DENIED': 'APPROVAL_REQUESTED',
    }
    const triggerId = pendingMap[event.event_type]
    if (triggerId) {
      const match = interventions.value.find(i => i.event_type === triggerId && !i.resolved)
      if (match) match.resolved = true
    }
  }

  function dismiss(id: string) {
    const item = interventions.value.find(i => i.id === id)
    if (item) item.resolved = true
  }

  return { interventions, pending, add, resolve, dismiss }
})
```

### 6.4 `useAuthStore` — `stores/auth.ts`

Thin wrapper around Supabase Auth. Components use this rather than calling `useSupabaseSession()` directly.

```typescript
import { defineStore } from 'pinia'

export const useAuthStore = defineStore('auth', () => {
  const supabase = useSupabaseClient()
  const session = useSupabaseSession()

  const userId = computed(() => session.value?.user?.id ?? null)
  const accessToken = computed(() => session.value?.access_token ?? null)
  const isAuthenticated = computed(() => !!session.value)

  async function loginWithMagicLink(email: string) {
    return supabase.auth.signInWithOtp({ email, options: { emailRedirectTo: '/dashboard' } })
  }

  async function loginWithGoogle() {
    return supabase.auth.signInWithOAuth({ provider: 'google', options: { redirectTo: '/dashboard' } })
  }

  async function logout() {
    await supabase.auth.signOut()
    await navigateTo('/login')
  }

  return { session, userId, accessToken, isAuthenticated, loginWithMagicLink, loginWithGoogle, logout }
})
```

---

## 7. Page Components

### 7.1 Landing / Login (`pages/index.vue`, `pages/login.vue`)

ISR — rendered at build time, revalidated every 60 seconds. Pure marketing content. No Supabase Realtime. No Pinia canvas store.

`login.vue` renders a centered card with:
- Magic link email input
- Google OAuth button
- No password field — the platform uses passwordless auth only

### 7.2 Dashboard (`pages/dashboard.vue`)

The agent list overview. Fetches `GET /api/v1/users/{uid}/swarm-state` once on mount, then subscribes to a lightweight Supabase Realtime channel that only listens for FSM state changes (not full event payloads — those are for the canvas).

Displays per-agent:
- Agent name and current FSM state badge
- Animated budget gauge (current balance / initial budget)
- Token ROI figure
- Last active timestamp
- Quick actions: View Canvas, Command Console, Terminate

### 7.3 Infinite Canvas (`pages/canvas/[id].vue`)

The primary interface. Pure CSR — no SSR. The page:

1. Reads `darwinId` from the route param
2. Fetches the agent's current state and all historical events on mount
3. Subscribes to Supabase Realtime
4. Renders `<CanvasViewport />` which contains the entire timeline

Route guard: if the agent `darwinId` doesn't belong to the current user, redirect to `/dashboard`.

```vue
<!-- pages/canvas/[id].vue -->
<script setup lang="ts">
definePageMeta({ middleware: 'auth' })

const route = useRoute()
const darwinId = route.params.id as string

const canvas = useCanvasStore()
const agent = useAgentStore()

onMounted(async () => {
  await agent.fetchAgent(darwinId)

  // Load history first — then subscribe so we don't miss events during fetch
  const { data } = await useFetch(`/api/v1/darwins/${darwinId}/events`, {
    headers: authHeaders(),
    query: { limit: 2000, order: 'timestamp_ms.asc' },
  })
  canvas.loadHistoricalEvents(data.value?.events ?? [])

  const { subscribe } = useRealtimeCanvas(darwinId)
  subscribe()
})

onUnmounted(() => {
  const { unsubscribe } = useRealtimeCanvas(darwinId)
  unsubscribe()
})
</script>

<template>
  <div class="h-screen w-screen overflow-hidden bg-darwin-bg flex flex-col">
    <!-- Top bar: agent name, FSM state, budget gauge, heartbeat -->
    <CanvasTopBar :darwin-id="darwinId" />

    <!-- Pending intervention banners (amber) -->
    <InterventionBanner />

    <!-- The canvas itself -->
    <CanvasViewport :darwin-id="darwinId" class="flex-1" />

    <!-- Right-side panels (toggled by top bar buttons) -->
    <Teleport to="body">
      <CommandConsole :darwin-id="darwinId" />
      <EntityPanel :darwin-id="darwinId" />
      <SkillsPanel :darwin-id="darwinId" />
      <LedgerPanel :darwin-id="darwinId" />
      <MentalReplayPanel :darwin-id="darwinId" />
    </Teleport>
  </div>
</template>
```

### 7.4 Spawn Agent (`pages/spawn.vue`)

A multi-step form:
1. **Identity** — agent name, mission directive, LLM model selection
2. **Budget** — initial budget, VCC submission (or select from vault)
3. **Tools** — toggle which tools the agent has access to
4. **Review** — system prompt preview, token ROI estimate, deploy button

On submit: `POST /api/v1/darwins/spawn` → if successful, navigate to `/canvas/{newDarwinId}`.

### 7.5 Mobile Interstitial (`pages/resolve.vue`)

This page is opened from a Telegram deep link. The URL contains a short-lived JWT: `/resolve?token=<jwt>`.

```typescript
// composables/useInterventionJWT.ts
export function useInterventionJWT() {
  const route = useRoute()

  const decoded = computed(() => {
    const token = route.query.token as string
    if (!token) return null
    try {
      // JWT is base64 — decode the payload section without verifying
      // (verification happens server-side when the human submits)
      const payload = JSON.parse(atob(token.split('.')[1]))
      return payload as { darwin_id: string; session_id: string; purpose: string; exp: number }
    } catch {
      return null
    }
  })

  const isExpired = computed(() => {
    if (!decoded.value) return true
    return Date.now() / 1000 > decoded.value.exp
  })

  return { decoded, isExpired }
}
```

The page renders a stripped-down, full-screen `<MobileInterstitial />` component with no navigation, no sidebar — just the resolution UI.

---

## 8. The Infinite Canvas

### 8.1 Layout & Coordinate System

The canvas is a horizontally-scrolling SVG viewport. Time maps to the X axis.

```
X = (event.timestamp_ms - genesis_timestamp_ms) / MS_PER_PX * zoomLevel
```

At default zoom (`zoomLevel = 1`), `MS_PER_PX = 1000` — one second of real time equals one pixel. This means:
- A 5-minute active session spans ~300px
- An 8-hour hibernation spans ~28,800px — collapsed by the zoom to a visually compact dashed line

**Y axis:**
- Main trunk: fixed at `Y = 0` (centre of the track)
- Each neuron lane: offset by `LANE_HEIGHT = 60px` per lane below the trunk

```
Y_trunk = 0
Y_neuron_0 = LANE_HEIGHT      (60px)
Y_neuron_1 = LANE_HEIGHT * 2  (120px)
...
```

The total SVG height grows dynamically as neurons are spawned: `height = TRUNK_HEIGHT + (activeNeurons * LANE_HEIGHT) + PADDING`.

### 8.2 Node Types & Visual Language

Every `event_type` maps to a node with a distinct icon, colour, and interaction behaviour.

| Event type(s) | Node | Icon | Colour | Clickable |
|---|---|---|---|---|
| `AGENT_SPAWNED` | Genesis | Dna | `#6366f1` indigo | Yes — shows DNA summary |
| `THOUGHT` | Cognitive | Brain | `#8b5cf6` violet | Yes — opens Mental Replay |
| `TOOL_CALL` + `TOOL_RESULT` | Action (success) | Wrench | `#22c55e` green | Yes — shows tool I/O |
| `TOOL_CALL` + `TOOL_RESULT` | Action (retry) | Wrench | `#f59e0b` amber | Yes |
| `TOOL_CALL` + `TOOL_RESULT` | Action (failure) | Wrench | `#ef4444` red | Yes |
| `BROWSER_SCREENSHOT` | Screenshot | Camera | `#0ea5e9` sky | Yes — enlarges screenshot |
| `LEDGER_DEBIT` | Financial (debit) | TrendingDown | `#ef4444` red | Yes — shows transaction |
| `LEDGER_CREDIT` | Financial (credit) | TrendingUp | `#22c55e` green | Yes |
| `SKILL_CREATED` | Skill | Star | `#f59e0b` amber | Yes — shows skill doc |
| `SKILL_LOADED` | Skill (loaded) | BookOpen | `#f59e0b` amber | Yes |
| `ENTITY_CREATED` | Entity | Database | `#06b6d4` cyan | Yes — shows entity data |
| `EPISODIC_SUMMARY_WRITTEN` | Memory | Archive | `#a855f7` purple | Yes — shows summary |
| `ASK_HUMAN_SENT` | Intervention (pending) | User | `#f59e0b` amber, pulsing | Yes |
| `ASK_HUMAN_RESOLVED` | Intervention (resolved) | UserCheck | `#22c55e` green | Yes |
| `APPROVAL_REQUESTED` | Approval (pending) | ShieldQuestion | `#f59e0b` amber | Yes — shows Approve/Deny |
| `APPROVAL_GRANTED` | Approval (done) | ShieldCheck | `#22c55e` green | Yes |
| `APPROVAL_DENIED` | Approval (done) | ShieldX | `#ef4444` red | Yes |
| `CAPTCHA_DELEGATED` | CAPTCHA | ScanFace | `#f97316` orange | Yes — opens live session |
| `NEURON_SPAWNED` | Neuron spawn | GitBranch | `#0ea5e9` sky | No |
| `NEURON_COMPLETED` | Neuron done | GitMerge | `#22c55e` green | Yes — shows result |
| `NEURON_FAILED` | Neuron failed | GitMerge | `#ef4444` red | Yes — shows error |
| `SYNAPTIC_INTEGRATION` | Synthesis | Merge | `#6366f1` indigo, large | Yes — shows synthesis |
| `AGENT_WOKE` | Wake | Zap | `#22c55e` green | Yes — shows wake reason |
| `AGENT_HIBERNATED` | Hibernate | Moon | `#64748b` slate | Yes — shows episode summary |
| `AGENT_TERMINATED` | Terminated | Skull | `#ef4444` red | Yes — shows final P&L |

**Node anatomy** (all nodes share this base structure):

```
┌──────────────────┐
│  [icon]          │  ← 32×32px circle, colour-coded
│  [label]         │  ← event_type simplified (e.g., "Thought", "Purchase")
│  [cost badge]    │  ← "$0.042" if token_cost or fiat_cost > 0
│  [timestamp]     │  ← relative time (e.g., "2m ago", "yesterday")
└──────────────────┘
         │
         │ click → MentalReplayPanel slide-over
```

### 8.3 Neural Thread Branching

When a `NEURON_SPAWNED` event arrives, the canvas must:

1. Open a new horizontal lane below the trunk at the event's X position
2. Draw a diagonal connector from the trunk's centre to the lane start
3. Begin rendering the neuron's events in that lane
4. When `NEURON_COMPLETED` or `NEURON_FAILED` arrives, end the lane with a termination node
5. When `SYNAPTIC_INTEGRATION` arrives (all neurons done), draw diagonal connectors from all ended lanes back up to the trunk

```
Trunk:   ─────────────────[Branch]────────────────────────[Synthesis]──────────
                              \                                    /
Neuron A:                      ─────[events]───[COMPLETED]───────
                              \                                    \
Neuron B:                      ─────[events]───[FAILED]────────────
```

The connectors are SVG `<path>` elements using cubic bezier curves for the smooth diagonal effect.

**Lane colour coding:**
- Active neuron lane: `border-sky-500/40` dashed line
- Completed: `border-green-500/40`
- Failed: `border-red-500/40`

### 8.4 Metabolic State Visualization

The trunk line between nodes changes appearance based on the agent's metabolic state:

**ACTIVE (container running):**
```css
.trunk-active {
  stroke: theme('colors.violet.500');
  stroke-width: 2px;
  filter: drop-shadow(0 0 4px theme('colors.violet.500'));
  animation: pulse-glow 2s ease-in-out infinite;
}
```

**HIBERNATING (container destroyed, zero cost):**
```css
.trunk-hibernating {
  stroke: theme('colors.slate.600');
  stroke-width: 1px;
  stroke-dasharray: 6 4;
  /* No animation — it is literally doing nothing */
}
```

**TERMINATED:**
```css
.trunk-terminated {
  stroke: theme('colors.red.800');
  stroke-width: 1px;
  opacity: 0.4;
}
```

The `HeartbeatBar` component renders a live mini-sparkline of `tokens_burned_last_30s` values received via Realtime, shown directly below the trunk when ACTIVE.

### 8.5 Canvas Virtualization

A canvas that has been alive for weeks will have thousands of nodes. Rendering all of them simultaneously will tank performance. We use viewport-based virtualization: only nodes whose X position falls within the visible viewport (plus a 200px buffer on each side) are mounted in the DOM.

```typescript
// composables/useVirtualScroll.ts
export function useVirtualScroll(darwinId: string) {
  const canvas = useCanvasStore()
  const viewportWidth = ref(window.innerWidth)
  const BUFFER_PX = 200

  const visibleEvents = computed(() => {
    const { scrollX, zoomLevel } = canvas
    const leftEdge = scrollX - BUFFER_PX
    const rightEdge = scrollX + viewportWidth.value + BUFFER_PX

    return canvas.trunkEvents.filter(event => {
      const x = timestampToX(event.timestamp_ms, canvas.events[0]?.timestamp_ms ?? 0, zoomLevel)
      return x >= leftEdge && x <= rightEdge
    })
  })

  return { visibleEvents }
}

function timestampToX(timestamp_ms: number, genesis_ms: number, zoom: number): number {
  const MS_PER_PX = 1000
  return ((timestamp_ms - genesis_ms) / MS_PER_PX) * zoom
}
```

Hibernation periods present a special case: they can span hours or days of real time, which would produce enormous amounts of blank canvas space. We apply **temporal compression** to hibernation gaps — any hibernation period longer than 30 minutes is visually compressed to a fixed width of `120px` with a label showing the actual duration ("Hibernated 8h 22m").

### 8.6 Mental Replay Slide-Over

Clicking any node opens the `<MentalReplayPanel />` — a right-side slide-over that renders the specific LangSmith trace for that event.

```
┌─────────────────────────────────────────────────────┐
│ Mental Replay — "Thought" (14:32:07)         [✕]   │
├───────────────────────┬─────────────────────────────┤
│  EYES (what it saw)   │  MIND (what it thought)     │
│                       │                             │
│  [Screenshot or       │  [Chat-style LLM trace]     │
│   fetched text]       │  > Prompt: "You are..."     │
│                       │  > Response: "I cannot      │
│                       │    click checkout yet..."   │
│                       │                             │
│                       │  Tokens: 1,280 in / 80 out  │
│                       │  Cost: $0.0042              │
│                       │  Latency: 1,100ms           │
└───────────────────────┴─────────────────────────────┘
```

For neuron events, the panel shows the neuron-specific trace — including the exact prompt the parent agent gave the neuron, and the neuron's raw output.

Data is fetched from `GET /api/v1/darwins/{id}/logs?session_id={session_id}&event_id={event_id}` on panel open (lazy — not pre-fetched).

---

## 9. Component Library

### 9.1 `<CanvasViewport />`

The outermost canvas container. Handles pan (mouse drag / touch) and zoom (scroll wheel / pinch).

```vue
<script setup lang="ts">
const canvas = useCanvasStore()
const { visibleEvents } = useVirtualScroll(props.darwinId)

// Pan via mouse drag
let isDragging = false
let dragStartX = 0

function onMouseDown(e: MouseEvent) {
  isDragging = true
  dragStartX = e.clientX + canvas.scrollX
}
function onMouseMove(e: MouseEvent) {
  if (!isDragging) return
  canvas.setScroll(dragStartX - e.clientX)
}
function onMouseUp() { isDragging = false }

// Zoom via scroll wheel
function onWheel(e: WheelEvent) {
  e.preventDefault()
  const delta = e.deltaY > 0 ? 0.9 : 1.1
  canvas.setZoom(canvas.zoomLevel * delta)
}
</script>

<template>
  <div
    class="relative overflow-hidden cursor-grab active:cursor-grabbing"
    @mousedown="onMouseDown"
    @mousemove="onMouseMove"
    @mouseup="onMouseUp"
    @wheel.passive="false" @wheel="onWheel"
  >
    <!-- SVG coordinate space — translated by scrollX -->
    <svg
      class="absolute top-0 left-0"
      :style="{ transform: `translateX(${-canvas.scrollX}px)` }"
      :width="totalCanvasWidth"
      :height="totalCanvasHeight"
    >
      <!-- Main trunk -->
      <DarwinTrunk :events="visibleEvents" />

      <!-- Neuron lanes -->
      <NeuralThread
        v-for="[neuronId, lane] in canvas.neuronLanes"
        :key="neuronId"
        :lane="lane"
        :lane-index="laneIndex(neuronId)"
      />
    </svg>
  </div>
</template>
```

### 9.2 `<DarwinTrunk />`

Renders the horizontal trunk line and all main-thread nodes.

```vue
<script setup lang="ts">
const props = defineProps<{ events: DarwinEvent[] }>()
const canvas = useCanvasStore()
const agent = useAgentStore()

const trunkClass = computed(() => ({
  'trunk-active': agent.isActive,
  'trunk-hibernating': agent.isHibernating,
  'trunk-terminated': agent.isTerminated,
}))
</script>

<template>
  <!-- The trunk line -->
  <line
    x1="0" :x2="canvas.maxTimestamp"
    y1="80" y2="80"
    :class="trunkClass"
  />

  <!-- Nodes positioned along the trunk -->
  <CanvasNode
    v-for="event in events"
    :key="event.id"
    :event="event"
    :x="timestampToX(event.timestamp_ms)"
    :y="80"
  />
</template>
```

### 9.3 `<NeuralThread />`

Renders a single neuron lane — the branch line, all neuron events, and the diagonal connectors at branch and merge points.

```vue
<script setup lang="ts">
const props = defineProps<{
  lane: NeuronLane
  laneIndex: number
}>()

const Y_TRUNK = 80
const LANE_HEIGHT = 60
const laneY = computed(() => Y_TRUNK + (props.laneIndex + 1) * LANE_HEIGHT)
const branchX = computed(() => timestampToX(props.lane.spawn_timestamp_ms))
const endX = computed(() => props.lane.end_timestamp_ms
  ? timestampToX(props.lane.end_timestamp_ms)
  : null
)

// Bezier connector — trunk to lane
const branchPath = computed(() =>
  `M ${branchX.value} ${Y_TRUNK} C ${branchX.value + 40} ${Y_TRUNK}, ${branchX.value + 40} ${laneY.value}, ${branchX.value + 80} ${laneY.value}`
)
// Bezier connector — lane back to trunk (at merge)
const mergePath = computed(() => {
  if (!endX.value) return null
  return `M ${endX.value} ${laneY.value} C ${endX.value + 40} ${laneY.value}, ${endX.value + 40} ${Y_TRUNK}, ${endX.value + 80} ${Y_TRUNK}`
})
</script>

<template>
  <!-- Branch connector -->
  <path :d="branchPath" class="lane-connector" fill="none" />

  <!-- Lane line -->
  <line
    :x1="branchX + 80" :x2="endX ?? canvasWidth"
    :y1="laneY" :y2="laneY"
    :class="laneLineClass"
  />

  <!-- Neuron events -->
  <CanvasNode
    v-for="event in lane.events"
    :key="event.id"
    :event="event"
    :x="timestampToX(event.timestamp_ms)"
    :y="laneY"
  />

  <!-- Merge connector (if neuron completed/failed) -->
  <path v-if="mergePath" :d="mergePath" class="lane-connector" fill="none" />
</template>
```

### 9.4 `<CanvasNode />`

The generic node dispatcher. Receives an event and renders the appropriate specialised node component.

```vue
<script setup lang="ts">
const props = defineProps<{ event: DarwinEvent; x: number; y: number }>()
const canvas = useCanvasStore()

const nodeComponent = computed(() => {
  const map: Record<string, Component> = {
    AGENT_SPAWNED: GenesisNode,
    THOUGHT: ThoughtNode,
    TOOL_CALL: ToolNode,
    TOOL_RESULT: ToolNode,
    BROWSER_SCREENSHOT: ToolNode,
    LEDGER_DEBIT: FinancialNode,
    LEDGER_CREDIT: FinancialNode,
    SKILL_CREATED: SkillNode,
    SKILL_LOADED: SkillNode,
    ENTITY_CREATED: EntityNode,
    ENTITY_UPDATED: EntityNode,
    EPISODIC_SUMMARY_WRITTEN: MemoryNode,
    ASK_HUMAN_SENT: InterventionNode,
    ASK_HUMAN_RESOLVED: InterventionNode,
    APPROVAL_REQUESTED: InterventionNode,
    APPROVAL_GRANTED: InterventionNode,
    APPROVAL_DENIED: InterventionNode,
    CAPTCHA_DELEGATED: InterventionNode,
    NEURON_SPAWNED: null,         // no node — the branch itself is the indicator
    NEURON_COMPLETED: ToolNode,
    NEURON_FAILED: ToolNode,
    SYNAPTIC_INTEGRATION: SynapticNode,
    AGENT_WOKE: WakeNode,
    AGENT_HIBERNATED: HibernateNode,
    AGENT_TERMINATED: TerminatedNode,
  }
  return map[props.event.event_type] ?? null
})

function onClick() {
  canvas.selectEvent(props.event.id)
}
</script>

<template>
  <g
    v-if="nodeComponent"
    :transform="`translate(${x}, ${y})`"
    class="cursor-pointer"
    @click="onClick"
  >
    <component :is="nodeComponent" :event="event" />
  </g>
</template>
```

### 9.5 `<BudgetGauge />`

The animated horizontal budget bar pinned to the top of the canvas. Reflects the live balance from `useAgentStore`.

```vue
<script setup lang="ts">
const agent = useAgentStore()

const barWidth = computed(() => `${Math.max(0, agent.pctRemaining)}%`)
const barColor = computed(() => {
  if (agent.pctRemaining > 50) return 'bg-green-500'
  if (agent.pctRemaining > 20) return 'bg-amber-500'
  return 'bg-red-500'
})
</script>

<template>
  <div class="flex items-center gap-3 px-4 py-2 border-b border-white/10">
    <span class="text-xs text-slate-400 whitespace-nowrap">
      ${{ agent.balance.toFixed(2) }}
    </span>
    <div class="flex-1 h-1.5 bg-white/10 rounded-full overflow-hidden">
      <div
        class="h-full rounded-full transition-all duration-500"
        :class="barColor"
        :style="{ width: barWidth }"
      />
    </div>
    <span class="text-xs text-slate-400 whitespace-nowrap">
      ${{ agent.agent?.initial_budget.toFixed(2) }}
    </span>
  </div>
</template>
```

### 9.6 `<HeartbeatBar />`

A small live sparkline rendered under the active trunk showing `tokens_burned_last_30s` over the last 10 readings.

Receives data from the canvas store — the Realtime subscription populates it when metabolic stream events arrive.

```vue
<script setup lang="ts">
// Last 10 metabolic samples for the sparkline
const samples = ref<number[]>([])
const max = computed(() => Math.max(...samples.value, 1))

function addSample(tokensPerBatch: number) {
  samples.value.push(tokensPerBatch)
  if (samples.value.length > 10) samples.value.shift()
}

// Expose addSample for the Realtime handler to call
defineExpose({ addSample })
</script>

<template>
  <div v-if="samples.length" class="flex items-end gap-0.5 h-4">
    <div
      v-for="(s, i) in samples"
      :key="i"
      class="w-1.5 bg-violet-500/60 rounded-sm transition-all duration-300"
      :style="{ height: `${(s / max) * 100}%` }"
    />
  </div>
</template>
```

### 9.7 `<MentalReplayPanel />`

The right-side slide-over that opens when a node is clicked.

```vue
<script setup lang="ts">
const canvas = useCanvasStore()
const isOpen = computed(() => !!canvas.selectedEventId)
const selectedEvent = computed(() =>
  canvas.events.find(e => e.id === canvas.selectedEventId) ?? null
)

const trace = ref<TraceNode[] | null>(null)
const isLoading = ref(false)

watch(() => canvas.selectedEventId, async (eventId) => {
  if (!eventId || !selectedEvent.value) return
  isLoading.value = true
  trace.value = null
  const data = await $fetch(`/api/v1/darwins/${darwinId}/logs`, {
    query: {
      session_id: selectedEvent.value.session_id,
      event_id: eventId,
      neuron_id: selectedEvent.value.neuron_id,
    },
    headers: authHeaders(),
  })
  trace.value = data.trace
  isLoading.value = false
})

function close() { canvas.selectEvent(null) }
</script>

<template>
  <Transition name="slide-right">
    <div v-if="isOpen" class="fixed right-0 top-0 h-screen w-[640px] bg-darwin-surface border-l border-white/10 flex flex-col z-50 shadow-2xl">
      <!-- Header -->
      <div class="flex items-center justify-between p-4 border-b border-white/10">
        <div>
          <h2 class="text-sm font-semibold text-white">Mental Replay</h2>
          <p class="text-xs text-slate-400">{{ selectedEvent?.event_type }} · {{ relativeTime(selectedEvent?.created_at) }}</p>
        </div>
        <button @click="close" class="text-slate-400 hover:text-white">
          <X :size="18" />
        </button>
      </div>

      <!-- Content: two-panel layout -->
      <div class="flex-1 grid grid-cols-2 gap-0 overflow-hidden">
        <!-- Left: EYES — screenshot or fetched content -->
        <div class="border-r border-white/10 overflow-auto p-4">
          <p class="text-xs text-slate-500 mb-2 uppercase tracking-wider">Eyes</p>
          <img
            v-if="selectedEvent?.payload?.screenshot_url"
            :src="selectedEvent.payload.screenshot_url"
            class="w-full rounded border border-white/10"
          />
          <pre
            v-else-if="selectedEvent?.payload?.content"
            class="text-xs text-slate-300 whitespace-pre-wrap"
          >{{ selectedEvent.payload.content }}</pre>
          <p v-else class="text-xs text-slate-600 italic">No visual data for this event</p>
        </div>

        <!-- Right: MIND — LangSmith trace -->
        <div class="overflow-auto p-4">
          <p class="text-xs text-slate-500 mb-2 uppercase tracking-wider">Mind</p>
          <div v-if="isLoading" class="text-xs text-slate-500">Loading trace…</div>
          <div v-else-if="trace" class="space-y-2">
            <div
              v-for="node in trace"
              :key="node.node_id"
              class="text-xs bg-white/5 rounded p-2"
            >
              <span class="text-violet-400 font-mono">{{ node.type }}</span>
              <p class="mt-1 text-slate-300">{{ node.content }}</p>
              <div class="flex gap-3 mt-1 text-slate-500">
                <span v-if="node.token_cost">${{ node.token_cost.toFixed(5) }}</span>
                <span v-if="node.latency_ms">{{ node.latency_ms }}ms</span>
              </div>
            </div>
          </div>
        </div>
      </div>

      <!-- Footer: cost summary -->
      <div class="p-3 border-t border-white/10 flex gap-4 text-xs text-slate-400">
        <span>Token cost: ${{ selectedEvent?.token_cost?.toFixed(5) ?? '—' }}</span>
        <span>Infra cost: ${{ selectedEvent?.fiat_cost?.toFixed(5) ?? '—' }}</span>
      </div>
    </div>
  </Transition>
</template>
```

### 9.8 `<CommandConsole />`

The right-side command panel. Opened via a button in the top bar.

Contains five sub-sections, each collapsible:

**1. Cognitive Uplink**
```vue
<div class="space-y-2">
  <textarea
    v-model="uplinkMessage"
    placeholder="Inject a mid-lifecycle directive... e.g. 'Pivot to aerospace momentum for 4 hours.'"
    class="w-full text-sm bg-white/5 rounded p-2 text-slate-200 placeholder:text-slate-600 resize-none h-24"
  />
  <button @click="sendUplink" class="darwin-btn-primary w-full text-sm">
    ⚡ Inject Uplink
  </button>
</div>
```

**2. Secrets Vault** — `<SecretsVault />` sub-component (see 9.9)

**3. Metabolic Injection**
```vue
<div class="space-y-2">
  <div class="flex items-center gap-2">
    <input v-model.number="injectionAmount" type="number" min="1" step="10"
           class="darwin-input w-24" placeholder="50" />
    <span class="text-sm text-slate-400">USD</span>
  </div>
  <p class="text-xs text-slate-500">
    Current balance: ${{ agent.balance.toFixed(2) }} → After: ${{ (agent.balance + injectionAmount).toFixed(2) }}
  </p>
  <button @click="injectBudget" class="darwin-btn-secondary w-full text-sm">
    💉 Inject Budget
  </button>
</div>
```

**4. Autonomy Toggle**
```vue
<div class="flex items-center justify-between">
  <div>
    <p class="text-sm text-white">Autonomy Mode</p>
    <p class="text-xs text-slate-400">Disable all human approval gates</p>
  </div>
  <button
    @click="toggleAutonomy"
    :class="agent.isAutonomyEnabled ? 'bg-violet-600' : 'bg-white/10'"
    class="relative w-10 h-5 rounded-full transition-colors"
  >
    <span :class="agent.isAutonomyEnabled ? 'translate-x-5' : 'translate-x-0.5'"
          class="absolute top-0.5 w-4 h-4 bg-white rounded-full transition-transform" />
  </button>
</div>
```

**5. Kill Switch**
```vue
<button
  @click="showKillConfirm = true"
  class="w-full py-2 rounded text-sm font-semibold bg-red-900/40 hover:bg-red-900/70 text-red-400 border border-red-900 transition-colors"
>
  ☠ Terminate Agent
</button>
<DarwinModal v-model="showKillConfirm" title="Terminate Agent">
  <p class="text-sm text-slate-300">
    This is permanent. The container will be incinerated, the VCC frozen, and memory archived.
    The agent cannot be revived.
  </p>
  <template #footer>
    <button @click="showKillConfirm = false" class="darwin-btn-ghost">Cancel</button>
    <button @click="confirmTerminate" class="darwin-btn-danger">Terminate</button>
  </template>
</DarwinModal>
```

### 9.9 `<SecretsVault />`

The credential management panel within the Command Console. Once a credential is saved, it displays only a masked reference (`sk-live-...89a2`) — the raw value is never readable from the UI.

```vue
<script setup lang="ts">
const form = reactive({ secret_type: 'vcc', pan: '', cvv: '', expiry: '', spend_limit: 50 })
const savedSecrets = ref<VaultRef[]>([])

// After submission, only the masked reference is stored in component state
async function saveSecret() {
  const resp = await $fetch(`/api/v1/users/${userId}/vault`, {
    method: 'POST',
    headers: authHeaders(),
    body: { darwin_id: darwinId, secret_type: form.secret_type, payload: { ...form } },
  })
  // Clear raw values immediately
  Object.assign(form, { pan: '', cvv: '', expiry: '' })
  savedSecrets.value.push({ ref: resp.ref, type: form.secret_type, spend_limit: resp.spend_limit })
}
</script>

<template>
  <div class="space-y-3">
    <!-- Saved secrets list (masked) -->
    <div v-for="secret in savedSecrets" :key="secret.ref"
         class="flex items-center justify-between text-xs bg-white/5 rounded p-2">
      <span class="font-mono text-slate-300">{{ maskRef(secret.ref) }}</span>
      <span class="text-slate-500">{{ secret.type }} · ${{ secret.spend_limit }} limit</span>
    </div>

    <!-- Add new secret form -->
    <div class="space-y-2 border-t border-white/10 pt-3">
      <select v-model="form.secret_type" class="darwin-input w-full text-sm">
        <option value="vcc">Virtual Credit Card</option>
        <option value="stripe_key">Stripe API Key</option>
        <option value="broker_api_key">Broker API Key</option>
        <option value="iban">IBAN</option>
      </select>

      <!-- VCC fields -->
      <template v-if="form.secret_type === 'vcc'">
        <input v-model="form.pan" type="text" maxlength="19"
               placeholder="4111 1111 1111 1111" class="darwin-input w-full font-mono" />
        <div class="grid grid-cols-2 gap-2">
          <input v-model="form.cvv" type="password" maxlength="4"
                 placeholder="CVV" class="darwin-input" />
          <input v-model="form.expiry" type="text" maxlength="5"
                 placeholder="MM/YY" class="darwin-input" />
        </div>
        <input v-model.number="form.spend_limit" type="number"
               placeholder="Spend limit ($)" class="darwin-input w-full" />
      </template>

      <button @click="saveSecret" class="darwin-btn-primary w-full text-sm">
        Save to Vault
      </button>
    </div>
  </div>
</template>
```

### 9.10 `<EntityPanel />`

A right sidebar showing all entities the agent has created in its SQLite database, grouped by `entity_type`. Each entity is expandable to show its full data payload.

```vue
<script setup lang="ts">
const props = defineProps<{ darwinId: string }>()

// Entities arrive in real-time via canvas events (ENTITY_CREATED / ENTITY_UPDATED)
// We also fetch the full list on mount from the backend
const canvas = useCanvasStore()

const entities = computed(() => {
  const entityEvents = canvas.events.filter(e =>
    ['ENTITY_CREATED', 'ENTITY_UPDATED'].includes(e.event_type)
  )
  // Build a map of entity_id → latest data (UPDATE overwrites CREATE)
  const map = new Map<string, Record<string, unknown>>()
  entityEvents.forEach(e => {
    map.set(e.payload.entity_id as string, {
      type: e.payload.entity_type,
      data: e.payload.data,
      updated_at: e.created_at,
    })
  })
  return Array.from(map.values())
})

const grouped = computed(() => {
  return entities.value.reduce((acc, e) => {
    const type = e.type as string
    if (!acc[type]) acc[type] = []
    acc[type].push(e)
    return acc
  }, {} as Record<string, typeof entities.value>)
})
</script>

<template>
  <div class="overflow-auto">
    <div v-for="(items, type) in grouped" :key="type" class="mb-4">
      <h3 class="text-xs font-semibold text-slate-400 uppercase tracking-wider px-4 py-1 sticky top-0 bg-darwin-surface">
        {{ type }} ({{ items.length }})
      </h3>
      <div v-for="entity in items" :key="entity.data.id as string"
           class="px-4 py-2 border-b border-white/5 hover:bg-white/5 cursor-pointer">
        <p class="text-sm text-white">{{ entityDisplayName(entity) }}</p>
        <p class="text-xs text-slate-500">{{ relativeTime(entity.updated_at) }}</p>
        <!-- Expandable JSON view -->
        <details class="mt-1">
          <summary class="text-xs text-slate-600 cursor-pointer">View data</summary>
          <pre class="text-xs text-slate-400 mt-1 overflow-auto">{{ JSON.stringify(entity.data, null, 2) }}</pre>
        </details>
      </div>
    </div>
  </div>
</template>
```

### 9.11 `<SkillsPanel />`

Lists all skill documents compiled by the agent. Renders the markdown content inline.

```vue
<script setup lang="ts">
const canvas = useCanvasStore()

const skills = computed(() =>
  canvas.events
    .filter(e => e.event_type === 'SKILL_CREATED')
    .map(e => e.payload)
    .reverse()  // newest first
)
</script>

<template>
  <div class="space-y-2 p-4">
    <div v-for="skill in skills" :key="skill.skill_id as string"
         class="bg-white/5 rounded border border-white/10 p-3">
      <div class="flex items-start justify-between">
        <div>
          <p class="text-sm font-medium text-amber-400">{{ skill.title }}</p>
          <p class="text-xs text-slate-400 mt-0.5">Royalty: ${{ skill.royalty_usdc }} USDC per use</p>
        </div>
        <DarwinBadge color="amber">Skill</DarwinBadge>
      </div>
      <details class="mt-2">
        <summary class="text-xs text-slate-500 cursor-pointer">View document</summary>
        <div class="mt-2 text-xs text-slate-300 prose prose-invert prose-sm max-w-none"
             v-html="renderMarkdown(skill.content_preview as string)" />
      </details>
    </div>
  </div>
</template>
```

### 9.12 `<LedgerPanel />`

Full transaction history for this agent.

```vue
<script setup lang="ts">
const canvas = useCanvasStore()

const transactions = computed(() =>
  canvas.events
    .filter(e => ['LEDGER_DEBIT', 'LEDGER_CREDIT'].includes(e.event_type))
    .map(e => e.payload)
    .reverse()
)

const totalRevenue = computed(() =>
  transactions.value
    .filter(t => (t.amount as number) > 0)
    .reduce((sum, t) => sum + (t.amount as number), 0)
)
const totalBurned = computed(() =>
  transactions.value
    .filter(t => (t.amount as number) < 0)
    .reduce((sum, t) => sum + Math.abs(t.amount as number), 0)
)
</script>

<template>
  <div>
    <!-- Summary -->
    <div class="grid grid-cols-2 gap-2 p-4 border-b border-white/10">
      <div class="bg-green-900/20 rounded p-2 text-center">
        <p class="text-xs text-slate-400">Revenue</p>
        <p class="text-lg font-semibold text-green-400">${{ totalRevenue.toFixed(2) }}</p>
      </div>
      <div class="bg-red-900/20 rounded p-2 text-center">
        <p class="text-xs text-slate-400">Burned</p>
        <p class="text-lg font-semibold text-red-400">${{ totalBurned.toFixed(2) }}</p>
      </div>
    </div>

    <!-- Transaction list -->
    <div class="overflow-auto">
      <div v-for="(tx, i) in transactions" :key="i"
           class="flex items-center justify-between px-4 py-2 border-b border-white/5 text-sm">
        <div>
          <p class="text-slate-200">{{ tx.description }}</p>
          <p class="text-xs text-slate-500">{{ tx.type }}</p>
        </div>
        <span :class="(tx.amount as number) > 0 ? 'text-green-400' : 'text-red-400'" class="font-mono">
          {{ (tx.amount as number) > 0 ? '+' : '' }}${{ Math.abs(tx.amount as number).toFixed(2) }}
        </span>
      </div>
    </div>
  </div>
</template>
```

### 9.13 `<InterventionNode />`

Renders on the canvas for `ASK_HUMAN_SENT`, `APPROVAL_REQUESTED`, `CAPTCHA_DELEGATED`. When pending, it pulses amber. When resolved, it becomes static green or red.

The `APPROVAL_REQUESTED` variant renders two buttons inline on the canvas node itself (not just in the Mental Replay panel) because the human needs to respond without digging through panels.

```vue
<script setup lang="ts">
const props = defineProps<{ event: DarwinEvent }>()
const agent = useAgentStore()
const notifications = useNotificationStore()

const isPending = computed(() =>
  notifications.pending.some(n => n.id === props.event.id)
)

const isApproval = computed(() => props.event.event_type === 'APPROVAL_REQUESTED')

async function approve() {
  await agent.patchConfig(darwinId, { budget_injection: 0 }) // triggers approval flow
  // The backend handles waking the agent; the canvas will update via Realtime
}
async function deny() {
  // POST to backend — agent wakes with denied=true
}
</script>

<template>
  <g>
    <!-- Node circle -->
    <circle
      r="18"
      :class="isPending ? 'fill-amber-500/20 stroke-amber-500 animate-pulse' : 'fill-green-500/20 stroke-green-500'"
      stroke-width="1.5"
    />
    <component :is="nodeIcon" class="text-amber-400" :size="14" />

    <!-- Inline approve/deny for approval nodes -->
    <foreignObject v-if="isApproval && isPending" x="24" y="-20" width="180" height="44">
      <div class="flex gap-1">
        <button @click.stop="approve"
                class="px-2 py-1 text-xs bg-green-600 hover:bg-green-500 text-white rounded transition-colors">
          ✓ Approve
        </button>
        <button @click.stop="deny"
                class="px-2 py-1 text-xs bg-red-900/60 hover:bg-red-900 text-red-300 rounded transition-colors">
          ✕ Deny
        </button>
      </div>
    </foreignObject>
  </g>
</template>
```

### 9.14 `<MobileInterstitial />`

Full-screen component rendered by `pages/resolve.vue`. Three variants based on `decoded.purpose`:

**`2fa` variant:**
```vue
<template>
  <div class="min-h-screen bg-darwin-bg flex flex-col items-center justify-center p-6">
    <div class="w-full max-w-sm space-y-4">
      <div class="text-center">
        <Lock :size="32" class="text-violet-400 mx-auto mb-3" />
        <h1 class="text-lg font-semibold text-white">2FA Code Required</h1>
        <p class="text-sm text-slate-400 mt-1">{{ decoded.prompt }}</p>
      </div>
      <input
        v-model="code"
        type="tel"
        maxlength="8"
        placeholder="000000"
        class="w-full text-center text-3xl font-mono tracking-widest darwin-input h-16"
        inputmode="numeric"
        autofocus
      />
      <div class="flex items-center gap-2 text-xs text-slate-500">
        <Timer :size="12" />
        <span>Expires in {{ countdown }}s</span>
      </div>
      <button @click="submit" :disabled="code.length < 6" class="darwin-btn-primary w-full h-12 text-base">
        Submit Code
      </button>
    </div>
  </div>
</template>
```

**`captcha` variant:**
```vue
<!-- Full-screen Browserbase live session WebSocket feed -->
<template>
  <div class="min-h-screen bg-darwin-bg flex flex-col">
    <div class="p-4 flex items-center gap-3 border-b border-white/10">
      <ScanFace :size="20" class="text-orange-400" />
      <h1 class="text-sm font-semibold text-white">Solve CAPTCHA</h1>
    </div>
    <!-- The live browser session is rendered as an iframe pointing to the stream relay -->
    <iframe
      :src="`/sessions/${sessionId}/stream?token=${token}`"
      class="flex-1 border-0"
      allow="camera; microphone"
    />
    <div class="p-4 border-t border-white/10">
      <button @click="resumeAgent" class="darwin-btn-primary w-full">
        ✓ CAPTCHA Solved — Resume Agent
      </button>
    </div>
  </div>
</template>
```

**`approval` variant:**
```vue
<template>
  <div class="min-h-screen bg-darwin-bg flex flex-col items-center justify-center p-6">
    <div class="w-full max-w-sm space-y-4">
      <ShieldQuestion :size="32" class="text-amber-400 mx-auto" />
      <h1 class="text-lg font-semibold text-white text-center">Approval Required</h1>
      <div class="bg-white/5 rounded p-4 space-y-2 text-sm">
        <p class="text-slate-400">Amount</p>
        <p class="text-2xl font-semibold text-white">${{ payload.amount.toFixed(2) }}</p>
        <p class="text-slate-400 mt-2">Justification</p>
        <p class="text-slate-200">{{ payload.justification }}</p>
        <p v-if="payload.token_roi_estimate" class="text-xs text-violet-400 mt-2">
          {{ payload.token_roi_estimate }}
        </p>
      </div>
      <div class="grid grid-cols-2 gap-3">
        <button @click="deny" class="darwin-btn-ghost h-14 text-red-400 border-red-900">
          ✕ Deny
        </button>
        <button @click="approve" class="darwin-btn-primary h-14 bg-green-700 hover:bg-green-600">
          ✓ Approve
        </button>
      </div>
    </div>
  </div>
</template>
```

### 9.15 `<SpawnForm />`

Multi-step form rendered in `pages/spawn.vue`.

**Step 1 — Identity:**
- Agent name (text)
- Mission directive (textarea — large, this is the seed of the system prompt)
- LLM model selector (radio group: Claude Sonnet, Claude Haiku, GPT-4o, GPT-4o Mini)

**Step 2 — Budget:**
- Initial budget slider ($5 → $500)
- VCC selection (pick from vault or add new via `<SecretsVault />` inline)
- Visual preview of the hard cap enforcement ("The Visa network will decline any charge over this amount")

**Step 3 — Tools:**
- Toggle grid — every available tool with a one-line description and cost indicator
- Sensible defaults pre-checked

**Step 4 — Review:**
- System prompt preview (first 400 chars of what the agent will receive)
- Token ROI table (estimated burn rate at the selected model price)
- Confirm and Deploy button

---

## 10. API Client Layer

All backend calls go through a thin wrapper that handles auth headers consistently.

```typescript
// composables/useApi.ts
export function useApi() {
  const auth = useAuthStore()

  function headers() {
    return { Authorization: `Bearer ${auth.accessToken}` }
  }

  async function get<T>(path: string, query?: Record<string, string | number>): Promise<T> {
    return $fetch(`/api/v1${path}`, { headers: headers(), query })
  }

  async function post<T>(path: string, body: object): Promise<T> {
    return $fetch(`/api/v1${path}`, { method: 'POST', headers: headers(), body })
  }

  async function patch<T>(path: string, body: object): Promise<T> {
    return $fetch(`/api/v1${path}`, { method: 'PATCH', headers: headers(), body })
  }

  return { get, post, patch }
}
```

Nuxt's `$fetch` is used (not axios, not raw `fetch`) because it automatically handles SSR/CSR environment differences, base URL configuration, and error parsing.

The backend base URL is proxied through Nuxt's server routes (`/api/v1/*`) to avoid exposing the Railway URL to the browser and to handle CORS in one place.

```typescript
// server/routes/api/[...].ts
export default defineEventHandler(async (event) => {
  const path = event.path.replace('/api/', '')
  return proxyRequest(event, `${process.env.BACKEND_URL}/${path}`)
})
```

---

## 11. TypeScript Types

### `types/events.ts`

```typescript
export type EventType =
  | 'AGENT_SPAWNED' | 'AGENT_WOKE' | 'AGENT_HIBERNATED' | 'AGENT_TERMINATED'
  | 'THOUGHT' | 'TOOL_CALL' | 'TOOL_RESULT' | 'LLM_INFERENCE'
  | 'NEURON_SPAWNED' | 'NEURON_COMPLETED' | 'NEURON_FAILED' | 'SYNAPTIC_INTEGRATION'
  | 'BROWSER_FETCH' | 'BROWSER_SESSION_OPEN' | 'BROWSER_SCREENSHOT'
  | 'BROWSER_CLICK' | 'BROWSER_TYPE' | 'BROWSER_SESSION_CLOSE'
  | 'LEDGER_DEBIT' | 'LEDGER_CREDIT' | 'VCC_INJECTED' | 'USDC_TX'
  | 'ASK_HUMAN_SENT' | 'ASK_HUMAN_RESOLVED'
  | 'APPROVAL_REQUESTED' | 'APPROVAL_GRANTED' | 'APPROVAL_DENIED'
  | 'CAPTCHA_DELEGATED' | 'CAPTCHA_SOLVED'
  | 'SKILL_CREATED' | 'SKILL_LOADED' | 'EPISODIC_SUMMARY_WRITTEN'
  | 'ENTITY_CREATED' | 'ENTITY_UPDATED'
  | 'CIRCUIT_BREAKER_TRIPPED'
  | 'UPLINK_INJECTED' | 'CONFIG_UPDATED'

export interface DarwinEvent {
  id: string
  darwin_id: string
  session_id: string
  neuron_id: string | null
  parent_event_id: string | null
  event_type: EventType
  payload: Record<string, unknown>
  timestamp_ms: number
  token_cost: number | null
  fiat_cost: number | null
  created_at: string
}

// Typed payload accessors (cast payload after checking event_type)
export interface ThoughtPayload {
  content: string
  model: string
  prompt_tokens: number
  completion_tokens: number
  latency_ms: number
}

export interface ToolResultPayload {
  tool_name: string
  status: 'success' | 'failure' | 'retry'
  output_preview: string
  latency_ms: number
}

export interface LedgerPayload {
  amount: number
  balance_after: number
  type: string
  description: string
  external_ref?: string
}

export interface NeuronSpawnedPayload {
  neuron_id: string
  task: string
  parent_event_id: string
}

export interface SynapticIntegrationPayload {
  neuron_ids_merged: string[]
  failed_neurons: string[]
  synthesis: string
}

export interface AskHumanPayload {
  prompt: string
  screenshot_url: string | null
  timeout_seconds: number
  type: '2fa' | 'captcha' | 'general'
}

export interface ApprovalPayload {
  amount: number
  justification: string
  token_roi_estimate: string | null
}

export interface EntityPayload {
  entity_type: string
  entity_id: string
  data: Record<string, unknown>
  operation: 'CREATE' | 'UPDATE'
}
```

### `types/canvas.ts`

```typescript
export interface NeuronLane {
  neuron_id: string
  status: 'active' | 'completed' | 'failed'
  events: DarwinEvent[]
  spawn_timestamp_ms: number
  end_timestamp_ms: number | null
}

export interface CanvasLayout {
  genesis_timestamp_ms: number
  ms_per_px: number
  zoom: number
  total_width_px: number
  total_height_px: number
}

export interface TraceNode {
  node_id: string
  parent_node_id: string | null
  type: 'THOUGHT' | 'TOOL_CALL' | 'TOOL_RESULT' | 'LLM_INFERENCE'
  timestamp_ms: number
  content: string
  tool_name: string | null
  tool_input: Record<string, unknown> | null
  tool_output: Record<string, unknown> | null
  token_cost: number
  latency_ms: number
}
```

---

## 12. Design System

The design uses a dark, high-contrast theme with colour-coded metabolic states. All colours are defined as Tailwind extensions.

```typescript
// tailwind.config.ts
export default {
  theme: {
    extend: {
      colors: {
        darwin: {
          bg:       '#090c12',  // near-black background
          surface:  '#111827',  // card/panel surface
          border:   '#1f2937',  // subtle border
          muted:    '#374151',  // disabled/muted elements
        },
        // Metabolic state colours (also used for trunk glow)
        active:     '#8b5cf6',  // violet — agent is burning compute
        hibernating:'#475569',  // slate — zero cost resting state
        terminated: '#7f1d1d',  // dark red — permanent end
      },
      fontFamily: {
        mono: ['JetBrains Mono', 'monospace'],
      },
      animation: {
        'pulse-glow': 'pulse-glow 2s ease-in-out infinite',
        'slide-right': 'slide-right 0.2s ease-out',
      },
      keyframes: {
        'pulse-glow': {
          '0%, 100%': { filter: 'drop-shadow(0 0 4px #8b5cf6)' },
          '50%':       { filter: 'drop-shadow(0 0 12px #8b5cf6)' },
        },
        'slide-right': {
          from: { transform: 'translateX(100%)' },
          to:   { transform: 'translateX(0)' },
        },
      },
    },
  },
}
```

**Typography:**
- Body: system-ui, 14px
- Mono (node labels, costs, token counts): JetBrains Mono, 12px
- Headings: system-ui, semibold

**Shared CSS classes** (defined in `assets/css/main.css`):

```css
.darwin-btn-primary {
  @apply px-4 py-2 rounded bg-violet-600 hover:bg-violet-500 text-white font-medium transition-colors;
}
.darwin-btn-secondary {
  @apply px-4 py-2 rounded bg-white/10 hover:bg-white/15 text-white font-medium transition-colors;
}
.darwin-btn-ghost {
  @apply px-4 py-2 rounded hover:bg-white/5 text-slate-300 transition-colors;
}
.darwin-btn-danger {
  @apply px-4 py-2 rounded bg-red-700 hover:bg-red-600 text-white font-medium transition-colors;
}
.darwin-input {
  @apply bg-white/5 border border-white/10 rounded px-3 py-2 text-sm text-white
         placeholder:text-slate-600 focus:outline-none focus:ring-1 focus:ring-violet-500;
}
```

---

## 13. Deployment

The frontend deploys to **Vercel** with zero configuration. Nuxt 3 has first-class Vercel support — the `nuxt build` output is automatically detected and deployed as Edge Functions (for ISR pages) and static assets (for SSR-disabled canvas routes).

```json
// vercel.json — minimal config needed
{
  "buildCommand": "nuxt build",
  "outputDirectory": ".output/public",
  "framework": "nuxtjs"
}
```

Environment variables are set in the Vercel dashboard and injected at build time.

---

## 14. Environment Variables

```bash
# Supabase (client-side — public)
SUPABASE_URL=https://xxx.supabase.co
SUPABASE_ANON_KEY=...               # safe to expose — RLS enforces access control

# Backend proxy (server-side — never exposed to browser)
BACKEND_URL=https://darwin-backend.railway.app

# App
APP_URL=https://app.darwin.ai
NODE_ENV=production
```

The `SUPABASE_ANON_KEY` is safe to expose client-side — it only allows operations that pass Row Level Security policies, and all user data is gated behind auth.uid() = user_id policies.

The `BACKEND_URL` is server-side only — it never reaches the browser, preventing direct API access without going through Nuxt's server proxy.

---

## 15. Testing Strategy

### 15.1 Unit tests — Vitest

Test stores and composables in isolation.

Key targets:
- `useCanvasStore.appendEvent` — deduplication (same id twice = one entry), neuron lane routing
- `useCanvasStore.trunkEvents` — filters out neuron events correctly
- `useNotificationStore.resolve` — matches resolution event to correct trigger type
- `useCanvasLayout.timestampToX` — correct pixel math at different zoom levels
- `useVirtualScroll.visibleEvents` — only returns events within viewport + buffer
- `eventColor` util — every event_type returns a valid colour class

### 15.2 Component tests — Vue Test Utils

Test individual components with stubbed stores.

Key targets:
- `<CanvasNode />` renders the correct specialised node for every `event_type`
- `<InterventionNode />` shows approve/deny buttons only when `isPending === true`
- `<BudgetGauge />` bar width matches `pctRemaining` from the store
- `<SecretsVault />` clears `pan`, `cvv`, `expiry` from local state after submission

### 15.3 E2E tests — Playwright

Key flows:
- **Canvas loads with historical events:** navigate to `/canvas/{id}`, assert genesis node is visible at left edge
- **Live event appears:** mock a Supabase Realtime insert, assert a new node appears on the canvas within 500ms
- **Mental Replay opens:** click a thought node, assert the slide-over opens and displays "Eyes" and "Mind" panels
- **Neuron branching:** insert `NEURON_SPAWNED` event, assert a new lane appears below the trunk; insert `NEURON_COMPLETED`, assert lane terminates with green node
- **Approval flow:** insert `APPROVAL_REQUESTED` event, assert inline Approve/Deny buttons appear on the canvas node; click Approve, assert `PATCH /darwins/{id}/config` was called
- **Kill switch with confirmation:** open Command Console, click Terminate, assert confirmation modal appears, confirm, assert canvas shows TERMINATED node

### 15.4 Critical invariants

1. **Raw VCC digits must never appear in any component's template or reactive state** after submission — only the masked reference
2. **The canvas must not re-render the entire node tree when one event is appended** — use Vue devtools to verify that only the new node's component re-renders
3. **A Realtime event that arrives while the Mental Replay panel is open must not close the panel** — the panel is keyed to `selectedEventId`, which does not change on new events
4. **Temporal compression must not re-order events** — `events` array must always be sorted by `timestamp_ms` ascending
5. **The Supabase Realtime channel must be unsubscribed when the canvas page is unmounted** — verify in Supabase dashboard that no orphaned channels accumulate
