# Desktop Port Selection

When starting the ComfyUI backend, the desktop (Electron) app determines which port to use through a layered override system, then scans for an available port before spawning the Python process.

## Summary

The default port is `8000`. User launch-arg settings and dev-mode environment variables can override it. Before starting the server, `findAvailablePort()` probes the target port by attempting to bind a TCP server; if the port is in use it increments by 1, scanning up to 1000 ports above the target. The resolved port is passed as `--port <port>` to the ComfyUI Python process and used to construct the URL loaded into the Electron BrowserWindow.

## Detailed Flow

### 1. Default Server Args

**File:** `desktop/src/constants.ts`

The baseline configuration is defined in `DEFAULT_SERVER_ARGS`:

```typescript
export const DEFAULT_SERVER_ARGS: ServerArgs = {
  listen: '127.0.0.1',
  port: '8000',
  'enable-manager': '',
};
```

### 2. User & Dev Overrides

**File:** `desktop/src/main-process/comfyDesktopApp.ts`

`buildServerArgs()` merges three layers of configuration:

```typescript
async buildServerArgs({ useExternalServer, COMFY_HOST, COMFY_PORT }: DevOverrides): Promise<ServerArgs> {
  const serverArgs: ServerArgs = {
    ...DEFAULT_SERVER_ARGS,
    ...useComfySettings().get('Comfy.Server.LaunchArgs'),
  };

  if (COMFY_HOST) serverArgs.listen = COMFY_HOST;
  if (COMFY_PORT) serverArgs.port = COMFY_PORT;

  if (!useExternalServer) {
    const targetPort = Number(serverArgs.port);
    const port = await findAvailablePort(serverArgs.listen, targetPort, targetPort + 1000);
    serverArgs.port = String(port);
  }

  return serverArgs;
}
```

The priority order (highest wins):

1. **Dev environment variables** — `COMFY_HOST` / `COMFY_PORT` (only active when the app is unpackaged or launched with `--dev-mode`)
2. **User settings** — `Comfy.Server.LaunchArgs` from `electron-store`
3. **Defaults** — `DEFAULT_SERVER_ARGS` (`127.0.0.1:8000`)

**File:** `desktop/src/main-process/devOverrides.ts`

Dev overrides are read from `process.env` only when `!app.isPackaged` or `--dev-mode` is passed:

```typescript
constructor() {
  if (app.commandLine.hasSwitch('dev-mode') || !app.isPackaged) {
    this.COMFY_HOST = process.env.COMFY_HOST;
    this.COMFY_PORT = process.env.COMFY_PORT;
    // ...
  }
}
```

In production builds, these fields remain `undefined` and do not override anything.

### 3. Port Availability Scan

**File:** `desktop/src/utils.ts`

`findAvailablePort()` tries to bind a TCP server on the target port. If the port is taken, it increments by 1 and retries, up to `endPort`:

```typescript
export function findAvailablePort(host: string, startPort: number, endPort: number): Promise<number> {
  return new Promise((resolve, reject) => {
    function tryPort(port: number) {
      if (port > endPort) {
        reject(new Error(`No available ports found between ${startPort} and ${endPort}`));
        return;
      }

      const server = net.createServer();
      server.listen(port, host, () => {
        server.once('close', () => {
          resolve(port);
        });
        server.close();
      });
      server.on('error', () => {
        tryPort(port + 1);
      });
    }

    tryPort(startPort);
  });
}
```

Called with `(host, targetPort, targetPort + 1000)`, so the scan range is the target port through target + 1000. If all 1001 ports are busy, the promise rejects with an error.

This scan is **skipped** when `useExternalServer` is `true` (the user has an existing ComfyUI server running elsewhere).

### 4. Passing the Port to ComfyUI

**File:** `desktop/src/main-process/comfyServer.ts`

The resolved `serverArgs` (including `port`) are converted to CLI arguments by `buildLaunchArgs()`:

```typescript
static buildLaunchArgs(args: Record<string, string>) {
  return Object.entries(args)
    .flatMap(([key, value]) => [`--${key}`, value])
    .filter((value) => value !== '');
}
```

This produces args like `--listen 127.0.0.1 --port 8000 --enable-manager`, which are passed to the Python process.

### 5. Loading the Frontend

**File:** `desktop/src/main-process/appWindow.ts`

Once the server starts, the BrowserWindow loads the ComfyUI URL using the same `serverArgs`:

```typescript
public async loadComfyUI(serverArgs: ServerArgs) {
  const host = serverArgs.listen === '0.0.0.0' ? 'localhost' : serverArgs.listen;
  const url = this.frontendUrlOverride ?? `http://${host}:${serverArgs.port}`;
  await this.window.loadURL(url);
}
```

If `listen` is `0.0.0.0` (all interfaces), the URL uses `localhost` instead. A `frontendUrlOverride` (from `DEV_FRONTEND_URL`) can bypass this entirely for development.

### 6. Startup Orchestration

**File:** `desktop/src/desktopApp.ts`

The full sequence is orchestrated in the `DesktopApp.run()` method:

1. `buildServerArgs(overrides)` — resolve port with all overrides and availability scan
2. `startComfyServer(comfyDesktopApp, serverArgs)` — spawn the Python process with `--port`
3. `loadFrontend(serverArgs)` — load `http://host:port` in the BrowserWindow

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                     App Startup                             │
│                                                             │
│  1. DEFAULT_SERVER_ARGS          port = "8000"              │
│       ↓                                                     │
│  2. User settings overlay        port = user pref or "8000" │
│       ↓                                                     │
│  3. Dev overrides (COMFY_PORT)   port = env var or above    │
│       ↓                                                     │
│  4. findAvailablePort()          port = first free port     │
│     (tries port..port+1000)      in range                   │
│       ↓                                                     │
│  5. ComfyServer.start()          python main.py --port PORT │
│       ↓                                                     │
│  6. appWindow.loadComfyUI()      http://127.0.0.1:PORT      │
└─────────────────────────────────────────────────────────────┘
```

## Key Points

- The port is **not random** — it starts at a deterministic target (default `8000`) and walks upward to find the first free port.
- The scan range is 1001 ports wide (`target` through `target + 1000`).
- In production, only user settings can change the starting port. Dev environment variables are ignored in packaged builds.
- When using an external server (`USE_EXTERNAL_SERVER=true`), the port scan is skipped entirely — the configured port is used as-is.
