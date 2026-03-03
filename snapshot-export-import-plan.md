# Snapshot Export/Import Plan

## Goal

Allow users to export a single snapshot (or all snapshots for an installation) to a portable JSON file, import snapshot files into an existing installation, and create a brand-new installation pre-configured from a snapshot file.

---

## Export Format

A single envelope format covers both single-snapshot and all-snapshots export:

```json
{
  "type": "comfyui-launcher-snapshot",
  "version": 1,
  "exportedAt": "2026-03-02T12:00:00.000Z",
  "installationName": "My GPU Install",
  "snapshots": [
    {
      "version": 1,
      "createdAt": "2026-02-26T12:00:00.000Z",
      "trigger": "boot",
      "label": null,
      "comfyui": { "ref": "v0.3.10", "commit": "abc1234", "releaseTag": "v0.2.1", "variant": "win-nvidia-cu128" },
      "customNodes": [ ... ],
      "pipPackages": { ... }
    }
  ]
}
```

**Fields:**

| Field | Description |
|---|---|
| `type` | Magic string for validation (`"comfyui-launcher-snapshot"`) |
| `version` | Envelope schema version (always `1`) |
| `exportedAt` | ISO 8601 timestamp of the export |
| `installationName` | Source installation name (informational only) |
| `snapshots` | Array of `Snapshot` objects (same schema as the internal `.json` files) |

For **single-snapshot** export, the `snapshots` array has one element. For **all-snapshots** export, it contains every snapshot in the installation, sorted newest-first.

The file extension is `.json`. Default filename:
- Single: `snapshot-{installName}-{trigger}-{date}.json` (e.g., `snapshot-MyInstall-boot-20260226.json`)
- All: `snapshots-{installName}-{date}.json` (e.g., `snapshots-MyInstall-20260302.json`)

---

## Phase 1: Export & Import into Existing Installation

### Import Logic

1. User picks a `.json` file via the OS open dialog
2. Read and parse the file
3. Validate: check `type === "comfyui-launcher-snapshot"` and `version === 1` and `snapshots` is a non-empty array
4. For each snapshot in the array, validate it has the required fields (`version`, `createdAt`, `trigger`, `comfyui`, `customNodes`, `pipPackages`)
5. Write each snapshot as a new file in `{installPath}/.launcher/snapshots/` using the standard filename format (`{timestamp}-{trigger}-{suffix}.json`). Use the **original** `createdAt` timestamp for the filename to preserve chronological ordering.
6. Deduplicate: before writing, check if a snapshot with the same `createdAt` and `trigger` already exists. If so, skip it.
7. Update `snapshotCount` and `lastSnapshot` on the installation record.
8. Return the count of imported snapshots.

**Import does NOT restore the environment.** It only adds the snapshot files to the history. The user can then use the existing "Restore" button on any imported snapshot to apply it.

### Implementation

#### Layer 1: Backend — `src/main/lib/snapshots.ts`

Add two new exported functions:

```typescript
export interface SnapshotExportEnvelope {
  type: 'comfyui-launcher-snapshot'
  version: 1
  exportedAt: string
  installationName: string
  snapshots: Snapshot[]
}

export function buildExportEnvelope(
  installationName: string,
  snapshotEntries: SnapshotEntry[]
): SnapshotExportEnvelope

export async function importSnapshots(
  installPath: string,
  envelope: SnapshotExportEnvelope
): Promise<{ imported: number; skipped: number }>
```

`buildExportEnvelope` is pure — it just wraps the snapshot data. The caller (IPC handler) handles the file dialog and disk write.

`importSnapshots` validates each snapshot, checks for duplicates (matching `createdAt` + `trigger`), writes new files, and returns counts.

#### Layer 2: IPC — `src/main/lib/ipc.ts`

Three new IPC handlers:

```typescript
ipcMain.handle('export-snapshot', async (_event, installationId: string, filename: string) => { ... })
ipcMain.handle('export-all-snapshots', async (_event, installationId: string) => { ... })
ipcMain.handle('import-snapshots', async (_event, installationId: string) => { ... })
```

**`export-snapshot`:** Load snapshot → build single-item envelope → `dialog.showSaveDialog` → write JSON.

**`export-all-snapshots`:** `listSnapshots` → build full envelope → save dialog → write JSON.

**`import-snapshots`:** `dialog.showOpenDialog` → read file → validate envelope → `importSnapshots()` → update installation record.

#### Layer 3: Preload — `src/preload/index.ts`

Expose three new API methods:

```typescript
exportSnapshot: (installationId, filename) =>
  ipcRenderer.invoke('export-snapshot', installationId, filename),
exportAllSnapshots: (installationId) =>
  ipcRenderer.invoke('export-all-snapshots', installationId),
importSnapshots: (installationId) =>
  ipcRenderer.invoke('import-snapshots', installationId),
```

#### Layer 4: Types — `src/types/ipc.ts`

Add to `ElectronApi`:

```typescript
exportSnapshot(installationId: string, filename: string): Promise<{ ok: boolean; message?: string }>
exportAllSnapshots(installationId: string): Promise<{ ok: boolean; message?: string }>
importSnapshots(installationId: string): Promise<{ ok: boolean; imported?: number; skipped?: number; message?: string }>
```

#### Layer 5: UI — `src/renderer/src/components/SnapshotTab.vue`

**Header buttons** (next to "Save Snapshot"):
- **Import** — calls `window.api.importSnapshots(installationId)`, reloads list on success, shows result alert.
- **Export All** — calls `window.api.exportAllSnapshots(installationId)`, shown only when snapshots exist.

**Inspector button** (when a snapshot is selected):
- **Export** — calls `window.api.exportSnapshot(installationId, filename)`. Appears alongside Restore/Delete.

#### Layer 6: i18n — `locales/en.json`

Add keys under `"snapshots"`:

```json
"exportSnapshot": "Export",
"exportAllSnapshots": "Export All",
"importSnapshots": "Import",
"importSuccess": "Imported {imported} snapshot(s), skipped {skipped} duplicate(s).",
"importInvalidFile": "Invalid snapshot file. Expected a ComfyUI Launcher snapshot export.",
"importNoSnapshots": "The file contains no snapshots.",
"exportSuccess": "Snapshot exported successfully."
```

---

## Phase 2: New Installation from Snapshot

### Overview

Create a brand-new standalone installation and automatically restore it to match a snapshot's state. The flow:

1. User clicks "New from Snapshot" (in the install list toolbar)
2. File picker opens → user selects a snapshot export file
3. Launcher reads the envelope, takes the **newest** snapshot (first in array)
4. Extracts `comfyui.variant` from the snapshot → maps to current platform's variant
5. Fetches the **latest** release that has this variant available (the snapshot's exact `releaseTag` may be old)
6. Creates the installation record with `pendingSnapshotRestore` set to the file path
7. Starts the normal install flow (download → extract → setup env)
8. After `postInstall` completes, the `install-instance` handler detects `pendingSnapshotRestore` and chains into snapshot restore (import snapshot + run `restoreCustomNodes` + `restorePipPackages`)
9. Progress UI shows the extra restore steps seamlessly

### Variant Mapping

The snapshot stores a full variant ID like `win-nvidia-cu128`. When importing on a different machine:

1. Strip the platform prefix: `win-nvidia-cu128` → `nvidia-cu128`
2. Re-add current platform's prefix: `linux-nvidia-cu128`
3. When fetching releases, look for the best match:
   - Exact match of the re-prefixed variant
   - If not found, strip the CUDA suffix and match the base GPU type (e.g., `nvidia` → pick any `nvidia-*` variant from the release)
   - If still no match, fall back to the user's detected GPU variant (same as the normal install wizard would pick)

This means a snapshot exported from a Windows NVIDIA machine will auto-select the Linux NVIDIA variant when imported on Linux — matching the same GPU type with whatever CUDA version the latest release bundles.

### Implementation

#### IPC handler — `src/main/lib/ipc.ts`

New handler `create-from-snapshot`:

```typescript
ipcMain.handle('create-from-snapshot', async (_event) => {
  // 1. File dialog
  const win = BrowserWindow.fromWebContents(_event.sender)
  const { canceled, filePaths } = await dialog.showOpenDialog(win!, {
    filters: [{ name: 'Snapshot', extensions: ['json'] }],
    properties: ['openFile'],
  })
  if (canceled || filePaths.length === 0) return { ok: false }

  // 2. Read and validate
  const content = await fs.promises.readFile(filePaths[0]!, 'utf-8')
  const envelope = JSON.parse(content)
  // ... validate envelope format ...

  // 3. Extract variant from newest snapshot
  const targetSnapshot = envelope.snapshots[0]
  const snapshotVariant = targetSnapshot.comfyui.variant  // e.g. "win-nvidia-cu128"

  // 4. Map to current platform variant
  const stripped = stripPlatform(snapshotVariant)           // "nvidia-cu128"
  const localVariant = PLATFORM_PREFIX[process.platform] + stripped

  // 5. Fetch latest release, find matching variant
  const releases = await fetchJSON(`.../${RELEASE_REPO}/releases?per_page=5`)
  const { releaseData, variantData } = findBestVariantMatch(releases, localVariant, stripped)
  // ... if no match, return error with message ...

  // 6. Build installation record
  const instData = standalone.buildInstallation({
    release: { value: releaseData.tag_name, label: releaseData.tag_name, data: releaseData },
    variant: { value: variantData.variantId, label: variantData.label, data: variantData },
  })
  const name = await getUniqueName(envelope.installationName || 'ComfyUI')
  const installPath = path.join(defaultInstallDir(), name.replace(/[<>:"/\\|?*]+/g, '_'))

  const entry = await installations.add({
    name, installPath,
    pendingSnapshotRestore: filePaths[0],
    ...instData
  })

  return { ok: true, entry }
})
```

#### Post-install restore hook — `src/main/lib/ipc.ts`

In the `install-instance` handler, after `postInstall` returns and before `sendProgress('done', ...)`:

```typescript
// After postInstall completes, check for pending snapshot restore
const freshInst = await installations.get(installationId)
const pendingFile = freshInst?.pendingSnapshotRestore as string | undefined
if (pendingFile && fs.existsSync(pendingFile)) {
  const sendOutput = (text: string): void => {
    try { if (!sender.isDestroyed()) sender.send('comfy-output', { installationId, text }) } catch {}
  }

  // Add restore steps to progress
  sendProgress('steps', { steps: [
    { phase: 'restore-nodes', label: i18n.t('standalone.snapshotRestoreNodesPhase') },
    { phase: 'restore-pip', label: i18n.t('standalone.snapshotRestorePipPhase') },
  ] })

  // Read and import the snapshot
  const content = await fs.promises.readFile(pendingFile, 'utf-8')
  const envelope = JSON.parse(content) as SnapshotExportEnvelope
  await importSnapshots(freshInst.installPath, envelope)
  const targetSnapshot = envelope.snapshots[0]!

  // Restore nodes, then pip
  await restoreCustomNodes(freshInst.installPath, freshInst, targetSnapshot, sendProgress, sendOutput, abort.signal)
  await restorePipPackages(freshInst.installPath, freshInst, targetSnapshot,
    (phase, data) => sendProgress(phase === 'restore' ? 'restore-pip' : phase, data),
    sendOutput, abort.signal)

  // Save post-restore snapshot and clear pending flag
  const filename = await saveSnapshot(freshInst.installPath, freshInst, 'post-restore')
  const snapshotCount = await getSnapshotCount(freshInst.installPath)
  await update({ pendingSnapshotRestore: undefined, lastSnapshot: filename, snapshotCount })
}
```

#### Preload + types

```typescript
// preload/index.ts
createFromSnapshot: () => ipcRenderer.invoke('create-from-snapshot'),

// types/ipc.ts — add to ElectronApi
createFromSnapshot(): Promise<{ ok: boolean; entry?: Installation; message?: string }>
```

#### UI — `src/renderer/src/views/InstallationList.vue`

Add a "New from Snapshot" button next to "New Install" in the toolbar:

```html
<button @click="handleNewFromSnapshot">{{ $t('list.newFromSnapshot') }}</button>
```

The handler calls `window.api.createFromSnapshot()`, and on success, triggers the install progress modal the same way `handleSave` does in `NewInstallModal.vue`:

```typescript
async function handleNewFromSnapshot(): void {
  const result = await window.api.createFromSnapshot()
  if (!result.ok || !result.entry) return
  emit('show-progress', {
    installationId: result.entry.id,
    title: `Installing — ${result.entry.name}`,
    apiCall: () => window.api.installInstance(result.entry!.id),
  })
}
```

#### i18n

```json
"newFromSnapshot": "New from Snapshot",
"snapshotNoVariantMatch": "No compatible variant found for this snapshot on your system.",
"snapshotRestoreAfterInstall": "Restoring snapshot…"
```

---

## Files Modified (Both Phases)

| File | Phase 1 | Phase 2 |
|---|---|---|
| `src/main/lib/snapshots.ts` | `SnapshotExportEnvelope`, `buildExportEnvelope()`, `importSnapshots()` | — |
| `src/main/lib/ipc.ts` | `export-snapshot`, `export-all-snapshots`, `import-snapshots` handlers | `create-from-snapshot` handler, post-install restore hook |
| `src/preload/index.ts` | 3 new methods | 1 new method |
| `src/types/ipc.ts` | 3 new methods on `ElectronApi` | 1 new method on `ElectronApi` |
| `src/renderer/src/components/SnapshotTab.vue` | Export/Import buttons | — |
| `src/renderer/src/views/InstallationList.vue` | — | "New from Snapshot" button |
| `locales/en.json` | Export/import i18n keys | New-from-snapshot i18n keys |

No new files needed.

---

## Design Decisions

1. **Dedicated IPC handlers, not actions.** Export/import need native file dialogs (`dialog.showSaveDialog` / `showOpenDialog`), which require `BrowserWindow` access only available in IPC handlers, not in the action system's `ActionTools`.

2. **Single envelope format.** The same `comfyui-launcher-snapshot` wrapper is used for both single and bulk export. Import always processes an array, making the logic uniform. A single-snapshot export has one entry; bulk has all.

3. **Import adds to history; does not restore.** Importing a snapshot just places it in the timeline. Applying it to the environment is the existing "Restore" action. This separation keeps import fast, safe, and non-destructive.

4. **Deduplication by `createdAt` + `trigger`.** A snapshot with the same creation timestamp and trigger type is considered a duplicate. This prevents double-importing the same file.

5. **Filename preserves original timestamp.** When importing, the written filename uses the snapshot's `createdAt` (not the import time). This ensures the imported snapshot appears at the correct chronological position in the timeline.

6. **No platform restrictions on export.** Snapshots are platform-agnostic descriptions (package names + versions, node IDs + versions). The variant field is informational for import-into-existing. For new-from-snapshot, it drives variant auto-selection.

7. **New-from-snapshot uses latest release, not the snapshot's exact release.** The snapshot's `releaseTag` may refer to an old or unavailable release. Using the latest release with a compatible variant is more reliable. The snapshot restore handles aligning custom nodes and pip packages regardless of the base release.

8. **Post-install restore chains inside `install-instance`.** Rather than requiring the user to manually click Restore after install, the restore runs automatically as part of the install flow. The progress UI already handles multi-step operations, so the restore steps appear seamlessly after download/extract/setup.

9. **`pendingSnapshotRestore` on the installation record.** This is the bridge between "create" (which returns immediately with the new ID) and "install" (which runs async). The field is cleared after restore completes. If the install fails or is cancelled, the field remains — harmless, since the installation would be deleted anyway.

---

## Estimated Scope

**Phase 1** (~160 lines):
- ~40 lines in `snapshots.ts`
- ~60 lines in `ipc.ts`
- ~10 lines in `preload/index.ts` + `types/ipc.ts`
- ~30 lines in `SnapshotTab.vue`
- ~8 lines in `en.json`

**Phase 2** (~120 lines):
- ~70 lines in `ipc.ts` (new handler + post-install hook)
- ~10 lines in `preload/index.ts` + `types/ipc.ts`
- ~25 lines in `InstallationList.vue`
- ~8 lines in `en.json`

**Phase 3** (future — UX mitigation for snapshot restore on existing installs):
- Add a diff preview UI before applying an imported snapshot to an existing installation
- Show node differences (added/removed/changed), pip package differences (added/removed/version changes), and ComfyUI version differences
- Reuse the existing snapshot comparison layout (similar to viewing a snapshot in history, comparing "current" state vs. the snapshot to restore)
- Prevents accidental environment damage from importing minimal or mismatched snapshots

Total Phases 1+2: ~280 lines of new code.

**Phase 4** (future — release version matching for create-from-snapshot):
- Currently, create-from-snapshot always installs the **latest** ComfyUI release, then restores custom nodes and pip packages from the snapshot
- Enhancement: allow matching the snapshot's exact ComfyUI release version (via `releaseTag` or `ref` in the envelope) instead of always using latest
- This becomes especially important when pytorch or python versions change between releases, since the prepackaged python environments are tied to specific releases — installing latest + restoring old snapshot packages could cause compatibility issues
- UX options to consider: a toggle or dropdown in the Load Snapshot modal letting the user choose between "Latest release" and "Match snapshot release (vX.Y.Z)", with a warning if the snapshot release is no longer available
