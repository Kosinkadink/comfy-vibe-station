# Workspace Structure

This directory is a **multi-repo workspace** — it is itself a git repository, but most of its subdirectories are **independent git repositories** with their own history, branches, and remotes.

## Nested Repositories

The following subdirectories are separate git repos (cloned from [Comfy-Org](https://github.com/Comfy-Org)):

- `ComfyUI/` — Core backend (Python)
- `ComfyUI_frontend/` — Frontend web UI (Vue/TypeScript)
- `ComfyUI-Manager/` — Extension manager
- `desktop/` — Electron desktop app
- `ComfyUI-Launcher/` — Launcher for managing installations
- `ComfyUI-Launcher-Environments/` — Environment definitions for the launcher
- `workflow_templates/` — Official workflow templates
- `docs/` — Documentation site source
- `embedded-docs/` — Built-in help pages for core nodes
- `pyisolate/` — Python extension isolation
- `comfy-kitchen/` — Fast kernel library for diffusion inference
- `comfy-aimdo/` — AI Model Dynamic Offloader
- `comfyui-benchmark/` — Benchmarking extension

## Important Notes

- **Git operations** (commit, branch, log, etc.) apply to whichever repo's directory you are in. Running `git` from the workspace root operates on the outer wrapper repo, not the nested ones.
- **Cross-repo docs** (`.md` files in the workspace root) belong to the wrapper repo and describe how the nested repos relate to each other.
- See `README.md` for links to each repository's GitHub remote and descriptions.
