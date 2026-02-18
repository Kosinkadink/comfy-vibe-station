# Extra Model Paths

ComfyUI supports pointing multiple directories at each model folder type (checkpoints, loras, vae, etc.) via `extra_model_paths.yaml`. This is the mechanism that both the desktop app and a launcher would use to share a global model store across multiple ComfyUI installations.

## Summary

At startup, ComfyUI loads model path configs from two sources: an `extra_model_paths.yaml` file in the ComfyUI root (auto-loaded if present), and any files passed via `--extra-model-paths-config <path>` (CLI arg, supports multiple files). Each YAML file contains config groups with a `base_path`, an optional `is_default` flag, and `folder_name: path(s)` entries. Paths are resolved (env vars, `~`, relative-to-yaml) and appended to the global `folder_names_and_paths` registry in `folder_paths.py`. The `is_default` flag causes paths to be prepended instead, making them the primary search/download targets.

The desktop app already uses this system — it generates an `extra_models_config.yaml` at startup with a hardcoded list of folder types. Both the desktop and any launcher face the same problem: the YAML must explicitly list every folder type, and ComfyUI keeps adding new ones. A proposed change to ComfyUI adds an `all_model_folders` flag that, when set to `true`, automatically registers `base_path/{folder_name}` for every model folder type — keeping configs future-proof without enumerating types.

## YAML Format

```yaml
config_group_name:
    base_path: /path/to/models/root   # resolved: expandvars → expanduser → relative-to-yaml-dir
    is_default: true                   # optional; prepends paths instead of appending
    checkpoints: checkpoints/          # relative to base_path
    loras: loras/
    text_encoders: |                   # multiple paths via YAML block scalar
        text_encoders/
        clip/
```

**Special keys** (popped before processing):
- `base_path` — root for relative paths in this group. Supports `~`, `%APPDATA%`, `$HOME`, etc.
- `is_default` — if `true`, paths are inserted at position 0 (searched first, used for downloads)
- `all_model_folders` — if `true`, registers `base_path/{folder_name}` for all model folder types (excludes `custom_nodes`). Can be combined with explicit entries

**All other keys** are treated as `folder_name → path(s)`. Values are split on newlines, each line joined with `base_path`, normalized, and registered via `folder_paths.add_model_folder_path()`.

**Legacy name mapping** is applied automatically: `unet` → `diffusion_models`, `clip` → `text_encoders`.

## Loading Order

**File:** `ComfyUI/main.py` — `apply_custom_paths()`

1. Auto-load `extra_model_paths.yaml` from the ComfyUI root directory (if it exists)
2. Load each file from `--extra-model-paths-config` CLI args (in order)

Both call `utils.extra_config.load_extra_path_config(yaml_path)`.

## How Paths Are Registered

**File:** `ComfyUI/folder_paths.py` — `add_model_folder_path()`

```python
def add_model_folder_path(folder_name, full_folder_path, is_default=False):
    folder_name = map_legacy(folder_name)
    if folder_name in folder_names_and_paths:
        # Existing type: append (or prepend if is_default)
        paths, _exts = folder_names_and_paths[folder_name]
        if full_folder_path not in paths:
            if is_default:
                paths.insert(0, full_folder_path)
            else:
                paths.append(full_folder_path)
    else:
        # New type: create with empty extension filter
        folder_names_and_paths[folder_name] = ([full_folder_path], set())
```

Key behaviors:
- Duplicate paths are silently ignored (but may be reordered if `is_default`)
- New folder types created via YAML get an **empty extension set**, meaning all files are returned (vs hardcoded types which filter by `.safetensors`, `.ckpt`, etc.)
- The path list order determines search priority — `get_full_path()` returns the first match

## Hardcoded Folder Types

**File:** `ComfyUI/folder_paths.py` (module-level)

These are registered at import time under `{base_path}/models/`:

| Folder Type | Default Subdirectory | Extensions |
|------------|---------------------|------------|
| `checkpoints` | `models/checkpoints` | `.ckpt .pt .pt2 .bin .pth .safetensors .pkl .sft` |
| `configs` | `models/configs` | `.yaml` |
| `loras` | `models/loras` | (same as checkpoints) |
| `vae` | `models/vae` | (same) |
| `text_encoders` | `models/text_encoders` + `models/clip` | (same) |
| `diffusion_models` | `models/unet` + `models/diffusion_models` | (same) |
| `clip_vision` | `models/clip_vision` | (same) |
| `style_models` | `models/style_models` | (same) |
| `embeddings` | `models/embeddings` | (same) |
| `diffusers` | `models/diffusers` | `folder` |
| `vae_approx` | `models/vae_approx` | (same as checkpoints) |
| `controlnet` | `models/controlnet` + `models/t2i_adapter` | (same) |
| `gligen` | `models/gligen` | (same) |
| `upscale_models` | `models/upscale_models` | (same) |
| `latent_upscale_models` | `models/latent_upscale_models` | (same) |
| `custom_nodes` | `custom_nodes` | (none) |
| `hypernetworks` | `models/hypernetworks` | (same as checkpoints) |
| `photomaker` | `models/photomaker` | (same) |
| `classifiers` | `models/classifiers` | `""` |
| `model_patches` | `models/model_patches` | (same as checkpoints) |
| `audio_encoders` | `models/audio_encoders` | (same) |

Custom nodes and extensions can register additional types at runtime via `add_model_folder_path()`.

## How the Desktop App Uses This

**File:** `desktop/src/config/comfyServerConfig.ts`

The desktop app generates its own `extra_models_config.yaml` (stored in `app.getPath('userData')`) and passes it via `--extra-model-paths-config`:

```typescript
// comfyServer.ts — coreLaunchArgs
'extra-model-paths-config': ComfyServerConfig.configPath,
```

### Config Generation

The desktop maintains a `knownModelKeys` array — a hardcoded list of folder type names:

```typescript
const knownModelKeys = [
  'checkpoints', 'classifiers', 'clip', 'clip_vision', 'configs',
  'controlnet', 'diffusers', 'diffusion_models', 'embeddings', 'gligen',
  'hypernetworks', 'loras', 'photomaker', 'style_models', 'text_encoders',
  'unet', 'upscale_models', 'vae', 'vae_approx',
  // Custom node model types (hardcoded workaround)
  'animatediff_models', 'animatediff_motion_lora', 'animatediff_video_formats',
  'ipadapter', 'liveportrait', 'insightface', 'layerstyle', 'LLM',
  'Joy_caption', 'sams', 'blip', 'CogVideo', 'xlabs', 'instantid',
];
```

`getBaseModelPathsFromRepoPath()` generates a path entry for each key:

```typescript
for (const key of knownModelKeys) {
  paths[key] = path.join(repoPath, 'models', key) + path.sep;
}
```

This produces a YAML like:

```yaml
comfyui_desktop:
  is_default: "true"
  base_path: /path/to/user/data
  custom_nodes: custom_nodes/
  checkpoints: /path/to/comfyui/models/checkpoints/
  loras: /path/to/comfyui/models/loras/
  # ... one entry per knownModelKeys item
```

### The Sync Problem

The desktop's `knownModelKeys` list is **already out of sync** with `folder_paths.py`:
- **Missing from desktop**: `latent_upscale_models`, `model_patches`, `audio_encoders` (recently added to ComfyUI)
- **Extra in desktop**: `animatediff_*`, `ipadapter`, `liveportrait`, etc. (custom node types, flagged with a TODO comment)

This is the exact same problem a launcher would face.

## Proposed ComfyUI Change: Wildcard Base Path

### Problem

Every consumer of `extra_model_paths.yaml` (desktop app, launcher, user configs) must enumerate every folder type explicitly. When ComfyUI adds new types (e.g., `audio_encoders`, `model_patches`), existing configs silently miss them. There's no way to say "use this directory as a model root for all types."

### Proposed Solution

Add a new `all_model_folders` flag. When set to `true` alongside `base_path`, it automatically registers `{base_path}/{folder_name}` for every model folder type already in `folder_names_and_paths` at load time. `custom_nodes` is excluded — sharing custom nodes between installations causes dependency conflicts, and this feature is intended for model files. The flag can be combined with explicit folder entries to add additional paths for specific types.

**YAML usage:**

```yaml
# Covers all current and future model folder types
launcher_global_store:
    base_path: /shared/models
    is_default: true
    all_model_folders: true
```

This would register `/shared/models/checkpoints`, `/shared/models/loras`, `/shared/models/vae`, etc. for every type in the registry. Explicit entries can be added alongside to provide extra paths for specific types:

```yaml
launcher_global_store:
    base_path: /shared/models
    is_default: true
    all_model_folders: true
    checkpoints: /extra/checkpoints    # adds an additional path for checkpoints
```

### Implementation

**Branch:** [`jk/all-model-folders-extra-paths`](https://github.com/Comfy-Org/ComfyUI/tree/jk/all-model-folders-extra-paths)

**File:** `ComfyUI/utils/extra_config.py`

The change is minimal — after popping the special keys (`base_path`, `is_default`, `all_model_folders`), if the flag is set, iterate all registered model folder types before processing any explicit entries:

```python
        is_default = False
        if "is_default" in conf:
            is_default = conf.pop("is_default")
        # NEW: all_model_folders flag
        all_model_folders = False
        if "all_model_folders" in conf:
            all_model_folders = conf.pop("all_model_folders")
        if all_model_folders and base_path:
            for folder_name in list(folder_paths.folder_names_and_paths.keys()):
                if folder_name == "custom_nodes":
                    continue
                full_path = os.path.normpath(os.path.join(base_path, folder_name))
                logging.info("Adding extra search path {} {}".format(folder_name, full_path))
                folder_paths.add_model_folder_path(folder_name, full_path, is_default)
        # Explicit entries are still processed normally after the wildcard
        for x in conf:
            ...
```

### Behavior Details

- Requires explicit opt-in via `all_model_folders: true` — a `base_path` without the flag behaves exactly as before
- Uses `folder_names_and_paths.keys()` at call time, so it covers all types registered so far (hardcoded + any previously loaded configs)
- The subdirectory names match the canonical folder type names (e.g., `checkpoints`, `loras`, `diffusion_models`) — NOT the filesystem defaults which sometimes differ (e.g., ComfyUI uses `models/unet` for `diffusion_models`)
- Can be combined with explicit folder entries — the wildcard runs first, then explicit entries add additional paths
- `custom_nodes` is excluded from wildcard expansion — sharing custom nodes between installations causes dependency conflicts, and this feature targets model files
- Fully backwards-compatible — existing configs without `all_model_folders` are unaffected

### Impact

| Consumer | Before | After |
|----------|--------|-------|
| Desktop `knownModelKeys` | Must list all types, already out of sync | Could use wildcard config, always in sync |
| Launcher YAML | Must enumerate every type manually | Single `base_path` line covers everything |
| User configs | Must copy/update from example file | Simple wildcard for shared model dirs |

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                        ComfyUI Startup                              │
│                                                                     │
│  1. folder_paths.py (module load)                                   │
│     └─ Register 21 hardcoded types under models/                    │
│                                                                     │
│  2. main.py: apply_custom_paths()                                   │
│     ├─ Load extra_model_paths.yaml (if in ComfyUI root)             │
│     └─ Load --extra-model-paths-config files (CLI arg)              │
│                                                                     │
│  3. utils/extra_config.py: load_extra_path_config()                 │
│     ├─ Parse YAML → config groups                                   │
│     ├─ Resolve base_path (expandvars, expanduser, relative)         │
│     ├─ [PROPOSED] Wildcard: base_path only → all registered types   │
│     └─ For each folder_name: path → add_model_folder_path()         │
│                                                                     │
│  4. folder_paths.py: add_model_folder_path()                        │
│     ├─ map_legacy() (unet→diffusion_models, clip→text_encoders)     │
│     ├─ is_default? → insert at index 0                              │
│     └─ else → append                                                │
│                                                                     │
│  Result: folder_names_and_paths dict                                │
│     { "checkpoints": (["/local/models/checkpoints",                 │
│                         "/shared/models/checkpoints"], {exts}) }     │
└─────────────────────────────────────────────────────────────────────┘

Consumers:
  get_full_path(folder, file)  → first match across all paths
  get_filename_list(folder)    → deduplicated list from all paths
  GET /models                  → list all folder type names
  GET /models/{folder}         → list files in that type
```

## Launcher Implications

For ComfyUI-Launcher to implement a global model store:

1. **Create a shared models directory** (e.g., `~/.comfyui-launcher/models/`) with subdirectories matching ComfyUI's folder type names (`checkpoints/`, `loras/`, `vae/`, etc.)

2. **Generate a YAML config** pointing to it — with the proposed `all_model_folders` flag, this is just:
   ```yaml
   launcher_shared:
       base_path: ~/.comfyui-launcher/models
       is_default: true
       all_model_folders: true
   ```

3. **Pass it when spawning ComfyUI** via `--extra-model-paths-config /path/to/launcher_models.yaml`

4. **Without the proposed change**, the launcher must enumerate every folder type explicitly and keep the list in sync — the same problem the desktop has today with `knownModelKeys`. With `all_model_folders: true`, the YAML is future-proof

5. **The `GET /models` endpoint** can be used at runtime to discover all registered types, which could help generate or validate the YAML
