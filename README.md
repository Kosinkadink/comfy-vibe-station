# ComfyUI Workspace

This workspace contains the official [Comfy-Org](https://github.com/Comfy-Org) repositories for ComfyUI development.

## Repositories

| Directory | Repository | Description |
|-----------|-----------|-------------|
| `ComfyUI/` | [Comfy-Org/ComfyUI](https://github.com/Comfy-Org/ComfyUI) | Core backend — the graph/nodes engine, model loading, and execution runtime (Python) |
| `ComfyUI_frontend/` | [Comfy-Org/ComfyUI_frontend](https://github.com/Comfy-Org/ComfyUI_frontend) | Official frontend — the Vue/TypeScript web UI |
| `ComfyUI-Manager/` | [Comfy-Org/ComfyUI-Manager](https://github.com/Comfy-Org/ComfyUI-Manager) | Extension manager — install, remove, and manage custom nodes |
| `desktop/` | [Comfy-Org/desktop](https://github.com/Comfy-Org/desktop) | Desktop application — Electron wrapper for Windows & macOS |
| `ComfyUI-Launcher/` | [Kosinkadink/ComfyUI-Launcher](https://github.com/Kosinkadink/ComfyUI-Launcher) | Electron app for managing multiple ComfyUI installations |
| `ComfyUI-Launcher-Environments/` | [Kosinkadink/ComfyUI-Launcher-Environments](https://github.com/Kosinkadink/ComfyUI-Launcher-Environments) | Environment definitions for ComfyUI-Launcher |
| `workflow_templates/` | [Comfy-Org/workflow_templates](https://github.com/Comfy-Org/workflow_templates) | Official workflow templates and subgraph blueprints for ComfyUI |
| `docs/` | [Comfy-Org/docs](https://github.com/Comfy-Org/docs) | Documentation site — source for [docs.comfy.org](https://docs.comfy.org) (Mintlify) |
| `pyisolate/` | [Comfy-Org/pyisolate](https://github.com/Comfy-Org/pyisolate) | Python extension isolation — run extensions in isolated venvs with seamless RPC |
| `comfy-kitchen/` | [Comfy-Org/comfy-kitchen](https://github.com/Comfy-Org/comfy-kitchen) | Fast kernel library for diffusion inference with multiple compute backends |
| `comfy-aimdo/` | [Comfy-Org/comfy-aimdo](https://github.com/Comfy-Org/comfy-aimdo) | AI Model Dynamic Offloader — PyTorch VRAM allocator with on-demand weight offloading |

## Workspace Documentation

Cross-repo notes, architecture walkthroughs, and research docs live as `.md` files in the **root** of this repository (not in `docs/`, which is a cloned repo). Examples:

- [`desktop-model-downloads.md`](desktop-model-downloads.md) — How the desktop app downloads models directly to `models/` directories
- [`manager-restart.md`](manager-restart.md) — How ComfyUI-Manager restart is handled by the launcher vs desktop

## Cloned On

**2026-02-16** from the default branches of each repository.
