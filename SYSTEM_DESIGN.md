# Download Manager — Full System Design
**Author:** Keith Paul Kato
**Version:** 2.1
**Type:** Open Source, Local Desktop Application
**Stack:** Electron + React + TypeScript · Python FastAPI · SQLite · Manifest V3

---

## Table of Contents

1. [Vision and Goals](#1-vision-and-goals)
2. [Architecture Overview](#2-architecture-overview)
3. [Technology Stack](#3-technology-stack)
4. [Domain Layer](#4-domain-layer)
5. [Application Layer](#5-application-layer)
6. [Infrastructure Layer](#6-infrastructure-layer)
7. [API Specification](#7-api-specification)
8. [Database Schema](#8-database-schema)
9. [Detailed Workflows](#9-detailed-workflows)
10. [Browser Extension Design](#10-browser-extension-design-manifest-v3)
11. [Desktop UI Structure](#11-desktop-ui-structure)
12. [Electron Security](#12-electron-security)
13. [Security and Ethics](#13-security-and-ethics)
14. [Configuration and Environment](#14-configuration-and-environment)
15. [Logging and Observability](#15-logging-and-observability)
16. [Testing Strategy](#16-testing-strategy)
17. [Implementation Roadmap](#17-implementation-roadmap)
18. [Folder Structure](#18-folder-structure)
19. [Definition of Done](#19-definition-of-done)

---

## 1. Vision and Goals

Build a professional, open-source, browser-integrated download manager with an IDM-like workflow. The system must be modular, testable, and easy to extend by contributors. Because the app is open source and local-only, it ships without an authentication layer — all logic runs on the user's machine.

### 1.1 Core Goals
- Add downloads from the desktop UI or browser extension
- Support pause, resume, retry, queue management, and scheduling
- Use HTTP Range requests for segmented, resumable, parallel downloading
- Show real-time progress, speed, ETA, and status via WebSocket
- Persist all state in SQLite across sessions
- Follow Clean Architecture so every layer is independently testable
- Never bypass DRM, paid access, or copyright protections

### 1.2 Non-Goals
- No cloud sync, no remote management, no telemetry
- No HLS/DASH stream capture or media extraction
- No multi-user accounts, sharing, or collaboration features
- HTTP/HTTPS only in v1 — FTP, BitTorrent, and other protocols are out of scope

---

## 2. Architecture Overview

The system follows **Clean Architecture** with an **Event-Driven Progress** layer on top.

```
┌─────────────────────────────────────────────────┐
│             External Clients                     │
│   Desktop UI (Electron/React)                   │
│   Browser Extension (Manifest V3)               │
├─────────────────────────────────────────────────┤
│           Interface Adapters                     │
│   REST Controllers · WebSocket Gateway           │
│   Extension Bridge · DTO Mappers                │
├─────────────────────────────────────────────────┤
│           Application Layer                      │
│   Use Cases · Ports · Queue Service             │
│   Scheduler · Progress Service                  │
├─────────────────────────────────────────────────┤
│             Domain Layer                         │
│   Entities · Value Objects · Policies           │
│   Domain Events · Business Rules                │
├─────────────────────────────────────────────────┤
│           Infrastructure Layer                   │
│   HTTP Client · File System · SQLite            │
│   Thread Pool · Notifications · OS Bridge       │
└─────────────────────────────────────────────────┘
```

**Dependency Rule:** Dependencies point inward only. The domain layer never imports FastAPI, React, SQLite, or OS-specific code.

---

## 3. Technology Stack

| Layer | Technology | Reason |
|---|---|---|
| Desktop Shell | Electron (context-isolated) | Cross-platform desktop with web UI |
| UI Framework | React + TypeScript | Component-based, type-safe UI |
| Styling | Tailwind CSS + shadcn/ui | Clean, reusable components |
| Backend API | Python FastAPI | Fast local API, native WebSocket |
| Download Engine | Python httpx + asyncio | Async HTTP, range requests, streaming |
| Concurrency | asyncio + ThreadPoolExecutor | Non-blocking parallel segment workers |
| Database | SQLite + Alembic | Lightweight local storage with migrations |
| Schema Migrations | Alembic | Versioned SQLite schema upgrades |
| Browser Extension | Manifest V3 | Chrome and Edge support |
| IPC | REST (commands) + WebSocket (events) | Separation of concerns |
| Packaging | electron-builder | Cross-platform installer |

---

## 4. Domain Layer

The domain layer is pure Python. No framework imports.

### 4.1 Entities

#### DownloadTask
```
id: UUID
url: str
file_name: str
save_path: str
total_size: int | None
downloaded_size: int
status: DownloadStatus
resume_supported: bool
segment_count: int
category: str
speed_limit: int | None
created_at: datetime
started_at: datetime | None
completed_at: datetime | None
error_message: str | None
checksum: str | None
checksum_algorithm: str | None  (md5 | sha256 | None)
```

#### DownloadSegment
```
id: UUID
download_id: UUID
segment_index: int
start_byte: int
end_byte: int
downloaded_bytes: int
temp_file_path: str
status: SegmentStatus
retry_count: int
last_error: str | None
```

#### DownloadQueue
```
id: UUID
name: str
max_parallel_downloads: int
status: QueueStatus  (Active | Paused | Stopped)
speed_limit: int | None
```

### 4.2 Value Objects and Enums

```python
class DownloadStatus(Enum):
    PENDING = "pending"
    QUEUED = "queued"
    DOWNLOADING = "downloading"
    PAUSED = "paused"
    MERGING = "merging"
    COMPLETED = "completed"
    FAILED = "failed"
    CANCELLED = "cancelled"

class SegmentStatus(Enum):
    PENDING = "pending"
    DOWNLOADING = "downloading"
    COMPLETED = "completed"
    FAILED = "failed"
    RETRYING = "retrying"
```

### 4.3 Policies

| Policy | Responsibility |
|---|---|
| SegmentationPolicy | Decides segment count based on file size, server Range support, and user settings |
| RetryPolicy | Controls per-segment retry count, backoff delay, and maximum total retries |
| SpeedLimitPolicy | Calculates token-bucket rate for a given speed limit in bytes/sec |
| ChecksumPolicy | Determines when and how to verify file integrity after merge |

### 4.4 Domain Events

```
DownloadCreated(download_id)
DownloadStarted(download_id)
DownloadPaused(download_id, saved_bytes)
DownloadResumed(download_id)
DownloadCompleted(download_id, file_path)
DownloadFailed(download_id, error)
DownloadCancelled(download_id)
SegmentFailed(download_id, segment_index, error, will_retry: bool)
SegmentCompleted(download_id, segment_index)
MergeStarted(download_id)
MergeCompleted(download_id, checksum_verified: bool)
```

---

## 5. Application Layer

### 5.1 Use Cases

| Use Case | Steps |
|---|---|
| AddDownloadUseCase | Validate URL → Probe metadata → Create DownloadTask → Plan segments → Save → Queue |
| StartDownloadUseCase | Load task → Apply SegmentationPolicy → Create workers → Start ProgressService |
| PauseDownloadUseCase | Signal workers to stop → Persist segment byte positions → Keep temp files |
| ResumeDownloadUseCase | Load segments → Resume each from saved position (if Range supported) |
| CancelDownloadUseCase | Stop workers → Delete temp files → Mark cancelled |
| RetryDownloadUseCase | Check RetryPolicy → Reset failed segments → Re-queue or re-start |
| ScheduleDownloadUseCase | Assign start time or queue rule → Persist schedule |
| DeleteDownloadUseCase | Stop if active → Delete temp files and record → Optionally delete final file |
| ChangeSettingsUseCase | Validate settings → Persist → Propagate to active services |

### 5.2 Ports (Interfaces)

```python
class DownloadRepository(Protocol):
    def save(self, task: DownloadTask) -> None: ...
    def get_by_id(self, id: UUID) -> DownloadTask | None: ...
    def list(self, filters: DownloadFilters) -> list[DownloadTask]: ...
    def update_status(self, id: UUID, status: DownloadStatus) -> None: ...
    def delete(self, id: UUID) -> None: ...

class SegmentRepository(Protocol):
    def save_all(self, segments: list[DownloadSegment]) -> None: ...
    def get_by_download(self, download_id: UUID) -> list[DownloadSegment]: ...
    def update_progress(self, id: UUID, downloaded_bytes: int) -> None: ...

class MetadataProbe(Protocol):
    async def probe(self, url: str) -> FileMetadata: ...

class EventBus(Protocol):
    def publish(self, event: DomainEvent) -> None: ...
    def subscribe(self, event_type: type, handler: Callable) -> None: ...

class NotificationService(Protocol):
    def notify(self, title: str, body: str) -> None: ...
```

### 5.3 Progress Service

The ProgressService subscribes to segment events and emits real-time snapshots:

```python
@dataclass
class ProgressSnapshot:
    download_id: UUID
    total_size: int | None
    downloaded_bytes: int
    speed_bps: float          # bytes per second (rolling 3s window)
    eta_seconds: float | None
    percent: float | None
    active_segments: int
    status: DownloadStatus
```

Speed is calculated using a **rolling 3-second window** to avoid jitter.

---

## 6. Infrastructure Layer

### 6.1 HTTP Download Worker

Each segment worker:
1. Opens an HTTP request with `Range: bytes=start-end`
2. Streams the response into a temp file at the saved byte offset
3. Reports progress bytes to ProgressService every 512 KB
4. On network error: applies RetryPolicy (exponential backoff, max 5 retries per segment)
5. On success: marks segment as Completed and emits SegmentCompleted

```python
CHUNK_SIZE_BYTES = 512 * 1024  # 512 KB — balances syscall overhead vs progress granularity

async def download_segment(segment: DownloadSegment, task: DownloadTask) -> None:
    resume_offset = segment.start_byte + segment.downloaded_bytes
    headers = {"Range": f"bytes={resume_offset}-{segment.end_byte}"}
    async with httpx.AsyncClient() as client:
        async with client.stream("GET", task.url, headers=headers) as response:
            response.raise_for_status()
            async for chunk in response.aiter_bytes(chunk_size=CHUNK_SIZE_BYTES):
                write_chunk_to_temp_file(segment.temp_file_path, chunk)
                report_progress(segment.id, len(chunk))
```

### 6.2 File Merger

After all segments complete:
1. Sort segments by index
2. Open final output file for writing
3. Append each temp file in order
4. Delete all temp files
5. Verify checksum if task has an expected checksum
6. Emit MergeCompleted(checksum_verified)

**Checksum verification** uses MD5 or SHA-256 depending on what the server provided in response headers (`Content-MD5`, `Digest`).

### 6.3 Metadata Probe

```python
@dataclass
class FileMetadata:
    file_name: str
    total_size: int | None
    content_type: str
    resume_supported: bool       # True if server returns Accept-Ranges: bytes
    suggested_segments: int      # Based on SegmentationPolicy
    checksum: str | None
    checksum_algorithm: str | None
```

The probe sends a HEAD request first. Falls back to GET with stream if HEAD is blocked.

### 6.4 SQLite Repository

Uses raw SQLite via `aiosqlite` for async access. Schema versioned with Alembic.

### 6.5 Error Recovery Matrix

| Error | Segment Behavior | Task Behavior |
|---|---|---|
| Network timeout | Retry with backoff (max 5 per segment) | Continue other segments |
| 416 Range Not Satisfiable | Restart segment from byte 0 | Continue |
| 503 / 429 | Retry after Retry-After header delay | Continue |
| Disk full | Pause all workers | Mark task Failed with disk error |
| All segments exhausted retries | — | Mark task Failed, notify user |
| Merge checksum mismatch | — | Mark task Failed, keep temp files for re-merge |

---

## 7. API Specification

### REST Endpoints

| Method | Path | Body / Params | Response |
|---|---|---|---|
| POST | /api/downloads | `{url, save_path?, category?, speed_limit?}` | DownloadDTO |
| GET | /api/downloads | `?status=&category=&page=&limit=` | Page\<DownloadDTO\> |
| GET | /api/downloads/{id} | — | DownloadDTO |
| POST | /api/downloads/{id}/start | — | DownloadDTO |
| POST | /api/downloads/{id}/pause | — | DownloadDTO |
| POST | /api/downloads/{id}/resume | — | DownloadDTO |
| POST | /api/downloads/{id}/cancel | — | DownloadDTO |
| POST | /api/downloads/{id}/retry | — | DownloadDTO |
| DELETE | /api/downloads/{id} | `?delete_file=true/false` | 204 |
| GET | /api/queues | — | list\<QueueDTO\> |
| POST | /api/queues | `{name, max_parallel, speed_limit?}` | QueueDTO |
| PUT | /api/queues/{id} | `{max_parallel, status}` | QueueDTO |
| GET | /api/settings | — | SettingsDTO |
| PUT | /api/settings | `{key: value, ...}` | SettingsDTO |
| WS | /ws/progress | — | ProgressSnapshot stream |

### WebSocket Event Format

```json
{
  "event": "progress",
  "download_id": "uuid",
  "downloaded_bytes": 52428800,
  "total_size": 104857600,
  "speed_bps": 5242880,
  "eta_seconds": 10.0,
  "percent": 50.0,
  "status": "downloading",
  "active_segments": 4
}
```

Other event types: `download.created`, `download.completed`, `download.failed`, `download.paused`, `merge.started`, `merge.completed`

---

## 8. Database Schema

### downloads
```sql
CREATE TABLE downloads (
    id TEXT PRIMARY KEY,
    url TEXT NOT NULL,
    file_name TEXT NOT NULL,
    save_path TEXT NOT NULL,
    total_size INTEGER,
    downloaded_size INTEGER NOT NULL DEFAULT 0,
    status TEXT NOT NULL DEFAULT 'pending',
    category TEXT NOT NULL DEFAULT 'general',
    speed_limit INTEGER,
    resume_supported INTEGER NOT NULL DEFAULT 0,
    segment_count INTEGER NOT NULL DEFAULT 1,
    checksum TEXT,
    checksum_algorithm TEXT,
    error_message TEXT,
    created_at TEXT NOT NULL,
    started_at TEXT,
    completed_at TEXT
);
```

### segments
```sql
CREATE TABLE segments (
    id TEXT PRIMARY KEY,
    download_id TEXT NOT NULL REFERENCES downloads(id) ON DELETE CASCADE,
    segment_index INTEGER NOT NULL,
    start_byte INTEGER NOT NULL,
    end_byte INTEGER NOT NULL,
    downloaded_bytes INTEGER NOT NULL DEFAULT 0,
    temp_file_path TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'pending',
    retry_count INTEGER NOT NULL DEFAULT 0,
    last_error TEXT
);
```

### queues
```sql
CREATE TABLE queues (
    id TEXT PRIMARY KEY,
    name TEXT NOT NULL UNIQUE,
    max_parallel_downloads INTEGER NOT NULL DEFAULT 3,
    status TEXT NOT NULL DEFAULT 'active',
    speed_limit INTEGER
);
```

### queue_items
```sql
CREATE TABLE queue_items (
    id TEXT PRIMARY KEY,
    queue_id TEXT NOT NULL REFERENCES queues(id),
    download_id TEXT NOT NULL REFERENCES downloads(id),
    position INTEGER NOT NULL,
    priority INTEGER NOT NULL DEFAULT 0
);
```

### settings
```sql
CREATE TABLE settings (
    key TEXT PRIMARY KEY,
    value TEXT NOT NULL
);
```

### schema_version (managed by Alembic)
```sql
CREATE TABLE alembic_version (
    version_num TEXT NOT NULL
);
```

---

## 9. Detailed Workflows

### 9.1 Add Download
```
User/Extension → POST /api/downloads
  → AddDownloadUseCase
    → Validate URL format
    → MetadataProbe.probe(url)
      → HEAD request → parse Content-Length, Accept-Ranges, Content-Type, filename
    → SegmentationPolicy.plan(metadata, settings)
    → Create DownloadTask (status=PENDING)
    → Create DownloadSegments
    → DownloadRepository.save(task)
    → SegmentRepository.save_all(segments)
    → QueueService.enqueue(task)
    → EventBus.publish(DownloadCreated)
    → WebSocket broadcasts download.created
  → Return DownloadDTO
```

Key decisions in this flow:
- If `Content-Length` is unknown, the task is still created, but progress percent and ETA remain `None` until the size becomes known.
- If the server does not support `Accept-Ranges`, the SegmentationPolicy creates a single segment and marks `resume_supported=false`.
- If filename metadata is missing, the application derives a safe filename from the URL path or falls back to a generated default.

### 9.2 Segmented Download
```
StartDownloadUseCase
  → Load task and segments from DB
  → Mark task DOWNLOADING
  → For each segment:
      → Create SegmentWorker(segment, task)
      → Submit to ThreadPoolExecutor
  → ProgressService starts listening to segment events
  → Workers run concurrently:
      → Each writes to temp_{download_id}_{segment_index}.part
      → Reports progress bytes periodically
      → On completion: emits SegmentCompleted
  → When all segments complete:
      → FileMerger.merge(task, segments)
        → Append temp files in order
        → Delete temp files
        → Verify checksum
      → Mark task COMPLETED
      → NotificationService.notify(...)
      → EventBus.publish(DownloadCompleted)
```

Operational notes:
- Segment count is capped by both user settings and file size so the system does not create tiny inefficient segments.
- Each worker writes only inside its byte range and never mutates another segment's temp file.
- Progress updates are throttled before persistence so the DB is not written on every network chunk.
- Final completion only occurs after merge and optional checksum verification succeed.

### 9.3 Pause and Resume
```
Pause:
  → Signal each active worker to stop after current chunk
  → Workers flush current position to DB before stopping
  → Mark task PAUSED
  → Keep all .part files on disk

Resume:
  → Load incomplete segments (downloaded_bytes < end_byte - start_byte)
  → For each: start new worker from start_byte + downloaded_bytes
  → Mark task DOWNLOADING
```

Resume rules:
- If the server previously supported Range but later rejects ranged requests, the resume attempt fails gracefully and the UI offers a restart from zero.
- Temp files are treated as the source of truth only after their byte counts are reconciled with persisted segment progress.
- A resumed task reuses the original queue position unless the user explicitly reprioritizes it.

### 9.4 Segment Failure and Retry
```
SegmentWorker catches exception:
  → If retry_count < RetryPolicy.max_retries:
      → Wait backoff_delay * 2^retry_count seconds
      → Increment retry_count in DB
      → Restart download from last saved byte
      → Emit SegmentFailed(will_retry=True)
  → Else:
      → Mark segment FAILED
      → Emit SegmentFailed(will_retry=False)
      → Check if all segments exhausted
        → If yes: mark task FAILED, notify user
```

Failure handling rules:
- Transient network errors, timeouts, and 5xx responses are retryable by default.
- Permanent errors such as 401, 403, 404, or repeated checksum mismatch mark the download as failed without infinite retries.
- A task is only marked `FAILED` when no viable recovery path remains for the remaining segments.

### 9.5 Queue Scheduling and Concurrency
```
QueueService loop
  → Load active queues ordered by priority
  → For each queue:
      → Count active downloads
      → If active_downloads < max_parallel_downloads:
          → Pick next QUEUED item by position and schedule rules
          → Invoke StartDownloadUseCase
      → Else:
          → Leave remaining items in QUEUED state
  → Sleep short interval or wake on queue change event
```

Scheduling behavior:
- Queue-level concurrency limits are enforced before any task-level workers are created.
- Tasks with a future scheduled start time remain queued but ineligible until their time window opens.
- Global limits can additionally cap total active downloads across all queues.

### 9.6 Cancel and Cleanup
```
User → POST /api/downloads/{id}/cancel
  → CancelDownloadUseCase
    → Signal active workers to stop
    → Wait for in-flight chunk writes to finish safely
    → Delete segment temp files if present
    → Mark task CANCELLED
    → Persist terminal state and timestamps
    → EventBus.publish(DownloadCancelled)
    → WebSocket broadcasts download.cancelled
```

Cleanup guarantees:
- Partial files are deleted for cancelled tasks unless the product later adds a "keep partial data" option.
- The final merged file is never deleted unless cancellation occurs during post-processing before completion.
- Cancel is idempotent: repeating the request for an already-cancelled task returns success without side effects.

### 9.7 Browser Extension Handoff
```
User clicks "Download with App" in extension
  → Content script collects page URL, media URL, referrer, and cookies allowed by browser policy
  → Background service worker sends payload to local desktop API
  → Desktop API validates origin and payload shape
  → AddDownloadUseCase creates task
  → API returns task id and initial metadata
  → Extension shows success/failure notification to user
```

Boundary rules:
- The extension forwards only the metadata required to reproduce the request locally.
- Restricted headers or protected browser credentials are never fabricated or escalated beyond browser permissions.
- If the desktop app is offline, the extension surfaces a clear local-connection error and does not silently drop the request.

---

## 10. Browser Extension Design (Manifest V3)

### 10.1 Structure
```
browser-extension/
├── manifest.json
├── background.js       (service worker)
├── popup.html
├── popup.js
├── content.js          (optional page scanner)
└── icons/
```

### 10.2 MV3 Keep-Alive Strategy

MV3 service workers terminate after ~30 seconds of inactivity. To maintain localhost connectivity, the extension uses `chrome.alarms` — the only Chrome API guaranteed to wake the worker reliably under MV3:

- Wake the service worker every 24 seconds (`periodInMinutes: 0.4`) — chosen to stay safely under the 30 s idle-shutdown window
- On wake: ping `/api/health` to verify the app is running
- Persist connection status in `chrome.storage.session`
- The popup reads status from session storage; it never calls the worker directly

```javascript
// background.js
const HEALTH_URL = 'http://127.0.0.1:6543/api/health';

chrome.alarms.create('keepAlive', { periodInMinutes: 0.4 });

chrome.alarms.onAlarm.addListener(async (alarm) => {
    if (alarm.name !== 'keepAlive') return;
    try {
        const res = await fetch(HEALTH_URL);
        await chrome.storage.session.set({ connected: res.ok });
    } catch {
        await chrome.storage.session.set({ connected: false });
    }
});
```

### 10.3 Extension Features

| Feature | Implementation |
|---|---|
| Right-click capture | `chrome.contextMenus` on link and page targets |
| Send to app | POST to `http://127.0.0.1:6543/api/downloads` |
| Popup status | Read from `chrome.storage.session` |
| Open desktop app | `chrome.tabs.create` with deep link or custom protocol |
| Permission scope | `contextMenus`, `alarms`, `storage`, `host_permissions: 127.0.0.1:6543` |

---

## 11. Desktop UI Structure

### 11.1 Screen Map
```
App Window
├── Sidebar
│   ├── All Downloads
│   ├── Downloading
│   ├── Completed
│   ├── Paused
│   ├── Failed
│   ├── Categories (General, Video, Audio, Document, Archive, Other)
│   └── Queues
├── Top Toolbar
│   ├── Add Download (primary CTA)
│   ├── Start All / Pause All
│   └── Search / Filter
├── Download List (main area)
│   ├── Table Mode (file name, size, speed, ETA, status, actions)
│   └── Card Mode (visual tiles with progress bar)
├── Details Panel (right, collapsible)
│   ├── Progress circle
│   ├── Speed, ETA, size, segments, resume support
│   ├── URL, save path, category
│   └── Segment progress breakdown
└── Status Bar (bottom)
    ├── Total speed
    ├── Active downloads count
    └── Extension connection status
```

### 11.2 Dialogs
- **Add Download:** URL, file name (auto-filled), save folder, category, speed limit (optional), Start Now / Download Later / Cancel
- **Settings:** Default folder, max connections, global speed limit, extension toggle, theme, notifications, startup behavior

### 11.3 Visual Design System
- Status colors: `blue` → downloading, `green` → completed, `yellow` → paused, `red` → failed, `gray` → pending
- Progress bar fills animated with smooth transition
- Speed shown as human-readable: KB/s, MB/s
- ETA formatted: `2m 34s`, `< 1 minute`, `Unknown`

---

## 12. Electron Security

| Setting | Value | Reason |
|---|---|---|
| `contextIsolation` | `true` | Prevents renderer from accessing Node.js directly |
| `nodeIntegration` | `false` | Prevents malicious downloads from calling Node APIs |
| `preload script` | Defined | Exposes only a safe `window.api` bridge to renderer |
| `webSecurity` | `true` | Keeps same-origin policy active |
| `allowRunningInsecureContent` | `false` | No mixed content |

The preload script exposes:
```typescript
window.api = {
  downloads: { add, list, start, pause, resume, cancel, retry, delete },
  settings: { get, update },
  queues: { list, create, update },
  onProgress: (callback) => ipcRenderer.on('progress', callback),
}
```

---

## 13. Security and Ethics

### 13.1 Local Security Controls
- API bound to `127.0.0.1:6543` only — not accessible from the network
- File name sanitization prevents path traversal (`../`, absolute paths rejected)
- Executable file types (`.exe`, `.bat`, `.sh`, `.ps1`) trigger a warning dialog before downloading
- Error logs do not store full URLs unnecessarily — truncated to domain only in logs

### 13.2 Ethical Boundaries

| Allowed | Not Allowed |
|---|---|
| Public files, software installers, open datasets | DRM-protected streaming video |
| Files the user has direct URL access to | Paid content behind login |
| Open source assets | Scraping protected platforms |
| Files the user owns | Removing access controls |

The app does not implement stream capture, HLS segment harvesting, or any mechanism to bypass authentication.

---

## 14. Configuration and Environment

The app reads configuration from three sources, in order of precedence: CLI flags → environment variables → `settings` table (SQLite). Defaults are baked into the application; the user does not need to provide any configuration to run it.

### 14.1 Runtime Paths

| Platform | Data Directory |
|---|---|
| Linux | `$XDG_DATA_HOME/download-manager` (fallback: `~/.local/share/download-manager`) |
| macOS | `~/Library/Application Support/DownloadManager` |
| Windows | `%APPDATA%\DownloadManager` |

Inside the data directory:
- `app.db` — SQLite database (created on first run, migrated via Alembic)
- `logs/` — rotating log files
- `temp/` — `.part` files for in-progress downloads (configurable)

### 14.2 Environment Variables

| Variable | Default | Purpose |
|---|---|---|
| `DM_API_HOST` | `127.0.0.1` | Bind address — must remain loopback in production |
| `DM_API_PORT` | `6543` | API + WebSocket port |
| `DM_DATA_DIR` | platform default | Override data directory location |
| `DM_LOG_LEVEL` | `INFO` | `DEBUG`, `INFO`, `WARNING`, `ERROR` |
| `DM_MAX_PARALLEL_DOWNLOADS` | `3` | Global default for new queues |

### 14.3 Port Selection and Conflict Handling

The default API port `6543` is unregistered with IANA, reducing collision risk with common services. On startup the API attempts to bind; if the port is in use, the app:

1. Logs the conflict with the offending port
2. Surfaces a desktop notification with a "Change Port" action
3. Exits with code `2` if no override is provided

The Electron shell reads the active port from a per-user `runtime.json` file written on successful bind, so the renderer and browser extension always reach the correct process.

---

## 15. Logging and Observability

All logs are local-only. The app never transmits log data to any remote endpoint.

### 15.1 Log Levels and Routing

| Sink | Level | Format | Rotation |
|---|---|---|---|
| Console (dev only) | DEBUG+ | Human-readable, colorized | n/a |
| `logs/app.log` | INFO+ | JSON lines | 10 MB × 5 files |
| `logs/error.log` | WARNING+ | JSON lines | 10 MB × 5 files |

### 15.2 Structured Log Fields

Every log line includes:
```
timestamp · level · logger · download_id? · segment_index? · message · error_type? · duration_ms?
```

URLs are truncated to `scheme://host` in log output; query strings and paths are never logged unless `DM_LOG_LEVEL=DEBUG` is set.

### 15.3 Metrics (Local)

A lightweight `/api/metrics` endpoint exposes counters for the desktop UI's debug panel:
- `downloads_total{status}` — count by terminal status
- `bytes_downloaded_total` — lifetime byte counter
- `segment_retries_total{reason}` — retry counts by failure category
- `active_workers` — current segment worker count

No Prometheus scraping, no external collectors — these are surfaced in-app only.

### 15.4 Health Check

`GET /api/health` returns:
```json
{
  "status": "ok",
  "version": "2.1.0",
  "uptime_seconds": 12345,
  "db_migrations_current": true,
  "active_downloads": 2
}
```

Used by the browser extension's keep-alive ping and by the desktop UI's connection indicator.

---

## 16. Testing Strategy

| Type | What | Tool |
|---|---|---|
| Unit | Domain rules, policies, progress math, URL validation | pytest |
| Use Case | All 9 use cases with mocked ports | pytest + unittest.mock |
| Integration | FastAPI endpoints, SQLite repository, WebSocket events | pytest + httpx |
| Download Engine | Range requests, segment merging, retry, network drops | pytest + respx (mock HTTP) |
| Checksum | Merge output matches expected hash | pytest |
| UI Component | Button states, progress display, filter, details panel | Vitest + Testing Library |
| E2E | Add → start → complete full flow on local file server | Playwright |
| Security | Path traversal, localhost binding, unsafe file warning | pytest |

---

## 17. Implementation Roadmap

| Phase | Deliverables | Notes |
|---|---|---|
| 1 · Foundation | Clean Architecture folders, domain entities, SQLite schema with Alembic, first migration | Set up Alembic from day one |
| 2 · Basic Download + Minimal UI | Single-file direct download, progress via WebSocket, minimal Electron window showing progress | Build UI alongside engine so you can test visually |
| 3 · Resume + Checksum | Range request support, pause/resume with temp file persistence, checksum verification after download | |
| 4 · Segmented Download | Segment planning, parallel workers, file merger, per-segment retry | Error recovery matrix fully implemented |
| 5 · Full Desktop UI | Complete dashboard, Add Download dialog, details panel, sidebar, settings | |
| 6 · Browser Extension | Context menu, popup, MV3 keep-alive, localhost communication | |
| 7 · Queue + Scheduler | Queues, priorities, scheduling rules, speed limits | |
| 8 · Hardening | Full test suite, security review, Alembic migration tests, electron-builder packaging, README | |

---

## 18. Folder Structure

```
download-manager/
├── apps/
│   ├── desktop/
│   │   ├── electron/
│   │   │   ├── main.ts
│   │   │   ├── preload.ts
│   │   │   └── ipc-handlers.ts
│   │   └── renderer/
│   │       ├── src/
│   │       │   ├── components/
│   │       │   │   ├── DownloadList/
│   │       │   │   ├── AddDownloadDialog/
│   │       │   │   ├── DetailsPanel/
│   │       │   │   ├── Sidebar/
│   │       │   │   ├── QueueManager/
│   │       │   │   └── Settings/
│   │       │   ├── hooks/
│   │       │   │   ├── useDownloads.ts
│   │       │   │   ├── useProgress.ts
│   │       │   │   └── useSettings.ts
│   │       │   ├── stores/
│   │       │   └── App.tsx
│   │       └── tailwind.config.ts
│   ├── api/
│   │   ├── presentation/
│   │   │   ├── routers/
│   │   │   │   ├── downloads.py
│   │   │   │   ├── queues.py
│   │   │   │   └── settings.py
│   │   │   ├── websocket/
│   │   │   │   └── progress_gateway.py
│   │   │   └── schemas/
│   │   │       ├── download_dto.py
│   │   │       └── progress_dto.py
│   │   ├── application/
│   │   │   ├── use_cases/
│   │   │   │   ├── add_download.py
│   │   │   │   ├── start_download.py
│   │   │   │   ├── pause_download.py
│   │   │   │   ├── resume_download.py
│   │   │   │   ├── cancel_download.py
│   │   │   │   ├── retry_download.py
│   │   │   │   ├── schedule_download.py
│   │   │   │   ├── delete_download.py
│   │   │   │   └── change_settings.py
│   │   │   ├── ports/
│   │   │   │   ├── download_repository.py
│   │   │   │   ├── segment_repository.py
│   │   │   │   ├── metadata_probe.py
│   │   │   │   ├── event_bus.py
│   │   │   │   └── notification_service.py
│   │   │   └── services/
│   │   │       ├── progress_service.py
│   │   │       ├── queue_service.py
│   │   │       └── scheduler_service.py
│   │   ├── domain/
│   │   │   ├── entities/
│   │   │   │   ├── download_task.py
│   │   │   │   ├── download_segment.py
│   │   │   │   └── download_queue.py
│   │   │   ├── value_objects/
│   │   │   │   ├── download_status.py
│   │   │   │   └── segment_status.py
│   │   │   ├── policies/
│   │   │   │   ├── segmentation_policy.py
│   │   │   │   ├── retry_policy.py
│   │   │   │   ├── speed_limit_policy.py
│   │   │   │   └── checksum_policy.py
│   │   │   └── events/
│   │   │       └── domain_events.py
│   │   ├── infrastructure/
│   │   │   ├── http/
│   │   │   │   ├── segment_worker.py
│   │   │   │   ├── metadata_probe_impl.py
│   │   │   │   └── file_merger.py
│   │   │   ├── persistence/
│   │   │   │   ├── sqlite_download_repository.py
│   │   │   │   ├── sqlite_segment_repository.py
│   │   │   │   └── migrations/        (Alembic)
│   │   │   ├── events/
│   │   │   │   └── in_memory_event_bus.py
│   │   │   └── os/
│   │   │       └── os_notification_service.py
│   │   ├── tests/
│   │   │   ├── unit/
│   │   │   ├── use_cases/
│   │   │   ├── integration/
│   │   │   └── security/
│   │   └── main.py
│   └── browser-extension/
│       ├── manifest.json
│       ├── background.js
│       ├── popup.html
│       ├── popup.js
│       ├── content.js
│       └── icons/
├── docs/
│   ├── architecture.md
│   ├── api-spec.md
│   ├── database-schema.md
│   ├── ui-ux-guidelines.md
│   └── contributing.md
├── SYSTEM_DESIGN.md
├── SKILL.md
├── README.md
├── docker-compose.yml          (optional dev environment)
└── .github/
    └── workflows/
        ├── test.yml
        └── build.yml
```

---

## 19. Definition of Done

- [ ] Download can be added from desktop UI and browser extension
- [ ] User can start, pause, resume, cancel, retry, and delete downloads
- [ ] Segmented downloading works on Range-supporting servers
- [ ] Each segment retries independently on failure (max 5 retries, exponential backoff)
- [ ] File merger verifies checksum when available
- [ ] Progress, speed, ETA, and status update live via WebSocket without polling
- [ ] Downloads and settings persist after app restart
- [ ] SQLite schema is versioned with Alembic; migrations run on startup
- [ ] Electron uses context isolation and preload script
- [ ] API is bound to 127.0.0.1 only
- [ ] File names are sanitized; path traversal is blocked
- [ ] Executable downloads trigger a warning dialog
- [ ] MV3 extension uses keep-alive alarm; reconnects if app restarts
- [ ] All use cases have unit and integration tests
- [ ] UI is responsive, readable, and professionally organized
- [ ] electron-builder produces installers for Windows, macOS, and Linux
