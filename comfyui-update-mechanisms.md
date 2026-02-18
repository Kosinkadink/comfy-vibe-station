# ComfyUI Update Mechanisms

How ComfyUI updates work across the portable (standalone) and desktop (Electron) distributions.

---

# Portable (Windows Standalone)

## Directory Layout

```
ComfyUI_windows_portable/
├── python_embeded/          # Embedded Python (no system install needed)
├── ComfyUI/                 # Git repo clone
├── run_nvidia_gpu.bat       # Launch script
└── update/
    ├── update.py                                    # Core updater (pygit2-based)
    ├── update_comfyui.bat                           # Update to latest dev (master)
    ├── update_comfyui_stable.bat                    # Update to latest stable tag
    └── update_comfyui_and_python_dependencies.bat   # Full deps reinstall (dangerous)
```

## Source Location

The scripts live in the ComfyUI repo under `.ci/update_windows/`:

- [`update.py`](ComfyUI/.ci/update_windows/update.py) — core update logic
- [`update_comfyui.bat`](ComfyUI/.ci/update_windows/update_comfyui.bat) — nightly bat wrapper
- [`update_comfyui_stable.bat`](ComfyUI/.ci/update_windows/update_comfyui_stable.bat) — stable bat wrapper

The `update_comfyui_and_python_dependencies.bat` is **generated at build time** by GitHub Actions (see [`windows_release_dependencies.yml`](ComfyUI/.github/workflows/windows_release_dependencies.yml)).

## Update Flow (`update_comfyui.bat`)

### 1. Bat Wrapper

```bat
..\python_embeded\python.exe .\update.py ..\ComfyUI\
if exist update_new.py (
  move /y update_new.py update.py
  echo Running updater again since it got updated.
  ..\python_embeded\python.exe .\update.py ..\ComfyUI\ --skip_self_update
)
```

Runs `update.py` via the embedded Python. If the updater itself got updated (see self-update below), it replaces itself and re-runs.

The stable variant passes `--stable` to `update.py`.

### 2. Core Logic (`update.py`)

Uses **pygit2** (not the git CLI) for all git operations:

1. **Stash** any local changes (creates a stash with identity `comfyui <comfy@ui>`)
2. **Create a backup branch** named `backup_branch_YYYY-MM-DD_HH_MM_SS`
3. **Checkout `master`** — if the branch doesn't exist locally, creates it from `origin/master`
4. **Pull** latest changes via fetch + fast-forward (or merge if needed; aborts on conflicts)
5. **Stable mode** (`--stable` flag): after pulling master, finds the latest `refs/tags/v*.*.*` tag by sorting version numbers and checks it out

### 3. Self-Update Mechanism

After pulling, `update.py` compares itself against the freshly-pulled repo copy at `.ci/update_windows/update.py`. If they differ (and the new file is >10 bytes), it writes `update_new.py` and **exits early**. The `.bat` wrapper detects this file, replaces `update.py`, and re-runs with `--skip_self_update` to prevent infinite loops.

### 4. Dependency Sync

Compares `ComfyUI/requirements.txt` against a local `update/current_requirements.txt` cache. If they differ, runs:

```
python -s -m pip install -r requirements.txt
```

Then copies the new requirements file to the cache so subsequent runs skip the install.

### 5. Stable Script Propagation

If `update_comfyui_stable.bat` doesn't exist (or is <10 bytes) in the `update/` folder, copies it from the repo's `.ci/update_windows/` directory. This ensures older portable installs gain the stable update option.

## `new_updater.py` — Startup Bootstrap

[`ComfyUI/new_updater.py`](ComfyUI/new_updater.py) runs at ComfyUI startup and handles migrating **old-format** update scripts. It:

1. Reads the existing `update/update_comfyui.bat`
2. Checks if it starts with the legacy pattern (`..\\python_embeded\\python.exe .\\update.py`)
3. If so, overwrites `update/update.py` with the latest from `.ci/update_windows/`
4. Patches `update_comfyui_and_python_dependencies.bat` to replace legacy invocations with `call update_comfyui.bat nopause`
5. Overwrites `update/update_comfyui.bat` with the new version

This ensures the updater itself stays current even between manual updates.

## `update_comfyui_and_python_dependencies.bat`

Generated during CI ([`windows_release_dependencies.yml`](ComfyUI/.github/workflows/windows_release_dependencies.yml)):

```bat
call update_comfyui.bat nopause
echo This will try to update pytorch and all python dependencies.
echo If you just want to update normally, close this and run update_comfyui.bat instead.
pause
..\python_embeded\python.exe -s -m pip install --upgrade torch torchvision torchaudio \
  --extra-index-url https://download.pytorch.org/whl/cu130 \
  -r ../ComfyUI/requirements.txt pygit2
pause
```

This is the "nuclear" option — it first runs the normal update, then reinstalls PyTorch and all Python dependencies. Can break custom nodes that pin specific package versions.

## Build-Time Assembly (GitHub Actions)

The [`stable-release.yml`](ComfyUI/.github/workflows/stable-release.yml) workflow assembles the portable `.7z` package:

1. Downloads embedded Python and installs pip
2. Installs pre-cached wheel dependencies
3. Installs comfyui-specific pip packages (`comfyui-frontend-package`, `comfyui-workflow-templates`, `comfyui-embedded-docs`)
4. Clones TAESD models into `models/vae_approx/`
5. Copies `.ci/update_windows/*` → `update/`
6. Copies `.ci/windows_nvidia_base_files/*` → root (run scripts, README)
7. Copies the CI-generated `update_comfyui_and_python_dependencies.bat` → `update/`
8. Packages everything into a `.7z` archive

## Key Design Decisions

- **pygit2 over git CLI**: The portable package doesn't include git, so all git operations go through pygit2 (a Python libgit2 binding bundled as a pip package)
- **Backup branches**: Every update creates a timestamped backup branch, giving users a rollback path
- **Stash-before-update**: Local modifications are stashed rather than discarded
- **Self-updating updater**: The update scripts can update themselves, handled via a two-pass `.bat` approach to avoid file-in-use issues
- **Cached requirements**: Dependencies are only reinstalled when `requirements.txt` actually changes, keeping normal updates fast

---

# Desktop (Electron App)

The desktop app ([`desktop/`](desktop/)) replaces the portable batch scripts with a fully managed Electron + `uv` workflow. Updates happen at two levels: the **Electron shell** itself and the **Python packages** it manages.

## Electron App Auto-Update

The app uses [ToDesktop](https://www.todesktop.com/) (`@todesktop/runtime`) for self-updating the Electron binary:

- Checks for updates every **1 hour** against `https://updater.comfy.org` ([`comfyDesktopApp.ts`](desktop/src/main-process/comfyDesktopApp.ts))
- Exposes `checkForUpdates` and `restartAndInstall` to the renderer via IPC ([`AppHandlers.ts`](desktop/src/handlers/AppHandlers.ts), [`preload.ts`](desktop/src/preload.ts))
- When an update is ready, always prompts the user to install and restart

## Python Package Updates

Instead of pygit2 and batch scripts, the desktop app manages Python through **`uv`** (a fast Python package manager bundled as a binary in `assets/uv/`).

### Architecture

```
desktop/
├── src/
│   ├── virtualEnvironment.ts          # VirtualEnvironment class — uv-based venv management
│   ├── install/
│   │   └── installationManager.ts     # InstallationManager — orchestrates install & updates
│   └── main-process/
│       └── comfyInstallation.ts       # ComfyInstallation — validates install state
└── assets/
    ├── ComfyUI/                       # Bundled ComfyUI source (cloned at build time)
    │   ├── requirements.txt           # Core requirements
    │   └── manager_requirements.txt   # Manager requirements
    └── uv/                            # Bundled uv binaries (win/linux/darwin)
```

### VirtualEnvironment Class

[`virtualEnvironment.ts`](desktop/src/virtualEnvironment.ts) — the core of the desktop update system:

- **Creates** a `.venv` in the user's `basePath` using `uv venv` with a configured Python version (default 3.12)
- **Installs packages** via `uv pip install` with configurable mirrors for PyPI and PyTorch (CUDA, ROCm, CPU)
- **Validates requirements** using `uv pip install --dry-run` — parses the output to determine if packages are up to date, need a known upgrade, or have an unexpected discrepancy

### Update Detection (`hasRequirements`)

On every launch, `ComfyInstallation.validate()` calls `VirtualEnvironment.hasRequirements()`, which:

1. Runs `uv pip install --dry-run -r requirements.txt` for both ComfyUI core and ComfyUI-Manager
2. Checks if output says `Would make no changes` → packages are current
3. If packages need installing, classifies the change:
   - **Manager upgrade**: exactly 1–3 known packages (`uv`, `toml`, `chardet`)
   - **Core upgrade**: only recognized packages change (`aiohttp`, `av`, `yarl`, `pydantic`, `sqlalchemy`, etc.) with no unexpected removals
   - **Error**: unknown packages are being added/removed — flags for manual intervention
4. Returns `'OK'`, `'package-upgrade'`, or `'error'`

### Update Execution (`updatePackages`)

If `needsRequirementsUpdate` is true (upgrade detected), `InstallationManager.updatePackages()`:

1. Loads a `desktop-update` page in the Electron window
2. Calls `installComfyUIRequirements()` — runs `uv pip install -r requirements.txt` with configured mirrors
3. Calls `installComfyUIManagerRequirements()` — runs `uv pip install -r manager_requirements.txt`
4. Checks if NVIDIA driver version is below minimum (580) and warns the user
5. Re-validates the installation

### NVIDIA Torch Upgrade Check

`VirtualEnvironment.needsNvidiaTorchUpgrade()` reads installed torch/torchaudio/torchvision versions and checks they:
- Include the required CUDA tag (currently `+cu130`)
- Meet minimum version thresholds

This check exists but **automatic NVIDIA torch upgrades are currently disabled** (commented out) so users control large downloads themselves.

### Build-Time Setup

[`makeComfy.js`](desktop/scripts/makeComfy.js) runs during the desktop build to:

1. Clone ComfyUI at a specific tag (`v{version}`) or branch into `assets/ComfyUI/`
2. Clone ComfyUI-Manager into `assets/ComfyUI/custom_nodes/` (legacy) or use `manager_requirements.txt` (current)
3. Build the frontend and download `uv` binaries for all platforms

[`updateFrontend.js`](desktop/scripts/updateFrontend.js) automates frontend version bumps by fetching the latest release from `Comfy-Org/ComfyUI_frontend` and creating a PR.

### Reinstall & Recovery

The desktop app provides several recovery mechanisms:

- **Reinstall venv**: If Python import verification fails, offers to delete `.venv` and recreate it
- **Reinstall requirements**: `reinstallRequirements()` tries a manual install, and if that fails, recreates the entire venv
- **Clear uv cache**: `clearUvCache()` runs `uv cache clean`
- **Full reinstall**: IPC handler for `REINSTALL` deletes the installation and relaunches the app

## Key Design Decisions (Desktop vs Portable)

| Aspect | Portable | Desktop |
|--------|----------|---------|
| **Package manager** | pip (via embedded Python) | uv (bundled binary) |
| **Git operations** | pygit2 (Python binding) | Not needed — ComfyUI source bundled at build time |
| **Update trigger** | Manual (user runs `.bat`) | Automatic on every launch |
| **Torch management** | `update_comfyui_and_python_dependencies.bat` | Automatic detection, manual upgrade (disabled by default) |
| **Self-update** | Two-pass `.bat` with `update_new.py` | ToDesktop auto-updater for Electron shell |
| **Rollback** | Timestamped backup git branches | Reinstall venv / full reinstall |
| **Requirement diffing** | File comparison (`filecmp`) | `uv pip install --dry-run` output parsing |
