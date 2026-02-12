# OpenClaw Command Center — Implementation Guide

> **Version:** 1.0.0 · **Architecture:** Single-file HTML · **Lines:** 8,849 · **Dependencies:** Zero (Google Fonts optional)

---

## Table of Contents

1. [Overview](#overview)
2. [Install Link Bootstrapping](#install-link-bootstrapping)
3. [Architecture](#architecture)
4. [Storage Engine](#storage-engine)
5. [Gateway Protocol](#gateway-protocol)
6. [Gateway-Side Implementation](#gateway-side-implementation)
7. [Security Model](#security-model)
8. [Admin/Worker RBAC System](#adminworker-rbac-system)
9. [Canvas API Reference](#canvas-api-reference)
10. [Contributing](#contributing)

---

## Install Link Bootstrapping

The dashboard now supports one-click install by passing gateway credentials in URL params or hash.

### Supported formats

- Query params:
  - `gateway` / `gw` / `ws` / `url` / `gateway_url` / `gateway-url` / `gatewayUrl` / `agent` / `agent_url` / `agent-link` / `openclaw_gateway` / `openclaw_gateway_url` (any with a gateway URL)
  - `token` / `auth` / `auth_token` / `agent_token` / `gw_token` / `openid_token` / `access_token`
- Fragment/hash forms:
  - `#agent=<ws-or-http-url>` where the value is a gateway URL (or OpenClaw link)
  - Query-style `?token=...` works with hash URLs too.

If `agent` or `gateway` resolves to a full `ws://`/`wss://` URL with `token` in that URL query, that token is also honored.
Bootstrap values are scrubbed from browser URL/search state after parsing to avoid leaving tokens in the address bar and history.

### Runtime behavior

- Parsed values are applied before first render and persisted to localStorage.
- The settings form reflects those values when opened.
- On load, the dashboard auto-connects and shows a success toast indicating bootstrap source.

Supported examples:

```
https://<host>/index-2.html?gateway=ws://127.0.0.1:18789&token=oc_live_123
https://<host>/index-2.html#agent=ws://gateway.openclaw.app:18789?token=oc_live_123
https://<host>/index-2.html?agent=https://proxy.example.com/connect?gateway=ws://gateway.openclaw.app:18789&token=oc_live_123
```

---

## Overview

The OpenClaw Command Center (CC) is a single HTML file that serves as the management dashboard for OpenClaw agent teams. It connects to an OpenClaw Gateway via WebSocket and provides:

- **Org Structure** — drag-and-drop node graph editor for agent hierarchy and routing
- **Agent Chat** — per-agent message threads relayed through the gateway
- **Task Board** — Kanban-style task management agents can read/write
- **Cron Jobs** — scheduled directives with natural language scheduling
- **Briefing Room** — multi-agent meetings with agenda tracking
- **Marketing Campaigns** — campaign planning with agent assignments
- **Calendar** — event scheduling with agent assignments
- **Agent Logs** — filterable activity log across all agent operations
- **System Prompts** — per-agent prompt editor with live push to gateway
- **API Vault** — secrets management with per-agent access control
- **RevenueCat Analytics** — revenue dashboard for iOS/Android apps
- **Character Customizer** — procedural 8-bit pixel art avatars

The entire dashboard is a single `index-2.html` file. No build step. No npm. No framework. Open it in a browser, point it at a gateway, and it works.

---

## Architecture

```
┌─────────────────────────────────────────────────┐
│                   Browser                        │
│                                                  │
│  ┌─────────────────────────────────────────────┐ │
│  │          OpenClaw Command Center            │ │
│  │              (index-2.html)                 │ │
│  │                                             │ │
│  │  ┌──────────┐  ┌──────────┐  ┌───────────┐ │ │
│  │  │  State   │  │  Render  │  │  Canvas   │ │ │
│  │  │  Engine  │  │  Layer   │  │  API      │ │ │
│  │  └─────┬────┘  └──────────┘  └─────┬─────┘ │ │
│  │        │                           │        │ │
│  │  ┌─────▼───────────────────────────▼─────┐  │ │
│  │  │           Storage Engine              │  │ │
│  │  │  localStorage ← IndexedDB ← Gateway   │  │ │
│  │  └─────────────────────┬─────────────────┘  │ │
│  └────────────────────────┼─────────────────────┘ │
│                           │ WebSocket              │
└───────────────────────────┼─────────────────────────┘
                            │
                  ┌─────────▼──────────┐
                  │   OpenClaw Gateway  │
                  │                    │
                  │  ┌──────────────┐  │
                  │  │  Filesystem  │  │
                  │  │  state/      │  │
                  │  │  current.json│  │
                  │  │  backups/    │  │
                  │  └──────────────┘  │
                  │                    │
                  │  ┌──────────────┐  │
                  │  │   Agents     │  │
                  │  │  (LLM calls) │  │
                  │  └──────────────┘  │
                  └────────────────────┘
```

### State Management

All application state lives in a single `state` object in memory. There are no stores, no reducers, no observables. Functions mutate `state` directly and call the relevant `render*()` function to update the DOM.

```javascript
state.tasks.push(newTask);
_saveTasks();      // → localStorage + IndexedDB + Gateway
renderTasks();     // → DOM update
```

### Rendering

Each view has a `render*()` function that reads from `state` and writes `innerHTML`. The `renderAll()` function calls every renderer. This is intentionally simple — no virtual DOM, no diffing. For a dashboard with < 1,000 DOM nodes per view, direct innerHTML replacement at 60fps is fine.

---

## Storage Engine

The CC uses a three-tier persistence model. Data flows down through all three layers on every save. Recovery flows up from the most durable available source.

```
Tier 1: Gateway Filesystem    ← Most durable (survives everything)
  ↑
Tier 2: IndexedDB              ← Durable (50MB+, survives browser restarts)
  ↑
Tier 3: localStorage           ← Fast cache (5-10MB, volatile)
```

### Write Path

Every `_save*()` call triggers:

1. **localStorage** (sync) — immediate write for fast page reload
2. **IndexedDB** (async, debounced 1.5s) — full state snapshot, transactional
3. **Gateway** (async, on WebSocket) — `state_sync` message with full state

```javascript
// Example: saving a task triggers all three tiers
function _saveTasks() {
  _safeSave('oc_tasks', state.tasks);
  // _safeSave writes localStorage (sync)
  // Then calls _storage.queueSave() which debounces
  // persistFullState() writes to IDB + pushes to gateway
}
```

### Recovery Chain (Boot Sequence)

On page load:

1. **localStorage** loads synchronously (page renders immediately)
2. **IndexedDB** checked async — if localStorage was wiped but IDB has data, auto-recovers
3. **Gateway** — sends `state_request` on WebSocket connect. If gateway has newer data, restores it.

```
Boot
 ├─ Load from localStorage (sync, instant)
 ├─ Render UI
 ├─ Init IndexedDB (async)
 │   ├─ localStorage empty + IDB has data → auto-recover from IDB
 │   ├─ Both have data → sync IDB with current state
 │   └─ Neither has data → fresh install, persist defaults
 ├─ Connect to Gateway
 │   └─ Send state_request with local timestamp
 │       ├─ Gateway newer → restore from gateway, create backup
 │       ├─ Local newer → push to gateway
 │       └─ Same → in sync
 └─ Start backup timer (every 5 minutes)
```

### Automatic Backups

Every 5 minutes, the CC creates a timestamped snapshot in IndexedDB. The last 20 are kept. Each backup includes:

```javascript
{
  timestamp: Date.now(),
  label: 'auto' | 'boot' | 'manual' | 'pre-import' | 'gateway-restore' | 'conflict-resolve',
  data: { /* full state snapshot */ },
  size: 45230  // bytes
}
```

Users can browse and restore from any backup in Settings → Backup list.

### Durability Events

| Event | Trigger |
|---|---|
| Tab closes with unsaved data | `beforeunload` — emergency sync to localStorage |
| Tab goes to background | `visibilitychange` — full persist to IDB + gateway |
| Gateway reconnects | `state_request` — sync with server |
| Every 5 minutes | Auto-backup to IndexedDB |
| Before import/restore | Safety backup created first |

### What Survives What

| Event | localStorage | IndexedDB | Gateway | Recovery |
|---|---|---|---|---|
| Page reload | ✅ | ✅ | ✅ | Instant |
| Browser restart | ✅ | ✅ | ✅ | Instant |
| "Clear browsing data" | ❌ | ✅* | ✅ | Auto from IDB or gateway |
| "Clear all site data" | ❌ | ❌ | ✅ | Auto from gateway on connect |
| Switch browsers | ❌ | ❌ | ✅ | Auto from gateway on connect |
| Switch computers | ❌ | ❌ | ✅ | Auto from gateway on connect |
| Gateway wipe | ✅ | ✅ | ❌ | CC pushes local to new gateway |
| Nuclear (everything gone) | ❌ | ❌ | ❌ | JSON export file |

*IndexedDB survives most cleanup except explicit "Clear all site data"

---

## Gateway Protocol

The CC communicates with the OpenClaw Gateway over a single WebSocket connection. All messages are JSON.

### Connection

```javascript
// CC connects with optional auth token
ws = new WebSocket('ws://gateway-host:18789?token=...');

// On connect, CC sends:
{ type: 'subscribe', channels: ['sessions', 'presence', 'config'] }
{ type: 'get_models' }
{ type: 'state_request', localSavedAt: 1707782400000, schema: 1, version: '1.0.0' }
```

### Full Message Catalog

#### Outbound (CC → Gateway)

| Type | Purpose | Payload |
|---|---|---|
| `subscribe` | Register for event channels | `{ channels: string[] }` |
| `get_models` | Request available LLM models | `{}` |
| `message` | Send chat message to agent | `{ agentId, text }` |
| `prompt_update` | Push system prompt to agent | `{ agentId, prompt: {...} }` |
| `routing_update` | Push routing config to agent | `{ agentId, routingConfig: {...} }` |
| `routing_directive` | One-shot routing command | `{ agentId, directive, routingConfig }` |
| `cron_sync` | Push all cron jobs | `{ jobs: [...] }` |
| `vault_manifest` | Push vault structure (no values) | `{ secrets: [...] }` |
| `vault_response` | Respond to agent's secret request | `{ requestId, agentId, value }` |
| `canvas_eval_result` | Return value from canvas_eval call | `{ requestId, result }` |
| `meeting_start` | Begin a multi-agent briefing | `{ topic, agents: string[] }` |
| `state_sync` | Push full state snapshot | `{ savedAt, schema, version, state: {...} }` |
| `state_request` | Request stored state | `{ localSavedAt, schema, version }` |

#### Inbound (Gateway → CC)

| Type | Purpose | Payload |
|---|---|---|
| `message` | Chat message from agent | `{ agentId, text }` |
| `presence` | Agent status change | `{ agentId, status }` |
| `models` | Available LLM models | `{ models: [{id, name, provider, tier, size}] }` |
| `routing_ack` | Agent acknowledged routing update | `{ agentId }` |
| `vault_request` | Agent requests a secret | `{ requestId, agentId, secretKey }` |
| `canvas_eval` | Agent calls a CC API method | `{ agentId, method, args, requestId }` |
| `meeting_message` | Agent message in briefing | `{ agentId, text, messageType }` |
| `agent_comms` | Inter-agent message relay | `{ from, to, text }` |
| `state_restore` | Gateway sends stored state | `{ state: {...}, savedAt }` |
| `state_ack` | Gateway confirms state write | `{ savedAt, path }` |
| `state_conflict` | Gateway detected version conflict | `{ savedAt, state: {...} }` |

### Reconnection Strategy

The CC uses exponential backoff with jitter:

```
Attempt 1: ~1s    (1000ms + 0-1000ms jitter)
Attempt 2: ~2s    (2000ms + jitter)
Attempt 3: ~4s    (4000ms + jitter)
Attempt 4: ~8s    (8000ms + jitter)
Attempt 5: ~16s   (16000ms + jitter)
Attempt 6+: ~30s  (30000ms cap + jitter)
```

On successful reconnect, the attempt counter resets to 0.

---

## Gateway-Side Implementation

This section is for **gateway developers** implementing the CC persistence protocol. The CC side is fully built — the gateway just needs to handle three message types and write files to disk.

### File Structure

```
~/.openclaw/workspace/canvas/
  index.html              ← the CC dashboard
  state/
    current.json          ← latest full state snapshot
    backups/
      2026-02-12T14-30-00.json
      2026-02-12T14-25-00.json
      ...                 ← keep last 20
```

### Minimal Implementation (Node.js)

```javascript
const fs = require('fs');
const path = require('path');

const STATE_DIR = path.join(process.env.HOME, '.openclaw/workspace/canvas/state');
const CURRENT_PATH = path.join(STATE_DIR, 'current.json');
const BACKUP_DIR = path.join(STATE_DIR, 'backups');
const MAX_BACKUPS = 20;

// Ensure directories exist
fs.mkdirSync(BACKUP_DIR, { recursive: true });

function handleCanvasMessage(ws, msg) {
  switch (msg.type) {

    case 'state_sync': {
      // CC pushes full state — write to disk
      const data = JSON.stringify({
        ...msg.state,
        _savedAt: msg.savedAt,
        _schema: msg.schema,
        _version: msg.version,
      }, null, 2);

      // Write current state
      fs.writeFileSync(CURRENT_PATH, data);

      // Create timestamped backup
      const ts = new Date(msg.savedAt).toISOString().replace(/[:.]/g, '-');
      fs.writeFileSync(path.join(BACKUP_DIR, `${ts}.json`), data);

      // Prune old backups
      const backups = fs.readdirSync(BACKUP_DIR)
        .filter(f => f.endsWith('.json'))
        .sort()
        .reverse();
      backups.slice(MAX_BACKUPS).forEach(f =>
        fs.unlinkSync(path.join(BACKUP_DIR, f))
      );

      // Acknowledge
      ws.send(JSON.stringify({
        type: 'state_ack',
        savedAt: msg.savedAt,
        path: 'state/current.json',
      }));
      break;
    }

    case 'state_request': {
      // CC asks for stored state (on connect/reconnect)
      if (!fs.existsSync(CURRENT_PATH)) {
        // No stored state — CC will push its local state
        ws.send(JSON.stringify({
          type: 'state_restore',
          state: null,
          savedAt: 0,
        }));
        return;
      }

      const stored = JSON.parse(fs.readFileSync(CURRENT_PATH, 'utf8'));
      const gatewaySavedAt = stored._savedAt || 0;
      const clientSavedAt = msg.localSavedAt || 0;

      if (gatewaySavedAt !== clientSavedAt && gatewaySavedAt > 0 && clientSavedAt > 0) {
        // Both have data with different timestamps — potential conflict
        // Simple strategy: send stored state and let CC decide
        ws.send(JSON.stringify({
          type: 'state_restore',
          state: stored,
          savedAt: gatewaySavedAt,
        }));
      } else {
        // Normal restore
        ws.send(JSON.stringify({
          type: 'state_restore',
          state: stored,
          savedAt: gatewaySavedAt,
        }));
      }
      break;
    }
  }
}
```

### Minimal Implementation (Python)

```python
import json
import os
from pathlib import Path
from datetime import datetime

STATE_DIR = Path.home() / '.openclaw/workspace/canvas/state'
CURRENT_PATH = STATE_DIR / 'current.json'
BACKUP_DIR = STATE_DIR / 'backups'
MAX_BACKUPS = 20

BACKUP_DIR.mkdir(parents=True, exist_ok=True)

def handle_canvas_message(ws, msg):
    if msg['type'] == 'state_sync':
        data = {**msg['state'],
                '_savedAt': msg['savedAt'],
                '_schema': msg.get('schema', 1),
                '_version': msg.get('version', '1.0.0')}

        # Write current state
        CURRENT_PATH.write_text(json.dumps(data, indent=2))

        # Create timestamped backup
        ts = datetime.fromtimestamp(msg['savedAt'] / 1000).strftime('%Y-%m-%dT%H-%M-%S')
        (BACKUP_DIR / f'{ts}.json').write_text(json.dumps(data, indent=2))

        # Prune old backups
        backups = sorted(BACKUP_DIR.glob('*.json'), reverse=True)
        for old in backups[MAX_BACKUPS:]:
            old.unlink()

        # Acknowledge
        ws.send(json.dumps({
            'type': 'state_ack',
            'savedAt': msg['savedAt'],
            'path': 'state/current.json'
        }))

    elif msg['type'] == 'state_request':
        if not CURRENT_PATH.exists():
            ws.send(json.dumps({'type': 'state_restore', 'state': None, 'savedAt': 0}))
            return

        stored = json.loads(CURRENT_PATH.read_text())
        ws.send(json.dumps({
            'type': 'state_restore',
            'state': stored,
            'savedAt': stored.get('_savedAt', 0)
        }))
```

### Multi-Client Conflict Handling

If multiple CC instances connect to the same gateway (e.g., two browser tabs, two computers), the gateway can optionally detect conflicts:

```javascript
// Track the last known savedAt per connection
const clientTimestamps = new Map();

case 'state_sync': {
  const stored = fs.existsSync(CURRENT_PATH)
    ? JSON.parse(fs.readFileSync(CURRENT_PATH, 'utf8'))
    : null;

  // If another client wrote more recently, notify this client
  if (stored && stored._savedAt > msg.savedAt) {
    ws.send(JSON.stringify({
      type: 'state_conflict',
      savedAt: stored._savedAt,
      state: stored,
    }));
    return; // Don't overwrite newer data
  }

  // Otherwise, write normally
  // ... (same as above)
}
```

---

## Security Model

### XSS Prevention

All agent-provided content is escaped before innerHTML insertion using a DOM-based escaper:

```javascript
function _esc(str) {
  if (!str) return '';
  const div = document.createElement('div');
  div.appendChild(document.createTextNode(String(str)));
  return div.innerHTML;
}
```

Applied to all 6 injection surfaces: chat messages, war chatter, activity log descriptions, activity log details, briefing transcript messages, and toast notifications.

### canvas_eval Security

The `canvas_eval` gateway message lets agents call CC API methods remotely. Two layers of protection:

**Method Whitelist** — The following methods are blocked from remote execution:

```
importState, getState, createAgent, removeAgent, editAgent,
addSecret, updateSecret, deleteSecret
```

These cover state management, agent CRUD, and vault manipulation — operations that should only be performed by the human operator.

**Rate Limiting** — Each agent is limited to 50 `canvas_eval` calls per 10-second window. Exceeding the limit returns an error and the call is dropped.

### Vault Security

The API Vault stores secrets in `state.vault`. Each secret has an `allowList` of agent IDs that can access it. When an agent requests a secret via `vault_request`, the CC checks:

1. Does the secret exist?
2. Is the requesting agent on the `allowList`?
3. If yes, send the value. If no, deny.

**Current limitation:** Vault secrets are stored in plaintext in localStorage/IndexedDB. A future version could encrypt them with Web Crypto API using a user passphrase.

### State Sync Security

The `state_sync` message strips the `_config` object (which contains the gateway URL and auth token) before transmission. The gateway already has the auth token from the WebSocket connection — it doesn't need it in the state payload.

---

## Admin/Worker RBAC System

Each agent has an `admin` boolean flag that controls their system prompt and capabilities.

### Admin Agents

Admin agents receive a full dashboard context injection with 11 tool sections:

1. Agent management (create, edit, remove agents)
2. Org structure (connections, routing)
3. Communication (push comms, manage chatter)
4. Agent status management
5. Task management (full CRUD)
6. Cron job management
7. Calendar management
8. Campaign management
9. System prompt management
10. API Vault management
11. Activity log management

Admin prompts include a live snapshot of the current dashboard state: agent roster, active tasks, recent comms, cron jobs, and active campaigns.

### Worker Agents

Worker agents receive a restricted prompt with 4 sections:

1. Read tasks assigned to them
2. Update their own task status
3. Log their own activity
4. Read their own agent info

Workers cannot create/edit agents, manage the vault, modify cron jobs, or access other agents' data. The prompt explicitly instructs them to escalate cross-system needs to admin agents.

### Prompt Regeneration

When an agent's admin flag is toggled, the CC automatically regenerates their system prompt with the appropriate tool sections and pushes it to the gateway.

---

## Canvas API Reference

The CC exposes `window.__openclaw_cc` — a comprehensive API that agents (via `canvas_eval`) and developers (via browser console) can use to interact with the dashboard.

### Agent Management
```javascript
.createAgent({ name, role, model, color, skills, personality })
.editAgent(id, { name?, role?, model?, color?, skills?, personality?, admin? })
.removeAgent(id)
.getAgents()
.getModels()
```

### Org Structure
```javascript
.addConnection(fromId, fromPort, toId, toPort)
.removeConnection(connectionId)
.getConnections(agentId?)
.getConnectedAgents(agentId)
.moveOrgNode(agentId, x, y)
.clearConnections()
.generateRoutingConfig(agentId)
.exportAllRoutingConfigs()
```

### Communication
```javascript
.pushComms({ from, to, text })
.clearComms()
```

### Agent Status
```javascript
.setAgentStatus(id, 'active' | 'busy' | 'idle' | 'error' | 'offline')
.getAgentStatus(id)
```

### Tasks
```javascript
.addTask({ title, agent, status?, priority?, due?, desc? })
.moveTask(id, newStatus)
.updateTask(id, { ...fields })
.deleteTask(id)
.getTasks({ agent?, status? })
.clearTasks({ agent?, status? })
```

### Cron Jobs
```javascript
.addCronJob({ name, agents, schedule, message })
.updateCronJob(id, { ...fields })
.deleteCronJob(id)
.toggleCronJob(id)
.getCronJobs({ agent?, enabled? })
.logCronRun(id, success, error?)
```

### Activity Log
```javascript
.logActivity({ type, agent, desc, detail? })
.getActivityLog({ agent?, type?, since? })
.clearActivityLog({ agent?, type? })
```

### System Prompts
```javascript
.getPrompt(agentId)
.setPrompt(agentId, { identity, role, tools, personality, instructions })
.pushPrompt(agentId)       // Push to gateway
.pushAllPrompts()           // Push all agents' prompts
```

### API Vault
```javascript
.addSecret({ name, envKey, fields: [{ label, key, value }], note? })
.getSecret('SECRET_NAME.FIELD_KEY', agentId)
.listSecrets(agentId?)
.setAllowList(secretId, [agentIds])
.deleteSecret(id)
```

### Calendar
```javascript
.addCalEvent({ title, date, agent?, time?, desc? })
.updateCalEvent(id, { ...fields })
.deleteCalEvent(id)
.getCalEvents({ agent?, from?, to? })
```

### Marketing Campaigns
```javascript
.addCampaign({ name, status?, agents?, budget?, startDate?, endDate? })
.updateCampaign(id, { ...fields })
.deleteCampaign(id)
.getCampaigns({ agent?, status? })
.addCampaignTask(campaignId, { title, agent?, status? })
.updateCampaignTask(campaignId, taskId, { ...fields })
```

### RevenueCat Analytics
```javascript
.rcUpdateMetrics({ mrr?, revenue?, activeSubscribers?, ... })
.rcAddTransaction({ productId, type, revenue, ... })
.rcGetMetrics()
.rcSetApps(apps)
```

### Briefings
```javascript
.meetingMessage({ agent, text, type? })
.getMeetingHistory()
```

### State Management
```javascript
.getState()          // Returns full serializable state
.importState(data)   // Overwrites state from snapshot
```

---

## Contributing

### Adding a New View

1. Add a `<div class="view-panel" id="your-view">` to the `#content` container
2. Add a nav button with `data-view="your-view"` to `#sidebar`
3. Create a `renderYourView()` function and add it to `renderAll()`
4. Add the view title mapping in `switchView()`

### Adding a New Gateway Message Type

1. Add the handler in `handleGatewayMessage()` with `if (msg.type === 'your_type')`
2. Document it in this guide under the message catalog
3. If it modifies state, call the appropriate `_save*()` function

### Adding a New Canvas API Method

1. Add the method to `window.__openclaw_cc`
2. If it should be agent-callable, ensure it's NOT in the `BLOCKED_METHODS` list
3. If it should be admin-only, add it to the blocked list

### Testing

Open the browser console and use `window.__openclaw_cc` directly:

```javascript
// Create an agent
__openclaw_cc.createAgent({ name: 'TEST', role: 'QA', model: 'claude-sonnet-4-5-20250929' });

// Add a task
__openclaw_cc.addTask({ title: 'Test task', agent: 'test', status: 'todo' });

// Check state
__openclaw_cc.getState();
```

### Data Format

All state is JSON-serializable. The full state snapshot schema:

```javascript
{
  _schema: 1,                    // Schema version (for migrations)
  _version: '1.0.0',            // App version
  _savedAt: 1707782400000,      // Timestamp of last save

  agents: [{ id, name, role, color, model, admin, skills, personality }],
  tasks: [{ id, title, agent, status, priority, due, desc, created }],
  calEvents: [{ id, title, date, agent, time, desc }],
  campaigns: [{ id, name, status, agents, budget, startDate, endDate }],
  campaignTasks: { campaignId: [{ id, title, agent, status }] },
  chatMessages: { agentId: [{ from, text, ts }] },
  warChatter: [{ from, to, text, ts }],
  orgPositions: { agentId: { x, y } },
  orgConnections: [{ id, from, fromPort, to, toPort }],
  cronJobs: [{ id, name, agents, schedule, message, enabled, lastRun }],
  activityLog: [{ id, type, agent, desc, detail, ts }],
  agentPrompts: { agentId: { identity, role, tools, personality, instructions } },
  vault: [{ id, name, envKey, fields, note, allowList }],
  agentStatus: { agentId: 'active' | 'busy' | 'idle' | 'error' | 'offline' },
  avatarConfigs: { agentId: { skin, hair, hairColor, eyes, mouth, ... } },
  rc: { mrr, revenue, activeSubscribers, ... },
  meeting_history: [{ topic, messages, startedAt, endedAt, actionItems }],
  gatewayModels: [{ id, name, provider, tier, size }],

  _config: {                     // Stripped from state_sync messages
    gw_url: 'ws://...',
    gw_token: '...'
  }
}
```

### Data Size Caps

To prevent unbounded growth:

| Data | Cap |
|---|---|
| Chat messages per agent | 200 |
| War chatter messages | 500 |
| Activity log entries | 500 |
| Meeting messages per meeting | 300 |
| IndexedDB backups | 20 |

---

## License

See LICENSE file in the repository root.
