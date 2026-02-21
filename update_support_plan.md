# ComfyUI-Launcher: Update Support Plan

Comprehensive plan for updating, rollback, and installation management for ComfyUI Standalone in ComfyUI-Launcher.

---

## Context

Updating ComfyUI is a fragile process. Different update methods carry different risks of breaking installs. Since pushing new features requires updating, the Launcher must provide reliable update paths with clear rollback options so users feel safe.

The primary install method is **Standalone** — prepackaged Python environments on a per-environment, per-accelerator basis.

### Ground Truths

- The three base PyTorch packages (`torch`, `torchvision`, `torchaudio`) are **unsafe to update** without a full new installation and migration. They are the single biggest source of custom node breakage.
- `requirements.txt` may change between ComfyUI versions and must be reconciled.
- Custom nodes install arbitrary Python packages, gradually mutating the environment. Over time, conflicts can arise and nodes may break.
- `pygit2` ships with the Standalone environment, so git operations are available.

### Update Tracks

Two tracks of update markers, **independent of the update method**:

| Track | Source | Description |
|-------|--------|-------------|
| **Stable** | Tagged releases (`v*.*.*`) on `Comfy-Org/ComfyUI` | Tested, recommended for most users |
| **Latest from Git** | `master` branch on `Comfy-Org/ComfyUI` | Bleeding edge, may have regressions |

---

## Update Methods

### 1. Commit-Based Method

Updates the ComfyUI code in-place using git operations (via the existing portable update scripts that use `pygit2`). The Python environment is preserved; only code files and potentially pip packages change.

**Supports:** Both Stable and Latest from Git tracks.

**How it works:**
1. Stash local changes, create a timestamped backup branch
2. Pull latest from remote (or checkout a specific stable tag)
3. Compare `requirements.txt` against current state
4. If requirements changed, run `uv pip install -r requirements.txt`
5. Store rollback metadata (previous commit hash + custom node state)

**Rollback:** Revert to previous commit. If the Python environment was not modified (no requirements changes), rollback is trivial. If requirements were installed, a full environment rollback may be needed (see caveats below).

**Caveats:**
- Requirements changes may introduce package conflicts with custom nodes. A **dry-run check** (`uv pip install --dry-run`) should be performed before applying. If conflicts are detected, the user should be warned and offered a copy-commit-based update as a safer alternative.
- The user can still choose in-place update despite dry-run warnings — some conflicts are benign (custom nodes often pin versions unnecessarily). The caveats should be clearly explained.
- Shoddy network or unavailable packages in mirrors could cause partial installs.
- PyTorch versions are tied to the Standalone release and must not be touched.

**Future enhancement:** If a connection to the GitHub repo cannot be made, a zipped file containing the full `.git` history up to the target version could be downloaded from Standalone release artifacts as a fallback. This is not required for the initial implementation.

### 2. Copy-Commit-Based Method

Creates a complete copy of the current installation, then applies a commit-based update on the copy. The original installation is completely untouched.

**Supports:** Both Stable and Latest from Git tracks.

**How it works:**
1. Copy the entire installation directory (the active environment, not the master copy)
2. Apply the commit-based update method on the copied installation
3. Custom nodes and user data are already present in the copy — no migration needed

**Rollback:** Not needed — the original install is unmodified. The user can delete the copy if unsatisfied.

**This is the recommended safe path** when:
- The dry-run check detects potential conflicts during a commit-based update
- The user wants to try new features without any risk to their current setup

**Note:** The installation copy feature should also be offered as a standalone operation outside of the update flow, for users who want to duplicate an install for their own purposes.

### 3. Release-Based Method

Creates a brand new installation from the latest Standalone release, then migrates custom nodes and user data from an existing installation. This is the only method that updates the PyTorch version.

**Supports:** Stable track primarily.

**How it works:**
1. Download and install a new Standalone release (new environment, new PyTorch)
2. Migrate custom nodes and user data from the source installation (see Migration below)

**Rollback:** Not needed — the source install is unmodified.

**Caveats:**
- The new installation may use a different PyTorch version, which could break some custom nodes after migration.
- This is the only reasonable way to update PyTorch. That said, PyTorch updates are extremely rarely required — even old versions work with modern ComfyUI, just with reduced performance from missing optimizations.

---

## Supporting Operations

### Installation Copy

Copy an entire Standalone installation, producing a fully independent duplicate. This is used by the copy-commit-based method but is also useful as a standalone feature.

**Key concern — Python environment portability:**
- Embedded Python uses relative paths in `python._pth`, so it copies safely.
- `pyvenv.cfg` in venvs contains absolute paths (`home`, `base-prefix`) that will point to the original location after copy. This works for running ComfyUI directly (launching `python main.py`) but may break console entry points or tools that rely on these paths.
- Console scripts in `Scripts/` have shebangs with absolute paths to the Python interpreter.
- **Action required:** Verify empirically that a copied install works correctly (run ComfyUI, check `uv pip list`, test custom node installs). If `pyvenv.cfg` causes issues, implement a post-copy fixup that rewrites its paths.

**Future concern — uv hardlinks/symlinks:** Currently we use direct copies for site-packages, not `uv` link modes. If we ever switch to `uv` hardlinks/symlinks for space efficiency, copying would need special handling (e.g., `--link-mode=copy`). This should be noted and addressed when/if that optimization is implemented.

### Custom Node & User Data Migration

Two distinct sub-operations with different complexity:

#### File Migration (Simple)
Copy custom node directories and user data folders (`custom_nodes/`, `user/`, `output/`, `input/`, `models/` or symlinks) from one installation to another.

Used by: Copy-commit-based (implicitly, since the whole install is copied), Release-based, and as a standalone feature.

#### Dependency Reconciliation (Complex)
After migrating custom node files to a new environment (especially one with a different PyTorch version), re-run `requirements.txt` for each custom node to ensure dependencies are satisfied in the target environment.

Used by: Release-based method only. Not needed for copy-commit-based (same env, same packages).

**Approach:** For each migrated custom node that has a `requirements.txt`, run `uv pip install -r requirements.txt` against the target environment. Consider a dry-run first to detect conflicts. Referencing ComfyUI-Manager v4's node scanning and dependency installation may be useful here.

This migration operation should be coded as a reusable primitive, as it enables:
- Migrating between any two installations
- Future feature: sharing a full install environment specification with another user on a different machine

---

## Dry-Run Conflict Handling

Before any in-place commit-based update that involves requirements changes:

1. Run `uv pip install --dry-run -r requirements.txt` against the current environment
2. If no conflicts → proceed with update
3. If conflicts detected → warn the user with specifics, then offer:
   - **Copy-commit-based update** (recommended) — safe, creates a new install
   - **In-place update anyway** — user accepts the risk; caveats are clearly explained
   - **Cancel** — do nothing

Note: Custom node developers often pin versions unnecessarily. A pinned version conflict does not necessarily mean breakage. The user should always have a path forward to try the update.

---

## UX Decisions

- **Environment selection is hidden.** The multi-env machinery remains as an internal implementation detail for copying installations and rollback, but users see and manage installations, not environments.
- **Installations are the unit of visibility.** Users can see their installations, copy them, update them, and delete them. This avoids confusion about rollback environments silently consuming disk space.
- **Installations track their origin.** Each installation records which Standalone release it was created from, enabling informed decisions about PyTorch version compatibility.

---

## Development Plan

### Step 1: Commit-Based Updating

Implement the commit-based update method for Standalone, based on the Portable's update mechanism (`update.py` using `pygit2`).

- Support both Stable and Latest from Git tracks
- Handle `requirements.txt` reconciliation via `uv pip install`
- Implement dry-run conflict detection (`uv pip install --dry-run`)
- Store rollback metadata (previous commit, custom node state snapshot)
- When dry-run detects conflicts, warn user and offer copy-commit-based as alternative (with option to proceed in-place anyway)

### Step 2: Installation Copy

Implement copying a full Standalone installation.

- Copy the active environment directory (not the master copy)
- Verify Python environment portability (test `pyvenv.cfg`, console scripts, `uv pip list`)
- Implement post-copy path fixups if needed
- Register the copy as a new installation in the Launcher
- Track which Standalone release the copy originated from
- Expose as both a standalone feature and a building block for step 4

### Step 3: Custom Node & User Data Migration

Implement migration of custom nodes and user data between installations.

- **File migration:** Copy `custom_nodes/`, user data directories
- **Dependency reconciliation:** Re-run `requirements.txt` for migrated nodes against target environment
- View and compare custom node state between two installations
- Reference ComfyUI-Manager v4 for node scanning and identification patterns
- Code as a reusable primitive for use by step 5 and future features

### Step 4: Copy-Commit-Based Update

Combine steps 2 and 1 into an automated flow.

- Copy the installation (step 2)
- Apply commit-based update on the copy (step 1)
- The manual way (copy + update separately) should remain available
- This is the recommended path when dry-run detects conflicts

### Step 5: Release-Based Update

Combine creating a new Standalone installation with custom node/user data migration.

- Download and install a new Standalone release as a new installation
- Run migration (step 3) from the source installation
- Handle PyTorch version differences gracefully (warn user, run dependency reconciliation)

### Future Enhancements

- **Disk space checks:** Gate large operations (update downloads, install copies, env creation) on available disk space using `lib/disk.js`. See `TODO_DISK_SPACE_CHECKS.md` on the `phase1-env-management` branch for specifics.
- **Zipped `.git` fallback:** Download `.git` history from release artifacts when GitHub is unreachable, supporting offline/restricted-network Stable updates.
- **uv hardlink/symlink optimization:** If env creation switches to `uv` link modes, update the copy logic to handle non-self-contained site-packages.
- **Install environment sharing:** Export an installation specification (Standalone version + custom node list + versions) for another user to reproduce.

---

## Reference Material

- `comfy-vibe-station/comfyui-update-mechanisms.md` — How portable and desktop updates work today
- `comfy-vibe-station/comfyui-manager-functionality.md` — Manager v3/v4 node management, useful for step 3
- `ComfyUI-Launcher/DESIGN_PROCESS.md` — Launcher architecture and source module conventions
- `ComfyUI-Launcher/sources/portable.js` — Existing portable update implementation (commit-based via `update.py`)
- `ComfyUI-Launcher/sources/standalone.js` — Current standalone source module
- `stale/standalone-environments.md` — Archived phase1 design doc (code snippets may be useful)
