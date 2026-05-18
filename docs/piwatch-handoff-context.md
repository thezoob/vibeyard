# PiWatch Vibeyard Handoff Context

## Why This Exists

Zubair is considering switching PiWatch’s desktop session surface from Supacode to Vibeyard. The goal is not to migrate immediately, but to test whether Vibeyard can become a reliable Pi command center.

## Current PiWatch/Supacode Problem

PiWatch currently depends on Supacode CLI/tab concepts:

- `supacode repo open`
- `supacode tab new`
- `supacode tab focus`
- `supacode tab close`
- worktree IDs
- tab IDs / surface IDs

This works, but has caused stale-tab/deeplink issues such as Supacode showing “Tab not found” when PiWatch switches chats. PiWatch also needs reliable helper-session restart, exact session resume, abort, steering, and live output routing.

## What Was Checked

I inspected Vibeyard’s GitHub repo and this local fork.

Confirmed architecture:

- Electron app.
- Terminal sessions run through `node-pty`.
- Provider architecture exists in `src/main/providers/`.
- `src/main/pty-manager.ts` handles PTY spawning:
  - gets provider
  - builds env
  - builds args
  - resolves binary
  - calls `pty.spawn(...)`
- `src/main/ipc-handlers.ts` exposes internal renderer IPC:
  - `pty:create`
  - `pty:createShell`
  - `pty:write`
  - `pty:resize`
  - `pty:kill`
- `src/preload/preload.ts` exposes these to renderer.
- `src/shared/types.ts` hard-codes provider IDs:
  - `claude`
  - `codex`
  - `copilot`
  - `gemini`

Confirmed missing:

- no `pi` provider
- no external automation CLI/API
- no stable public session-control API
- no direct equivalent to Supacode tab/worktree control

## Key Code References

### `src/main/ipc-handlers.ts`

`pty:create` signature:

```ts
ipcMain.handle(
  'pty:create',
  async (
    _event,
    sessionId: string,
    cwd: string,
    cliSessionId: string | null,
    isResume: boolean,
    extraArgs: string,
    providerId: ProviderId = 'claude',
    initialPrompt?: string,
    systemPrompt?: string
  ) => { ... }
)
```

It calls:

```ts
await spawnPty(sessionId, cwd, cliSessionId, isResume, extraArgs, providerId, initialPrompt, systemPrompt, onData, onExit)
```

### `src/main/pty-manager.ts`

Important path:

```ts
const provider = getProvider(providerId);
const env = provider.buildEnv(sessionId, { ...process.env });
const args = provider.buildArgs({ cliSessionId, isResume, extraArgs, initialPrompt, systemPrompt });
const resolvedShell = provider.resolveBinaryPath();
const ptyProcess = pty.spawn(shell, spawnArgs, { cwd, env, cols: 120, rows: 30 });
```

### `src/main/providers/claude-provider.ts`

Good reference for provider implementation. Claude resume behavior:

```ts
if (opts.cliSessionId) {
  if (opts.isResume) {
    args.push('-r', opts.cliSessionId);
  } else {
    args.push('--session-id', opts.cliSessionId);
  }
}
```

### `src/main/providers/codex-provider.ts`

Codex resume behavior:

```ts
if (opts.isResume && opts.cliSessionId) {
  args.push('resume', opts.cliSessionId);
}
```

## Proposed Minimal Spike

### Step 1: Add `pi` provider

Add `pi` to `ProviderId` in `src/shared/types.ts`.

Create `src/main/providers/pi-provider.ts` implementing `CliProvider`.

Suggested behavior:

- `meta.id = 'pi'`
- `displayName = 'Pi'`
- `binaryName = 'pi'`
- capabilities:
  - session resume: true
  - hook status: false initially
  - images: depends on Pi capability; can start false/minimal
- `resolveBinaryPath()` uses `resolveBinary('pi', binaryCache)`.
- `buildEnv()`:

```ts
const env = { ...baseEnv };
env.PI_WATCH_SINGLE_BRIDGE = 'false';
env.PI_WATCH_ADVERTISE = 'false';
env.PI_WATCH_PORT = '0';
env.PATH = getFullPath();
return env;
```

- `buildArgs()`:

```ts
const args: string[] = [];
if (opts.isResume && opts.cliSessionId) {
  args.push('--resume', '--session', opts.cliSessionId);
}
if (opts.extraArgs) args.push(...opts.extraArgs.split(/\s+/).filter(Boolean));
if (opts.initialPrompt) args.push(opts.initialPrompt);
return args;
```

Register it in `src/main/providers/registry.ts`.

### Step 2: Verify by UI first

Before adding automation API, check whether the UI can create a `pi` provider session manually after provider registration.

Expected test:

1. Run Vibeyard from fork.
2. Open PiWatch project.
3. Start a Pi provider session.
4. Confirm a PiWatch bridge appears in:

```bash
~/.pi/agent/pi-watch-bridges.json
```

If this fails, fix provider args/env first.

### Step 3: Add automation API

After provider works, add local HTTP automation in the Electron main process.

Minimum endpoint for proof:

```http
POST /sessions
{
  "projectPath": "/path/to/project",
  "providerId": "pi",
  "sessionPath": "/Users/.../.pi/agent/sessions/...jsonl",
  "name": "PiWatch helper"
}
```

Expected behavior:

- create/find project record
- create session record
- set it active
- call existing PTY creation path
- return stable Vibeyard session ID

Then add:

- `GET /sessions`
- `POST /sessions/:id/focus`
- `POST /sessions/:id/write`
- `DELETE /sessions/:id`

## Design Caution

Vibeyard session state seems renderer-owned. Main process has PTY control, but project/session mutations live heavily in renderer state.

Two possible approaches:

### Fast Spike

Main HTTP server sends an IPC event to renderer like `automation:create-session`, and renderer performs existing state mutations + `window.api.pty.create(...)`. Renderer replies with ack.

### Cleaner Long-Term

Extract project/session mutation logic into a shared service callable by automation and renderer.

For the spike, use the fast route.

## PiWatch Repo Follow-Up

Only after Vibeyard proves create/resume/focus/close/list, patch PiWatch broker with a surface abstraction:

```ts
PI_WATCH_DESKTOP_SURFACE=supacode|vibeyard
PI_WATCH_VIBEYARD_URL=http://127.0.0.1:<port>
```

Keep Supacode as fallback until Vibeyard behavior is proven.

## Success Criteria

A successful Vibeyard spike means:

1. `pi` provider builds and runs.
2. Vibeyard can launch `pi --resume --session <jsonl>`.
3. PiWatch bridge registers from the Vibeyard-launched Pi process.
4. External local API can create that session without manual UI.
5. API can focus/list/close/write to the session.
6. PiWatch can switch chats without Supacode stale-tab errors.

## Existing Plan

See:

`docs/piwatch-patch-plan.md`
