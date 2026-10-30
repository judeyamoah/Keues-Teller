# AGENTS.md

Project context for agents and contributors. Read this before modifying code.

## What it is

**Keues-Counter** is the frontend for the **service counter station** in the **Keues** queue management system. It runs on a screen at each counter (butcher, fishmonger, greengrocer, etc.) and allows users to:

- Select a location → flow → counter and save that configuration to disk (a local JSON file via the Rust backend).
- Call the next ticket and finish it (*TicketMachine* flow).
- Indicate that the counter is free (*SetFree* flow).
- Manually call numbers (+1/−1/+10/−10, the counter's local number, *ManualCall* flow).

It is a **Tauri 2 + React + Vite** app. It connects to an ASP.NET backend (the Keues API), which forwards events to the **Monitor** project (which does not exist yet; the SetFree/ManualCall endpoints return 404 until the user creates them).

**TicketMachine** (the ticket machine in the same system) is **a separate project: read-only, DO NOT modify**.

## Language (mandatory)

**The app must be English-only.** All visible text and labels, comments, error messages, and identifier names (variables, functions, IPC methods) must be in English. Do not add Spanish strings or comments; all new UI text must be written in English. When modifying code, retain the existing English names (e.g. `saveConfiguration`/`loadConfiguration`, `showConfig`, `counterName`, `selectedFlowType`).

## Stack

- Tauri 2 (Rust backend in `src-tauri/`), React 19, Vite 8, TypeScript 6, Mantine 9 (UI) + tabler-icons, @emotion/react.
- Scripts (`package.json`):
  - `pnpm dev` → Vite (renderer at `localhost:11220`).
  - `pnpm start` → `tauri dev` (starts Vite + the Tauri app). **This is the user's normal command.**
  - `pnpm build` → renderer (`dist/`).
  - `pnpm dist` → `tauri build` → builds everything and generates the **NSIS** installer.
- Mandatory checks after modifying code (must pass without errors):
  - `pnpm exec tsc -b`
  - `pnpm exec eslint .`
  - `pnpm exec vite build`
  - `cargo check` and `cargo clippy` in `src-tauri/`

## Domain flow (validated against the real API at `localhost:5125`)

**Location → Flows + Counters.**

- **Location** (`/api/locations`): a site/establishment. Fields: `id`, `name`, `color`.
- **Flow** (`/api/flows?locationId=X`): a menu tree. It contains `flowJson` (the tree's JSON); nodes of type `ticket` have a `queueId`. `getFlowQueueIds(flow)` extracts those `queueId`s. Fields: `id`, `name`, `flowType`, `flowJson`.
- **Counter** (`/api/counters?locationId=X`): a physical service counter. Fields: `id`, `name`, `code`, `color`, `queues: string[]` (queues assigned to the counter).
- The counter calls tickets from its queues. The counter selector filters using `counter.queues ∩ queueIds from the flow`.

## API contract (`keuesApi.ts`)

All list responses are wrapped in `{data: [...]}`. Use `json.data`.

- `GET /api/locations` → `{data: [Location]}`
- `GET /api/flows?locationId=X` → `{data: [Flow]}`
- `GET /api/flows/{id}` → **direct object** (not `{data}`). Always use this to get fresh `flowJson` (never use a flow stored in React state, as it may be stale).
- `GET /api/counters?locationId=X` → `{data: [Counter]}`
- `GET /api/queues?locationId=X` → `{data: [Queue]}`
- `POST /api/users/login` → body `{email, password}`; 200 `{jwt}` (also sets an HTTP-only `access_token` cookie); 400 `{error}` for invalid credentials. `login` throws an error containing that message.
- `POST /api/counters/{counterId}/check-access` → no body. **200 = access granted; any other response = no access** (403 if access is denied). `checkAccess` returns `response.ok` (it does not throw on 403).
- `POST .../call-next-ticket` → body `{counterId, flowId}`; response `{ticketId, code}` or `null` if there are no tickets.
- `POST .../attend-ticket` → body `{counterId, ticketId, flowId}` → 200.
- `POST .../set-free` → body `{counterId, flowId}`. The route is still a **stub** and returns 404. `setCounterFree` throws an error if `!ok`; the SetFree panel calls it even if it fails.
- `POST .../call-manual-ticket` → body `{code, flowId, locationId, counterId}`. The route is still a **stub** and returns 404. `manualCall` throws an error if `!ok`; the ManualCall panel calls it even if it fails.

### Authentication (access token)

- The login JWT is stored **in memory only** (`src/api/auth.ts`: `getToken`/`setToken`); it is not persisted or included in `AppConfiguration`. It is lost when the app closes and will be requested again when needed.
- `keuesApi.ts` adds `Authorization: Bearer <jwt>` to all requests when a token exists (`authHeaders()`), although the backend also accepts the `access_token` cookie.
- `ConfigScreen` checks `check-access` when a counter is selected and when the user clicks "Connect" with a counter already selected. If access is denied, it opens `LoginModal` (email + password) → `users/login` → saves the JWT → retries `check-access`. The modal can be cancelled so the user can choose another counter; while the counter is not authorised, "Save and start" remains disabled. For now, **access is not checked** when `CounterPanel` starts.

### FlowType (.NET enum)

- `0` = **TicketMachine**: "Call next" + "Finish" buttons.
- `1` = **SetFree**: only the "Mark as free" button (the endpoint does not exist yet → 404).
- `2` = **ManualCall**: +1/−1/+10/−10, the counter's local number (the endpoint does not exist yet → 404).

## Structure

- `src-tauri/src/main.rs`: `load_config`/`save_config` commands (JSON in the app's config directory, `deviceId` using `uuid`), `get_proxy_base`/`set_proxy_target`, and updater plugin registration.
- `src-tauri/src/proxy.rs`: local HTTP + WebSocket reverse proxy at `127.0.0.1:<port>` that forwards to the actual server (avoids mixed content/CORS issues from the WebView).
- `src/main.tsx`: `MantineProvider` with `forceColorScheme="light"` (forces the light theme).
- `src/App.tsx`: loads the config → if one exists, goes straight to `CounterPanel`; `showConfig` state allows returning to Settings without losing the config in memory; `onSaved` saves and returns to the panel. **Owns the SignalR connection**: connects using the config and reconnects whenever the config changes (when saving, even if the user changed the location/flow); the connection remains active while the user is in Settings.
- `src/types/config.ts`: `AppConfiguration` (`server, locationId, flowId, counterId, flowType, locationName?, flowName?, counterName?, counterCode?, deviceName?, deviceId?`).
- `src/types/models.ts`: `Location`, `Counter`, `Ticket`, `Flow`, `FlowNode`.
- `src/api/appBridge.ts`: bridge to the Tauri backend (`saveConfiguration`/`loadConfiguration`/updater via `invoke` and `@tauri-apps/plugin-updater`).
- `src/api/net.ts`: `isTauri`, `proxyBase`, `serverBase` and `configureTarget` (the frontend points to the local proxy).
- `src/api/keuesApi.ts`: all API-call logic (`getLocations`, `getFlows`, `getFlow`, `getCounters`, `getQueues`, `getFlowQueueIds`, `login`, `checkAccess`, `callNext`, `attendTicket`, stubs `setCounterFree`, `manualCall`).
- `src/api/auth.ts`: login JWT token stored **in memory only** (`getToken`/`setToken`).
- `src/constants/app.ts`: `APP_VERSION` imported from `package.json` (increment the version when publishing; currently `1.0.0`).
- `src/components/config/ConfigScreen.tsx`: Location → Flow → Counter selector. Pre-fills values from `initialConfig`; `changeFlow` fetches fresh data using `getFlow` + `getCounters` in `Promise.all` and filters counters by queues; saves names (`locationName/flowName/counterName/counterCode`) as well as IDs; "Back" button (`onCancel`); the Flow select label displays a badge with the type (`TicketMachine`/`SetFree`/`ManualCall`); when a counter is selected/connected, it runs `check-access` and opens `LoginModal` if access is denied.
- `src/components/config/LoginModal.tsx`: authorisation modal (email + password) that calls `users/login` and retries `check-access`.
- `src/components/flows/`: `PanelShell.tsx` and `NumberDisplay.tsx` (shared wrapper for the three cards: radial background, decorative blue gradient top border, header with title/subtitle; large number + caption), `CounterPanel.tsx` (dispatches by `flowType`, counter badge at bottom-left, SignalR status badge, "Settings" button at top-right, `VersionBadge`), `CallTicketPanel.tsx`, `SetFreePanel.tsx`, `ManualCallPanel.tsx`.
- `src/components/VersionBadge.tsx`: `v{APP_VERSION}` badge at the bottom-right (so support staff can identify the version).

## Visual design (counter panels)

Counter panels follow these design principles: **light background**, white cards, and dark, high-contrast colours for important data. This is the official colour palette for the service counter.

### General principles

- **Light background** (`#f8f9fa` — very light grey, almost white).
- White cards (`#ffffff`) with a soft shadow and border (`#e5e7eb`).
- Important numbers and data use **dark, high-contrast colours**.
- Large, legible typography (the operator should be able to see the ticket number at a glance).
- A consistent palette across all panels (blue `#2563eb` + dark blue `#1a1a2e`), regardless of flow type.

### Buttons: Mantine theme colours only (mandatory rule)

Interactive buttons must use **only Mantine theme colours**: `variant="filled"` + `color="blue" | "green" | "red"` (or `variant="default"/"outline"/"subtle"`). **Prohibited**: custom gradients on buttons and manually defined `&:hover`/`&:active` styles (transform, boxShadow, inline backgrounds) — Mantine already manages hover (lighter) and active (darker) states automatically. Permitted exceptions for custom styles: **state feedback** unrelated to hover (e.g. the `[data-sent]` dimming effect on the SetFree circle). Gradients `#2563eb → #3b82f6` are reserved for **non-interactive decorative backgrounds** (the `PanelShell` header and the card's top border).

### Colour palette

| Use | Colour |
|---|---|
| Main background | `#f8f9fa` |
| Card background | `#ffffff` |
| Current ticket / large number | `#1a1a2e` (very dark blue) |
| Decorative backgrounds (header/card border) | gradient `#2563eb` → `#3b82f6` |
| Interactive buttons | Mantine theme: `blue`, `green`, `red` (see panel-specific sections) |
| Secondary text / labels | `#6b7280` |
| Very faint text (empty state) | `#d1d5db` |
| Borders and separators | `#e5e7eb` |
| Connected status badge | Mantine green |
| Disconnected status badge | Mantine red |

### CallTicketPanel (flowType 0)

- `PanelShell` (title = flowName, subtitle = counterName) + `NumberDisplay` ("Current ticket").
- "Call next": `variant="filled"` `color="blue"`, `size="xl"`, full width; `loading={calling}` + double-click guard.
- "Finish": primary action below — `variant="filled"` `color="green"`, `size="xl"`, full width, disabled when there is no active ticket.
- "Cancel ticket": discreetly placed at the bottom — `variant="subtle"`, muted grey, `IconCircleX` icon; opens a **confirmation modal** ("Cancel ticket C-42? The customer will lose their turn.") with "Keep ticket" / "Cancel ticket" (red).
- Errors appear in an `Alert` (red if there is an active ticket, yellow if there are no tickets).

### SetFreePanel (flowType 1)

- `PanelShell` (title = counterName).
- Circular "FREE" button: `radius="100%"`, 180×180, `variant="filled"` `color="blue"`, `IconArrowUp` icon + "FREE" text; `[data-sent]` dimming provides state feedback.
- Success is shown via a notification (green); errors appear in a yellow `Alert`.

### ManualCallPanel (flowType 2)

- `PanelShell` (title = flowName, subtitle = counterName) + `NumberDisplay` ("Current number"). Initial value is 1.
- Circular 110×110 buttons in a `SimpleGrid`: ±1 uses standard `variant="default"`; ±10 uses `variant="filled"` `color="blue"`.
- "Reset to 1": `variant="outline"`, `color="dark"`.
- API calls are **debounced and grouped over 500 ms** (the UI updates immediately; only the latest value is sent).
- Errors appear in a yellow `Alert`.

### Common elements (managed by CounterPanel)

| Element | Position | Details |
|---|---|---|
| Counter badge | Fixed bottom-left | Blue `variant="light"` |
| VersionBadge | Fixed bottom-right | `v{APP_VERSION}`, grey outline badge |
| "Settings" button | Fixed top-right | `variant="default"`, `IconSettings` 16px |
| SignalR status badge | Fixed bottom-left next to the counter | Green/red/orange/yellow depending on status |

## Persistence (local configuration)

- The Rust backend saves the config to **`app_config_dir()/config.json`** → on Linux: `~/.config/com.keues.counter/config.json`.
- The config is the direct JSON object (without a wrapper key); it includes `deviceId` (UUID v4 generated/validated by Rust).
- Automatic migration from the Electron format: if no new config exists, it reads `~/.config/keues-counter/config.json` (key `config`) and imports it once.
- The saved config may be old (without `counterName`): `CounterPanel` falls back to looking up the counter through the API, and the names are persisted when the config is saved again.

## Network / proxy

- The frontend does **not** call the server directly: it points to the local Rust reverse proxy (`http://127.0.0.1:<port>`, see `src/api/net.ts`), which forwards HTTP + WebSocket traffic to the real IP (`configureTarget`).
- This avoids mixed content/CORS issues when the server is on another IP (`http://IP_LAN`); it works the same on Linux and Windows.

## Environment / gotchas

- To compile/run Tauri on Linux, the required `-dev` packages (`libwebkit2gtk-4.1-dev`, `libgtk-3-dev`, etc.) and the Rust toolchain must be installed.
- The updater (`tauri-plugin-updater`) requires a signing key pair (`tauri signer generate`); the private key must be stored as a CI secret and the public key in `tauri.conf.json` (`plugins.updater.pubkey`).
- Frontend `APP_VERSION` comes from `package.json`; the installer/updater version comes from `src-tauri/tauri.conf.json`. Keep both in sync when publishing.
- Releases are **manual** using Release Please (`npx release-please release-pr` / `release`). There is no GitHub Actions workflow. Configuration is in `release-please-config.json` + `.release-please-manifest.json` and synchronises `package.json`, `src-tauri/tauri.conf.json`, and `src-tauri/Cargo.toml`.
- The SetFree/ManualCall endpoints do not exist yet: a 404 error is expected.
