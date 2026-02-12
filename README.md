# 🦞 OpenClaw Command Center

A zero-dependency, single-file AI agent management dashboard. One HTML file. No build step. No npm. No framework.

Connect it to an OpenClaw Gateway and manage your entire agent team from a glass-panel UI.

---

## Quick Start

```bash
# Option 1: Just open it
open index-2.html

# Option 2: Serve it (for WebSocket connectivity)
python3 -m http.server 8080
# Then open http://localhost:8080
```

Point the gateway URL in Settings (⚙) to your OpenClaw Gateway and you're live.

### One-Click Install Link

Host `index-2.html` on GitHub Pages or any static host, then share this URL format with users:

```
https://<your-host>/index-2.html?gateway=ws://<gateway-host>:18789&token=<AUTH_TOKEN>
```

Examples:

```bash
https://opencalw.example/index-2.html?gateway=ws://127.0.0.1:18789
https://opencalw.example/index-2.html?gateway=wss://gateway.openclaw.app:18789&token=oc_live_123
```

You can also use a fragment link with a full agent URL:

```
https://<your-host>/index-2.html#agent=ws://gateway.openclaw.app:18789?token=oc_live_123
```

On load, the dashboard auto-fills/saves those values and connects to the gateway.  
Prefer short-lived or low-privilege tokens in public links.
The dashboard parses OpenClaw-style `gatewayUrl`/`auth` style params too, and automatically removes bootstrap query/hash values after first load so tokens are not left visible in the address bar.

In the dashboard, open Settings (⚙) and use **COPY INSTALL LINK** to generate the exact shareable URL for that environment.

---

## Features

| Module | Description |
|---|---|
| **Org Structure** | Drag-and-drop node graph editor. Define agent hierarchy, connections, and routing. |
| **Agent Chat** | Per-agent message threads. Messages relay through the gateway to live LLM agents. |
| **Task Board** | Kanban board (todo → active → done). Agents can read/write tasks via the API. |
| **Cron Jobs** | Scheduled directives with natural language scheduling. Push to agents on a timer. |
| **Briefing Room** | Multi-agent meetings. Set an agenda, pick participants, agents discuss in real-time. |
| **Marketing** | Campaign planning with budgets, timelines, and agent assignments. |
| **Calendar** | Event scheduling with agent assignments and monthly/weekly views. |
| **Agent Logs** | Filterable activity log across all operations. Filter by agent, type, and time. |
| **System Prompts** | Per-agent prompt editor with identity, role, tools, personality, and instructions. Live push to gateway. |
| **API Vault** | Secrets management with per-agent access control (allowlists). |
| **RevenueCat** | Revenue analytics dashboard for iOS/Android apps. |
| **Avatars** | Procedural 8-bit pixel art character customizer for each agent. |

---

## Storage & Persistence

Your data is stored in three tiers simultaneously:

```
Gateway Filesystem  →  Most durable (survives everything)
IndexedDB           →  Durable (survives browser restarts, most cleanup)
localStorage        →  Fast cache (instant page loads)
```

- **Auto-backup** every 5 minutes (last 20 kept in IndexedDB)
- **Gateway sync** — state persists on the gateway host's filesystem
- **Auto-recovery** — if localStorage is wiped, recovers from IndexedDB or gateway
- **Export/Import** — manual JSON backup files
- **beforeunload guard** — emergency save on tab close
- Switch browsers, switch computers, clear cache — connect to the same gateway and your data is there.

---

## Gateway Protocol

The CC communicates over a single WebSocket. Full protocol spec in [IMPLEMENTATION_GUIDE.md](IMPLEMENTATION_GUIDE.md).

**Outbound (CC → Gateway):** `subscribe`, `get_models`, `message`, `prompt_update`, `routing_update`, `routing_directive`, `cron_sync`, `vault_manifest`, `vault_response`, `canvas_eval_result`, `meeting_start`, `state_sync`, `state_request`

**Inbound (Gateway → CC):** `message`, `presence`, `models`, `routing_ack`, `vault_request`, `canvas_eval`, `meeting_message`, `agent_comms`, `state_restore`, `state_ack`, `state_conflict`

### Gateway Persistence (for gateway developers)

The CC side is fully implemented. Gateway just needs to handle two message types:

```javascript
case 'state_sync':
  // Write msg.state to ~/.openclaw/workspace/canvas/state/current.json
  // Send back: { type: 'state_ack', savedAt: msg.savedAt }

case 'state_request':
  // Read current.json from disk
  // Send back: { type: 'state_restore', state: data, savedAt: data._savedAt }
```

Full implementation examples (Node.js and Python) in the [Implementation Guide](IMPLEMENTATION_GUIDE.md#gateway-side-implementation).

---

## Canvas API

Agents interact with the dashboard through `canvas_eval` messages. The CC exposes 40+ methods via `window.__openclaw_cc`:

```javascript
// Agents can call these through the gateway:
__openclaw_cc.addTask({ title: 'Deploy v2', agent: 'forge', status: 'todo' })
__openclaw_cc.logActivity({ type: 'system', agent: 'viper', desc: 'Campaign launched' })
__openclaw_cc.pushComms({ from: 'atlas', to: 'oracle', text: 'Research complete' })
__openclaw_cc.setAgentStatus('forge', 'busy')
```

Security: `canvas_eval` has a method whitelist (blocking state/agent/vault mutations) and per-agent rate limiting (50 calls per 10s).

---

## Security

- **XSS prevention** — all agent-provided content escaped via DOM-based sanitizer
- **canvas_eval whitelist** — dangerous methods (importState, createAgent, deleteSecret, etc.) blocked from remote execution
- **Rate limiting** — 50 canvas_eval calls per 10s per agent
- **Vault access control** — per-secret agent allowlists
- **State sync strips auth tokens** — gateway URL and token never sent in state payloads
- **Exponential backoff** — reconnection uses 1s→30s backoff with jitter

---

## Admin / Worker RBAC

Each agent has an `admin` flag:

- **Admin agents** get full dashboard context + 11 tool sections (agent CRUD, org structure, vault, cron, campaigns, etc.)
- **Worker agents** get restricted access — can only read/update their own tasks, log their own activity, and escalate to admins

Toggling admin status auto-regenerates and pushes the agent's system prompt.

---

## Tech Stack

- **HTML/CSS/JS** — vanilla, no framework
- **Glassmorphism UI** — iOS-inspired dark/light themes
- **IndexedDB** — primary durable storage
- **WebSocket** — gateway communication
- **Canvas API** — procedural pixel art avatars
- **CSS Grid/Flexbox** — responsive layout

**8,849 lines. 132 functions. 338KB. One file.**

---

## Requirements

- A modern browser (Chrome, Firefox, Safari, Edge)
- An OpenClaw Gateway (optional — works offline without one)

---

## License

See [LICENSE](LICENSE) file.
