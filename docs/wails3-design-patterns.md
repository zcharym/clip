# Wails v3 + React design patterns

A practical design guide distilled from [Clip](https://github.com/clip-rss/clip), a cross-platform RSS reader built with **Go 1.25**, **Wails v3 (alpha)**, **React 18**, **Zustand**, and **SQLite**.

Use this when improving another Wails v3 desktop app. The goal is not to copy Clip’s product features, but to reuse the **layering, binding conventions, event model, local-first data path, and native-OS seams** that keep a Wails app maintainable.

---

## 1. Architecture at a glance

Clip treats Wails as a **thin IPC layer**, not as the application core.

```
┌─────────────────────────────────────────────────────────┐
│  React UI (components, hooks, CSS modules)              │
│       ↓ Zustand stores (optimistic UI, session state)   │
│  frontend/src/Utils/Api  ← single re-export of bindings │
└──────────────────────────┬──────────────────────────────┘
                           │  generated Wails bindings
                           │  (Promise reject on Go error)
┌──────────────────────────▼──────────────────────────────┐
│  api.*Service   ← the only types registered with Wails  │
│       ↓ inject store / fetcher / scheduler / cipher     │
│  internal/*     ← testable Go domain, no Wails imports  │
│       ↓                                                 │
│  SQLite (WAL) + OS APIs (dock, notifications, updater)  │
└─────────────────────────────────────────────────────────┘
         Go → frontend: named events (items:updated, …)
```

**Rules that travel well to other codebases**

| Layer | Owns | Must not own |
|---|---|---|
| `frontend/src/Components` | Rendering, local drafts, focus | Direct `bindings/…` imports |
| `frontend/src/Stores` | Client cache, optimistic writes | SQL, HTTP, OS APIs |
| `frontend/src/Utils/Api` | Binding re-exports, event wrappers, `toApiError` | Product logic |
| `api/` | Wails-bound methods, DTO mapping, i18n of API errors | Business loops, SQL |
| `internal/` | Persistence, fetch, schedule, notify, crypto | `application.Service` types |
| `main.go` | Composition root: construct, inject, register, shutdown | Domain algorithms |

If a package under `internal/` imports `github.com/wailsapp/wails/v3`, the domain is no longer unit-testable without a running app. Clip avoids that by defining small interfaces (`Emitter`, `Notifier`, `Sender`, `FeedStore`) and wiring Wails adapters in `main.go`.

---

## 2. Wails v3 service design

### 2.1 One service per domain

Clip registers services in `main.go`:

- App-owned: `SystemService`, `FeedService`, `ItemService`, `CategoryService`, `SettingsService`, `WebDAVConfigService`, `OPMLService`, `OPMLBackupService`
- Wails-owned: `notifications.NotificationService`, `dock.DockService`

Each app service is a struct with **injected collaborators**, constructed by `NewXService(...)`. Methods are the public API the frontend calls.

Do not put “god services” on the `App` struct. Wails v3 binds **exported methods on registered services**, so a fat `App` becomes an unversioned RPC surface.

### 2.2 Error contract: `(result, error)`, no envelopes

From `api/api.go`:

> Methods return `(result, error)`. Wails turns a non-nil error into a rejected Promise. The frontend uses `try/catch`. There is no custom `{ ok, data, error }` wrapper.

Benefits:

- Generated TypeScript already matches Go signatures.
- Callers cannot ignore failures by checking a boolean they forgot.
- Tests on the Go side stay ordinary `err != nil`.

Frontend unwraps Wails’ `CallError` JSON once, in one helper:

```ts
// frontend/src/Utils/Api/index.ts
export function toApiError(err: unknown): string {
  const raw = err instanceof Error ? err.message : String(err)
  try {
    const message: unknown = JSON.parse(raw)?.message
    if (typeof message === 'string' && message) return message
  } catch { /* not JSON */ }
  return raw
}
```

Wails puts the whole `{"message","cause","kind"}` blob into `Error.message`. Without this helper, every toast shows raw JSON.

### 2.3 Models are store structs with `json` tags

Bound methods take and return `store.Feed`, `store.Item`, `store.Settings`, etc. `wails3 generate bindings` emits matching TypeScript. Frontend re-exports those types from `frontend/src/Types/Models.ts` — business code never imports the generated deep path.

Hide internal fields from the wire with `json:"-"` (Clip hides `Feed.LastAttempted`). Do not ship process-only state to the UI.

### 2.4 Binding generator pitfalls (Clip hit these)

Wails v3 scans **every exported method and every field type** on a service. That has consequences:

1. **Do not put `func` fields on a service.** Binding generation tries to JSON-encode them and warns `function types are not supported by encoding/json`. Use an interface instead (`SettingsObserver`), even if the field is unexported.
2. **Wiring helpers must be package functions, not methods.** `ObserveSettings(svc, observer)` is a package-level function so it is **not** generated as a frontend binding. Any exported method on a service becomes part of the public RPC API.
3. **Inject late-bound callbacks as fields, then wrap them.** `SystemService` holds `CheckUpdateFn`, `OnlineChangedFn`, `LanguageFn` set from `main` after construction. The bound methods (`CheckForUpdates`, `SetOnline`) are thin, serializable wrappers. This breaks circular construction (`Updater` needs `App`, `App` needs services).
4. **Regenerate, never hand-edit** `frontend/bindings/`. After changing `api/` methods or `store` models: `wails3 generate bindings`.

### 2.5 Binding facade (mandatory)

All frontend code imports services from `frontend/src/Utils/Api`:

```ts
export {
  FeedService,
  ItemService,
  CategoryService,
  SettingsService,
  // ...
} from '../../../bindings/github.com/clip-rss/clip/api'
```

When the Go module path or generated folder layout changes, **one file** updates. Components and stores stay stable.

Do the same for event names and payloads (`frontend/src/Types/Events.ts` + `Utils/Api/Events.ts`).

### 2.6 `0` means “none” on the wire

JavaScript has no `*int64`. Clip maps `categoryID == 0` to `nil` via `nullableID()`. Pick one sentinel, document it, and use it everywhere (uncategorized, root folder, “all feeds”).

---

## 3. Push events, don’t poll

Long-running Go work must not be a request the UI waits on. Clip’s scheduler, OPML import, notifications, and updater all **emit named events**.

### 3.1 Keep event names in one place on each side

Go (`internal/scheduler/scheduler.go`):

```go
const (
    ItemsUpdatedEvent   = "items:updated"
    FeedErrorEvent      = "feed:error"
    FeedRefreshingEvent = "feed:refreshing"
)
```

TypeScript (`frontend/src/Types/Events.ts`) duplicates the **string** and the payload shape. There is no generated event schema — treat the strings as a contract and keep them next to comments that name the Go constant.

Namespace custom events (`clip:update:available`, `clip:updater:user:browser`) so they never collide with `wails:updater:*`.

### 3.2 Adapter in `main.go`, interface in domain

```go
type Emitter interface {
    Emit(name string, data any)
}

type wailsEmitter struct{}
func (wailsEmitter) Emit(name string, data any) {
    if app := application.Get(); app != nil {
        app.Event.Emit(name, data)
    }
}
```

`application.Get()` decouples construction order: the scheduler can start before the window exists, and tests inject a `fakeEmitter`.

### 3.3 Subscribe with an unsubscribe

Every `Events.On` wrapper returns a cancel function. Hooks call it in `useEffect` cleanup. Leaked listeners are a classic Wails bug (duplicate toasts, double navigation).

```ts
export function onItemsUpdated(handler: (p: ItemsUpdatedPayload) => void): () => void {
  return Events.On(ItemsUpdatedEvent, (ev) => handler(ev.data as ItemsUpdatedPayload))
}
```

### 3.4 Event-driven reload must not wipe UI state

When `items:updated` fires, Clip’s `ArticleStore.reload()`:

- Re-fetches the **light** list for the current selection
- **Merges previously loaded `content`** so the open article does not flash blank
- Keeps `selectedItemId` if the row still exists
- Consumes `pendingSelectId` from a notification click

A naïve `load()` that resets selection will interrupt reading every auto-refresh.

---

## 4. Local-first data (SQLite)

Desktop apps should remain useful with the network down. Clip’s source of truth is `<UserConfigDir>/clip/clip.db`.

### 4.1 Engine choices that matter

- `modernc.org/sqlite` (pure Go, no CGo) — simpler cross-compile
- `MaxOpenConns(1)` — SQLite + multiple writers is a footgun
- `PRAGMA journal_mode=WAL`, `foreign_keys=ON`, `busy_timeout=5000`, `synchronous=NORMAL`

### 4.2 Schema for “sync from the internet”

Patterns worth copying even if the product is not RSS:

| Concern | Clip’s approach |
|---|---|
| Identity of remote rows | `seen_items (feed_id, item_key)` survives prune, so deleted rows do not reappear as “new” |
| Multiple keys | `url:` + `source:` fingerprint (GUID / hash) — digest feeds reuse one URL |
| Retention | Per-feed `max_items`; delete oldest after insert |
| Search | FTS5 trigram on the fields users actually query (title, summary, note — **not** full HTML) |
| Settings | Single JSON blob in a KV table (`settings` key `"app"`), not dozens of columns |
| Migrations | `user_version` pragma, additive, tested |

### 4.3 Light vs full payloads

List endpoints return `ItemLight` (no `content`). Opening a row calls `GetItem`. Crossing the Wails bridge with megabytes of HTML for a virtualized list is the fastest way to make a desktop UI feel like a slow website.

On the Go side, keep two column lists (`itemColumns` / `itemColumnsLight`) rather than `SELECT *` and stripping in JS.

### 4.4 Restore while the DB is open

You cannot overwrite a live SQLite file. Clip **stages** a validated backup as `clip.db.pending` and swaps it on next launch (`applyPendingRestore`), including WAL/SHM cleanup. Any other Wails app that offers “restore backup” needs the same two-phase dance.

### 4.5 Secrets do not live in plaintext in the DB

Backup/export of `clip.db` is a support path. WebDAV passwords are AES-GCM encrypted with a sibling `.synckey` (0600). The comments in `internal/secret` are honest about the threat model: this stops a leaked DB file, **not** an attacker who can read the whole config directory. If a product needs that, use OS keychain/DPAPI.

Cipher init failure **disables that feature**, it does not crash the app.

---

## 5. Frontend state (Zustand)

### 5.1 Stores map 1:1 to UI columns / concerns

| Store | Responsibility | Persistence |
|---|---|---|
| `SettingsStore` | Backend settings; optimistic `update` with rollback | SQLite via Go |
| `ThemeStore` | Resolved theme + native title bar | Settings + first-paint cache |
| `LayoutStore` | Column widths, focus mode, note drawer | Widths only (`partialize`) |
| `SidebarStore` | Feeds, folders, selection, unread | None (reload from Go) |
| `ArticleStore` | Items, search, notes, selection | None |
| `ReaderStore` | Typography | Settings |
| `SearchHistoryStore` | Recent queries | `localStorage` |
| `ToastStore` / `UpdateStore` / `BackupStore` | Ephemeral chrome | None |

**Session vs durable:** focus mode and the note drawer are intentionally **not** persisted. Restoring “fullscreen reading” on launch is surprising. Persist layout chrome; keep modes ephemeral.

### 5.2 Optimistic writes with rollback

Star, read, notes, and settings all:

1. Patch local state
2. Call the bound method
3. On failure, restore previous fields and set `error: toApiError(err)`

Settings also has `applyExternal()` for “backend already wrote this” (sync pull). Calling `update()` again would echo the value back to the server.

### 5.3 Cross-store wiring without a god store

- `ArticleStore` calls `SidebarStore.load()` after read-state changes (unread badges).
- `ThemeStore` **subscribes** to `SettingsStore` so a language/theme pulled from backup applies without a restart.
- `App.tsx` subscribes to settings for `i18n.changeLanguage` and CSS flags (`reduce-motion`, focus rings).

Prefer `store.subscribe` over React context for “backend setting changed → side effect”. Context re-renders the tree; a subscription can be surgical.

### 5.4 Legacy migration is a boot step

Prefs that used to live in `localStorage` (theme, reader typography) migrate **once after** `SettingsStore.load()`, then the backend is canonical. Order matters: migrate-before-load would overwrite server state with empty defaults.

---

## 6. UI composition

### 6.1 Slot layout, not a router

Clip is a single window. `App` passes slots into `Layout` (`toolbar`, `sidebar`, `list`, `reader`). Overlays (`FocusMode`, modals, toasts) are **siblings**, not nested in the third column. That keeps `position: fixed` and keyboard isolation simple.

Resizable columns: a `Divider` reports mouse delta; the store clamps. Reader is `flex: 1`.

### 6.2 Virtualize long lists

`@tanstack/react-virtual` with a fixed row height (88px). Combined with `ListItemsLight` and a client-side cap (2000 rows), the middle column stays cheap. If a Wails app lists files, logs, or messages, copy this pair: **light DTO + virtualizer**.

### 6.3 Sanitize untrusted HTML twice

- Go sanitizes on ingest (`internal/fetcher/sanitize.go`) before SQLite.
- React sanitizes on render (`DOMPurify` in `Utils/Sanitize.ts`) and forces `rel="noopener noreferrer"` + lazy images.

Never `dangerouslySetInnerHTML` with feed/email/markdown HTML that only one layer cleaned. External links go through `Browser.OpenURL` with a `window.open` fallback — the WebView must not navigate away from the app.

### 6.4 Crash boundary at the root

`CrashBoundary` catches React errors, `window.error`, and `unhandledrejection`, then shows a copy-pasteable report (platform, arch, version from `System.Environment()` + `SystemService.Version()`). Desktop users cannot “open DevTools and tell you the red box”.

---

## 7. Offline and background work

### 7.1 Assume offline until the WebView says otherwise

On startup Clip sets `scheduler.SetOfflineMode(true)` **before** `sch.Start()`. The WebView has not reported `navigator.onLine` yet; a burst of HTTP on a dead network is wasted work and noisy errors.

Frontend `useOnlineStatus` listens to `online`/`offline` and calls `SystemService.SetOnline`, which forwards to the scheduler. UI shows an `OfflineBanner`. Cached SQLite content remains readable.

Caveat (documented in the hook): `navigator.onLine` means “has a local network interface”, not “can reach the internet”.

### 7.2 Scheduler pattern (copy this)

- Tick loop (1 minute) + wake-on-online
- Per-entity interval **or** global default; `0` = manual only
- Exponential backoff on error (cap 24h)
- Global concurrency semaphore + per-entity lock
- Conditional HTTP (ETag / Last-Modified) kept in **process memory**
- Domain interfaces (`FeedStore`, `FeedFetcher`, `Emitter`, `Notifier`) so tests need no Wails

Do not run background HTTP from React `setInterval`. The window can be minimized, frozen, or gone; Go owns the process lifetime.

### 7.3 Don’t notify on the first sync

Clip notifies only when `NewItems > 0` **and** `feed.LastUpdated != nil`. The first fetch after subscribe would otherwise spam the OS. Any “watch remote resource” app needs the same distinction.

### 7.4 Shutdown order

```go
app.OnShutdown(func() {
    sch.Stop()
    api.StopOPMLBackup(opmlBackupSvc)
    _ = st.Close()
})
```

Stop producers, then interrupt in-flight jobs, then close SQLite. Closing the DB while the scheduler still writes corrupts WAL.

---

## 8. Native OS integration

Keep platform quirks **out of components**. Isolate them in small hooks and `//go:build` files.

### 8.1 Platform from Go, not from UA sniffing

`SystemService.Platform()` returns `"mac"` | `"windows"` from `runtime.GOOS`. `usePlatform()` is `null` until that Promise resolves — badge/title-bar code must no-op on `null`.

### 8.2 Notifications

| Piece | Where |
|---|---|
| Decision (`each` / `summary` / `off`, collapse after 5) | Pure `notify.PlanLocalized` — unit tested |
| Send | `Sender` interface; `main.go` wraps Wails `NotificationService` |
| Permission | `RequestNotificationAuthorization` on `ApplicationStarted` |
| Click | `OnNotificationResponse` → unminimise/focus window → emit `notification:open` |
| Frontend | `useNotificationNavigation`: `GetItem` → `scheduleSelect` → sidebar `select` |

Put the article/entity id in `NotificationOptions.Data` (`articleId`). Click handling is otherwise guesswork.

### 8.3 Dock / taskbar badges

Unread lives in SQLite; the **frontend** sums `SidebarStore.feeds[].unreadCount` and calls `DockService`. That sounds backwards until you notice:

- The sidebar already has the counts (one source of truth).
- macOS wants a number; Windows overlay is ~16px so Clip draws a **solid red dot** (`SetCustomBadge` with matching text/background colour).
- Honor a user setting (`showUnreadBadge`) and remove the badge at 0.

### 8.4 Native chrome vs CSS theme

- macOS: `MacTitleBarHiddenInset` + translucent backdrop — CSS theme is enough.
- Windows: `SystemService.SetTheme` in `system_windows.go` calls `w32.SetTheme(hwnd, isDark)`. Non-Windows file is a no-op (`//go:build !windows`).

Call `SetTheme` whenever the resolved theme changes, including first paint (the inline script only sets a CSS class).

### 8.5 Persist window size on close

Read `WindowWidth`/`WindowHeight` from settings at startup (with min clamps). On `WindowClosing`, write them back. Do not persist minimized or sub-minimum sizes.

### 8.6 Menus call the same functions as the UI

“Check for Updates…” in the application menu and `SystemService.CheckForUpdates()` share `updCtrl.check()`. Duplicate menu vs button logic diverges immediately.

---

## 9. Internationalization

Split by **who produces the string**:

| Producer | Location |
|---|---|
| React UI | `frontend/src/I18n/locales/{en,zh,zh-TW}.json` via i18next |
| Go API errors, dialogs, notifications, native menus | `internal/i18n` map |
| Software Update **subwindow** | Same locale JSON, `updater` segment injected into HTML at window create time |

Do not maintain a third copy of strings inside `window.html`. Clip panics at startup if the `updater` segment is missing — fail loud in development.

Language is stored in settings. `App` applies it on boot **and** subscribes so a sync/backup of language updates the UI without restart. Backend error messages call `backendLanguage(store)` at the moment of the error, not at process start.

Boot language before settings load: `navigator.languages` (zh-TW/HK/MO/Hant → `zh-TW`, other zh → `zh`, else `en`).

---

## 10. Theme without a white flash

Wails’ first paint happens **before** any bound call returns. If the true theme lives in SQLite, the window will flash the default background.

Clip’s three-layer theme:

1. **Inline script in `index.html`** synchronously reads `localStorage['clip-theme']` and adds `theme-dark` / `theme-sepia` on `<html>`.
2. **`PrefsCache.ts`** is the only owner of that cache format (key + version). The inline script must stay compatible; comments in both files say so.
3. **`ThemeStore`** applies the same class, calls `SystemService.SetTheme`, writes the cache, and persists via `SettingsStore.update({ theme })`. `system` listens to `prefers-color-scheme`.

Class names must **not** be Tailwind’s `dark` / `sepia` utilities — those apply real CSS filters to the whole document. Clip uses `theme-dark` / `theme-sepia`.

Reader background is independent of app theme (a user may want a sepia article on a dark chrome).

---

## 11. Keyboard-driven UI

One `window` `keydown` listener (`useHotkeys`):

- Normalize combos as `mod+…` so Ctrl and Cmd share bindings.
- Skip IME composition (`e.isComposing`).
- Skip editable targets unless `allowInInput`.
- Skip Radix modals (`[role="dialog"][data-state="open"]`) so Esc/focus trap still work.
- Focus overlay is `role="dialog"` **without** `data-state`, so global shortcuts still run there.

Keep a read-only cheat sheet in Settings that is generated from the same combo strings. **If the sheet lists a key, it must be bound** (Clip currently documents `j`/`k` as general next/prev article, but those keys are only wired inside focus mode — a documentation/implementation drift to avoid).

---

## 12. Software updates (Wails Updater)

Clip does **not** use `updater.CheckAndInstall` (that downloads immediately). It:

- Sets `updater.WindowNone` and builds its own window from an embedded HTML template
- Checks only; the user chooses Install / Close / “open in browser”
- Rebuilds the window every check — Wails destroys webview windows on close; `Reload()` is a no-op on macOS, so reused windows stick on stale JS state
- Replays the last updater event when the new window sends `window:ready` (fixes the race where Check finishes before the page subscribed)
- Uses a **separate HTTP transport** for update downloads vs feed fetches (long file vs many small requests; same user proxy, different connection pool)
- Cleans orphaned `wails-update-*` dirs in `os.TempDir()` on startup (download finished, user quit without restart)

If another app only needs “notify me”, a silent `Check` + `clip:update:available` event is enough. Do not block startup on the network.

---

## 13. Testing strategy that survives Wails

Wails bindings and the webview are integration concerns. Clip keeps them off the unit-test critical path.

**Go**

- Domain packages take interfaces; tests use fakes (`fakeEmitter`, temp SQLite via `NewWithPath`).
- Pure functions for policy (`notify.Plan`, OPML parse, backoff).
- `go test ./...` from the README.

**Frontend**

- Vitest + jsdom, **no** Wails/Tailwind Vite plugins in `vitest.config.ts`.
- Stores and helpers (`toApiError`, `badgeAction`, `comboFromEvent`, article filters) are tested without a window.
- Do not import generated `bindings/**` from tests; import the facade and mock it if needed.

**What not to unit-test:** `main.go` window options, generated bindings, OS notification permission dialogs. Those belong in a manual/smoke pass on each OS.

---

## 14. Feature → implementation map (Clip-specific, for orientation)

How the advertised product features sit on the patterns above:

| Feature | Frontend | Go |
|---|---|---|
| Three-column layout | `Layout` + `LayoutStore` + virtual `ArticleList` | `ListItemsLight` / `GetItem` |
| Subscriptions + folders | `SidebarStore`, add/edit modals | `FeedService`, `CategoryService` |
| OPML | Settings / backup UI | `OPMLService` (txn import, no fetch), optional WebDAV |
| Scheduled fetch | Event subscriptions, refresh actions | `fetcher` (custom RSS/Atom) + `scheduler` |
| Focused reading | `ReadingView` + `FocusMode` overlay | Full `Item.Content` in SQLite |
| Search | Toolbar debounce → `runSearch` | FTS5 on title/summary/note |
| Notes | `NotePanel` 500ms debounce | `ItemService.AddNote` |
| Notifications + badge | `useNotificationNavigation`, `useDockBadge` | `notify` + Wails services |
| Shortcuts | `useAppHotkeys` | — |
| Theme + i18n | ThemeStore, i18next, locale JSON | `internal/i18n`, `SetTheme`, updater HTML inject |
| Offline | `useOnlineStatus`, `OfflineBanner` | Scheduler offline flag, local `content` |

---

## 15. Checklist for another Wails v3 codebase

Use this as a review list, not a rewrite plan.

### Composition and bindings

- [ ] `main.go` is a composition root; domain lives in `internal/`
- [ ] One Wails service per domain; constructors inject dependencies
- [ ] Methods return `(T, error)`; no ad-hoc result envelopes
- [ ] Frontend imports bindings from **one** facade module
- [ ] Types re-exported from generated models; no deep `bindings/github.com/...` in components
- [ ] No `func` fields on bound structs; wiring helpers are package functions
- [ ] `wails3 generate bindings` is documented; `frontend/bindings/` is not hand-edited
- [ ] `0` / empty-string sentinels for optional IDs are documented

### Events and background work

- [ ] Long jobs emit named events; UI does not poll
- [ ] Event name strings exist on both sides with typed payloads
- [ ] Domain `Emitter` interface; Wails adapter in `main`
- [ ] Every `Events.On` has an unsubscribe in React cleanup
- [ ] Event-driven reloads preserve selection and already-fetched heavy fields
- [ ] Process starts in offline mode until the WebView reports online
- [ ] Shutdown: stop scheduler → cancel jobs → close DB

### Data and performance

- [ ] Local DB is the read path; network fills it
- [ ] List APIs omit large blobs (`content`, file bytes, HTML)
- [ ] Long lists are virtualized
- [ ] Search indexes the fields users type into, not raw HTML
- [ ] Backup restore is staged for next launch
- [ ] Secrets are not plaintext in the exported DB

### Native desktop

- [ ] Platform comes from Go (`runtime.GOOS`), not user-agent
- [ ] Theme first paint does not flash (inline script + cache)
- [ ] Windows title bar theme is set from the same preference as CSS
- [ ] Notifications: permission, click-to-focus, payload id, no first-sync spam
- [ ] Badge respects a user setting and clears at zero
- [ ] Window size persisted with min clamps
- [ ] Menu items call the same functions as in-app buttons

### UI resilience

- [ ] Optimistic mutations roll back on rejected Promises
- [ ] `toApiError` unwraps Wails `CallError` JSON
- [ ] Untrusted HTML sanitized on ingest **and** render
- [ ] External URLs open in the system browser
- [ ] Root crash boundary + copyable report
- [ ] i18n: UI strings in the frontend, API/OS strings in Go, one locale source for extra windows
- [ ] Keyboard: `mod` abstraction, IME + input + modal guards
- [ ] Settings cheat sheet matches real bindings

### Tests

- [ ] Domain logic unit-tested with fakes (no running Wails)
- [ ] Frontend unit tests do not load the Wails Vite plugin
- [ ] Policy (notify, retry, parse) is a pure function

---

## 16. What not to copy blindly

- **Custom RSS/Atom parser** — Clip needed encoding/challenge handling; most apps should use a maintained library.
- **In-memory HTTP validators** — ETags die on process restart. Persist them if conditional GET must survive relaunch.
- **FTS on notes/titles only** — if users expect “search the whole document”, index that field and accept the cost.
- **J/K only in focus mode** — documented as global; implemented as local. When copying the hotkey framework, bind what you advertise.
- **Windows badge as a red dot** — product-specific; macOS numbers may still be right for your app.
- **Wails v3 is still alpha** (Clip pins `v3.0.0-alpha.98`). Updater window HTML, event names, and binding generator behavior can change; isolate those behind adapters the same way Clip isolates `Emitter` and `Sender`.

---

## 17. Suggested package layout for a new app

```
cmd-or-main/
  main.go                 # composition, window, menus, shutdown
api/
  <domain>.go             # Wails-bound services only
internal/
  store/                  # SQLite, models, migrations
  <domain>/               # fetch, schedule, notify, … (no Wails import)
frontend/src/
  Utils/Api/              # binding + event facade
  Types/                  # re-exported models + event payloads
  Stores/                 # Zustand
  Hooks/                  # platform, hotkeys, badge, online
  Components/             # UI
  I18n/locales/           # UI translations
frontend/bindings/        # generated; do not edit
```

Start with `SystemService` (platform, theme, online, version), one domain service, SQLite, the binding facade, and a single event. Add dock/notifications/updater only when the core loop is solid.

That is the transferable core of Clip: **Wails is IPC and native chrome; Go is the product; React is a view over a local cache.**
