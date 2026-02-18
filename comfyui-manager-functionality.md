# ComfyUI-Manager Functionality

Deep-dive into how ComfyUI-Manager works — as a reference for replicating environment management features in ComfyUI-Launcher.

---

## Two Modes of Operation

ComfyUI-Manager exists in two forms:

| | Legacy (v3, `main` branch) | PyPI Package (v4, `manager-v4` branch) |
|---|---|---|
| **Installation** | `git clone` into `custom_nodes/ComfyUI-Manager/` | `pip install comfyui-manager` (from `manager_requirements.txt`) |
| **Activation** | Auto-discovered as a custom node by ComfyUI | Requires `--enable-manager` CLI flag |
| **Entry point** | `__init__.py` at repo root (ComfyUI custom node protocol: `NODE_CLASS_MAPPINGS`, `WEB_DIRECTORY`) | `comfyui_manager/__init__.py` with explicit public API |
| **Imports** | `sys.path.append("glob/")` hacks | Proper relative imports (`from .common import ...`) |
| **ComfyUI version** | `manager_requirements.txt` → `comfyui_manager==4.1b1` | Same package, different entry path |
| **CLI** | `cm-cli.py` ad-hoc script | `cm-cli` console entrypoint via `pyproject.toml [project.scripts]` |

### How `--enable-manager` Works (PyPI Mode)

**File:** [`ComfyUI/main.py`](ComfyUI/main.py)

ComfyUI's `main.py` checks `--enable-manager` early and `import comfyui_manager` (the pip-installed package). It then calls four public functions at specific lifecycle points:

```python
# 1. Early import + validation
if args.enable_manager:
    if importlib.util.find_spec("comfyui_manager"):
        import comfyui_manager
        # Validates it's a real __init__.py (not a stale namespace package)
    else:
        args.enable_manager = False  # graceful fallback

# 2. Pre-startup (before custom nodes load)
if args.enable_manager:
    comfyui_manager.prestartup()  # security checks, logging, snapshot restore, lazy installs

# 3. During custom node loading — block old git-based manager
if args.enable_manager:
    if comfyui_manager.should_be_disabled(module_path):  # True if 'comfyui-manager' in dir name
        continue  # skip loading

# 4. Middleware injection
if args.enable_manager:
    middlewares.append(comfyui_manager.create_middleware())  # centralized security middleware

# 5. Server start (after aiohttp app is created)
if args.enable_manager and not args.disable_manager_ui:
    comfyui_manager.start()  # registers all HTTP route handlers
```

### Public API Functions

| Function | Purpose |
|----------|---------|
| `prestartup()` | Imports `prestartup_script.py` as side-effect — runs security scan, pip installs, snapshot restore, logging setup |
| `should_be_disabled(fullpath)` | Returns `True` for paths containing `'comfyui-manager'` — prevents the old git clone from loading alongside the pip package |
| `create_middleware()` | Returns an aiohttp middleware that tracks connected clients and enforces per-handler security policies via `HANDLER_POLICY` enum |
| `start()` | Imports `manager_server` and `share_3rdparty` (registering routes), sets security mode from config |

### Additional CLI Flags

| Flag | Description |
|------|-------------|
| `--enable-manager` | Enable ComfyUI-Manager (required) |
| `--enable-manager-legacy-ui` | Load the v3 UI from `comfyui_manager/legacy/` instead of the new `glob/` implementation |
| `--disable-manager-ui` | Disable the Manager UI and endpoints; background features (security checks, scheduled installs) still run |

### v4 Package Structure

```
comfyui_manager/                    # pip-installed package
├── __init__.py                     # Public API: prestartup(), start(), should_be_disabled(), create_middleware()
├── prestartup_script.py            # Pre-startup logic (same role as legacy, proper imports)
├── common/                         # Shared utilities (both glob + legacy use these)
│   ├── cm_global.py
│   ├── context.py                  # NEW: centralized path/env resolution (replaces scattered globals)
│   ├── manager_util.py
│   ├── manager_downloader.py
│   ├── manager_security.py         # NEW: HANDLER_POLICY enum, middleware security primitives
│   ├── cnr_utils.py
│   ├── security_check.py
│   ├── git_utils.py
│   ├── node_package.py
│   └── timestamp_utils.py
├── glob/                           # NEW implementation (v4 routes, Pydantic models)
│   ├── manager_core.py
│   ├── manager_server.py           # Routes at /v2/customnode/*, /v2/manager/*, /v2/snapshot/*
│   ├── share_3rdparty.py
│   ├── constants.py
│   └── utils/                      # Refactored utilities split by concern
│       ├── security_utils.py
│       ├── environment_utils.py
│       ├── formatting_utils.py
│       ├── model_utils.py
│       └── node_pack_utils.py
├── legacy/                         # Port of v3 code (for --enable-manager-legacy-ui)
│   ├── manager_core.py
│   ├── manager_server.py
│   └── share_3rdparty.py
├── data_models/                    # NEW: Pydantic v2 models generated from openapi.yaml
│   └── generated_models.py
├── js/                             # Frontend JavaScript
└── channels.list.template
cm_cli/                             # Top-level CLI package
├── __init__.py
└── __main__.py
```

### Key v4 vs v3 Differences

| Aspect | v3 (legacy custom node) | v4 (PyPI package) |
|--------|------------------------|-------------------|
| **Security** | Per-route checks in each handler | Centralized middleware with `HANDLER_POLICY` enum |
| **Path management** | Scattered globals in `manager_core.py` | Centralized in `common/context.py` |
| **API routes** | `/customnode/...`, `/manager/...`, `/snapshot/...` | Also adds `/v2/customnode/...`, `/v2/manager/...`, `/v2/snapshot/...` |
| **Data models** | Plain dicts throughout | Pydantic v2 models from `openapi.yaml` |
| **User directory** | `user/default/ComfyUI-Manager` | `folder_paths.get_system_user_directory("manager")` |
| **Dependencies** | `matrix-nio` required | `matrix-nio` optional |

---

## Legacy Architecture (v3, `main` branch)

The remainder of this document describes the v3 architecture since it's what's currently deployed and has the most detailed implementation to reference. The v4 PyPI package retains the same core functionality but with improved structure.

```
ComfyUI-Manager/                    # git clone in custom_nodes/
├── __init__.py              # Entry point — loads manager_server.py into ComfyUI
├── prestartup_script.py     # Runs BEFORE ComfyUI loads nodes (security checks, lazy installs, snapshot restore, logging)
├── cm-cli.py                # Standalone CLI (typer-based)
├── glob/                    # Core Python modules (added to sys.path)
│   ├── cm_global.py         # Global variables, extension API registry, pip blacklists
│   ├── cnr_utils.py         # Comfy Node Registry (CNR) API client
│   ├── git_utils.py         # Git repo introspection (commit hash, remote URL, no GitPython locking)
│   ├── manager_core.py      # UnifiedManager, snapshots, config, install/update/enable/disable logic
│   ├── manager_downloader.py # Download helpers (basic, torchvision, aria2, huggingface)
│   ├── manager_migration.py # Legacy path migration, security checks for old ComfyUI
│   ├── manager_server.py    # aiohttp route handlers (/manager/*, /customnode/*, /snapshot/*, /externalmodel/*)
│   ├── manager_util.py      # pip commands, version comparison, caching, PIPFixer
│   ├── node_package.py      # InstalledNodePackage dataclass
│   ├── security_check.py    # Malware detection (known bad nodes/packages)
│   └── share_3rdparty.py    # Workflow sharing integrations
├── js/                      # Frontend JavaScript (custom nodes manager, model manager, snapshot UI)
├── node_db/                 # Static databases of known custom nodes
└── snapshots/               # Snapshot JSON files (user data)
```

---

## 1. Custom Node Management

### Node Sources

Manager supports three types of custom node installations:

| Type | How it's identified | Source |
|------|-------------------|--------|
| **CNR** (ComfyRegistry) | Has `.tracking` file + `pyproject.toml` with `project.name` | Downloaded as `.zip` from `api.comfy.org/nodes/{id}/install?version=X` |
| **Nightly** (git) | Has `.git/` dir + `.git/.cnr-id` mapping | Cloned via `git.Repo.clone_from()` (GitPython) |
| **Unknown** (git) | Has `.git/` dir, no CNR mapping | Cloned via git, not in CNR registry |

### Node Install Flows

#### CNR Install (`cnr_install`)
1. Call `api.comfy.org/nodes/{id}/install?version=X` → get `download_url`
2. Download zip to `custom_nodes/CNR_temp_{uuid}.zip` (unpredictable name for security)
3. Extract into `custom_nodes/{node_id}/`
4. Write `.tracking` file listing all extracted files (used for clean upgrades)
5. Run post-install: `requirements.txt` (pip install each line) → `install.py` (if exists)

#### Git Install (`repo_install`)
1. `git.Repo.clone_from(url, path, recursive=True)`
2. On Windows (non-instant mode): delegates to `git_helper.py` subprocess to avoid file locking
3. Run post-install scripts

#### CNR Version Switch (`cnr_switch_version`)
1. Download new version zip
2. Extract over existing directory
3. Diff `.tracking` (old files) vs extracted (new files) → delete garbage files
4. Update `.tracking` with new file list
5. Run post-install

### Post-Install Script Execution

**File:** `manager_core.py` → `execute_install_script()`

1. If `requirements.txt` exists: pip install each line (with pip overrides and blacklist checks)
2. If `install.py` exists: run `python install.py` in the node directory
3. `PIPFixer.fix_broken()` runs after to restore torch/opencv/frontend if broken

### Lazy vs Instant Execution

- **Instant** (`instant_execution=True`): scripts run immediately in-process (used by CLI)
- **Lazy** (default, especially on Windows): writes commands to `startup-scripts/install-scripts.txt`, executed on next ComfyUI restart by `prestartup_script.py`

### Enable/Disable

- **Disable**: moves node directory to `custom_nodes/.disabled/{node_id}@{version}` (version suffix for multi-version support)
- **Enable**: moves back from `.disabled/` to `custom_nodes/{node_id}`
- ComfyUI-Manager itself is protected — install/disable/uninstall operations are blocked for it

### Uninstall

- Removes the directory from both active and inactive (`.disabled/`) locations
- Uses `shutil.rmtree()`, with fallback to lazy delete on failure (schedule for restart)

---

## 2. Pip Package Management

### Package Installation

**File:** `manager_util.py` → `make_pip_cmd()`

Uses either `pip` or `uv` (configurable via `use_uv` config):
- `pip` mode: `python -s -m pip install <package>` (embedded Python) or `python -m pip install`
- `uv` mode: `python -m uv pip install` or standalone `uv pip install`

Falls back from pip → uv automatically if pip is unavailable.

### Pip Blacklists & Overrides

- **Blacklist** (`pip_blacklist`): torch, torchaudio, torchsde, torchvision — never installed/upgraded
- **Downgrade blacklist** (`pip_downgrade_blacklist`): torch, transformers, safetensors, kornia — won't downgrade
- **Overrides** (`pip_overrides.json`): remap package names (e.g., swap to custom forks)
- **User blacklist** (`pip_blacklist.list`): user-defined packages to never install

### PIPFixer

**File:** `manager_util.py` → `PIPFixer`

After each batch of node installs, checks and fixes:
1. **Torch rollback**: if torch/torchvision/torchaudio versions changed, restores original versions
2. **OpenCV alignment**: if multiple opencv variants installed at different versions, upgrades all to max
3. **Frontend package**: if `comfyui-frontend-package` got removed, reinstalls it
4. **Custom fix list** (`pip_auto_fix.list`): user-defined packages to auto-restore

---

## 3. ComfyUI Core Update

### Nightly Update (`update_path`)

Uses GitPython (`git.Repo`):
1. If HEAD is detached, switch to default branch (master/main)
2. Fetch from remote
3. Compare local HEAD vs remote HEAD
4. If different: `git pull`
5. Returns `"updated"`, `"skip"`, or `"fail"`

### Stable Update (`update_to_stable_comfyui`)

1. Fetch all tags
2. Find latest `v*.*.*` tag by version sort
3. Checkout that tag (detached HEAD)

### Via Task Queue

Updates go through an async task queue (`manager_server.py` → `task_worker()`):
- `update-comfyui`: updates ComfyUI core
- `update-main`: updates all node packs
- `update`: updates a single node pack

Results are sent to the frontend via WebSocket (`cm-queue-status`).

---

## 4. Snapshot System

Snapshots capture the full environment state for reproducibility and rollback.

### Snapshot Content

```json
{
  "comfyui": "<git commit hash>",
  "git_custom_nodes": {
    "https://github.com/user/repo": {
      "hash": "<commit hash>",
      "disabled": false
    }
  },
  "cnr_custom_nodes": {
    "node-pack-id": "1.2.3"
  },
  "file_custom_nodes": [
    { "filename": "standalone.py", "disabled": false }
  ],
  "pips": {
    "package-name": "version",
    "package-with-url": "https://..."
  }
}
```

### Save Snapshot

1. Get ComfyUI git commit hash
2. Scan `custom_nodes/` and `.disabled/` for all installed nodes
3. For each: resolve CNR version or git commit hash
4. Capture all pip packages via `pip freeze`
5. Save as timestamped JSON to `snapshots/` directory

### Restore Snapshot

**File:** `manager_core.py` → `restore_snapshot()`

Processes three categories:

1. **CNR nodes**: disable unlisted, switch versions for mismatched, install missing
2. **Nightly/git nodes**: disable unlisted, enable listed, checkout specific commits, clone missing
3. **Unknown nodes**: same logic as nightly but using raw git URLs

Also optionally restores pip packages (with flags: `--pip-non-url`, `--pip-non-local-url`, `--pip-local-url`).

An "autosave" snapshot is created before any update-all or restore operation.

---

## 5. Model Management

### Model Database

Static JSON file (`model-list.json`) listing downloadable models with metadata:
- `type` (checkpoint, lora, vae, etc.) → mapped to ComfyUI folder paths
- `url`, `filename`, `save_path`

### Model Download

Three download strategies:
1. **torchvision** `download_url`: for GitHub, HuggingFace, Heidelberg URLs
2. **Agent-based**: `urllib` with browser User-Agent header
3. **aria2**: if `COMFYUI_MANAGER_ARIA2_SERVER` env var is set
4. **HuggingFace Hub**: for `<huggingface>` type — downloads entire repos

### Install Check

For each model in the list, checks if the file already exists in the corresponding model directory using `folder_paths.get_filename_list()`.

---

## 6. Prestartup Script (`prestartup_script.py`)

Runs before ComfyUI loads any nodes. Critical for environment management:

### Execution Order

1. **Security scan**: checks for known malware (ComfyUI_LLMVISION, lolMiner, compromised ultralytics)
2. **Read config**: load `config.ini`, set `uv` mode
3. **Load pip overrides and blacklists**
4. **Restore snapshot** (if `startup-scripts/restore-snapshot.json` exists)
5. **Execute lazy install scripts** (if `startup-scripts/install-scripts.txt` exists):
   - `#LAZY-INSTALL-SCRIPT`: pip install + install.py for a node
   - `#LAZY-CNR-SWITCH-SCRIPT`: download zip, extract, swap files, install deps
   - `#LAZY-DELETE-NODEPACK`: delete a node directory
   - Regular commands: `pip install`, etc.
6. **PIPFixer**: restore torch, opencv, frontend if broken
7. **Restart**: if any startup scripts executed, auto-restart ComfyUI (via `os.execv` or comfy CLI reboot file)
8. **Logging setup**: file-based logging with stdout/stderr wrapping, import failure tracking
9. **Windows event loop policy**: optional `WindowsSelectorEventLoopPolicy`

---

## 7. Configuration System

### `config.ini` (INI format via `configparser`)

| Key | Default | Description |
|-----|---------|-------------|
| `preview_method` | `none` | Latent preview method (auto/latent2rgb/taesd/none) |
| `git_exe` | `''` | Custom git executable path |
| `use_uv` | `false` | Use `uv` instead of `pip` |
| `channel_url` | GitHub raw URL | Custom node database channel |
| `share_option` | `all` | Workflow sharing visibility |
| `bypass_ssl` | `false` | Skip SSL verification |
| `file_logging` | `true` | Enable file-based logging |
| `security_level` | `normal` | Access control level (strong/normal/normal-/weak) |
| `network_mode` | `public` | Network access mode (public/private/offline) |
| `db_mode` | `cache` | Database fetch mode (local/cache/remote) |
| `update_policy` | `stable-comfyui` | ComfyUI update target |
| `always_lazy_install` | `false` | Force lazy install mode |
| `model_download_by_agent` | `false` | Use agent-based downloads |
| `downgrade_blacklist` | `''` | Additional packages to never downgrade |

### Security Levels

| Level | Local access | Remote access |
|-------|-------------|---------------|
| `strong` | Block all installs | Block all |
| `normal` | Allow installs | Block high-risk |
| `normal-` | Allow all | Allow installs |
| `weak` | Allow all | Allow all |

On old ComfyUI (without System User API), security is forced to `strong`.

---

## 8. Comfy Node Registry (CNR) API

**File:** `cnr_utils.py`

### Endpoints

| Endpoint | Purpose |
|----------|---------|
| `GET /nodes?page=N&limit=30&comfyui_version=X&form_factor=Y` | Paginated node list |
| `GET /nodes/{id}/install?version=X` | Get download URL for a specific version |
| `GET /nodes/{id}/versions` | List all available versions |

### Data Caching

- CNR data is fetched and cached to disk (JSON files in `.cache/`)
- Cache validity: 1 day
- On startup with `dont_wait=True`: uses stale cache if fresh data isn't available
- `form_factor` sent to API: `desktop-win`, `desktop-mac`, `git-windows`, `git-mac`, `git-linux`, `other`

---

## 9. CLI (`cm-cli.py`)

Standalone command-line interface using `typer`:

| Command | Description |
|---------|-------------|
| `install <node_spec>` | Install node packs (supports `name@version`, URLs) |
| `reinstall <node_spec>` | Uninstall then reinstall |
| `uninstall <node_spec>` | Remove node packs |
| `update <node_spec>` | Update node packs |
| `update all` | Update all installed packs |
| `disable/enable <node_spec>` | Toggle node packs |
| `fix <node_spec>` | Re-run install scripts (fix dependencies) |
| `show [installed\|enabled\|...]` | List node status |
| `save-snapshot` | Save environment snapshot (JSON or YAML) |
| `restore-snapshot <file>` | Restore from snapshot |
| `restore-dependencies` | Re-run all install scripts for all nodes |
| `install-deps <file>` | Install from dependency spec |
| `export-custom-node-ids` | Export all known node IDs |

Requires `COMFYUI_PATH` env var or discovers ComfyUI by relative path.

---

## 10. HTTP API Routes

### Node Management

| Route | Method | Description |
|-------|--------|-------------|
| `/customnode/getlist` | GET | Get all available custom nodes (with install status) |
| `/customnode/installed` | GET | Get currently installed node packs |
| `/customnode/getmappings` | GET | Node name → node pack mapping |
| `/customnode/fetch_updates` | GET | Check for available updates |
| `/customnode/versions/{name}` | GET | Get all CNR versions of a node |
| `/customnode/disabled_versions/{name}` | GET | Get locally disabled versions |
| `/customnode/alternatives` | GET | Get alternative nodes list |

### Task Queue

| Route | Method | Description |
|-------|--------|-------------|
| `/manager/queue/install` | POST | Queue node install |
| `/manager/queue/uninstall` | POST | Queue node uninstall |
| `/manager/queue/update` | POST | Queue node update |
| `/manager/queue/update_all` | GET | Queue update-all (with autosave snapshot) |
| `/manager/queue/fix` | POST | Queue dependency fix |
| `/manager/queue/status` | GET | Check queue status |

### Snapshots

| Route | Method | Description |
|-------|--------|-------------|
| `/snapshot/getlist` | GET | List saved snapshots |
| `/snapshot/save` | GET | Save current snapshot |
| `/snapshot/restore` | GET | Schedule snapshot restore (on next restart) |
| `/snapshot/remove` | GET | Delete a snapshot |
| `/snapshot/get_current` | GET | Get current environment state |

### Models

| Route | Method | Description |
|-------|--------|-------------|
| `/externalmodel/getlist` | GET | Get model list with install status |
| `/manager/queue/install_model` | POST | Queue model download |

### Config & Misc

| Route | Method | Description |
|-------|--------|-------------|
| `/manager/setting` | GET | Get/set Manager config |
| `/manager/preview_method` | GET | Get/set preview method |
| `/manager/reboot` | GET | Restart ComfyUI |
| `/manager/notice` | GET | Get news/notices |
| `/manager/startup_alerts` | GET | Get migration/security alerts |

---

## Key Design Patterns for Launcher Replication

### What Manager Does That Launcher Would Need

1. **Custom node install/uninstall/update**: CNR zip download, git clone, post-install scripts
2. **Pip management**: with blacklists, overrides, auto-fix (torch rollback, opencv alignment)
3. **Enable/disable**: moving directories to/from `.disabled/`
4. **Version switching**: CNR version switch with file tracking
5. **Snapshots**: capture and restore full environment state
6. **ComfyUI updates**: git-based nightly/stable
7. **Model downloads**: to correct model directories
8. **Security**: malware scanning, security levels, path traversal prevention

### What's Different for Launcher

- Manager runs **inside** ComfyUI; Launcher runs **outside** as a separate Electron app
- Manager's lazy install relies on ComfyUI restart; Launcher can execute directly
- Manager uses GitPython; Launcher could use git CLI or libgit2
- Manager's `prestartup_script.py` hooks into ComfyUI's boot; Launcher would need its own boot sequence
- Manager stores state in ComfyUI's user directory; Launcher has its own data paths
- Launcher manages **multiple** ComfyUI installations, so node/pip operations need to target specific environments

### Critical Manager Dependencies

```
GitPython          # git operations (clone, pull, fetch, commit hash)
requests           # HTTP downloads
aiohttp            # async HTTP (CNR API, data fetching)
toml               # pyproject.toml parsing (CNR node identification)
PyGithub           # GitHub API (stats)
huggingface-hub    # HF model downloads
tqdm               # Progress bars
pyyaml             # YAML snapshot format
chardet            # Encoding detection for requirements files
uv                 # Alternative to pip (optional)
typer + rich       # CLI framework
```
