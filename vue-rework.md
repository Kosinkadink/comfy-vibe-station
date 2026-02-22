# ComfyUI-Launcher Vue Rework Plan

ComfyUI-Launcher is being transitioned to TypeScript + Vite + Vue 3.

## Stack

### Core
- **electron-vite 5.x** — unified Electron + Vite build (main/preload/renderer)
- **Vite 7.x** — bundler (unless electron-vite requires different)
- **Vue 3** — renderer framework (`<script setup lang="ts">` style)
- **TypeScript** — strict mode, no `any`
- **Pinia** — state management
- **vue-i18n** — internationalization (replaces custom `data-i18n` walker)
- **VueUse** — composable utilities

### UI
- **Tailwind CSS 4** — replaces `renderer/styles.css`
- **lucide-vue-next** — replaces inline SVG sprite in `index.html`
- **SVGO** — SVG asset optimization at build time

### Electron
- **electron-store** — replaces custom `settings.js` / `installations.js` JSON read/write
- **electron-log** — structured logging in main process

### Tooling (post-rework)
- **ESLint** + **Prettier** — linting and formatting
- **Vitest** — unit testing

---

## Current Architecture

| Layer | Files | Description |
|-------|-------|-------------|
| **Main process** | `main.js`, `settings.js`, `installations.js`, `lib/*.js` (20 files) | CJS, Electron app lifecycle, IPC handlers, process management |
| **Preload** | `preload.js` | CJS, exposes 30+ IPC methods to renderer via `contextBridge` |
| **Renderer** | `index.html` + `renderer/*.js` (15 files) | Vanilla DOM manipulation on `window.Launcher` namespace |
| **Locales** | `locales/en.json`, `locales/zh.json` | i18n message bundles |

---

## Target Structure

```
src/
  main/
    index.ts                  ← Electron entry (window, tray, quit)
    ipc.ts                    ← IPC handler registration
    settings.ts               ← electron-store wrapper
    installations.ts           ← electron-store wrapper
    lib/
      actions.ts
      cache.ts
      comfyui-releases.ts
      copy.ts
      delete.ts
      download.ts
      extract.ts
      fetch.ts
      git.ts
      gpu.ts
      i18n.ts
      installer.ts
      migrate.ts
      models.ts
      paths.ts
      process.ts
      release-cache.ts
      updater.ts
      util.ts
  preload/
    index.ts                  ← contextBridge with typed API
  renderer/
    main.ts                   ← Vue app entry
    App.vue                   ← App shell (sidebar + content + modal layer)
    components/
      InstanceCard.vue        ← reusable card (from buildCard/buildManageBtn/etc.)
      DetailSection.vue       ← collapsible section with fields/items/actions
      SettingField.vue        ← dynamic field renderer (text/select/boolean/path/pathList/number)
      DirCard.vue             ← model directory card with actions
      ProgressStep.vue        ← stepped progress indicator
      UpdateBanner.vue        ← auto-update banner
      ModalDialog.vue         ← generic modal (alert/confirm/prompt/select)
    views/
      InstallationList.vue    ← list.js (main view with filter tabs, drag reorder)
      DetailModal.vue         ← detail.js (installation detail overlay)
      RunningView.vue         ← running.js (running/errors/in-progress sections)
      SettingsView.vue        ← settings.js (dynamic settings form)
      ModelsView.vue          ← models.js (model directory management)
      NewInstallModal.vue     ← new-install.js (cascading source/field selects)
      TrackModal.vue          ← track.js (track existing installation)
      ConsoleModal.vue        ← console.js (terminal output view)
      ProgressModal.vue       ← progress.js (flat/stepped progress with terminal)
    composables/
      useElectronApi.ts       ← typed wrapper around window.api with lifecycle cleanup
      useModal.ts             ← Promise-based modal API (confirm/alert/prompt/select)
      useTheme.ts             ← theme application via VueUse useColorMode or manual
    stores/
      sessionStore.ts         ← running instances, active sessions, error instances (merged from util.js + sessions.js)
      installationStore.ts    ← installation list cache + actions
    types/
      ipc.ts                  ← IPC channel types shared across all layers
      installation.ts         ← Installation, Source, Action, DetailSection interfaces
  locales/
    en.json
    zh.json
```

---

## Phases

### Phase 1: Scaffold build pipeline ✅ COMPLETE

Set up electron-vite + TypeScript + Vue tooling.

**What was done:**
- `electron.vite.config.ts` — main/preload/renderer build with Vue + Tailwind plugins
- `tsconfig.json` (root project refs) + `tsconfig.node.json` (main/preload) + `tsconfig.web.json` (renderer)
- `src/main/index.ts` — minimal Electron main process (BrowserWindow, dev URL / prod file loading)
- `src/preload/index.ts` + `index.d.ts` — typed contextBridge with placeholder `ping` API
- `src/renderer/index.html` — HTML entry with CSP (allows ws: for HMR)
- `src/renderer/src/main.ts` — Vue app entry with Pinia
- `src/renderer/src/App.vue` — minimal scaffold component with Tailwind classes
- `src/renderer/src/assets/main.css` — Tailwind CSS 4 import
- `src/renderer/src/env.d.ts` — Vite client + Vue module type declarations
- `electron-builder.yml` — build config (replaces `build` section from package.json)
- `resources/icon.png` — app icon for electron-vite
- Updated `package.json` — `main: "./out/main/index.js"`, electron-vite scripts, all new deps
- Updated `.gitignore` — added `out/`

**Verified:** `electron-vite build` ✅, `tsc` typecheck ✅, `vue-tsc` typecheck ✅

**Old files preserved** (still functional, will be removed after Phase 2):
- `main.js`, `preload.js`, `index.html`, `renderer/*.js`, `settings.js`, `installations.js`, `lib/*.js`

---

### Phase 2: Renderer — Vue 3 rewrite

Convert the 15 renderer files into Vue components + Pinia stores. This is ~70% of the total work.

**Branch:** `vue-rework` in `ComfyUI-Launcher`
**Working directory:** `c:/Users/Kosinkadink/workplaces/002/comfy-vibe-station/ComfyUI-Launcher`
**Build command:** `npx electron-vite build` (must pass after changes)
**Typecheck:** `npm run typecheck` (must pass after changes)

**Old renderer files to read** (these contain all the logic to port):
- `renderer/util.js` — READ FIRST: view switching, state maps (running/sessions/errors), card helpers, theme
- `renderer/sessions.js` — session buffer for terminal output, IPC listeners for comfy-output/comfy-exited
- `renderer/modal.js` — Promise-based modal system (alert/confirm/confirmWithOptions/prompt/select)
- `renderer/i18n.js` — custom i18n with `data-i18n` attribute walker and `window.t()` helper
- `renderer/list.js` — installation list with filtering, drag reorder, action buttons, async rendering
- `renderer/detail.js` — installation detail modal with sections, editable fields, action chains
- `renderer/running.js` — running/errors/in-progress instances view
- `renderer/settings.js` — dynamic settings form from API-driven sections
- `renderer/models.js` — model directory management with dir cards
- `renderer/new-install.js` — new installation wizard with cascading field selects
- `renderer/track.js` — track existing installation by probing a directory
- `renderer/console.js` — terminal output view for running instances
- `renderer/progress.js` — progress modal (flat/stepped modes) with terminal output
- `renderer/update-banner.js` — auto-update state machine (available/downloading/ready/error)
- `renderer/styles.css` — full CSS (design tokens, card styles, modal styles, etc.)

**Also read for context:**
- `preload.js` — current IPC API surface (30+ methods) that `window.api` exposes
- `index.html` — current HTML structure (sidebar, views, modals, SVG sprite)
- `locales/en.json` — i18n message keys

**New files to create/modify** are under `src/renderer/src/`. The Vue app entry (`main.ts`) and `App.vue` already exist from Phase 1.

Phase 2 should be done in sub-phases (2a → 2b → 2c → 2d) to keep changes manageable. Each sub-phase should result in a buildable, typechecking state.

#### 2a: Core infrastructure

| Old | New | Notes |
|-----|-----|-------|
| `renderer/i18n.js` | `vue-i18n` plugin in `main.ts` | `useI18n()` composable, `$t()` in templates |
| `renderer/util.js` (view switching) | Reactive `activeView` ref in `App.vue` | Sidebar nav binds to it, views use `v-show` |
| `renderer/util.js` (state maps) | `stores/sessionStore.ts` | `_runningInstances`, `_activeSessions`, `_errorInstances` → Pinia reactive maps |
| `renderer/util.js` (card helpers) | `components/InstanceCard.vue` | Props for name/meta/draggable, slots for actions |
| `renderer/sessions.js` | Merge into `stores/sessionStore.ts` | IPC listeners in store `setup()` via `useElectronApi` |
| `renderer/modal.js` | `components/ModalDialog.vue` + `composables/useModal.ts` | Keep Promise-based API |

#### 2b: Views → Components

| Old | New | Key changes |
|-----|-----|-------------|
| `renderer/list.js` | `views/InstallationList.vue` | `v-for` + `computed` filtering, `_renderGen` pattern eliminated by reactivity |
| `renderer/detail.js` | `views/DetailModal.vue` | `DetailSection` sub-component, `v-model` for editable fields |
| `renderer/running.js` | `views/RunningView.vue` | `computed` sections from store — auto-updates |
| `renderer/settings.js` | `views/SettingsView.vue` | `SettingField` component with `v-model` per type |
| `renderer/models.js` | `views/ModelsView.vue` | `DirCard` component, path list reactivity |
| `renderer/new-install.js` | `views/NewInstallModal.vue` | `watch()` on upstream selections triggers downstream reload |
| `renderer/track.js` | `views/TrackModal.vue` | Probe results as reactive ref |
| `renderer/console.js` | `views/ConsoleModal.vue` | Terminal output via ref, auto-scroll via `nextTick` or VueUse `useScroll` |
| `renderer/progress.js` | `views/ProgressModal.vue` | `ProgressStep` sub-component, per-operation state in store |
| `renderer/update-banner.js` | `components/UpdateBanner.vue` | Reactive state ref, `v-if` per state type |

#### 2c: IPC bridge

`composables/useElectronApi.ts`:
- Typed wrapper around `window.api`
- IPC event listeners auto-cleanup via `onUnmounted`
- Types imported from `src/types/ipc.ts`

#### 2d: Tailwind migration

- Replace `renderer/styles.css` with Tailwind utility classes in Vue templates
- Extract shared design tokens (colors, spacing) into `tailwind.config.ts` or CSS `@theme`
- Component-scoped styles via `<style scoped>` only where Tailwind doesn't cover it

**Deliverable**: Full renderer feature parity in Vue. Delete `renderer/`, old `index.html`.

---

### Phase 3: Main process → TypeScript

Convert main process CJS files to TypeScript ESM, file by file.

**Order** (leaf dependencies first):
1. `lib/paths.ts`, `lib/util.ts`
2. `lib/gpu.ts`, `lib/git.ts`, `lib/download.ts`, `lib/extract.ts`, `lib/copy.ts`
3. `lib/cache.ts`, `lib/models.ts`, `lib/process.ts`
4. `lib/fetch.ts`, `lib/release-cache.ts`, `lib/comfyui-releases.ts`
5. `lib/installer.ts`, `lib/actions.ts`, `lib/delete.ts`, `lib/migrate.ts`
6. `settings.ts` → electron-store, `installations.ts` → electron-store
7. `lib/i18n.ts`, `lib/updater.ts`
8. `lib/ipc.ts`
9. `main.ts`

**Per-file changes**:
- `require()` → `import`
- Add parameter/return types
- `module.exports` → named exports
- Define interfaces in `src/types/` for shared data structures
- Replace `console.log` with electron-log
- Replace manual JSON file I/O (`settings.js`, `installations.js`) with electron-store

**Deliverable**: Main process compiles as TypeScript. No behavior changes.

---

### Phase 4: Preload → TypeScript with shared types

1. Convert `preload.js` → `src/preload/index.ts`
2. Define `IpcApi` interface in `src/types/ipc.ts` — shared between preload and renderer
3. Type all `ipcRenderer.invoke()` calls and event payloads
4. Renderer's `useElectronApi` composable imports these same types
5. End-to-end type safety: changing an IPC channel signature produces compile errors in all three layers

**Deliverable**: Fully typed IPC contract across main ↔ preload ↔ renderer.

---

### Phase 5: Polish and cleanup

1. **Delete old files**: `renderer/*.js`, old `index.html`, root-level `main.js`, `preload.js`, `settings.js`, `installations.js`, `lib/*.js`
2. **ESLint + Prettier** config — follow Comfy org conventions
3. **Vitest tests** for:
   - Pinia stores (sessionStore, installationStore)
   - Composables (useModal, useElectronApi)
   - Key components (InstanceCard, DetailSection, SettingField)
4. **Update electron-builder config** for new output paths
5. **Update CI/GitHub Actions** for new build/lint/test commands
6. **Update `package.json`** `files` list

---

## Execution

| Phase | Effort | Dependencies |
|-------|--------|-------------|
| Phase 1: Scaffold | Small | None — must go first |
| Phase 2: Renderer (Vue) | **Large** | Phase 1 |
| Phase 3: Main → TS | Medium | Phase 1 (parallel with Phase 2) |
| Phase 4: Preload → TS | Small | Phase 3 |
| Phase 5: Polish | Small | All others |

Phases 2 and 3 can run in parallel after Phase 1, since main process and renderer are separate electron-vite build targets.
