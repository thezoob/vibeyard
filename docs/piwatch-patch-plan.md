# PiWatch + Vibeyard Patch Plan

Status: planning spike. Fork: https://github.com/thezoob/vibeyard

## Verified Findings

Vibeyard is not a drop-in Supacode replacement today.

What exists:
- Electron app with `node-pty` session backend.
- Providers are implemented under `src/main/providers/`.
- Current provider union is hard-coded in `src/shared/types.ts`:
  - `claude`
  - `codex`
  - `copilot`
  - `gemini`
- PTY creation is already abstracted by provider:
  - `src/main/ipc-handlers.ts` calls `spawnPty(...)` via `ipcMain.handle('pty:create', ...)`.
  - `src/main/pty-manager.ts` calls `provider.buildEnv(...)`, `provider.buildArgs(...)`, resolves provider binary, then `pty.spawn(...)`.
- Renderer calls `window.api.pty.create(...)` through preload.
- Public `bin/vibeyard.js` only launches/updates the app; no stable external automation commands were found.

What is missing for PiWatch:
- No `pi` provider.
- No public CLI/API for external automation:
  - open project
  - create session
  - focus session
  - list sessions
  - close session
  - resume exact saved session
  - inject text into a specific session
- No stable external session IDs equivalent to Supacode tab/surface IDs.

## Goal
Make Vibeyard usable as a PiWatch desktop surface without relying on Supacode tab APIs.

## Required Capabilities

PiWatch broker needs these operations:

1. `openProject(path)`
2. `createPiSession(projectPath, sessionPath?)`
3. `focusSession(sessionId)`
4. `closeSession(sessionId)`
5. `listSessions()`
6. `writeToSession(sessionId, text)`
7. `resumeExactPiSession(projectPath, piSessionJsonlPath)`
8. reliable stable IDs persisted across app state

## Proposed Patch Phases

### Phase 1 — Add Pi provider

Files likely touched:
- `src/shared/types.ts`
- `src/main/providers/pi-provider.ts` (new)
- `src/main/providers/registry.ts`
- provider tests

Implementation:
- Extend `ProviderId` to include `'pi'`.
- Add `PiProvider` implementing `CliProvider`.
- Binary: `pi`.
- Env:
  - `PI_WATCH_SINGLE_BRIDGE=false`
  - `PI_WATCH_ADVERTISE=false`
  - `PI_WATCH_PORT=0`
- Args:
  - new session: no resume args, plus any extra args
  - resume: `--resume --session <sessionPath>`
- No hook-status support initially unless Pi exposes equivalent lifecycle hooks.
- Minimal config/readiness support: binary present + optional version check.

### Phase 2 — Add external automation IPC/HTTP

Vibeyard’s current IPC is renderer-only. PiWatch needs an external control surface.

Recommended v1: local HTTP server in main process, bound to `127.0.0.1`.

Endpoints:
- `GET /health`
- `GET /projects`
- `POST /projects/open { path }`
- `GET /sessions`
- `POST /sessions { projectPath, providerId: 'pi', sessionPath?, name? }`
- `POST /sessions/:id/focus`
- `POST /sessions/:id/write { text }`
- `DELETE /sessions/:id`

Security:
- bind localhost only by default
- token stored in app config if remote control is ever exposed
- no unauthenticated LAN binding

### Phase 3 — Wire automation to app state

Need reusable main/renderer state operations instead of only UI-driven session creation.

Patch targets:
- `src/renderer/state.ts` has project/session mutation logic.
- `src/main/ipc-handlers.ts` already exposes PTY create/write/kill.
- Need bridge between main HTTP commands and renderer/app state, likely via `webContents.send` + renderer handler, or move session state mutation into shared/main-accessible service.

Lowest-risk option:
- main HTTP receives command
- main sends `automation:create-session` to renderer
- renderer uses existing app state methods to create session and calls `window.api.pty.create(...)`
- renderer replies via IPC ack

Cleaner long-term option:
- extract project/session store mutations into shared service callable by both renderer and automation layer.

### Phase 4 — PiWatch broker adapter

Add a desktop surface abstraction in PiWatch:

```ts
interface DesktopSurfaceProvider {
  openProject(path: string): Promise<void>;
  createPiSession(path: string, sessionPath?: string): Promise<{ sessionId: string }>;
  focusSession(sessionId: string): Promise<void>;
  closeSession(sessionId: string): Promise<void>;
  listSessions(): Promise<SurfaceSession[]>;
  writeToSession(sessionId: string, text: string): Promise<void>;
}
```

Implement:
- `SupacodeSurfaceProvider` — current behavior.
- `VibeyardSurfaceProvider` — calls Vibeyard local HTTP.

Make selectable via env:
- `PI_WATCH_DESKTOP_SURFACE=supacode|vibeyard`
- `PI_WATCH_VIBEYARD_URL=http://127.0.0.1:<port>`

### Phase 5 — Reliability tests

Test matrix:
- create new Pi session for current project
- resume exact Pi JSONL session
- focus existing session
- close stale session
- restart Vibeyard and verify persisted session metadata
- PiWatch session switch uses Vibeyard without Supacode tab-not-found style failures
- abort and steer still route through PiWatch bridge, not the terminal surface

## First Spike Recommendation

Do not build the entire migration first. Start with the minimum proof:

1. Add `pi` provider.
2. Add one automation endpoint: `POST /sessions`.
3. From shell, call endpoint to spawn `pi --resume --session <path>` inside Vibeyard.
4. Confirm PiWatch worker registers in `~/.pi/agent/pi-watch-bridges.json`.

If that works, continue. If not, stop; Vibeyard is not ready as the desktop surface.

## Verdict

Vibeyard is a good candidate for a custom Pi command center because it is open source and already has a provider/PTY architecture. But it needs patches before PiWatch can replace Supacode. The key missing piece is external automation, not PTY support.
