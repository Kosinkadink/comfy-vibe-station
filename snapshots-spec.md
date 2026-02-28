# ComfyUI-Launcher: Snapshots Specification

Consolidated spec for environment snapshots in ComfyUI-Launcher. Captures pip packages, custom nodes, and ComfyUI version at boot granularity, enabling history tracking and rollback.

---

## Concepts

### Soft Snapshot

A lightweight JSON file (~5 KB) capturing the full codewise state of an installation:

- **Pip packages**: Every installed Python package and its version (`uv pip freeze`)
- **Custom nodes**: Each node's identity, type (CNR/git/file), version or commit, and enabled/disabled state
- **ComfyUI version**: The `comfyui_ref` from `manifest.json` plus the actual git HEAD commit (these may differ after a commit-based update)

Soft snapshots are cheap and automatic. They are saved on every ComfyUI boot where the state differs from the last snapshot.

### Hard Snapshot

A full copy of the installation (~4.5 GB on Windows/NVIDIA). This is **not a new concept** — it is the existing Copy Installation feature. A hard snapshot is simply a copied installation with metadata linking it to the source installation's timeline.

Hard snapshots are installations. They appear in the installation list. Users "roll back" to a hard snapshot by launching that installation. No special rollback semantics, no deletion of "future" states — custom nodes store data inside their directories, so destructive rollback would lose user content.

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
| `trigger` | What caused this snapshot: `"boot"` (auto on ComfyUI start), `"manual"` (user-initiated), `"pre-update"` (auto before update) |
| `label` | Optional user-provided label (null for auto snapshots) |
| `comfyui.ref` | The `comfyui_ref` from `manifest.json` (e.g., a tag like `v0.3.10`) |
| `comfyui.commit` | Actual git HEAD commit hash of `ComfyUI/` (may differ from ref after commit-based updates) |
| `comfyui.releaseTag` | The Standalone release tag (e.g., `v0.2.1`) |
| `comfyui.variant` | The Standalone variant ID (e.g., `win-nvidia-cu128`) |
| `customNodes[]` | Array of all custom nodes (active + disabled) |
| `customNodes[].type` | `"cnr"` (registry), `"git"` (cloned repo), `"file"` (standalone .py) |
| `customNodes[].dirName` | Directory name in `custom_nodes/` (for CNR nodes, differs from `id`) |
| `pipPackages` | Map of package name → version string from `uv pip freeze` |

**Filename format:** `{timestamp}-{trigger}.json`
- Timestamp: `YYYYMMDD_HHmmss` (e.g., `20260226_120000`)
- Examples: `20260226_120000-boot.json`, `20260226_143000-pre-update.json`, `20260226_150000-manual.json`

### Installation Metadata Additions

In `installations.json`, each Standalone installation gains:

```json
{
  "lastSnapshot": "20260226_120000-boot",
  "snapshotCount": 12
}
```

`lastSnapshot` is the filename stem of the most recent snapshot. Used for quick change detection without reading the snapshot file — but the actual diff is done by comparing the captured state against the live environment, not by comparing snapshot filenames.

`snapshotCount` is a cached count for display purposes.

---

## Snapshot Capture

### When to Capture

A soft snapshot is captured **on every ComfyUI process start** — both cold launches from the Launcher and Manager-triggered restarts (detected via the `.reboot` marker in `ipc.ts:1352-1364`). The flow:

1. ComfyUI process starts (or restarts)
2. The Launcher captures the current environment state
3. Compares against the last saved snapshot
4. If anything changed → save a new snapshot
5. If nothing changed → skip

A ComfyUI **version change** also triggers a snapshot even if no pip packages or custom nodes changed (the plan explicitly requires this).

### Where to Hook

The snapshot capture runs **after** ComfyUI's process is spawned but doesn't block the UI — it runs in the background. Two hook points in `ipc.ts`:

1. **Cold launch** — after `tryLaunch()` succeeds and the session is registered (around line 1349, after `writePortLock`). The snapshot captures state before the user interacts with ComfyUI.

2. **Manager restart** — inside `attachExitHandler` when `checkRebootMarker(sessionPath)` is true (line 1354). After respawning, trigger another snapshot capture. This catches custom node installs/uninstalls that the Manager performed before requesting the restart.

Both hooks call the same `captureSnapshotIfChanged(installation)` function.

### Capture Cost

- `uv pip freeze`: ~1-2 seconds (fast, reads `.dist-info` metadata)
- Custom node scan: <0.5 seconds (filesystem reads only)
- Git HEAD read: <0.1 seconds (read `.git/HEAD` file)
- Total: ~2 seconds, runs async in the background, does not block UI or ComfyUI startup

### Auto-Pruning

Keep the last **50 auto snapshots** (`trigger: "boot"`) per installation. Manual and pre-update snapshots are never auto-pruned. At ~5 KB each, 50 auto snapshots = ~250 KB — negligible.

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

**Git operations** use direct file reads (`.git/HEAD`, `.git/refs/heads/`, `.git/config`) — no spawned processes, no git binary dependency:

```typescript
function readGitHead(nodePath: string): string | null {
  // Read .git/HEAD → "ref: refs/heads/main" → read refs/heads/main → sha
  // Or if detached: HEAD contains the sha directly
}

function readGitRemoteUrl(nodePath: string): string | null {
  // Parse .git/config → [remote "origin"] → url = ...
}
```

### 2. Pip Inspector — `lib/pip.ts`

Wraps `uv pip freeze` to capture the Python environment state.

```typescript
async function pipFreeze(
  uvPath: string,
  pythonPath: string
): Promise<Record<string, string>>
```

Returns a map of `packageName → version`. Parses `uv pip freeze` output (one `package==version` per line). Editable packages (installed via `-e`) are recorded with their URL/path as the version string.

### 3. Snapshot Manager — `lib/snapshots.ts`

Core snapshot operations:

```typescript
interface Snapshot {
  version: 1
  createdAt: string
  trigger: 'boot' | 'manual' | 'pre-update'
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
  trigger: 'boot' | 'manual' | 'pre-update'
): Promise<{ saved: boolean; filename?: string }>

// Save a snapshot unconditionally (used by manual save and pre-update)
async function saveSnapshot(
  installPath: string,
  installation: InstallationRecord,
  trigger: 'boot' | 'manual' | 'pre-update',
  label?: string
): Promise<string>  // returns filename

// List all snapshots for an installation, newest first
async function listSnapshots(installPath: string): Promise<Snapshot[]>

// Load a specific snapshot
async function loadSnapshot(installPath: string, filename: string): Promise<Snapshot>

// Delete a snapshot
async function deleteSnapshot(installPath: string, filename: string): Promise<void>

// Diff two snapshots (or a snapshot against current state)
function diffSnapshots(
  a: Snapshot,
  b: Snapshot
): SnapshotDiff

// Prune old auto snapshots beyond the retention limit
async function pruneAutoSnapshots(installPath: string, keep: number): Promise<number>
```

**Change detection** (`captureSnapshotIfChanged`):

1. Read the last snapshot file (from `installation.lastSnapshot`)
2. Capture current state: `scanCustomNodes()` + `pipFreeze()` + read `manifest.json` + read git HEAD
3. Compare:
   - `comfyui.commit` changed? → changed
   - `comfyui.ref` changed? → changed
   - Custom nodes list differs (added/removed/version changed/enabled changed)? → changed
   - Pip packages differ (added/removed/version changed)? → changed
4. If changed → save new snapshot, update `installation.lastSnapshot` and `snapshotCount`
5. If unchanged → return `{ saved: false }`

### 4. Snapshot Diff

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

After a successful launch (line ~1349):

```typescript
// After session is registered, capture snapshot in background
captureSnapshotIfChanged(inst.installPath, inst, 'boot')
  .then(({ saved, filename }) => {
    if (saved) {
      installations.update(installationId, {
        lastSnapshot: filename,
        snapshotCount: (inst.snapshotCount as number || 0) + 1,
      })
    }
  })
  .catch((err) => console.warn('Snapshot capture failed:', err))
```

Inside `attachExitHandler`, when a Manager restart is detected (line ~1354):

```typescript
if (checkRebootMarker(sessionPath)) {
  sendOutput('\n--- ComfyUI restarting ---\n\n')
  const spawned = spawnComfy()
  // ... existing restart logic ...

  // Capture snapshot after restart (Manager may have changed packages/nodes)
  const currentInst = await installations.get(installationId)
  if (currentInst) {
    captureSnapshotIfChanged(currentInst.installPath, currentInst, 'boot')
      .then(({ saved, filename }) => {
        if (saved) {
          installations.update(installationId, {
            lastSnapshot: filename,
            snapshotCount: (currentInst.snapshotCount as number || 0) + 1,
          })
        }
      })
      .catch((err) => console.warn('Snapshot capture failed:', err))
  }
  return
}
```

### In `sources/standalone.ts`

Before `update-comfyui` action executes (line ~605, after the git check):

```typescript
// Auto-snapshot before update
await saveSnapshot(installPath, installation, 'pre-update', 'before-update')
```

---

## Hard Snapshots

A hard snapshot is created via the existing Copy Installation flow (`performCopy` in `ipc.ts:139-194`). The only addition is metadata linking the copy to its source:

```json
{
  "copiedFrom": "inst-1234567890",
  "copiedAt": "2026-02-26T15:00:00.000Z",
  "copyReason": "manual" | "pre-update"
}
```

These fields are added to the new installation's record in `installations.json`. They enable the history view to show the lineage between installations.

No special "rollback to hard snapshot" mechanism. The user launches the old copy. The current installation and the copy are independent — the user can run either, delete either, or keep both.

---

## UI Integration

### Snapshot History Section (Detail View)

The detail modal now uses a **tabbed layout** (PR #36). Each section has an optional `tab` property (`'status'`, `'update'`, or `'settings'`) and the `DetailSection` interface includes `tab?: string`. Sections with the same `tab` value appear together; the tab bar is rendered by `DetailModal.vue` using `availableTabs` computed from the sections.

The snapshot history section should go on the **`'status'` tab**, since it describes the current and historical state of the installation. It sits after the Install Info section:

```typescript
{
  tab: 'status',
  title: t('standalone.snapshotHistory'),
  description: t('standalone.snapshotHistoryDesc', { count: snapshots.length }),
  collapsed: snapshots.length > 0,  // collapsed by default if there are snapshots
  items: snapshots.slice(0, 20).map(s => ({
    label: formatSnapshotLabel(s),  // "Boot · Feb 26, 2026 12:00 PM" or "Manual: my-label · ..."
    active: s.filename === installation.lastSnapshot,
    actions: [
      { id: 'snapshot-view', label: t('standalone.viewSnapshot'),
        data: { file: s.filename } },
      ...(s.trigger !== 'boot' || s.label ? [] : [
        { id: 'snapshot-delete', label: t('common.delete'), style: 'danger',
          data: { file: s.filename } },
      ]),
    ],
  })),
  actions: [
    { id: 'snapshot-save', label: t('standalone.saveSnapshot'),
      prompt: {
        title: t('standalone.saveSnapshotTitle'),
        message: t('standalone.saveSnapshotMessage'),
        placeholder: t('standalone.snapshotLabelPlaceholder'),
        field: 'label',
      } },
  ],
}
```

### Snapshot View (Future)

When the user clicks "View" on a snapshot, show a diff against the current state:

- ComfyUI version: `v0.3.10 (abc1234) → v0.3.12 (def5678)`
- Nodes added/removed/changed
- Pip packages added/removed/changed
- Total package count comparison

This is a read-only informational view. The implementation can use a modal or a dedicated page. Exact UI is deferred to implementation time.

### Snapshot Restore (Phase 2)

Restoring a soft snapshot means reverting the Python environment and custom nodes to the state captured in that snapshot. This is a multi-step operation that requires ComfyUI to be stopped.

#### Prerequisites

- **ComfyUI must not be running.** Any action that modifies the environment (restore, update, etc.) must check for a running ComfyUI process on that installation. If running, show a dialog: _"ComfyUI must be stopped to perform this action"_ with a **Stop** button that prompts to shut down the running instance.

#### Safety: Targeted Pre-Restore Backup

Before modifying the environment, create a temporary backup of only the packages that will change — their directories + `.dist-info/` folders + associated scripts in `Scripts/` (Windows) or `bin/` (Linux/macOS). Since the snapshot diff is already computed before restore begins, we know exactly which packages will be installed, downgraded, or removed.

This protects against partial restore failures — if the restore fails midway (e.g., network error, version unavailable), the backup can be used to revert to the pre-restore state. The backup is deleted on successful completion.

This is necessary because even an "undo" of a failed rollback could itself fail (e.g., network error), leaving the environment in a broken mixed state. A filesystem-level copy is the only reliable safety net.

**Note:** A targeted backup typically yields 50-200 MB (vs 1-3 GB for a full `site-packages/` copy). `uv`'s cache likely already has wheels for the pre-restore versions, but the cache can be pruned, so it is not a reliable safety net on its own.

#### Pip Package Restore

1. **Install missing + downgrade/upgrade changed**: `uv pip install package1==version1 package2==version2 ...` for all packages in the snapshot that differ from the current state.
2. **Remove extras**: Packages present in the current environment but absent from the snapshot should be uninstalled, **except** for protected packages. The protected set includes:
   - PyTorch ecosystem: `torch`, `torchvision`, `torchaudio`, `nvidia-*` (matches Manager's skip list)
   - Core tooling: `pip`, `setuptools`, `wheel`, `uv`
   - Transitive dependencies of protected packages (determined by checking if a package is required by any protected/remaining package)
3. **Failure handling**: Try bulk install first. On failure, fall back to one-by-one with `--no-deps` (matches Manager's approach). Track and report individual failures.

> **TODO (deferred):** The protected package set should be expanded beyond the hardcoded list above. Consider identifying protected packages by their index URL (PyTorch index) rather than by name. This would automatically cover new CUDA packages without maintaining a list.

#### Custom Node Restore

The Launcher should handle custom node restore at parity with what ComfyUI-Manager's `restore_snapshot` does today (see `manager_core.py:3114`). The Manager handles three node categories:

1. **CNR nodes** (registry-installed, identified by `.tracking` file):
   - **Missing from snapshot**: Disable (move to `.disabled/`)
   - **Version mismatch**: Switch version via CNR download + `.tracking` diff-based file cleanup
   - **Missing from current**: Install from registry at the snapshot's version
   - **Match**: Skip

2. **Git nodes** (cloned repos, identified by `.git/` directory):
   - **Missing from snapshot**: Disable
   - **Commit mismatch**: `git fetch` + `git checkout {commit}` to restore exact commit
   - **Missing from current**: Clone from remote URL, checkout to snapshot's commit
   - **Match**: Skip

3. **Standalone `.py` files**:
   - **Missing from snapshot**: Disable (move to `.disabled/`)
   - **Missing from current**: Report to user (cannot auto-restore a standalone file)
   - **Match**: Skip

4. **Post-install scripts**: After switching/installing nodes, run `requirements.txt` and `install.py` if present in the node directory (same as Manager does).

**Note:** If custom node restore scope becomes too large for a single phase, it can be split: Phase 2a (pip restore only) → Phase 2b (custom node restore). The safety infrastructure (stop check, backup, failure handling) would ship in Phase 2a and be reused in Phase 2b.

#### Integration with Update Rollback

The git-based update rollback (`update_support_plan.md`) handles **ComfyUI code** — it switches the git ref. Snapshot restore handles **dependencies and custom nodes**. These are complementary, not overlapping:

- A ComfyUI update should produce its own snapshot (already handled: `pre-update` trigger before, `boot` trigger after).
- Rolling back a ComfyUI update means: git rollback to restore code, then snapshot restore to restore the dependency/node state from the pre-update snapshot.
- There is no scenario where both operations conflict, because they operate on different layers.

---

## Integration with Update Flow

The update flow in `standalone.ts` (`update-comfyui` action, line 578) already captures pre/post state via `markers` (pre-update HEAD, backup branch, etc.). Snapshots integrate naturally:

1. **Before update**: `saveSnapshot(installPath, installation, 'pre-update')` — captures exact pip + node state
2. **After update**: The next ComfyUI boot auto-captures a `boot` snapshot with the new state
3. **History**: The user can compare the pre-update snapshot against the post-update boot snapshot to see exactly what changed

For `copy-update` (line 376): the copied installation inherits the source's snapshot history (since `.launcher/snapshots/` is inside the install directory and gets copied). The copy's first boot then creates a new snapshot showing the updated state.

---

## File Layout

```
{installPath}/
├── .launcher/
│   └── snapshots/
│       ├── 20260220_090000-boot.json
│       ├── 20260222_140000-boot.json
│       ├── 20260225_100000-pre-update.json
│       ├── 20260225_103000-boot.json
│       └── 20260226_120000-manual.json
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

### Phase 1: Capture & Compare (Initial Scope)

New files:
- `src/main/lib/nodes.ts` — Custom node scanner (`scanCustomNodes`, `readGitHead`, `readGitRemoteUrl`)
- `src/main/lib/pip.ts` — Pip inspector (`pipFreeze`)
- `src/main/lib/snapshots.ts` — Snapshot manager (`captureSnapshotIfChanged`, `saveSnapshot`, `listSnapshots`, `loadSnapshot`, `deleteSnapshot`, `diffSnapshots`, `pruneAutoSnapshots`)

Modified files:
- `src/main/lib/ipc.ts` — Add snapshot capture hooks at launch and restart
- `src/main/sources/standalone.ts` — Add snapshot history section (on `'status'` tab) to detail view; add pre-update snapshot to update action; add `snapshot-save`, `snapshot-view`, `snapshot-delete` action handlers
- `src/main/installations.ts` — No schema changes needed (uses existing `[key: string]: unknown` flexibility)
- `locales/en.json` — Add i18n keys for snapshot UI strings

### Phase 2: Pip Restore

- Add `snapshot-restore` action to `standalone.ts`
- Enforce "ComfyUI must be stopped" prerequisite — dialog with Stop button if running
- Implement `site-packages/` backup before restore, auto-delete on success, auto-revert on failure
- Implement pip restore in `lib/snapshots.ts`: install missing/changed packages, remove extras (with protected package set)
- Report restore results: successes, failures, skipped protected packages
- Add protected package list: `torch`, `torchvision`, `torchaudio`, `nvidia-*`, `pip`, `setuptools`, `wheel`, `uv`

### Phase 3: Custom Node Restore

- Extend `snapshot-restore` to handle custom node state (CNR version switch, git checkout, disable/enable, install missing)
- Run post-install scripts (`requirements.txt`, `install.py`) for switched/installed nodes
- Report node-level results: installed, switched, disabled, enabled, skipped, failed
- Handle standalone `.py` files: disable extras, report missing (cannot auto-restore)
- Reuse Phase 2 safety infrastructure (stop check, backup, failure handling)

### Phase 4: Hard Snapshot Lineage

- Add `copiedFrom` / `copiedAt` / `copyReason` metadata to `performCopy`
- Build a timeline view showing installation lineage (which install was copied from which)
- Consider a dedicated "Create Backup" action that wraps Copy Installation with `copyReason: 'backup'`

---

## Design Decisions

1. **Snapshot on every process start, not just cold launches.** Manager-triggered restarts are exactly when the environment changes (custom node install/uninstall). Capturing on every start means we never miss a change. The `.reboot` marker detection in `ipc.ts` provides the hook.

2. **Hard snapshots are installations, not a separate concept.** Aligns with `update_support_plan.md`'s principle: "Installations are the unit of visibility." No new abstraction, no "rollback to hard snapshot" semantics, no destructive deletion of future states.

3. **No dependency on ComfyUI-Manager's snapshot format.** The Launcher reads the same filesystem markers (`.tracking`, `.git/`, `pyproject.toml`) but maintains its own snapshot format. This avoids coupling to Manager's internal changes and works even if Manager is not installed.

4. **Git operations via file reads, not spawned processes.** Reading `.git/HEAD` and `.git/config` directly is fast, has no dependencies, and provides the limited info we need (commit hash, remote URL). Full git operations (checkout, fetch) are only needed for restore/update, not for capture.

5. **Restore is split across Phases 2 and 3.** Pip restore (Phase 2) ships first with safety infrastructure (stop check, `site-packages/` backup). Custom node restore (Phase 3) builds on the same infrastructure. This split keeps each phase focused and shippable independently.

7. **Environment-modifying operations require ComfyUI to be stopped.** Restore, update, and any action that modifies pip packages or custom nodes must verify ComfyUI is not running on that installation. On Windows especially, file locks on loaded `.pyd`/`.dll` files will cause pip operations to fail. A dialog prompts the user to stop the instance, with a button to do so.

8. **Targeted pre-restore backup.** Before modifying the environment, back up only the packages that will change — their directories + `.dist-info/` folders + associated scripts in `Scripts/` (Windows) or `bin/` (Linux/macOS). Since we already compute the snapshot diff, we know exactly which packages will be installed/downgraded/removed. This typically yields 50-200 MB instead of copying all of `site-packages/` (1-3 GB). Building wheels from installed packages is not a supported workflow in `pip` or `uv`, so a targeted filesystem copy is the most reliable lightweight option. The backup is deleted on success. Note: `uv`'s cache likely has wheels for the versions we're rolling back *to* (fast reinstall without network), but the cache can be pruned, so it's not a reliable safety net on its own.

6. **50 auto-snapshot retention limit.** Generous enough to cover weeks of daily use. At ~5 KB each, disk cost is negligible (~250 KB total). Manual and pre-update snapshots are never pruned — they're intentional and the user manages them.
