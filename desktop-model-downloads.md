# Desktop Model Downloads

When a user loads a workflow template that references models not present locally, the frontend shows a **Missing Models** dialog. The desktop (Electron) app downloads models directly into the correct ComfyUI `models/` subdirectory, while the browser version can only trigger a standard browser download to the user's Downloads folder.

## Summary

The frontend detects at build time whether it's running inside the Electron desktop app (`isDesktop`). When it is, the Missing Models dialog renders the `ElectronFileDownload` component instead of the browser's `FileDownload` component. On click, the download request travels through a Pinia store → IPC preload bridge → Electron main process `DownloadManager`, which writes the file directly to `models/{directory}/{filename}` using Electron's `session.downloadURL()` API. The browser fallback simply creates an `<a>` tag that opens the URL, sending the file to the user's Downloads folder.

## Detailed Flow

### 1. Missing Models Dialog

**File:** `ComfyUI_frontend/src/components/dialog/content/MissingModelsWarning.vue`

When a workflow is loaded, the backend reports any models it can't find. The frontend displays them in a `ListBox`. Each model entry includes:

- `directory` — the model subdirectory (e.g. `diffusion_models`, `checkpoints`, `clip`)
- `name` — the filename (e.g. `wan2.1_t2v_1.3B_bf16.safetensors`)
- `url` — the download URL (HuggingFace, CivitAI, etc.)

Before rendering download controls, the dialog validates:

- The URL starts with an allowed source (`https://huggingface.co/`, `https://civitai.com/`, or `http://localhost:`) or is in a hardcoded whitelist.
- The filename ends with `.safetensors` or `.sft`.

The dialog then conditionally renders the download component:

```vue
<Suspense v-if="isDesktop">
  <ElectronFileDownload :url="option.url" :label="option.label" :error="option.error" />
</Suspense>
<FileDownload v-else :url="option.url" :label="option.label" :error="option.error" />
```

The `label` prop is formatted as `"{directory} / {filename}"` (e.g. `"diffusion_models / model.safetensors"`).

### 2a. Browser Path — `FileDownload.vue`

**File:** `ComfyUI_frontend/src/components/common/FileDownload.vue`

In the browser, clicking Download calls `triggerBrowserDownload()` from the `useDownload` composable:

```typescript
const triggerBrowserDownload = () => {
  const link = document.createElement('a')
  link.href = url
  link.download = fileName || url.split('/').pop() || 'download'
  link.target = '_blank'
  link.rel = 'noopener noreferrer'
  link.click()
}
```

This creates a temporary `<a>` element and clicks it. The browser handles the download with its default behavior — the file goes to the user's Downloads folder. The user must then manually move it to the correct `models/` subdirectory.

### 2b. Desktop Path — `ElectronFileDownload.vue`

**File:** `ComfyUI_frontend/src/components/common/ElectronFileDownload.vue`

The desktop component splits the `label` prop into `savePath` and `filename`:

```typescript
const [savePath, filename] = props.label.split('/')
```

On click, it calls the Pinia store:

```typescript
const triggerDownload = async () => {
  await electronDownloadStore.start({
    url: props.url,
    savePath: savePath.trim(),   // e.g. "diffusion_models"
    filename: filename.trim()     // e.g. "model.safetensors"
  })
}
```

The component also shows rich UI that the browser version doesn't have:

- **Progress bar** with percentage during download
- **Pause/Resume** buttons
- **Cancel** button
- **Green checkmark** on completion

### 3. Electron Download Store

**File:** `ComfyUI_frontend/src/stores/electronDownloadStore.ts`

A Pinia store that bridges the renderer process to Electron's native APIs:

- Maintains a reactive `downloads` array with `{url, filename, progress, status}` entries.
- On initialization, fetches any in-progress downloads via `getAllDownloads()` and registers an `onDownloadProgress` callback to receive IPC updates.
- Proxies user actions to `window.electronAPI.DownloadManager`:
  - `start()` → `startDownload(url, savePath, filename)`
  - `pause()` → `pauseDownload(url)`
  - `resume()` → `resumeDownload(url)`
  - `cancel()` → `cancelDownload(url)`

### 4. IPC Preload Bridge

**File:** `desktop/src/preload.ts`

The preload script exposes a `DownloadManager` namespace on `window.electronAPI` via `contextBridge.exposeInMainWorld()`:

```typescript
DownloadManager: {
  startDownload: (url, path, filename) =>
    ipcRenderer.invoke(IPC_CHANNELS.START_DOWNLOAD, { url, path, filename }),
  pauseDownload: (url) =>
    ipcRenderer.invoke(IPC_CHANNELS.PAUSE_DOWNLOAD, url),
  resumeDownload: (url) =>
    ipcRenderer.invoke(IPC_CHANNELS.RESUME_DOWNLOAD, url),
  cancelDownload: (url) =>
    ipcRenderer.invoke(IPC_CHANNELS.CANCEL_DOWNLOAD, url),
  onDownloadProgress: (callback) =>
    ipcRenderer.on(IPC_CHANNELS.DOWNLOAD_PROGRESS, (_event, progress) => callback(progress)),
  getAllDownloads: () =>
    ipcRenderer.invoke(IPC_CHANNELS.GET_ALL_DOWNLOADS),
}
```

This bridge runs in an isolated context — the renderer never has direct access to Node.js or Electron APIs.

### 5. Download Manager (Main Process)

**File:** `desktop/src/models/DownloadManager.ts`

The core of the feature. A singleton that manages the full download lifecycle in Electron's main process.

#### Security Validation

Before starting any download, two checks must pass:

1. **Path containment** — The resolved save path must be inside the `models/` directory (prevents path traversal):
   ```typescript
   private isPathInModelsDirectory(filePath: string): boolean {
     const absoluteFilePath = path.resolve(filePath)
     const absoluteModelsDir = path.resolve(this.modelsDirectory)
     return absoluteFilePath.startsWith(absoluteModelsDir)
   }
   ```

2. **File type** — Only `.safetensors` files are allowed (checked against both the URL pathname and the filename).

#### Download Lifecycle

1. **Start** — `startDownload(url, savePath, filename)`:
   - Computes the final path: `{modelsDirectory}/{savePath}/{filename}`
   - Computes a temp path: `{modelsDirectory}/{savePath}/Unconfirmed {filename}.tmp`
   - Skips if the file already exists
   - Resumes if a download for this URL is already in progress but paused
   - Calls `session.defaultSession.downloadURL(url)` to begin the download

2. **Will-Download Event** — Electron fires this when the download starts:
   - Sets the save path to the temp file location
   - Captures the `DownloadItem` handle for pause/resume/cancel

3. **Progress** — The `updated` event fires as data arrives:
   - Computes `progress = receivedBytes / totalBytes`
   - Sends `DOWNLOAD_PROGRESS` IPC event to the renderer with `{url, progress, status, filename, savePath}`

4. **Completion** — The `done` event fires when the download finishes:
   - On `completed`: atomically renames temp file to final path via `fs.renameSync()`
   - On failure: reports error status, cleans up temp file via `fs.unlinkSync()`

5. **Pause/Resume/Cancel** — Forward directly to `DownloadItem.pause()`, `.resume()`, `.cancel()`

#### IPC Handler Registration

```typescript
ipcMain.handle(IPC_CHANNELS.START_DOWNLOAD, (event, { url, path, filename }) =>
  this.startDownload(url, path, filename))
ipcMain.handle(IPC_CHANNELS.PAUSE_DOWNLOAD, (event, url) => this.pauseDownload(url))
ipcMain.handle(IPC_CHANNELS.RESUME_DOWNLOAD, (event, url) => this.resumeDownload(url))
ipcMain.handle(IPC_CHANNELS.CANCEL_DOWNLOAD, (event, url) => this.cancelDownload(url))
ipcMain.handle(IPC_CHANNELS.GET_ALL_DOWNLOADS, () => this.getAllDownloads())
```

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────┐
│                    Renderer Process                      │
│                                                          │
│  MissingModelsWarning.vue                                │
│    ├─ isDesktop? ──► ElectronFileDownload.vue             │
│    │                   └─► electronDownloadStore (Pinia)  │
│    │                         └─► window.electronAPI       │
│    └─ browser?  ──► FileDownload.vue                      │
│                       └─► <a href="..."> (Downloads/)     │
└──────────────────────────┬──────────────────────────────-─┘
                           │ IPC (contextBridge)
┌──────────────────────────▼──────────────────────────────-─┐
│                     Main Process                          │
│                                                           │
│  DownloadManager (singleton)                              │
│    ├─ Validates path ∈ models/                            │
│    ├─ Validates .safetensors extension                    │
│    ├─ session.downloadURL(url)                            │
│    ├─ Writes to temp: models/{dir}/Unconfirmed {name}.tmp │
│    ├─ Sends progress via IPC_CHANNELS.DOWNLOAD_PROGRESS   │
│    └─ fs.renameSync(temp, models/{dir}/{name})            │
└───────────────────────────────────────────────────────────┘
```

## Key Difference: Desktop vs Browser

| Aspect | Desktop (Electron) | Browser |
|--------|-------------------|---------|
| Download target | `models/{directory}/{filename}` | User's Downloads folder |
| Progress tracking | Real-time progress bar | None |
| Pause/Resume | Supported | Not supported |
| Cancel | Supported | Not supported |
| User action needed | None — model is ready to use | Must manually move file |
| Mechanism | `session.downloadURL()` + `setSavePath()` | `<a>` tag click |
