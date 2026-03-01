# ComfyUI-Launcher: Snapshots Specification

Consolidated spec for environment snapshots in ComfyUI-Launcher. Captures pip packages, custom nodes, and ComfyUI version at boot granularity, enabling history tracking and rollback.

**Status:** Phases 1–4 implemented.

---

## Concepts

### Soft Snapshot

A lightweight JSON file (~5 KB) capturing the full codewise state of an installation:

- **Pip packages**: Every installed Python package and its version (`uv pip freeze`)
- **Custom nodes**: Each node's identity, type (CNR/git/file), version or commit, and enabled/disabled state
- **ComfyUI version**: The `comfyui_ref` from `manifest.json` plus the actual git HEAD commit (these may differ after a commit-based update)

Soft snapshots are cheap and automatic. They are saved on every ComfyUI boot or Manager-triggered restart where the state differs from the last snapshot.

### Hard Snapshot

A full copy of the installation (~4.5 GB on Windows/NVIDIA). This is **not a new concept** — it is the existing Copy Installation feature. A hard snapshot is simply a copied installation with metadata linking it to the source installation's timeline.

Hard snapshots are installations. They appear in the installation list. Users "roll back" to a hard snapshot by launching that installation. No special rollback semantics, no deletion of "future" states — custom nodes store data inside their directories, so destructive rollback would lose user content.

When a copy is created (via Copy Install, Copy & Update, or Release Update), the new installation record stores lineage metadata: `copiedFrom` (source ID), `copiedFromName` (source name at time of copy), `copiedAt` (timestamp), and `copyReason` (`'copy'`, `'copy-update'`, or `'release-update'`). This metadata is displayed in the detail view's install info section.

---

## Data Model

### Soft Snapshot File

Stored per-installation in `{installPath}/.launcher/snapshots/`:

```json
{
  "version": 1,
  "createdAt": "2026-02-26T12:00:00.000Z",
  "trigger": "boot",
  "label": null,
  "comfyui": {
    "ref": "v0.3.10",
    "commit": "abc1234def5678",
    "releaseTag": "v0.2.1",
    "variant": "win-nvidia-cu128"
  },
  "customNodes": [
    {
      "id": "comfyui-impact-pack",
      "type": "cnr",
      "version": "7.3.0",
      "dirName": "ComfyUI-Impact-Pack",
      "enabled": true
    },
    {
      "id": "ComfyUI-KJNodes",
      "type": "git",
      "url": "https://github.com/kijai/ComfyUI-KJNodes",
      "commit": "abc1234",
      "dirName": "ComfyUI-KJNodes",
      "enabled": true
    },
    {
      "id": "standalone-script.py",
      "type": "file",
      "enabled": true
    }
  ],
  "pipPackages": {
    "torch": "2.6.0+cu128",
    "torchvision": "0.21.0+cu128",
    "comfyui-manager": "4.1b1",
    "numpy": "1.26.4"
  }
}
```

**Fields:**

| Field | Description |
|-------|-------------|
| `version` | Schema version (always `1` for now) |
| `createdAt` | ISO 8601 timestamp |
| `trigger` | What caused this snapshot: `"boot"` (cold launch), `"restart"` (Manager-triggered restart), `"manual"` (user-initiated), `"pre-update"` (auto before update) |
| `label` | Optional user-provided label (null for auto snapshots) |
| `comfyui.ref` | The `comfyui_ref` from `manifest.json` (e.g., a tag like `v0.3.10`) |
| `comfyui.commit` | Actual git HEAD commit hash of `ComfyUI/` (may differ from ref after commit-based updates) |
| `comfyui.releaseTag` | The Standalone release tag (e.g., `v0.2.1`) |
| `comfyui.variant` | The Standalone variant ID (e.g., `win-nvidia-cu128`) |
| `customNodes[]` | Array of all custom nodes (active + disabled) |
| `customNodes[].type` | `"cnr"` (registry), `"git"` (cloned repo), `"file"` (standalone .py) |
| `customNodes[].dirName` | Directory name in `custom_nodes/` (for CNR nodes, differs from `id`) |
| `pipPackages` | Map of package name → version string from `uv pip freeze` |

**Filename format:** `{timestamp}-{trigger}-{suffix}.json`
- Timestamp: `YYYYMMDD_HHmmss_mmm` (includes milliseconds for ordering precision)
- Suffix: 6-character random hex string (prevents collisions from rapid captures)
- Examples: `20260226_120000_123-boot-a1b2c3.json`, `20260226_143000_456-pre-update-d4e5f6.json`

### Installation Metadata Additions

In `installations.json`, each Standalone installation gains:

```json
{
  "lastSnapshot": "20260226_120000_123-boot-a1b2c3",
  "snapshotCount": 12
}
```

`lastSnapshot` is the filename of the most recent snapshot. Used for quick change detection without reading the snapshot file — but the actual diff is done by comparing the captured state against the live environment, not by comparing snapshot filenames.

`snapshotCount` is a cached count for display purposes, recomputed from disk when snapshots are added or removed.

---

## Snapshot Capture

### When to Capture

A soft snapshot is captured **on every ComfyUI process start** — both cold launches from the Launcher and Manager-triggered restarts (detected via the `.reboot` marker in `ipc.ts`). The flow:

1. ComfyUI process starts (or restarts)
2. The Launcher captures the current environment state
3. For `boot` triggers: compares against the last saved snapshot; if unchanged, skips
4. For `restart` triggers: always saves (Manager may have changed state), then deduplicates intermediate restarts
5. For `manual`/`pre-update` triggers: always saves unconditionally

A ComfyUI **version change** also triggers a snapshot even if no pip packages or custom nodes changed (the plan explicitly requires this).

### Restart Deduplication

When Manager installs a custom node, it triggers multiple restarts — one for the node install, then another after pip packages are installed. This produces intermediate snapshots that capture a partial state (node present, but pip deps not yet installed). The `deduplicateRestartSnapshot` function detects this pattern:

- If two consecutive `restart` snapshots have identical ComfyUI version and custom nodes but differ only in pip packages, the older one is removed
- Only unlabeled restart snapshots are candidates for deduplication
- This keeps the snapshot history clean without losing meaningful state transitions

### Where to Hook

The snapshot capture runs **after** ComfyUI's process is spawned but doesn't block the UI — it runs in the background. Two hook points in `ipc.ts`:

1. **Cold launch** — after `tryLaunch()` succeeds and the session is registered. The snapshot captures state before the user interacts with ComfyUI.

2. **Manager restart** — inside `attachExitHandler` when `checkRebootMarker(sessionPath)` is true. After respawning, trigger another snapshot capture. This catches custom node installs/uninstalls that the Manager performed before requesting the restart.

Both hooks call the same `captureSnapshotIfChanged(installation)` function.

### Initial Snapshot

An initial `boot` snapshot is also captured during `postInstall` (after environment setup completes). This ensures the detail view shows a "Current" snapshot immediately after installation, before the user ever launches ComfyUI.

### Capture Cost

- `uv pip freeze`: ~1-2 seconds (fast, reads `.dist-info` metadata)
- Custom node scan: <0.5 seconds (filesystem reads only)
- Git HEAD read: <0.1 seconds (read `.git/HEAD` file)
- Total: ~2 seconds, runs async in the background, does not block UI or ComfyUI startup

### Auto-Pruning

Keep the last **50 auto snapshots** (`trigger: "boot"` or `"restart"`) per installation. Manual and pre-update snapshots are never auto-pruned. At ~5 KB each, 50 auto snapshots = ~250 KB — negligible.

---

## Implementation Components

### 1. Custom Node Scanner — `lib/nodes.ts`

Scans `custom_nodes/` and `custom_nodes/.disabled/` to build a typed list of installed nodes. Reads the same metadata files that ComfyUI-Manager uses, but directly from the filesystem (no Python dependency).

```typescript
interface ScannedNode {
  id: string              // CNR package name, git dir name, or filename
  type: 'cnr' | 'git' | 'file'
  dirName: string         // Directory name in custom_nodes/
  enabled: boolean        // true if in custom_nodes/, false if in .disabled/
  version?: string        // CNR version from pyproject.toml
  commit?: string         // Git HEAD commit hash
  url?: string            // Git remote origin URL
}

async function scanCustomNodes(comfyuiDir: string): Promise<ScannedNode[]>
```

**Node identification logic:**

| Marker | Type | ID source | Version source |
|--------|------|-----------|----------------|
| `.tracking` file exists | `cnr` | `pyproject.toml` → `project.name` | `pyproject.toml` → `project.version` |
| `.git/` directory exists | `git` | Directory name | `.git/HEAD` → resolve to commit hash |
| Neither (directory) | `git` (fallback) | Directory name | None |
| `.py` file (not directory) | `file` | Filename | None |

**Stable node key:** `nodeKey(node)` returns `${type}:${dirName}`, used for snapshot comparisons and diffs. If a node changes type (e.g., was git, now CNR with same dirName), they produce different keys and are treated as a remove + add — this is correct behavior since the install mechanism changed.

### 2. Git Operations — `lib/git.ts`

**Read-only operations** (used for capture — no git binary needed):

```typescript
function resolveGitDir(repoPath: string): string | null    // Handles worktrees/submodules
function readGitHead(repoPath: string): string | null       // Resolves HEAD → commit sha
function readGitRemoteUrl(repoPath: string): string | null  // Parses .git/config for origin URL
function hasGitDir(nodePath: string): boolean               // Quick existence check
```

Git remote URLs are redacted via `redactUrl()` to strip embedded credentials before storing in snapshots. Symbolic ref resolution includes a path-traversal guard (ref path must stay within `.git/`).

**Write operations** (used for restore — requires git binary in PATH):

```typescript
function isGitAvailable(): Promise<boolean>                              // Checks git --version
function gitClone(url, dest, sendOutput): Promise<number>                // git clone
function gitFetchAndCheckout(repoPath, commit, sendOutput): Promise<number>  // git fetch origin && git checkout
```

### 3. Pip Inspector — `lib/pip.ts`

Wraps `uv pip freeze` to capture the Python environment state.

```typescript
async function pipFreeze(
  uvPath: string,
  pythonPath: string
): Promise<Record<string, string>>
```

Returns a map of `packageName → version`. Parses `uv pip freeze` output (one `package==version` per line). Handles three output formats:
- Standard: `package==version` → `{ package: version }`
- Editable: `-e git+https://...@commit#egg=name` → `{ name: "-e ..." }`
- Direct reference (PEP 508): `package @ https://...` → `{ package: "https://..." }`

### 4. CNR Registry Client — `lib/cnr.ts`

Native implementation of the ComfyUI Node Registry install protocol. Does not depend on ComfyUI-Manager — works even if Manager is not installed.

```typescript
function getCnrInstallInfo(nodeId, version?): Promise<CnrInstallInfo | null>
function installCnrNode(nodeId, version, customNodesDir, sendOutput): Promise<string[]>
function switchCnrVersion(nodeId, newVersion, nodePath, sendOutput): Promise<string[]>
```

**CNR install protocol:**
1. `GET https://api.comfy.org/nodes/{id}/install?version={v}` → get `downloadUrl`
2. Download zip to temp file
3. Extract into `custom_nodes/{nodeId}/`
4. Write `.tracking` file listing all extracted files (one relative path per line)
5. Clean up temp file

**CNR version switch protocol:**
1. Read old `.tracking` file to get the old file list
2. Download and extract new version to a **temp directory** (not directly into the node path — this is critical for accurate garbage detection)
3. Walk the temp directory to get the true new file list
4. Copy extracted files into the node path (overwriting existing files)
5. Diff old vs new file lists → identify garbage files (in old but not in new)
6. Delete garbage files and their empty parent directories
7. Write new `.tracking` file

The temp-directory extraction step is essential: extracting directly into the node path and then walking it would return the union of old+new files, making garbage detection impossible.

**Security:** `nodeId` is validated via `isSafePathComponent()` before any filesystem or network operations. API URL parameters are URI-encoded.

**`.tracking` compatibility:** Our implementation walks the extracted directory to build the file list (excluding the `.tracking` file itself). Manager uses `zipfile.namelist()`. Both produce equivalent results — our approach excludes directory-only entries, but Manager's garbage collection only removes files and empty dirs anyway.

### 5. Snapshot Manager — `lib/snapshots.ts`

Core snapshot operations:

```typescript
interface Snapshot {
  version: 1
  createdAt: string
  trigger: 'boot' | 'restart' | 'manual' | 'pre-update'
  label: string | null
  comfyui: {
    ref: string
    commit: string | null
    releaseTag: string
    variant: string
  }
  customNodes: ScannedNode[]
  pipPackages: Record<string, string>
}

// Capture current state and save if changed from last snapshot
async function captureSnapshotIfChanged(
  installPath: string,
  installation: InstallationRecord,
  trigger: 'boot' | 'restart' | 'manual' | 'pre-update'
): Promise<{ saved: boolean; filename?: string; deduplicated?: string }>

// Save a snapshot unconditionally (used by manual save and pre-update)
async function saveSnapshot(
  installPath: string,
  installation: InstallationRecord,
  trigger: 'boot' | 'restart' | 'manual' | 'pre-update',
  label?: string
): Promise<string>  // returns filename

// List all snapshots for an installation, newest first
async function listSnapshots(installPath: string): Promise<SnapshotEntry[]>

// Load a specific snapshot
async function loadSnapshot(installPath: string, filename: string): Promise<Snapshot>

// Delete a snapshot
async function deleteSnapshot(installPath: string, filename: string): Promise<void>

// Diff two snapshots (or a snapshot against current state)
function diffSnapshots(a: Snapshot, b: Snapshot): SnapshotDiff

// Prune old auto snapshots beyond the retention limit
async function pruneAutoSnapshots(installPath: string, keep: number): Promise<number>
```

**Concurrency:** All snapshot CRUD operations are protected by a per-install mutex (`withLock`). Snapshot file writes use atomic rename (write to `.tmp`, then rename).

**Change detection** (`captureSnapshotIfChanged`):

1. Read the last snapshot file (from `installation.lastSnapshot`)
2. Capture current state: `scanCustomNodes()` + `pipFreeze()` + read `manifest.json` + read git HEAD
3. For `boot` triggers only: compare against last snapshot. If unchanged, return `{ saved: false }`
4. For `restart` triggers: always save, then attempt deduplication of intermediate restart snapshots
5. Save new snapshot, update `installation.lastSnapshot` and `snapshotCount`

### 6. Snapshot Diff

```typescript
interface SnapshotDiff {
  comfyuiChanged: boolean
  comfyui?: {
    from: { ref: string; commit: string | null }
    to: { ref: string; commit: string | null }
  }
  nodesAdded: ScannedNode[]
  nodesRemoved: ScannedNode[]
  nodesChanged: Array<{
    id: string
    type: string
    from: { version?: string; commit?: string; enabled: boolean }
    to: { version?: string; commit?: string; enabled: boolean }
  }>
  pipsAdded: Array<{ name: string; version: string }>
  pipsRemoved: Array<{ name: string; version: string }>
  pipsChanged: Array<{ name: string; from: string; to: string }>
}
```

---

## Hook Integration

### In `lib/ipc.ts`

After a successful launch:

```typescript
// After session is registered, capture snapshot in background
if (inst.sourceId === 'standalone') {
  captureSnapshotIfChanged(inst.installPath, inst, 'boot')
    .then(async ({ saved, filename }) => {
      if (saved) {
        const snapshotCount = await getSnapshotCount(inst.installPath)
        installations.update(installationId, { lastSnapshot: filename, snapshotCount })
      }
    })
    .catch((err) => console.warn('Snapshot capture failed:', err))
}
```

Inside `attachExitHandler`, when a Manager restart is detected:

```typescript
// Capture snapshot after Manager-triggered restart
if (inst.sourceId === 'standalone') {
  installations.get(installationId).then((currentInst) => {
    if (!currentInst) return
    captureSnapshotIfChanged(currentInst.installPath, currentInst, 'restart')
      .then(async ({ saved, filename }) => {
        if (saved) {
          const snapshotCount = await getSnapshotCount(currentInst.installPath)
          installations.update(installationId, { lastSnapshot: filename, snapshotCount })
        }
      })
      .catch((err) => console.warn('Snapshot capture failed:', err))
  })
}
```

**Running-session guard:** The `snapshot-restore` action is blocked at the IPC level if ComfyUI is running on that installation (`_runningSessions.has(installationId)`). This returns `snapshotRestoreStopRequired` without reaching the handler.

### In `sources/standalone.ts`

**Initial snapshot** — captured during `postInstall`, after the default environment is created:

```typescript
const filename = await snapshots.saveSnapshot(installation.installPath, installation, 'boot')
const snapshotCount = await snapshots.getSnapshotCount(installation.installPath)
await update({ lastSnapshot: filename, snapshotCount })
```

**Pre-update snapshot** — captured inside the `update-comfyui` handler before any changes:

```typescript
await snapshots.saveSnapshot(installPath, installation, 'pre-update', 'before-update')
```

---

## Snapshot Restore

Restoring a soft snapshot reverts the Python environment and custom nodes to the state captured in that snapshot. This is a two-phase operation that requires ComfyUI to be stopped.

### Prerequisites

- **ComfyUI must not be running.** Enforced at the IPC level in `ipc.ts` — the `snapshot-restore` action checks `_runningSessions.has(installationId)` and returns an error if true.

### Phase Ordering: Nodes First, Then Pip

The restore handler in `standalone.ts` runs two phases in sequence:

1. **Restore custom nodes** (`restoreCustomNodes`) — installs, switches versions, enables/disables nodes. Node installs may add pip dependencies via `requirements.txt`.
2. **Restore pip packages** (`restorePipPackages`) — syncs the Python environment to the exact target state, adding missing packages and removing extras.

This ordering matches ComfyUI-Manager's approach and prevents a scenario where pip restore removes packages that a subsequent node install would immediately re-add.

### Custom Node Restore (Phase 3)

Handles all three node types:

#### CNR nodes (registry-installed, identified by `.tracking` file)

| Current state | Target state | Action |
|---|---|---|
| Missing | In snapshot | Install from registry via `installCnrNode` |
| Version mismatch | In snapshot | Version switch via `switchCnrVersion` |
| Enabled | Disabled in snapshot | Move to `.disabled/` |
| Disabled | Enabled in snapshot | Move from `.disabled/` to `custom_nodes/` |
| Match | Match | Skip |

#### Git nodes (cloned repos, identified by `.git/` directory)

| Current state | Target state | Action |
|---|---|---|
| Missing | In snapshot (has URL) | `git clone {url}`, then `git checkout {commit}` |
| Commit mismatch | In snapshot | `git fetch origin`, then `git checkout {commit}` |
| Enabled | Disabled in snapshot | Move to `.disabled/` |
| Disabled | Enabled in snapshot | Move from `.disabled/` to `custom_nodes/` |
| Match | Match | Skip |

Git operations require the `git` binary in PATH. Availability is checked once upfront via `isGitAvailable()`. If unavailable, all git node operations are skipped with a clear message listing which nodes were affected.

#### Standalone `.py` files

| Current state | Target state | Action |
|---|---|---|
| Missing | In snapshot | Report to user (cannot auto-restore) |
| Present, not in snapshot | — | Delete from filesystem |
| Match | Match | Skip |

#### Post-install scripts

After any node install or version switch, run post-install scripts:
1. If `requirements.txt` exists: `uv pip install -r requirements.txt` (filtering out `torch`/`torchvision`/`torchaudio`/`torchsde` lines to prevent PyTorch breakage)
2. If `install.py` exists: `{python} -s install.py` in the node directory

Post-install failures are non-fatal — reported but do not block the restore.

#### Safety measures

- **ComfyUI-Manager is never modified.** Any node whose `id` contains `comfyui-manager` (case-insensitive) is skipped.
- **Path traversal protection.** `nodeId` and `dirName` from snapshot data are validated via `isSafePathComponent()` before use in filesystem operations.
- **Extra nodes are deleted, not disabled.** Nodes not present in the target snapshot are fully removed from the filesystem rather than moved to `.disabled/`. Disabling would leave them visible in Manager's UI, and re-enabling them would fail because the pip restore phase removes their dependencies. Deletion gives a clean state.
- **No backup/revert for node operations.** Unlike pip (which has a targeted backup), each node operation is individually atomic — install, switch, enable, or disable either succeeds or fails independently. Failed operations are tracked in `NodeRestoreResult.failed`.
- **Enable/disable collision handling.** Both `disableNode` and `enableNode` remove any pre-existing destination directory before renaming, preventing `ENOTEMPTY` errors from partial prior restores.

### Pip Package Restore (Phase 2)

#### Safety: Targeted Pre-Restore Backup

Before modifying the environment, create a temporary backup of only the packages that will change — their directories + `.dist-info/` folders + associated files identified from RECORD. Since the snapshot diff is already computed, we know exactly which packages will be installed, downgraded, or removed.

This protects against partial restore failures — if the restore fails midway (e.g., network error, version unavailable), the backup is used to revert to the pre-restore state. The backup is deleted on successful completion.

**Note:** A targeted backup typically yields 50-200 MB (vs 1-3 GB for a full `site-packages/` copy). `uv`'s cache likely already has wheels for the pre-restore versions, but the cache can be pruned, so it is not a reliable safety net on its own.

#### Restore Flow

1. **Capture current pip state** via `pipFreeze`
2. **Compute diff** against the target snapshot's `pipPackages`
3. **Create targeted backup** of packages that will be modified or removed
4. **Install missing + upgrade/downgrade changed**: `uv pip install package1==version1 package2==version2 ...` (bulk first, one-by-one with `--no-deps` as fallback)
5. **Remove extras**: Packages present in current environment but absent from snapshot, **except** protected packages
6. **On failure**: Revert from backup, uninstall any newly-added packages, report errors
7. **On success**: Delete backup

#### Protected Packages

Packages that are never modified during restore:
- **Exact matches:** `pip`, `setuptools`, `wheel`, `uv`
- **Prefix matches:** `torch*`, `nvidia*`, `triton*`, `cuda*`

> **TODO (deferred):** Expand protection via index-URL detection (packages from PyTorch index) and transitive dependency analysis (packages required by protected packages). See `snapshots-phase3-plan.md` § Deferred Item.

### Restore Results

After both phases complete, the handler builds a combined summary and:
- Saves a new `manual` snapshot labeled `after-restore` capturing the restored state
- Reports total successes and failures
- If pip restore had failures: the entire pip phase was reverted from backup, reported as `snapshotRestoreReverted`
- If only node operations failed: pip restore still ran (node failures are non-blocking)

---

## UI Integration

### Snapshot History Section (Detail View)

The snapshot history appears on the **`'status'` tab** of the detail modal. It shows the 20 most recent snapshots with the newest marked as "★ Current".

Each snapshot (except the current one) has two actions:
- **Restore** — restores the installation to that snapshot's state (with confirmation dialog)
- **Delete** — removes the snapshot file

A **Save Snapshot** button allows manual snapshot creation with an optional label.

### Snapshot View (Future)

When the user clicks "View" on a snapshot, show a diff against the current state:

- ComfyUI version: `v0.3.10 (abc1234) → v0.3.12 (def5678)`
- Nodes added/removed/changed
- Pip packages added/removed/changed
- Total package count comparison

This is a read-only informational view. Not yet implemented. The data infrastructure (`diffSnapshots`) is ready.

---

## Integration with Update Flow

The update flow in `standalone.ts` (`update-comfyui` action) integrates with snapshots:

1. **Before update**: `saveSnapshot(installPath, installation, 'pre-update', 'before-update')` — captures exact pip + node state
2. **After update**: The next ComfyUI boot auto-captures a `boot` snapshot with the new state
3. **History**: The user can compare the pre-update snapshot against the post-update boot snapshot to see exactly what changed
4. **Rollback**: Rolling back a ComfyUI update means: git rollback to restore code, then snapshot restore to restore the dependency/node state from the pre-update snapshot

For `copy-update`: the copied installation inherits the source's snapshot history (since `.launcher/snapshots/` is inside the install directory and gets copied). The copy's first boot then creates a new snapshot showing the updated state.

---

## File Layout

```
{installPath}/
├── .launcher/
│   └── snapshots/
│       ├── 20260220_090000_123-boot-a1b2c3.json
│       ├── 20260222_140000_456-restart-d4e5f6.json
│       ├── 20260225_100000_789-pre-update-a7b8c9.json
│       ├── 20260225_103000_012-boot-d0e1f2.json
│       └── 20260226_120000_345-manual-a3b4c5.json
├── standalone-env/
├── envs/
│   └── default/
├── ComfyUI/
│   ├── main.py
│   ├── custom_nodes/
│   │   ├── ComfyUI-Impact-Pack/
│   │   │   ├── .tracking
│   │   │   └── pyproject.toml
│   │   ├── ComfyUI-KJNodes/
│   │   │   └── .git/
│   │   └── .disabled/
│   │       └── SomeNode@1.0.0/
│   └── models/
└── manifest.json
```

---

## Implementation Plan

### Phase 1: Capture & Compare ✅

New files:
- `src/main/lib/nodes.ts` — Custom node scanner (`scanCustomNodes`, `nodeKey`, `identifyNode`)
- `src/main/lib/pip.ts` — Pip inspector (`pipFreeze`)
- `src/main/lib/snapshots.ts` — Snapshot manager (`captureSnapshotIfChanged`, `saveSnapshot`, `listSnapshots`, `loadSnapshot`, `deleteSnapshot`, `diffSnapshots`, `pruneAutoSnapshots`, `deduplicateRestartSnapshot`)

Modified files:
- `src/main/lib/git.ts` — Added `resolveGitDir`, `readGitHead`, `readGitRemoteUrl`, `hasGitDir`, `redactUrl`
- `src/main/lib/ipc.ts` — Added snapshot capture hooks at launch and restart, running-session guard for restore
- `src/main/sources/standalone.ts` — Added snapshot history section (on `'status'` tab), `snapshot-save`/`snapshot-delete` action handlers, initial snapshot in `postInstall`, pre-update snapshot in `update-comfyui`
- `locales/en.json` — Added i18n keys for snapshot UI strings

### Phase 2: Pip Restore ✅

Added to `src/main/lib/snapshots.ts`:
- `restorePipPackages` — full pip restore with bulk/fallback install strategy
- `isProtectedPackage` — hardcoded prefix/exact matching for torch/nvidia/pip/etc.
- `createTargetedBackup` / `restoreFromBackup` — filesystem-level backup and revert
- `findDistInfoDir` / `findPackageEntries` / `normalizeDistInfoName` — package-to-filesystem mapping via RECORD files
- `runUvPip` — spawns uv pip commands with output streaming

Added to `src/main/sources/standalone.ts`:
- `snapshot-restore` action handler

### Phase 3: Custom Node Restore ✅

New files:
- `src/main/lib/cnr.ts` — CNR registry client (`getCnrInstallInfo`, `installCnrNode`, `switchCnrVersion`, `walkDir`, `isSafePathComponent`)

Added to `src/main/lib/git.ts`:
- `isGitAvailable`, `gitClone`, `gitFetchAndCheckout`

Added to `src/main/lib/snapshots.ts`:
- `restoreCustomNodes` — orchestrates all node restore operations
- `isManagerNode`, `disableNode`, `enableNode`, `runPostInstallScripts`
- `NodeRestoreResult` interface

Updated in `src/main/sources/standalone.ts`:
- `snapshot-restore` handler now runs two phases (nodes first, then pip) with combined summary

### Phase 4: Hard Snapshot Lineage ✅

Modified in `src/main/lib/ipc.ts`:
- `performCopy` now accepts `copyReason` parameter and stamps lineage metadata (`copiedFrom`, `copiedFromName`, `copiedAt`, `copyReason`) on the new installation record
- Old lineage fields are explicitly stripped from inherited properties to prevent propagation through copy chains
- `copy` action uses `copyReason: 'copy'`, `copy-update` uses `'copy-update'`, `release-update` stamps `'release-update'`

Modified in `src/main/sources/standalone.ts`:
- Detail view shows a "Copied From" field in install info when `copiedFrom` is set, with reason-specific labels (Copied from / Copy & Update from / Release Update from) and the date

Modified in `locales/en.json`:
- Added i18n keys for lineage display (`lineage`, `lineageCopy`, `lineageCopyUpdate`, `lineageReleaseUpdate`)

---

## Design Decisions

1. **Snapshot on every process start, not just cold launches.** Manager-triggered restarts are exactly when the environment changes (custom node install/uninstall). Capturing on every start means we never miss a change. The `.reboot` marker detection in `ipc.ts` provides the hook.

2. **Hard snapshots are installations, not a separate concept.** Aligns with `update_support_plan.md`'s principle: "Installations are the unit of visibility." No new abstraction, no "rollback to hard snapshot" semantics, no destructive deletion of future states.

3. **No dependency on ComfyUI-Manager's snapshot format.** The Launcher reads the same filesystem markers (`.tracking`, `.git/`, `pyproject.toml`) but maintains its own snapshot format. This avoids coupling to Manager's internal changes and works even if Manager is not installed.

4. **Git operations split by phase.** Capture (Phase 1) uses direct file reads (`.git/HEAD`, `.git/config`) — fast, no dependencies. Restore (Phase 3) spawns the `git` binary for clone/fetch/checkout — these require the git CLI, which is checked upfront and fails gracefully if unavailable.

5. **Restore split across Phases 2 and 3.** Pip restore (Phase 2) shipped first with safety infrastructure (stop check, `site-packages/` backup). Custom node restore (Phase 3) builds on the same infrastructure. This split kept each phase focused and shippable independently.

6. **Environment-modifying operations require ComfyUI to be stopped.** Enforced at the IPC level for `snapshot-restore`. On Windows especially, file locks on loaded `.pyd`/`.dll` files will cause pip operations to fail.

7. **Targeted pre-restore backup.** Back up only the packages that will change. This typically yields 50-200 MB instead of copying all of `site-packages/` (1-3 GB). The backup is deleted on success.

8. **50 auto-snapshot retention limit.** Generous enough to cover weeks of daily use. At ~5 KB each, disk cost is negligible (~250 KB total). Manual and pre-update snapshots are never pruned — they're intentional and the user manages them.

9. **Restart deduplication.** Manager install sequences produce multiple restarts with intermediate states. Deduplicating consecutive restart snapshots with identical node state keeps the history clean.

10. **Nodes restored before pip.** Node installs may add pip dependencies via `requirements.txt`. Restoring nodes first, then syncing pip state, prevents removing packages that would be immediately re-added.
