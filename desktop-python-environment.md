# Desktop Python Environment & Install Error Handling

The desktop (Electron) app manages a Python virtual environment for ComfyUI using `uv`. This document describes the environment setup, validation checks, and how errors are detected and recovered from at each stage.

## Summary

The app uses `uv` to create a `.venv` with a managed Python interpreter, then installs PyTorch + ComfyUI requirements into it. There are three points where Python environment errors are caught:

1. **During fresh install** — `VirtualEnvironment.create()` catches `PythonImportVerificationError` (import check failure) and generic errors separately, offering venv reinstall or showing docs.
2. **During validation on subsequent launches** — `ComfyInstallation.validate()` checks the base path, venv directory, Python interpreter, `uv` binary, pip packages, git, and VC++ Redist. Any failures load a maintenance page with troubleshooting IPC actions.
3. **During server startup** — If the ComfyUI Python process exits with a `ModuleNotFoundError` in stderr, the app offers to nuke and recreate the venv, then retry.

## Fresh Install Flow

**File:** `desktop/src/install/installationManager.ts` — `freshInstall()`

### Steps

1. **Hardware validation** — Detects GPU type. If no supported GPU is found, loads a "not-supported" page.

2. **Git check** — Runs `git --version`. If missing, shows a dialog with an "Open git downloads page" button.

3. **User options** — Collects install path, device type (nvidia/amd/unsupported), mirror preferences.

4. **NVIDIA driver warning** — On Windows with nvidia selected, runs `nvidia-smi` to read the driver version. If below the minimum (580), shows a warning dialog.

5. **Venv creation** — Calls `virtualEnvironment.create(callbacks)`:

```
createEnvironment()
  ├─ venv exists + has requirements?  → skip install, verify imports
  ├─ venv exists + missing packages?  → manualInstall(), verify imports
  └─ no venv?
       ├─ createVenvWithPython()      — uv venv --python 3.12.x
       ├─ ensurePip()                 — python -m ensurepip --upgrade
       └─ installRequirements()       — compiled requirements, fallback to manual
```

### Error Handling During Install

**File:** `desktop/src/install/installationManager.ts` lines 246–307

The `try/catch` around `virtualEnvironment.create()` handles two error types:

#### `PythonImportVerificationError`

Thrown when `verifyPythonImports()` finds that the venv exists but key modules (`yaml`, `torch`, `uv`, `toml`, `numpy`, `PIL`, `sqlalchemy`) can't be imported. The user sees:

> *"We were unable to verify the state of your Python virtual environment. This will likely prevent ComfyUI from starting. Would you like to remove your .venv directory and reinstall it?"*

- **Reinstall venv** — Deletes `.venv`, calls `create()` again.
- **Ignore** — Continues with the broken venv.

#### Generic errors (venv creation / pip / requirements failed)

Any other error:

1. Reports to Sentry.
2. Shows a dialog:
   > *"Failed to create virtual environment"*
   >
   > *"ComfyUI Desktop was unable to set up the Python environment required to run."*
3. Offers "View Documentation" (opens docs.comfy.org) or "Exit".
4. Exits with code `3001`.

## Requirements Installation

**File:** `desktop/src/virtualEnvironment.ts`

### `installRequirements()` (first install)

Tries the compiled/locked requirements file first (`requirements.compiled`). If that fails (non-zero exit code), falls back to `manualInstall()`.

### `manualInstall()` (fallback)

Installs in three steps:
1. `installPytorch()` — installs `torch`, `torchvision`, `torchaudio` from the appropriate index URL (NVIDIA CUDA, AMD ROCm, or nightly).
2. `installComfyUIRequirements()` — `uv pip install -r requirements.txt` for ComfyUI core.
3. `installComfyUIManagerRequirements()` — `uv pip install -r requirements.txt` for ComfyUI-Manager.

Each step throws on non-zero exit code.

### `reinstallRequirements()` (troubleshooting)

Tries `manualInstall()`. If that fails, recreates the venv from scratch (`createVenv` → `upgradePip` → `manualInstall`).

## Validation on Subsequent Launches

**File:** `desktop/src/main-process/comfyInstallation.ts` — `validate()`

On every app start (when an installation already exists), `ensureInstalled()` runs validation. The `validate()` method checks each component in order and sets the result to `'OK'` or `'error'`:

| Check | What it does | Error condition |
|-------|-------------|-----------------|
| **Base path** | `pathAccessible(basePath)` + restriction flags | Inaccessible, inside app install dir, updater cache, or OneDrive |
| **Venv directory** | `venv.exists()` (path accessible + non-empty) | Directory missing or empty |
| **Python interpreter** | `canExecute(venv.pythonInterpreterPath)` | Missing or not executable |
| **uv** | `canExecute(venv.uvPath)` | Missing or not executable |
| **Python packages** | `venv.hasRequirements()` — `uv pip install --dry-run` | Dry run shows missing packages |
| **Git** | `canExecuteShellCommand('git --help')` | Not found in PATH |
| **VC++ Redist** (Windows) | `pathAccessible(vcruntime140.dll)` | DLL missing |

### `hasRequirements()` dry-run logic

Runs `uv pip install --dry-run -r requirements.txt` for both core and Manager. Parses the output:

- **"Would make no changes"** → `'OK'`
- **Only `uv`/`toml`/`chardet` would be added** → `'package-upgrade'` (known Manager upgrade)
- **Only known core packages** (`aiohttp`, `av`, `yarl`, `pydantic`, `sqlalchemy`, etc.) **would change** → `'package-upgrade'`
- **Anything else missing** → now falls through to `'package-upgrade'` as well (recent change treats all mismatches as upgradeable)

### Maintenance Page

**File:** `desktop/src/install/installationManager.ts` — `#onUpdateHandler()`

The first time any validation field is set to `'error'`, the app loads the maintenance page. The user stays on this page until all issues are resolved.

### Troubleshooting IPC Actions

**File:** `desktop/src/install/troubleshooting.ts`

The maintenance page exposes these IPC actions to the renderer:

| IPC Channel | Action |
|-------------|--------|
| `UV_INSTALL_REQUIREMENTS` | `venv.reinstallRequirements()` — reinstall pip packages |
| `UV_CLEAR_CACHE` | `venv.clearUvCache()` — `uv cache clean` |
| `UV_RESET_VENV` | Delete `.venv` → `createVenv()` → `upgradePip()` |
| `SET_BASE_PATH` | Open folder picker, validate path safety, save new base path |
| `VALIDATE_INSTALLATION` | Re-run `installation.validate()` |
| `COMPLETE_VALIDATION` | User clicks "Continue" — checks if all issues resolved |

After any fix action succeeds, `onInstallFix` triggers re-validation. The loop in `resolveIssues()` repeats until the installation is valid.

## Server Startup Error Recovery

**File:** `desktop/src/desktopApp.ts` — `run()`

After the server process is spawned, the app polls `http://host:port/queue` with `waitOn` (timeout: `ComfyServer.MAX_FAIL_WAIT`). If the process exits with a non-zero code:

1. `comfyServer.parseLastError()` checks the last line of stderr for `ModuleNotFoundError`.
2. If matched, shows:
   > *"Missing Python Module — We were unable to import at least one required Python module. Would you like to remove and reinstall the venv?"*
3. **Reset Virtual Environment** — Deletes `.venv`, calls `virtualEnvironment.create()`, retries server start.
4. **Ignore** — Falls through to the generic error page.

**File:** `desktop/src/main-process/comfyServer.ts` — `parseLastError()`

```typescript
parseLastError(): ServerStartError | undefined {
  return this.lastStdErr?.match(/(^|\n)ModuleNotFoundError: /) ? 'ModuleNotFoundError' : undefined;
}
```

The `lastStdErr` field is updated on every stderr write, so it captures only the most recent chunk.

### Unhandled Startup Errors

Any error not caught by the `ModuleNotFoundError` path shows a generic error dialog and quits the app:

> *"An unexpected error occurred whilst starting the app, and it needs to be closed."*

## Import Verification

**File:** `desktop/src/services/pythonImportVerifier.ts`

`runPythonImportVerifyScript()` generates and runs a Python script that attempts to `__import__()` a list of modules, outputting results as JSON. The desktop app checks these modules:

- `yaml`, `torch`, `uv`, `toml`, `numpy`, `PIL`, `sqlalchemy`

The script returns `{ success: boolean, failed_imports: string[] }`. On failure, the result includes the list of missing modules. This is used by `verifyPythonImports()` during `createEnvironment()`.

## Architecture Diagram

```
App Start
  │
  ├─ First launch? ──► freshInstall()
  │                      ├─ validateHardware()
  │                      ├─ check git
  │                      ├─ collect install options
  │                      ├─ warn if NVIDIA driver < 580
  │                      └─ virtualEnvironment.create()
  │                           ├─ createVenvWithPython()    ─┐
  │                           ├─ ensurePip()                ├─ on error:
  │                           ├─ installRequirements()      │   PythonImportVerificationError
  │                           │   └─ fallback: manualInstall()   → offer venv reinstall
  │                           └─ verifyPythonImports()      │   Other error
  │                                                         │   → show docs, exit(3001)
  │                                                         ─┘
  ├─ Existing install? ──► validate()
  │                          ├─ base path accessible?
  │                          ├─ base path safe? (not OneDrive/app dir/cache)
  │                          ├─ .venv exists?
  │                          ├─ python interpreter executable?
  │                          ├─ uv executable?
  │                          ├─ pip packages complete? (dry-run)
  │                          ├─ git in PATH?
  │                          └─ vcruntime140.dll? (Windows)
  │                          │
  │                          └─ any errors? ──► maintenance page
  │                                              ├─ Reinstall requirements
  │                                              ├─ Clear uv cache
  │                                              ├─ Reset venv
  │                                              ├─ Change base path
  │                                              └─ Re-validate
  │
  └─ Server start ──► comfyServer.start()
                        ├─ spawn python main.py --port ...
                        ├─ poll /queue until ready
                        └─ exit with error?
                             ├─ ModuleNotFoundError in stderr?
                             │   → offer venv reset + retry
                             └─ other error
                                 → generic error page
```
