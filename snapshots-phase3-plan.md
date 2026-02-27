# Snapshots: Deferred Item + Phase 3 Plan

## Deferred Item: Transitive Dependency Protection

### Problem

Phase 2's `isProtectedPackage` uses a hardcoded prefix list (`torch*`, `nvidia*`, `triton*`, `cuda*`, `pip`, `setuptools`, `wheel`, `uv`). This has two gaps:

1. **New CUDA ecosystem packages** (e.g., a future `nccl-cu128` or renamed nvidia package) won't be caught until we update the list.
2. **Transitive dependencies of protected packages** are unprotected. If `torch` depends on `filelock` and the target snapshot doesn't have `filelock`, the restore would remove it, breaking torch at runtime.

### Approach: Two complementary mechanisms

#### A. Index-URL-based detection

Packages installed from the PyTorch index (`https://download.pytorch.org/whl/...`) should all be protected. The index URL is recorded in the `direct_url.json` file inside each package's `.dist-info/` directory (PEP 610), or can be inferred from the version string (e.g., `+cu128` suffix).

**Implementation:**
- In `findDistInfoDir` we already locate the `.dist-info/` directory for a package.
- Read `direct_url.json` from each `.dist-info/` — if `url` matches `download.pytorch.org` or a known CUDA index, mark as protected.
- Fallback: if the version string in the snapshot contains `+cu` or `+rocm`, mark as protected.
- This runs once during diff computation, adding negligible cost (<100ms for ~300 packages).

**Files:** `snapshots.ts` — modify `isProtectedPackage` to accept an optional `sitePackages` path parameter and check `direct_url.json`.

#### B. Transitive dependency protection

Before removing "extra" packages (present in current, absent from target), verify none are required by a remaining/protected package.

**Implementation:**
- After computing `toRemove`, build a set of packages that will remain: everything in the target snapshot + all protected packages.
- For each remaining package, read its `.dist-info/METADATA` and parse `Requires-Dist:` lines to extract dependency names (ignore extras/markers for simplicity — conservative is safe here).
- Build a "required by remaining" set from all those dependency names.
- Filter `toRemove`: if a package is in the "required by remaining" set, move it to `protectedSkipped` instead.
- This is a single pass over METADATA files (~300 packages × ~1 KB parse = <0.5s).

**Files:** `snapshots.ts` — add `computeTransitiveDeps(sitePackages, remainingPackages)` helper, call from `restorePipPackages` before the removal step.

#### Complexity and risk

- **Index-URL detection**: Low complexity, low risk. Just file reads. Main risk is that `direct_url.json` doesn't exist for older pip-installed packages — the prefix fallback still covers those.
- **Transitive deps**: Medium complexity, low risk. Parsing `Requires-Dist` is straightforward (just extract the package name before any version specifier). Being conservative (protecting anything that might be needed) is safe — worst case we skip removing a package the user could remove manually.

#### Recommendation

Ship both as a single commit. They're independent but both improve the same protected-package logic. Combined, they cover the three known gaps: new CUDA packages, version-suffix detection, and transitive deps.

---

## Phase 3: Custom Node Restore

### Overview

Phase 3 extends `snapshot-restore` to also restore custom nodes to match the target snapshot's `customNodes[]` array. It reuses all Phase 2 safety infrastructure (running-session check, progress reporting, failure handling) and adds node-specific operations.

### Node categories and operations

The snapshot captures three node types. Each has different restore operations:

#### 1. CNR nodes (registry-installed, identified by `.tracking` file)

| Current state | Target state | Action |
|---|---|---|
| Missing | In snapshot | Install from registry via `api.comfy.org/nodes/{id}/install?version={v}` |
| Version mismatch | In snapshot | Version switch: download new version zip, extract, diff `.tracking` for cleanup |
| Enabled | Disabled in snapshot | Move to `.disabled/` |
| Disabled | Enabled in snapshot | Move from `.disabled/` to `custom_nodes/` |
| Match | Match | Skip |

**CNR install/switch protocol** (from `comfyui-manager-functionality.md`):
1. `GET api.comfy.org/nodes/{id}/install?version={v}` → get `download_url`
2. Download zip to temp file
3. Extract into `custom_nodes/{dirName}/`
4. Write `.tracking` file listing all extracted files
5. For version switch: diff old vs new `.tracking` → delete files only in old list
6. Run post-install scripts

**Key decision: implement CNR download natively or delegate to cm-cli?**

The desktop app (`desktop/src/services/cmCli.ts`) delegates to `cm-cli.py restore-snapshot`. But the Launcher can't assume Manager is installed (the spec explicitly says "works even if Manager is not installed"). The Launcher should implement the CNR download protocol natively in TypeScript using the same API endpoints. This is straightforward — it's just an HTTP download + zip extraction, both of which the Launcher already has infrastructure for (`lib/fetch.ts`, `lib/extract.ts`).

#### 2. Git nodes (cloned repos, identified by `.git/` directory)

| Current state | Target state | Action |
|---|---|---|
| Missing | In snapshot (has URL) | `git clone {url}`, then `git checkout {commit}` |
| Commit mismatch | In snapshot | `git fetch origin`, then `git checkout {commit}` |
| Enabled | Disabled in snapshot | Move to `.disabled/` |
| Disabled | Enabled in snapshot | Move from `.disabled/` to `custom_nodes/` |
| Match | Match | Skip |

**Git operations require spawning `git` CLI.** Unlike Phase 1's file-read-only approach (reading `.git/HEAD`), restore needs `git clone`, `git fetch`, `git checkout`. This introduces a dependency on the `git` binary being available. On Windows Standalone installs, git may not be in PATH. Options:
- **Option A:** Bundle a minimal git (e.g., `dugite` — the git binary that VS Code uses). Adds ~30 MB.
- **Option B:** Require git in PATH. If not found, skip git node operations and report to user.
- **Option C:** Use GitHub's archive download API for clones (no git needed), but checkout to specific commits isn't possible this way.

**Recommendation:** Option B for Phase 3. Git is already required for ComfyUI updates (`update_comfyui.py` uses git), so it's a reasonable prerequisite. Log a clear message if git is unavailable. Consider Option A for a future "batteries-included" phase.

#### 3. Standalone `.py` files

| Current state | Target state | Action |
|---|---|---|
| Missing | In snapshot | Report to user (cannot auto-restore a standalone file) |
| Present, not in snapshot | — | Move to `.disabled/` |
| Match | Match | Skip |

Standalone `.py` files can't be downloaded — they were manually placed. The restore can only disable extras and report missing files.

### Post-install scripts

After any node install or version switch, run post-install scripts (same as Manager):
1. If `requirements.txt` exists in the node directory: `uv pip install -r requirements.txt --python {pythonPath}` (filtering out torch/torchvision/torchaudio lines, same as `update-comfyui` and `migrate-from` already do)
2. If `install.py` exists: `{pythonPath} install.py` in the node directory

The Launcher already has this pattern in the `migrate-from` action (standalone.ts line 1024–1067) — reuse the same `PYTORCH_RE` filter and `uv pip install -r` approach.

### Implementation plan

#### New file: `src/main/lib/cnr.ts`

CNR registry API client. Minimal, focused on install/switch.

```typescript
interface CnrInstallInfo {
  downloadUrl: string
  version: string
}

// Fetch download URL for a specific node version from the registry
async function getCnrInstallInfo(nodeId: string, version: string): Promise<CnrInstallInfo>

// Download and extract a CNR node zip into custom_nodes/
async function installCnrNode(
  nodeId: string, version: string, customNodesDir: string,
  sendOutput: (text: string) => void
): Promise<void>

// Switch a CNR node to a different version (download, extract, diff .tracking, cleanup)
async function switchCnrVersion(
  nodeId: string, newVersion: string, nodePath: string,
  sendOutput: (text: string) => void
): Promise<void>
```

#### Modified file: `src/main/lib/snapshots.ts`

Add `restoreCustomNodes` function alongside existing `restorePipPackages`. Both are called from the `snapshot-restore` action handler.

```typescript
export interface NodeRestoreResult {
  installed: string[]     // newly installed nodes
  switched: string[]      // version-switched nodes
  enabled: string[]       // re-enabled nodes
  disabled: string[]      // disabled nodes
  skipped: string[]       // already matching
  failed: string[]        // operations that failed
  unreportable: string[]  // standalone .py files that can't be auto-restored
}

export async function restoreCustomNodes(
  installPath: string,
  installation: InstallationRecord,
  targetSnapshot: Snapshot,
  sendProgress: (phase: string, data: Record<string, unknown>) => void,
  sendOutput: (text: string) => void
): Promise<NodeRestoreResult>
```

The function:
1. Scans current custom nodes (reuses `scanCustomNodes` from `nodes.ts`)
2. Computes the diff against the target snapshot's `customNodes[]` (reuses `nodeKey` for stable matching)
3. Processes each category:
   - **Disable extras**: move unmatched enabled nodes to `.disabled/`
   - **Enable missing-but-disabled**: move from `.disabled/` to `custom_nodes/`
   - **Install missing CNR**: call `cnr.installCnrNode`
   - **Switch CNR version**: call `cnr.switchCnrVersion`
   - **Checkout git nodes**: `git fetch && git checkout {commit}`
   - **Clone missing git nodes**: `git clone {url} && git checkout {commit}`
   - **Report standalone .py**: add to `unreportable`
4. Runs post-install scripts for installed/switched nodes
5. Reports per-node results

#### Modified file: `src/main/sources/standalone.ts`

Update `snapshot-restore` action handler to call both `restorePipPackages` and `restoreCustomNodes` in sequence. Add a second progress step.

```typescript
sendProgress('steps', { steps: [
  { phase: 'restore-nodes', label: t('standalone.snapshotRestoreNodesPhase') },
  { phase: 'restore-pip', label: t('standalone.snapshotRestorePipPhase') },
] })
```

Node restore runs first (because node installs may add pip dependencies via `requirements.txt`), then pip restore syncs the final package state.

#### Modified file: `locales/en.json`

Add i18n strings for node restore progress and results.

### Enable/disable mechanics

Moving nodes between `custom_nodes/` and `custom_nodes/.disabled/`:

```typescript
async function disableNode(customNodesDir: string, dirName: string): Promise<void> {
  const src = path.join(customNodesDir, dirName)
  const disabledDir = path.join(customNodesDir, '.disabled')
  await fs.promises.mkdir(disabledDir, { recursive: true })
  await fs.promises.rename(src, path.join(disabledDir, dirName))
}

async function enableNode(customNodesDir: string, dirName: string): Promise<void> {
  const src = path.join(customNodesDir, '.disabled', dirName)
  await fs.promises.rename(src, path.join(customNodesDir, dirName))
}
```

Note: Manager uses `@{version}` suffix in disabled dir names for multi-version support. Our snapshots record `dirName` as-is (the actual directory name), so we match on that. If the disabled node has a `@version` suffix, we need to handle that during scanning — `scanCustomNodes` already reads from `.disabled/` and records the actual directory name, so this should work naturally.

### Safety considerations

1. **No backup/revert for node operations.** Unlike pip packages (which have a targeted backup), node operations are individually atomic — each enable/disable/install/switch either succeeds or fails independently. There's no equivalent of "half the environment is in an inconsistent state." Failed operations are reported individually.

2. **ComfyUI-Manager itself is never modified.** The Manager restore code (manager_core.py:3156, 3184, 3203) explicitly skips `comfyui-manager`. We do the same — skip any node where `id` contains `comfyui-manager`.

3. **Post-install failures are non-fatal.** If `requirements.txt` or `install.py` fails for a node, report it but don't block the rest of the restore. This matches the existing `migrate-from` behavior.

4. **Running-session check already exists** from Phase 2 — no additional safety needed.

### Ordering: nodes first, then pip

The spec notes that these are complementary layers. But there's a dependency: installing a new custom node may add new pip packages (via `requirements.txt`). If we did pip restore first, those packages would be removed as "extras," then re-added by the node install's post-install script.

Better ordering: **restore nodes first** (which may install new pip dependencies), **then restore pip** (which syncs to the exact target state, removing any packages that the node installs added but that weren't in the snapshot).

This matches Manager's order too — `manager_core.py` does all node operations first, then calls `restore_pip_snapshot` at the end (line 3353).

### Scope and phasing

All of Phase 3 can ship as a single commit. The three node types are independent but share infrastructure (enable/disable, post-install). Splitting further (e.g., "Phase 3a: enable/disable only, Phase 3b: install/switch") would add overhead without meaningful risk reduction.

Estimated new code: ~300 lines in `cnr.ts`, ~200 lines in `snapshots.ts`, ~30 lines in `standalone.ts`, ~15 lines in `en.json`.

### Open questions

1. **CNR API authentication.** Does `api.comfy.org/nodes/{id}/install` require authentication? If so, how does the Launcher obtain credentials? Check the Manager's implementation for auth headers.

2. **`.tracking` file format.** Need to verify the exact format — it appears to be a plain list of relative file paths, one per line. Confirm by reading Manager's `cnr_switch_version` implementation.

3. **Git binary availability.** Should Phase 3 fail gracefully (skip git nodes, report) or hard-fail if git is unavailable? Recommendation: fail gracefully with a clear message listing which nodes were skipped and why.
