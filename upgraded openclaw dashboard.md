# Skill: Upgraded OpenClaw Dashboard

## Purpose
This document is the operating playbook for future AI agents extending the OpenClaw Command Center project. It defines where to change code, how to add features safely, and how to keep upgrades backward-compatible.

## Scope
- Repository focus: single-file dashboard runtime in `index-2.html`
- Docs: `README.md`, `IMPLEMENTATION_GUIDE.md`
- Objective: add upgrades, integrations, and plugin-like features without breaking existing behavior

## Mental Model (current architecture)
- UI + state + persistence are all in `index-2.html`.
- Core state lives in the global `state` object.
- Rendering is function-based (`render*`) and updates DOM directly.
- Persistence layers:
  1) localStorage (fast cache)
  2) IndexedDB (durable browser tier)
  3) gateway file sync (`state_sync` / `state_request`)
- Gateway messaging is handled through `handleGatewayMessage` + `_storage` handlers.
- Public agent/control API is exposed via `window.__openclaw_cc`.

## Golden Rules for Every Upgrade
1. Keep `index-2.html` as source of truth unless docs clearly require file split.
2. Never break startup flow:
   - Bootstrap parsing
   - `initAvatars()`
   - `renderAll()`
   - Gateway connect and `asyncBoot()`
3. If adding anything in state:
   - include it in `STATE_KEYS`
   - include default value in `state`
   - include migration/default logic when state is loaded from older snapshots
   - ensure `_storage._applySnapshot()` handles it (or can safely ignore missing keys)
4. If adding persistence, route through existing helper paths:
   - state changes → `_safeSave` or dedicated `_save*`
   - full sync → `persistFullState()`
5. Keep plugin-like additions namespaced to avoid collisions in `window.__openclaw_cc` and state keys.

## Extending a Feature Area
To add or update a module (Org, Cron, Analytics, etc.):
1. UI: add HTML section and modal elements.
2. State: add defaults in `state`.
3. Mutations: add functions near similar feature section.
4. Persistence: add/extend `STATE_KEYS` and any save helpers.
5. Render: add/update renderer invoked from `renderAll()`.
6. API: expose needed methods via `window.__openclaw_cc`.
7. Integrations:
   - if remote agents need to call it, add safe handler in `handleGatewayMessage` (`canvas_eval` path if needed)
   - add rate-limit/safety checks for dangerous operations.

## Plugin Upgrade Pattern (Recommended for Future Add-ons)

Use a lightweight plugin surface that does not require a dynamic loader.

### 1) Define plugin registry (in JS state/helper area)
- Add:
  - `plugins: []` to state
  - `plugins` as a tracked `STATE_KEYS` entry
  - `registerPlugin`, `unregisterPlugin`, `enablePlugin`, `disablePlugin` methods on `window.__openclaw_cc`

### 2) Plugin contract (JSON shape)
```json
{
  "id": "my-plugin",
  "name": "My Plugin",
  "version": "0.1.0",
  "enabled": true,
  "config": {},
  "createdAt": 1700000000000
}
```

### 3) Execution hooks
- Add a central plugin dispatch table:
  - lifecycle: `onStateRestore`, `onTaskAdd`, `onChatMessage`, `onTick` (as needed)
- Keep callbacks synchronous, pure, and fail-safe (try/catch around plugin calls).
- Never let one plugin crash the dashboard loop; isolate with try/catch and log.

### 4) Plugin UI
- Add a plugin list panel in Settings or a dedicated “Plugins” view.
- Each plugin row should support enable/disable + delete + config paste/import.

## Safe Upgrade Steps (step-by-step)
1. Search for affected names before editing (`rg -n` in `index-2.html`).
2. Edit minimally around existing patterns.
3. Keep new methods small and deterministic.
4. Add user-visible feedback (`_showToast`) for any action that changes persistence.
5. Ensure settings path persists any new settings fields.
6. Document the behavior in `README.md` + `IMPLEMENTATION_GUIDE.md`.

## Backward Compatibility Checklist
- Old state snapshots still load without crashes.
- Missing fields fall back to defaults.
- `STATE_KEYS` additions do not prevent restore from older data.
- URL bootstrap and default gateway behavior remain intact.
- URL bootstrap supports OpenClaw-style `gatewayUrl`/`auth` style link formats.
- URL bootstrap values are scrubbed from browser address/query state after first parse (token hygiene).
- Gateway protocol messages unaffected unless intentionally versioned.

## Security Expectations
- Any new remote-capable method should consider:
  - `canvas_eval` allowlist and blocklist updates
  - rate limiting (`_canvasEvalAllowed`)
  - input validation and sanitization (`_esc`) for rendered content
- Do not persist secret material into non-secret fields.

## Performance Expectations
- Avoid deep synchronous loops inside high-frequency render paths.
- Use targeted render updates where practical; if global `renderAll()` is required, keep work bounded.
- Trim logs/chats with existing caps before persist when adding large history fields.

## Documentation Update Pattern
For each upgrade add:
- `README.md`: user-facing behavior and quick steps
- `IMPLEMENTATION_GUIDE.md`: architecture and protocol/state details
- Inline comments where behavior is non-obvious

## Common Mistakes to Avoid
- Forgetting to add new state keys to `STATE_KEYS` so data does not survive upgrades.
- Mutating state directly from async callbacks without save/render pipeline.
- Adding `window.__openclaw_cc` methods that can mutate core identity/security state without admin guard rails.
- Generating install links containing long-lived/high-privilege tokens in public spaces.

## Example Upgrade Playbook (template)
1. Create branch / backup.
2. Add plugin or module defaults.
3. Wire persistence keys + save helper.
4. Add rendering path and integrate into `renderAll()`.
5. Add API method and optional gateway event support.
6. Add guardrails and toasts.
7. Update docs.
8. Run quick manual checks:
   - open settings
   - save/connect
   - import/export or state sync path
   - plugin install/reload persistence
   - offline + reconnect
