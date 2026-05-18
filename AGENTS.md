# AGENTS.md

## Project Mission

This fork exists to evaluate and patch Vibeyard as a possible desktop surface for PiWatch, replacing or supplementing Supacode.

Primary objective: make Vibeyard able to launch, resume, focus, close, and control `pi` agent sessions through stable external automation.

## Current Status

- Fork: `https://github.com/thezoob/vibeyard`
- Local path: `/Users/zubairparkar/Documents/Ios_AI/vibeyard-pi`
- Initial plan: `docs/piwatch-patch-plan.md`
- No implementation has started yet.

## Important Findings

Vibeyard is not currently a drop-in Supacode replacement for PiWatch.

Confirmed:
- Vibeyard has a provider architecture under `src/main/providers/`.
- PTYs are created through `src/main/pty-manager.ts` using `node-pty`.
- Session creation is exposed internally through Electron IPC: `ipcMain.handle('pty:create', ...)` in `src/main/ipc-handlers.ts`.
- Renderer calls `window.api.pty.create(...)` through `src/preload/preload.ts`.
- Public `bin/vibeyard.js` only launches/updates the app and forwards args; it does not expose a stable automation CLI.
- Provider IDs are hard-coded in `src/shared/types.ts` as `claude | codex | copilot | gemini`.

Missing for PiWatch:
- `pi` provider.
- External automation API.
- Stable externally addressable session IDs for create/focus/close/write.
- Project/session control equivalent to Supacode CLI.

## Desired End State

PiWatch should be able to use Vibeyard through an adapter with operations like:

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

## Suggested Implementation Order

1. Add a `pi` provider.
2. Add minimal local automation endpoint to create a Pi session.
3. Verify PiWatch worker registration in `~/.pi/agent/pi-watch-bridges.json`.
4. Add list/focus/close/write endpoints.
5. Add PiWatch-side `VibeyardSurfaceProvider` adapter in the PiWatch repo later.

## Coding Guidance

- Keep the first spike minimal.
- Avoid large UI rewrites.
- Prefer adding provider/API seams over coupling PiWatch to renderer internals.
- Keep Vibeyard’s existing Claude/Codex/Gemini/Copilot behavior unchanged.
- Add focused tests around provider args and automation handlers.
- Bind any automation server to `127.0.0.1` by default.
- Do not expose unauthenticated LAN control.

## Key Files

- `docs/piwatch-patch-plan.md` — detailed plan.
- `src/shared/types.ts` — add `pi` to `ProviderId`.
- `src/main/providers/registry.ts` — register `PiProvider`.
- `src/main/providers/provider.ts` — provider interface reference.
- `src/main/providers/claude-provider.ts` — useful provider pattern.
- `src/main/pty-manager.ts` — PTY spawn path.
- `src/main/ipc-handlers.ts` — internal IPC handlers.
- `src/preload/preload.ts` — renderer API surface.
- `src/renderer/state.ts` — project/session mutations.
- `bin/vibeyard.js` — public CLI wrapper; currently not enough for PiWatch automation.

## Test Commands

From repo root:

```bash
npm install
npm test
npm run build
```

Use targeted tests if adding provider tests.

## Do Not

- Do not delete existing providers.
- Do not make automation network-visible by default.
- Do not assume Vibeyard app state is safely mutable from main without checking renderer/store flow.
- Do not migrate PiWatch off Supacode until create/resume/focus/close/list are proven.
