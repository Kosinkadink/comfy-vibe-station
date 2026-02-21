# ⚠️ STALE — Superseded by `update_support_plan.md`

This document was the design doc for the `phase1-env-management` branch, which was an early attempt at environment management. The approach has been rethought and replaced by the update support plan in `comfy-vibe-station/update_support_plan.md`. This document is kept for historical reference and code snippets only.

---

# Standalone Environment Management (ARCHIVED)

Design for virtual environment management, snapshots, and rollback in ComfyUI-Launcher's Standalone source.

---

## Current State

### What Exists

- **Environment creation**: `createEnv()` in `sources/standalone.js` creates a new venv via `uv venv`, then copies `site-packages` from the master `standalone-env/` into the new env
- **Environment switching**: `resolveActiveEnv()` picks the active env, `getLaunchCommand()` runs ComfyUI using that env's Python
- **Environment lifecycle**: Create / Delete / Set Active actions in the detail view
- **Installation metadata**: `installations.json` stores `activeEnv`, `envMethods` (map of env name → creation method)
- **ComfyUI Manager v4**: Installed as pip package `comfyui_manager` in the master env (and copied into each env). Activated via `--enable-manager` launch arg. Has a `cm-cli.py` script that works without a running ComfyUI instance, but it's v3 legacy code that uses `sys.path.append("glob/")` hacks and GitPython

### What's Missing

- No tracking of what custom nodes are installed or their versions
- No snapshot/restore capability
- No ComfyUI update mechanism (Standalone updates are "reinstall from a new release")
- No rollback if an update or custom node install breaks the environment
- No visibility into pip packages in an environment

---

## Design Goals

1. **Snapshot & rollback**: Capture the state of an environment (custom nodes + versions, pip packages) and restore it later — either in-place or by creating a new env from the snapshot
2. **ComfyUI updates**: Follow stable or latest tracks (like Portable), but using the Standalone release system. Updates should offer rollback via the snapshot mechanism
3. **Custom node visibility**: Show what custom nodes are installed, their versions, and which env they were installed under
4. **Disk-space conscious**: Use "soft" snapshots (metadata + selective pip reinstall) rather than full copies of multi-GB environments
5. **Offline-capable**: Work without a running ComfyUI instance, using direct filesystem inspection

---

## Architecture

### Why Not Use `cm-cli.py`

The Manager's CLI (`cm-cli.py`) can technically work without a running ComfyUI instance by setting `COMFYUI_PATH`. However, it's problematic for Launcher use:

1. **v3 legacy code** — Uses `sys.path.append("glob/")` hacks, imports `manager_core` from the old module layout. The v4 PyPI package (`comfyui_manager`) has a different internal structure, and there's no v4 CLI entrypoint defined in `pyproject.toml`
2. **Heavy dependencies** — Requires `GitPython`, `typer`, `rich`, `toml`, `PyGithub`, `huggingface-hub` etc. all importable in the calling Python
3. **Process model mismatch** — The CLI spawns as a Python process using the env's own Python. The Launcher is an Electron/Node.js app. We'd need to shell out to `python cm-cli.py` with `COMFYUI_PATH` set, parse stdout, handle errors — brittle
4. **Snapshot format coupling** — Manager's snapshot JSON format is an internal implementation detail that could change between versions

**Decision**: Build environment management directly in the Launcher (Node.js), using filesystem inspection for reads and `uv pip` for package operations. This matches the existing pattern (the Launcher already manages envs, downloads, and runs `uv` directly). We can read Manager's data files (`.tracking`, `pyproject.toml`, CNR metadata) without importing its Python code.

### Data Model

#### Snapshot (`snapshots/{timestamp}-{label}.json`)

Stored per-installation in `{installPath}/.launcher/snapshots/`:

```json
{
  "version": 1,
  "createdAt": "2026-02-19T12:00:00.000Z",
  "label": "before-update",
  "comfyui": {
    "ref": "v0.3.10",
    "releaseTag": "v0.2.1",
    "variant": "win-nvidia-cu128"
  },
  "env": "default",
  "customNodes": [
    {
      "id": "comfyui-impact-pack",
      "type": "cnr",
      "version": "7.3.0",
      "enabled": true,
      "tracking": ["file1.py", "file2.py"]
    },
    {
      "id": "ComfyUI-KJNodes",
      "type": "git",
      "url": "https://github.com/kijai/ComfyUI-KJNodes",
      "commit": "abc1234",
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

#### Installation metadata additions (`installations.json`)

```json
{
  "activeEnv": "default",
  "envMethods": { "default": "copy" },
  "updateTrack": "stable",
  "updateInfoByTrack": { ... },
  "envSnapshots": {
    "default": "2026-02-19_120000-auto"
  }
}
```

`envSnapshots` maps env name → label of the last auto-snapshot taken for that env. This lets us track the "last known good" state per environment.

### Directory Layout

```
{installPath}/
├── standalone-env/          # Master Python + packages (from release archive)
│   ├── python.exe / bin/python3
│   ├── uv.exe / bin/uv
│   ├── Lib/site-packages/   # or lib/pythonX.Y/site-packages
│   ├── wheels/              # (future: bundled wheel cache)
│   └── requirements-frozen.txt  # (future: frozen master requirements)
├── envs/
│   ├── default/             # Virtual environment (active)
│   └── pre-update-backup/   # Snapshot-restored env (rollback target)
├── ComfyUI/
│   ├── main.py
│   ├── custom_nodes/
│   │   ├── ComfyUI-Impact-Pack/
│   │   │   ├── .tracking        # CNR file list
│   │   │   ├── pyproject.toml   # Has project.name for CNR ID
│   │   │   └── requirements.txt
│   │   ├── ComfyUI-KJNodes/
│   │   │   ├── .git/
│   │   │   └── requirements.txt
│   │   └── .disabled/          # Manager's disabled nodes
│   │       └── SomeNode@1.0.0/
│   └── models/
├── .launcher/
│   ├── snapshots/
│   │   ├── 2026-02-19_120000-auto.json
│   │   └── 2026-02-19_143000-before-update.json
│   └── node-history.json    # (future: track install/uninstall timeline)
└── manifest.json            # Release metadata (comfyui_ref, python_version, variant)
```

---

## Components

### 1. Custom Node Scanner (`lib/nodes.js`)

Scans `custom_nodes/` to build a list of installed nodes without depending on Manager. Reads the same files Manager uses:

```javascript
// Returns: [{ id, type, version, enabled, path, ... }, ...]
async function scanCustomNodes(installPath) {
  const customNodesDir = path.join(installPath, "ComfyUI", "custom_nodes");
  const disabledDir = path.join(customNodesDir, ".disabled");
  const nodes = [];

  // Scan active nodes
  for (const entry of await readdir(customNodesDir)) {
    if (entry === ".disabled" || entry.startsWith(".")) continue;
    const nodePath = path.join(customNodesDir, entry);
    if (!isDirectory(nodePath)) {
      // File-based custom node (standalone .py)
      if (entry.endsWith(".py")) {
        nodes.push({ id: entry, type: "file", enabled: true, path: nodePath });
      }
      continue;
    }
    nodes.push({ ...await identifyNode(nodePath), enabled: true });
  }

  // Scan disabled nodes
  if (existsSync(disabledDir)) {
    for (const entry of await readdir(disabledDir)) {
      const nodePath = path.join(disabledDir, entry);
      if (isDirectory(nodePath)) {
        nodes.push({ ...await identifyNode(nodePath), enabled: false });
      }
    }
  }

  return nodes;
}

// Identify node type and version from its directory
async function identifyNode(nodePath) {
  const name = path.basename(nodePath);

  // CNR node: has .tracking file + pyproject.toml with project.name
  if (existsSync(path.join(nodePath, ".tracking"))) {
    const toml = readToml(path.join(nodePath, "pyproject.toml"));
    const id = toml?.project?.name || name;
    const version = toml?.project?.version || "unknown";
    return { id, type: "cnr", version, path: nodePath, dirName: name };
  }

  // Git node: has .git/ directory
  if (existsSync(path.join(nodePath, ".git"))) {
    const commit = readGitHead(nodePath);  // read .git/HEAD → ref → sha
    const url = readGitRemoteUrl(nodePath); // read .git/config → remote "origin" url
    return { id: name, type: "git", commit, url, path: nodePath, dirName: name };
  }

  // Unknown node: plain directory
  return { id: name, type: "unknown", path: nodePath, dirName: name };
}
```

**Git operations** use direct file reads (`.git/HEAD`, `.git/refs/heads/`, `.git/config`) rather than spawning git processes. This is fast and has no dependencies. For the limited info we need (current commit hash, remote URL), this is sufficient.

### 2. Pip Inspector (`lib/pip.js`)

Uses `uv pip freeze` and `uv pip install --dry-run` to inspect and compare environments:

```javascript
async function pipFreeze(uvPath, pythonPath) {
  // Returns Map<packageName, version>
  const output = await exec(uvPath, ["pip", "freeze", "--python", pythonPath]);
  return parseFreezeOutput(output);
}

async function pipDryRun(uvPath, pythonPath, requirementsFile) {
  // Returns { wouldInstall: [...], wouldRemove: [...], upToDate: bool }
  const output = await exec(uvPath, ["pip", "install", "--dry-run",
    "-r", requirementsFile, "--python", pythonPath]);
  return parseDryRunOutput(output);
}

async function pipInstallPackages(uvPath, pythonPath, packages, options = {}) {
  // Install specific package versions: ["torch==2.6.0+cu128", "numpy==1.26.4", ...]
  const args = ["pip", "install", "--python", pythonPath, ...packages];
  if (options.findLinks) args.push("--find-links", options.findLinks);
  if (options.noIndex) args.push("--no-index");
  return exec(uvPath, args);
}
```

### 3. Snapshot Manager (`lib/snapshots.js`)

```javascript
async function saveSnapshot(installPath, envName, label) {
  const nodes = await scanCustomNodes(installPath);
  const uvPath = getUvPath(installPath);
  const pythonPath = getEnvPythonPath(installPath, envName);
  const packages = await pipFreeze(uvPath, pythonPath);
  const manifest = readManifest(installPath);

  const snapshot = {
    version: 1,
    createdAt: new Date().toISOString(),
    label,
    comfyui: {
      ref: manifest.comfyui_ref,
      releaseTag: manifest.version,
      variant: manifest.id,
    },
    env: envName,
    customNodes: nodes.map(n => ({
      id: n.id,
      type: n.type,
      ...(n.version && { version: n.version }),
      ...(n.commit && { commit: n.commit }),
      ...(n.url && { url: n.url }),
      enabled: n.enabled,
    })),
    pipPackages: Object.fromEntries(packages),
  };

  const snapshotDir = path.join(installPath, ".launcher", "snapshots");
  await fs.promises.mkdir(snapshotDir, { recursive: true });
  const timestamp = new Date().toISOString()
    .replace(/[-:]/g, "").replace("T", "_").slice(0, 15);
  const filename = `${timestamp}-${label}.json`;
  await fs.promises.writeFile(
    path.join(snapshotDir, filename), JSON.stringify(snapshot, null, 2));
  return filename;
}

async function restoreSnapshot(installPath, snapshotFile, envName, { onProgress }) {
  const snapshot = JSON.parse(
    await fs.promises.readFile(snapshotFile, "utf8"));
  const uvPath = getUvPath(installPath);
  const pythonPath = getEnvPythonPath(installPath, envName);

  // Phase 1: Restore pip packages
  // Generate a constraints/requirements list from the snapshot
  const packages = Object.entries(snapshot.pipPackages)
    .map(([name, ver]) => `${name}==${ver}`);
  onProgress("pip", { status: "Restoring pip packages…" });
  await pipInstallPackages(uvPath, pythonPath, packages);

  // Phase 2: Report custom node state differences
  // (Custom nodes are filesystem-level; the launcher reports what's different
  //  but actual install/uninstall of custom nodes is done through Manager UI)
  const currentNodes = await scanCustomNodes(installPath);
  const diff = diffCustomNodes(snapshot.customNodes, currentNodes);
  return { pipRestored: true, nodeDiff: diff };
}
```

### 4. Environment-Aware Snapshot Flow

The core idea: **environments are the rollback mechanism, snapshots are the metadata**.

#### Before a risky operation (update, batch custom node install):

1. Auto-save a snapshot of the current env → `auto-{timestamp}.json`
2. Record the snapshot label in `envSnapshots[activeEnv]`
3. Proceed with the operation

#### If something breaks:

**Option A — Soft rollback (in-place)**:
1. Load the auto-snapshot
2. Run `uv pip install` with the exact package versions from the snapshot
3. Report any custom node version mismatches for manual resolution

**Option B — Clean rollback (new env from master)**:
1. Create a new env from master site-packages (existing `createEnv()`)
2. Apply the snapshot's pip packages on top
3. Switch active env to the new one
4. Old broken env can be deleted

#### Disk space comparison:

| Approach | Disk cost per rollback point |
|----------|------------------------------|
| Full env copy (current) | ~5-10 GB (full site-packages duplicate) |
| Soft snapshot (proposed) | ~5 KB (JSON metadata only) |
| Clean rollback from snapshot | ~5-10 GB but only when actually rolling back, can be deleted after |

---

## ComfyUI Updates (Standalone)

### Update Tracks

Standalone supports two update tracks, each with a different mechanism:

#### Stable Track

Follows tagged releases from `Kosinkadink/ComfyUI-Launcher-Environments`.

- **Check**: Fetch releases from the environments repo, compare `releaseTag` against current `manifest.json`
- **Update**: Download and extract a new release archive (new `standalone-env/` + new `ComfyUI/` code)
- **What changes**: Both ComfyUI code AND the Python environment (packages, potentially Python version) are updated together, since each release bundles both

This is the default and recommended track — releases are tested combinations of ComfyUI version + Python packages.

#### Latest from GitHub Track

Tracks the newest commits on `Comfy-Org/ComfyUI`'s main branch. Only updates the ComfyUI code, not the Python environment.

- **Check**: Fetch latest commit from `Comfy-Org/ComfyUI` main branch, compare against current `comfyui_ref` in `manifest.json`
- **Update mechanism**: Standalone releases include `ComfyUI/.git`. Future releases will use a full clone (~65 MB extra compressed, ~2-3% of total archive size) enabling offline `git checkout` to any past tag. The Launcher should prefer the local git history if available (full clone), otherwise fall back to `git fetch` from GitHub. Updates use `git fetch` + `git checkout` (or `git pull`) — fast and incremental, no full re-download needed. The Launcher drives this directly from Node.js (same approach as Portable's update, but without a dedicated update script)
- **What changes**: Only `ComfyUI/` source code. The `standalone-env/` and env pip packages stay the same. This means new ComfyUI features that require new pip dependencies won't work until the user manually installs them or a new Stable release catches up
- **Show info**: Commit count ahead of last stable tag, latest commit message (same as Portable's "v0.3.10 + 15 commits (abc1234)" format)

**Important difference from Portable**: Portable bundles `python_embeded/` which stays constant across git updates — only the ComfyUI code changes. Standalone is similar in this regard: `standalone-env/` stays constant, only `ComfyUI/` code is updated. However, if a new ComfyUI commit introduces new `requirements.txt` entries, those won't be installed automatically. The Launcher should detect this (via `uv pip install --dry-run -r requirements.txt`) and warn the user or offer to install missing dependencies into the active env.

### Update Flow

1. **Check for update**: Fetch releases (Stable) or latest commit (Latest from GitHub)
2. **Show update info**: Version comparison, release notes or commit log, what's changed
3. **Choose update strategy**: The user picks one of three approaches (see below). For Latest from GitHub, only Strategy A (in-place) is available since there's no new release archive to fork from
4. **Execute update** according to chosen strategy
5. **Post-update**: Save a snapshot of the new state, update `manifest.json` with new `comfyui_ref`

### Update Strategies

The update confirmation dialog offers three strategies:

#### Strategy A — In-Place Update (default)

Updates the current installation directly. Fastest, but carries some risk.

1. Auto-snapshot the current environment state
2. Download the new release archive
3. Extract to a temporary directory
4. Replace `standalone-env/` with the new master env
5. Replace `ComfyUI/` with the new ComfyUI code, **preserving**:
   - `custom_nodes/` (the user's installed nodes)
   - `custom_nodes/.disabled/` (disabled nodes)
   - `models/` (always preserved — ComfyUI's own models directory is always used regardless of shared model paths)
   - `user/` (ComfyUI's user data directory — workflows, settings, etc.)
   - `output/`, `input/` (user data)
6. Recreate the active env from the new master (or create a fresh one). The new master was built with the new release's `requirements.txt` and `manager_requirements.txt`, so core and manager dependencies are already satisfied
7. Sync any requirements changes: run `uv pip install -r ComfyUI/requirements.txt` and `uv pip install -r ComfyUI/manager_requirements.txt` against the new env to catch any dependencies that the master env doesn't already cover (edge case, but safe to run)
8. Re-run custom node dependency installation (their `requirements.txt` files)
9. Save a post-update snapshot

**Rollback if broken**:

After an in-place update, the old `standalone-env/` master and `ComfyUI/` core code have been replaced — they're gone. The auto-snapshot only contains metadata (package versions, custom node versions), not the actual files. This limits what rollback can do:

- **Pip/dependency issues** (most common): Attempt to install the old package versions from the snapshot into the current env via `uv pip install`. This works if the old versions are still in uv's cache or available online, and if they're compatible with the new Python version. If the Python version itself changed between releases, old package versions may not be installable.

- **ComfyUI code issues**: The snapshot records the exact `comfyui.ref` (e.g., `v0.3.10`) from the manifest, and standalone releases include `ComfyUI/.git`. The Launcher should defer to the local git history when possible:
  1. **Git checkout from local history** (preferred): If the `.git` has full history (non-shallow clone — future releases), `git checkout <old-ref>` restores the old code instantly with no download needed.
  2. **Git fetch + checkout** (fallback for shallow clones): If the ref isn't available locally (existing shallow clone installs), fetch just the needed tag from GitHub: `git fetch origin tag <old-ref> --depth 1`, then `git checkout <old-ref>`. Requires GitHub access but downloads only ~7 MB.
  3. **Source archive** (fallback if `.git` is damaged): Restore from the CDN-hosted bundled ComfyUI source zip (if available), or download from GitHub via `https://github.com/Comfy-Org/ComfyUI/archive/refs/tags/{ref}.zip`. Small download (~25 MB) compared to re-downloading the full standalone release archive (~GB+). Extract over `ComfyUI/`, preserving `custom_nodes/`, `models/`, `user/`, `output/`, `input/`.
  4. **Re-sync requirements after code rollback**: After checking out the old code, the old `requirements.txt` and `manager_requirements.txt` are restored. These may specify different package versions than what's currently in the env (which was built from the new master). Run `uv pip install -r ComfyUI/requirements.txt` and `uv pip install -r ComfyUI/manager_requirements.txt` to align the env with the rolled-back code. Alternatively, restore the full pip snapshot from the auto-snapshot taken before the update, which captures the exact package set that worked with the old code.

- **Fundamental breakage** (new Python version incompatible with custom nodes, etc.): Effectively requires a full reinstall from the old release. At this point the user is better off with Strategy B.

**In practice**: Strategy A rollback is best-effort. It handles the common case (a pip dependency broke) but can't guarantee full restoration. The update confirmation dialog should communicate this clearly and recommend Strategy B for users who need guaranteed rollback.

#### Strategy B — Fork as New Installation (zero-risk)

Creates a completely new, independent installation with the updated ComfyUI but the same custom nodes, user data, models, and configuration. The original installation is untouched — the user can run both side-by-side to compare.

This is the recommended option for professional users who need guaranteed stability and want to validate new features before committing.

1. Auto-snapshot the current environment state (for reference)
2. Download the new release archive
3. Create a new installation entry (e.g., "MySetup (v0.3.12)") in a new directory
4. Extract the new release into the new directory
5. Copy user data into the new installation:
   - `custom_nodes/` and `custom_nodes/.disabled/` — full copy
   - `user/` — full copy (workflows, settings)
   - `models/` — **symlink or junction** to the original's `models/` directory to avoid duplicating large model files
   - `output/`, `input/` — symlink or junction (or copy, user's choice)
6. Create a new env from the new master
7. Re-run custom node dependency installation
8. New installation appears in the Launcher list alongside the original

**Benefits**:
- Zero risk to the original installation — it's completely untouched
- User can launch both old and new versions to compare behavior
- Models are shared via symlinks/junctions so no extra disk space for them
- If the new version works well, the user can delete the old installation
- If the new version has issues, the user deletes the new one and continues with the original

**Disk cost**: New `standalone-env/` + `envs/` + `custom_nodes/` copy (~2-5 GB typical, models excluded via symlink). The original installation's disk usage is unchanged.

#### Strategy C — In-Place Update with Env Backup

A middle ground: updates in-place like Strategy A, but first clones the current env under a backup name so the user can switch back to the pre-update env if something breaks.

1. Clone the current active env to `pre-update-{version}` (full site-packages copy)
2. Proceed with in-place update (same as Strategy A steps 1-8)
3. If something breaks, user can switch active env back to the backup

**Trade-off**: Uses more disk space than Strategy A (full env copy) but less than Strategy B (no separate installation). Doesn't protect against ComfyUI code-level breakage, only pip/dependency issues.

### Update Detail Section

Added to `getDetailSections()` in `sources/standalone.js`:

```javascript
{
  title: t("standalone.updates"),
  fields: [
    { id: "updateTrack", label: t("standalone.updateTrack"),
      value: track, editable: true,
      refreshSection: true, editType: "select", options: [
        { value: "stable", label: t("standalone.trackStable") },
        { value: "latest", label: t("standalone.trackLatest") },
      ] },
    { label: t("standalone.installedVersion"),
      value: installation.releaseTag },
    { label: t("standalone.latestVersion"),
      value: info?.releaseName || "—" },
    { label: t("standalone.lastChecked"),
      value: info?.checkedAt ? ... : "—" },
    { label: t("standalone.updateStatus"),
      value: info?.available ? "Update available" : "Up to date" },
  ],
  actions: [
    ...(info?.available
      ? [{ id: "update-standalone", ... }] : []),
    { id: "check-update", label: t("actions.checkForUpdate"), ... },
  ],
}
```

---

## Custom Nodes Detail Section

New section in the Standalone detail view showing installed custom nodes:

```javascript
{
  title: t("standalone.customNodes"),
  description: t("standalone.customNodesDesc", { count: nodes.length }),
  items: nodes.map(node => ({
    label: node.id,
    badges: [
      { text: node.type,
        style: node.type === "cnr" ? "info" : "default" },
      { text: node.version || node.commit?.slice(0, 7) || "—" },
      ...(!node.enabled
        ? [{ text: "disabled", style: "muted" }] : []),
    ],
  })),
}
```

This is read-only for now — actual custom node management (install/uninstall/update) happens through the Manager UI inside ComfyUI. The Launcher just shows what's there.

---

## Snapshots Detail Section

```javascript
{
  title: t("standalone.snapshots"),
  description: t("standalone.snapshotsDesc"),
  items: snapshots.map(s => ({
    label: `${s.label}  ·  ${new Date(s.createdAt).toLocaleString()}`,
    sublabel: `${s.customNodes.length} nodes  ·  `
      + `${Object.keys(s.pipPackages).length} packages`,
    actions: [
      { id: "snapshot-restore",
        label: t("standalone.restoreSnapshot"),
        data: { file: s.filename } },
      { id: "snapshot-delete",
        label: t("common.delete"), style: "danger",
        data: { file: s.filename } },
    ],
  })),
  actions: [
    { id: "snapshot-save", label: t("standalone.saveSnapshot") },
  ],
}
```

---

## Custom Node Dependency Installation

### `installCustomNodeDeps()` (`lib/nodes.js`)

Scans all active custom nodes for `requirements.txt` and installs their dependencies into a target environment. Used after env creation and during snapshot restore to ensure custom node pip dependencies are satisfied.

```javascript
async function installCustomNodeDeps(installPath, uvPath, pythonPath, options = {}) {
  const nodes = await scanCustomNodes(installPath);
  const activeNodes = nodes.filter(n => n.enabled && n.type !== "file");
  for (const node of activeNodes) {
    const reqFile = path.join(node.path, "requirements.txt");
    if (!existsSync(reqFile)) continue;
    await runUv(uvPath, buildPipArgs(
      ["pip", "install", "-r", reqFile, "--python", pythonPath], options));
  }
}
```

**Where it's needed**:
- After `createEnv()` in `postInstall` and `handleAction("env-create")` — new envs have master packages but not custom node deps
- After Strategy A update (step 5) — new env from new master may not have custom node deps
- Strategy B fork — new installation copies `custom_nodes/` but needs deps installed into the new env
- Snapshot restore — after restoring node state, deps must be synced

### Custom Node State Restore (Future)

Currently, snapshot restore only restores pip packages and reports custom node differences. A full restore would need to handle node state changes:

| Change | Action | Mechanism |
|---|---|---|
| Node added since snapshot | Move to `.disabled/` or delete | Filesystem move/delete |
| Node removed since snapshot | Cannot restore without re-downloading | Report to user |
| CNR version changed | Re-download old version | `api.comfy.org/nodes/{id}/install?version=X` |
| Git commit changed | Checkout snapshot's commit | `git checkout {commit}` in node directory |
| Enabled/disabled changed | Move between `custom_nodes/` and `.disabled/` | Filesystem move |

**Two approaches for implementation**:

1. **Launcher-native**: Implement CNR download + git checkout directly. Full control, works offline for git nodes, but requires reimplementing Manager's `.tracking` file dance for CNR nodes.

2. **Delegate to Manager**: Write a `restore-snapshot.json` to `startup-scripts/` in Manager's format. Manager's `prestartup_script.py` handles the restore on next ComfyUI boot — it already knows how to download CNR versions, git checkout, and manage `.tracking`/`.disabled`. Trade-off: couples to Manager's snapshot format, only works if Manager is installed and `--enable-manager` is set (true for all Standalone installations).

**Decision**: Deferred. The pip-only restore + informational node diff is sufficient for the "undo last update" use case (updates preserve `custom_nodes/` via `PRESERVED_DIRS`). Full node restore matters more for Strategy B forks and manual "restore to a known-good state" scenarios.

---

## Phase 5 Analysis: Wheel Cache

### What Changes

Phase 5 replaces the `createEnv()` file-copy approach (copying ~5-10 GB of `site-packages` from the master env) with `uv pip install` from bundled `.whl` files. Two coordinated changes are needed:

**Build workflow** (`build-standalone-env.yml`): After the existing "Install dependencies" step, freeze the master environment and download matching wheels:

```yaml
- name: Bundle wheel cache
  shell: bash
  run: |
    if [[ "${{ runner.os }}" == "Windows" ]]; then
      PYTHON="standalone-env/python.exe"
      UV="standalone-env/uv.exe"
    else
      PYTHON="standalone-env/bin/python3"
      UV="standalone-env/bin/uv"
    fi

    "$UV" pip freeze --python "$PYTHON" > standalone-env/requirements-frozen.txt

    "$UV" pip download \
      -r standalone-env/requirements-frozen.txt \
      -d standalone-env/wheels \
      --python-version "${{ matrix.python_version || env.PYTHON_VERSION }}" \
      --python-platform "${{ matrix.python_platform }}"
```

Both `wheels/` and `requirements-frozen.txt` ship automatically since the workflow already packages the entire `standalone-env/` directory.

**Launcher** (`createEnv()` in `standalone.js`): Replace the copy path with wheel install, keeping the old path as a fallback for pre-wheel-cache installations:

```javascript
async function createEnv(installPath, envName, onProgress) {
  // ... uv venv ...

  const frozenReqs = path.join(installPath, "standalone-env", "requirements-frozen.txt");
  const wheelsDir = path.join(installPath, "standalone-env", "wheels");

  if (fs.existsSync(frozenReqs) && fs.existsSync(wheelsDir)) {
    // Wheel cache path: install from bundled wheels (fast, offline)
    const envPython = getEnvPythonPath(installPath, envName);
    await runUv(uvPath, ["pip", "install", "-r", frozenReqs,
      "--find-links", wheelsDir, "--no-index",
      "--python", envPython]);
    await codesignBinaries(findSitePackages(envPath));  // macOS only
  } else {
    // Legacy path: copy site-packages from master
    await copyDirWithProgress(masterSitePackages, envSitePackages, onProgress);
    await codesignBinaries(envSitePackages);
  }
}
```

### Touchpoints in Existing Code

| Location | Impact |
|---|---|
| `createEnv()` standalone.js | Primary change — wheel install replaces file copy |
| `postInstall()` standalone.js | Calls `createEnv()` — progress reporting changes (no file count) |
| `handleAction("env-create")` standalone.js | Same — progress UX changes |
| `handleAction("update-standalone")` standalone.js | Calls `createEnv()` after extracting new release — benefits most since the new archive has wheels |
| `performSoftRestore()` standalone.js | No change — already uses `pipInstallFromList()`, not `createEnv()` |
| Clean restore (standalone.js) | Calls `createEnv()` then `performSoftRestore()` — both benefit; fast env creation then snapshot delta |
| `lib/pip.js` | No changes — `buildPipArgs()` already supports `--find-links`/`--no-index` |
| `lib/snapshots.js` | No changes — snapshots record package versions, not installation method |
| `copyDirWithProgress`, `collectFiles`, `findSitePackages` | Retained for legacy fallback only; unused on wheel-cache installations |

### Benefits

- **Faster env creation**: `uv pip install` from local wheels is ~10-30x faster than copying millions of individual files. The torch wheel alone expands to thousands of `.py` files
- **Offline & reproducible**: `--no-index` guarantees zero network access; frozen requirements guarantee exact versions
- **Clean restore speedup**: The biggest user-facing win — "clean restore" (new env from master + snapshot packages) currently copies the entire master env first, then soft-restores. With wheel cache, env creation becomes trivially fast

### Trade-offs & Risks

1. **Archive size may increase**: `wheels/` could add 1-2 GB since the master `site-packages` is still present (needed by the master Python itself). Net size depends on how well 7z deduplicates between `site-packages` and `wheels/`. Likely ~10-20% increase. A future optimization could strip the master `site-packages` to only what the master Python needs to run (`uv`, `pip`, `setuptools`), but this is aggressive and should wait for real archive size data

2. **macOS codesigning**: `codesignBinaries()` ad-hoc signs `.dylib`/`.so` files after copy. With wheel install, `uv` extracts wheels and the resulting files still need signing. The function should run after wheel install, same as before — verify binaries end up in the expected `site-packages` location

3. **Progress reporting**: `copyDirWithProgress` gives granular file-by-file progress. Wheel install is fast enough to use an indeterminate spinner. The `onProgress` callback pattern in callers (`postInstall`, `handleAction("env-create")`, update flow) needs updating to handle the case where there's no total file count

4. **`uv pip download` platform fidelity**: Downloading wheels with `--python-platform` on GitHub Actions may not perfectly match the target in edge cases (e.g., CUDA-specific wheel tags, ABI-specific builds). Needs testing per variant, especially torch+CUDA builds that use custom index URLs. The `python_platform` matrix variable already exists in the workflow

5. **Backwards compatibility**: Old installations without `wheels/` or `requirements-frozen.txt` fall back to the copy approach. No migration needed

### Build Workflow Notes

- The workflow currently installs via `pip install` (not `uv pip install`), but `uv pip freeze` is compatible with standard site-packages — it reads the installed `.dist-info` metadata, not how packages were installed
- The `--python-platform` flag on `uv pip download` maps to the existing `python_platform` matrix variable (e.g., `x86_64-pc-windows-msvc`, `aarch64-apple-darwin`)
- The build workflow must ship first (in the Environments repo) — the Launcher change is useless without `wheels/` in the archive
- PyPI mirror config (from the "PyPI Mirrors" section) is not needed for Phase 5 since `--no-index` means no network. Mirrors matter for `performSoftRestore()` and custom node deps, which are Phases 1-4

### Rollout Order

1. Ship the build workflow change as a standalone PR to the Environments repo — add the freeze + download steps, verify `wheels/` in each variant's archive
2. Update `createEnv()` in the Launcher with the conditional fallback
3. Defer stripping master `site-packages` until real archive size data is available

---

## Artifact Hosting & China Accessibility

Release artifacts from `Kosinkadink/ComfyUI-Launcher-Environments` are currently hosted on GitHub. If artifacts are moved to a CDN more reliable in China (e.g., Alibaba Cloud OSS, Tencent COS), GitHub-dependent operations (git fetch, git checkout, source archive download) become unreliable for users in those regions.

### Impact on Update & Rollback

| Operation | GitHub dependency | With China CDN |
|-----------|-------------------|----------------|
| Stable track: check for update | Fetches release list from environments repo | Could use CDN-hosted release manifest |
| Stable track: download update | Downloads release archive | Works if archive is on CDN |
| Strategy A rollback (code) | `git checkout <old-ref>` or GitHub source archive | **Breaks** if GitHub is unreliable |
| Strategy B fork | Downloads new release archive | Works if archive is on CDN |
| Latest from GitHub track | Fetches commits from `Comfy-Org/ComfyUI` | **Requires GitHub by definition** — users opt into this |

### Full Git Clone in Releases

Future releases will switch from `--depth 1` (shallow clone, ~7 MB `.git`) to a full clone (~72 MB `.git`, ~65 MB compressed difference). This enables offline `git checkout` to any past ComfyUI tag without contacting GitHub — solving the Strategy A rollback problem for China-hosted artifacts.

- **Rollback**: `git checkout <old-ref>` works entirely offline using the local git history
- **Stable updates**: Each new release's archive includes the full ComfyUI repo. Forward updates extract the new code; rollback checks out the old ref from history
- **Disk cost**: ~65 MB extra per release archive (~2-3% of total). Reasonable given the multi-GB `standalone-env/`
- **Existing shallow installs**: The Launcher falls back to `git fetch` from GitHub if the local history doesn't contain the requested ref. This gracefully handles installs created before the switch to full clones

### Bundled ComfyUI Source Zip

In addition to the full clone in the main archive, releases will include a separate bundled ComfyUI source zip (with `.git/` history) as a release artifact. This eliminates the need for the GitHub source archive download fallback — if the `.git` inside an installation is damaged or missing, the Launcher can restore it from the CDN-hosted source zip instead of contacting GitHub.

This means **all Strategy A rollback paths work without GitHub access** when artifacts are CDN-hosted:
1. Local `git checkout` from the installation's own `.git` (full clone)
2. Restore `.git` + source from the CDN-hosted source zip if damaged
3. No GitHub dependency for the Stable track at all

### Release Manifest on CDN

For update checking without GitHub API access, releases could include a `latest.json` manifest hosted on the CDN:

```json
{
  "stable": {
    "tag": "v0.2.1",
    "comfyui_ref": "v0.3.12",
    "published_at": "2026-02-15T00:00:00Z",
    "artifacts_url": "https://cdn.example.com/releases/v0.2.1/"
  }
}
```

The Launcher would check this manifest first, falling back to the GitHub API if it's unavailable.

---

## PyPI Mirrors & Geographic Parity

All `uv pip` operations (snapshot restore, dependency sync, custom node requirements) must support PyPI mirror fallbacks to maintain geographic parity with Desktop, particularly for users in China where `pypi.org` is slow or unreliable.

### Desktop's Approach

Desktop defines fallback index URLs in `constants.ts` that are passed as `--extra-index-url` on every pip operation:

```
https://mirrors.aliyun.com/pypi/simple/    (Alibaba Cloud — China)
https://mirrors.cloud.tencent.com/pypi/simple/  (Tencent Cloud — China)
https://pypi.org/simple/                    (default)
```

Desktop also exposes three user-configurable mirror settings:
- `UV.PythonInstallMirror` — for Python interpreter downloads (`UV_PYTHON_INSTALL_MIRROR` env var)
- `UV.PypiInstallMirror` — primary pip index URL (`--index-url`)
- `UV.TorchInstallMirror` — PyTorch wheel index URL (`--extra-index-url`)

### Launcher Implementation

`lib/pip.js` should accept mirror configuration and apply it to all `uv` invocations:

```javascript
// Default fallback index URLs (same as Desktop)
const PYPI_FALLBACK_INDEX_URLS = [
  "https://mirrors.aliyun.com/pypi/simple/",
  "https://mirrors.cloud.tencent.com/pypi/simple/",
  "https://pypi.org/simple/",
];

function buildPipArgs(baseArgs, options = {}) {
  const args = [...baseArgs];
  if (options.indexUrl) args.push("--index-url", options.indexUrl);
  const fallbacks = (options.extraIndexUrls || PYPI_FALLBACK_INDEX_URLS)
    .filter(url => url !== options.indexUrl);
  for (const url of fallbacks) {
    args.push("--extra-index-url", url);
  }
  return args;
}
```

Mirror settings should be stored in the Launcher's settings (alongside existing settings like `cacheDir`, `theme`, etc.) and configurable in the Settings view. The standalone `getLaunchCommand()` should also pass `UV_PYTHON_INSTALL_MIRROR` in the environment if configured.

---

## Implementation Plan

### Phase 1: Foundation
- [ ] `lib/nodes.js` — Custom node scanner (scan `custom_nodes/`, identify CNR/git/file nodes)
- [ ] `lib/pip.js` — Pip inspector (freeze, dry-run, install via `uv`)
- [ ] `lib/snapshots.js` — Save/load/list/delete snapshots, diff function
- [ ] Add `.launcher/` to the installation directory layout

### Phase 2: Detail View Integration
- [ ] Custom Nodes section in standalone detail view (read-only list)
- [ ] Snapshots section with save/restore/delete actions
- [ ] Wire up IPC handlers for new actions in `standalone.js` `handleAction()`
- [ ] i18n keys for all new strings

### Phase 3: ComfyUI Updates
- [ ] `check-update` action for Standalone (fetch releases, compare versions)
- [ ] Update track selector (stable/latest) in detail view
- [ ] `update-standalone` action (download, extract, preserve user data, recreate env)
- [ ] Auto-snapshot before update, rollback on failure

### Phase 4: Rollback UX
- [ ] Soft restore (pip install from snapshot into current env)
- [ ] Clean restore (new env from master + snapshot apply)
- [ ] "Undo last update" convenience action
- [ ] Snapshot comparison view (diff two snapshots)

### Phase 5: Wheel Cache (depends on build workflow changes)
- [ ] Add `uv pip freeze` + `uv pip download` steps to `build-standalone-env.yml`
- [ ] Verify `wheels/` appears in release archives with correct content per variant
- [ ] Bundle `wheels/` + `requirements-frozen.txt` in release archives
- [ ] Update `createEnv()` to use `uv pip install --find-links --no-index` with fallback to legacy copy for pre-wheel-cache installations
- [ ] Update progress reporting (wheel install is fast; indeterminate spinner replaces file-count progress)
- [ ] Verify macOS codesigning still works (run `codesignBinaries()` after wheel install)
- [ ] Faster env creation + snapshot clean restore

---

## Decisions (formerly Open Questions)

1. **Custom node isolation**: Shared `custom_nodes/` with metadata tracking is the MVP. Per-env custom node sets (symlinking) deferred — adds significant complexity for a niche use case.

2. **Manager's own snapshots**: Manager's snapshot system is v3 legacy — we ignore it and use our own format. A separate "import from Manager snapshot" feature could be added later for users migrating from v3-based setups.

3. **Network operations during restore**: Try offline first (`--find-links` with wheel cache if available, leveraging uv's own cache), fall back to network downloads if needed.

4. **Maximum auto-snapshots**: Keep last 5 auto-snapshots per env, unlimited manual snapshots. Auto-snapshots are lightweight JSON metadata files (~5 KB each — custom node list + pip freeze output), not full env copies. Full env copies only occur if the user explicitly chooses a "clean rollback" (Option B), and can be deleted afterward.
