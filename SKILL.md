---
name: ontime-standalone-view
description: Generates standalone Ontime custom views as single-file index.html uploads. Use when creating new Ontime display views, timer screens, backstage monitors, stage displays, or any custom browser-based view that connects to Ontime via WebSocket.
---

# Ontime Standalone View Generator

Creates self-contained, single-file `index.html` custom views for [Ontime](https://getontime.no) that connect via WebSocket for real-time event, timer, and rundown data.

## When to Use

- Creating any Ontime custom view (stage display, backstage monitor, lobby screen, confidence monitor, multiview, etc.)
- User asks for an "Ontime view", "custom view", "stage timer", or similar

## Output Rules

1. Create a new directory → place a single `index.html` inside it.
2. **Fully standalone**: all CSS in `<style>`, all JS in `<script>`, system font stacks only, no external dependencies.
3. **Max 4 MB** (Ontime upload limit).
4. Must work when uploaded to Ontime, opened via `file://` (falls back to `localhost:4001`), or with `?server=<host>:<port>`.

## Upload & Serving

Upload via Ontime → Custom Views → create view name → upload `index.html`.
View URL: `http://<host>:<port>/external/<view-name>/`

**Critical**: Views are served under `/external/<name>/`. Relative fetches (`fetch("data/...")`) resolve wrong. Always use **absolute paths** via `getServerOrigin() + getBasePath()`.

## Instructions

### Step 1: Clarify View Type

| View Type | Data Needed |
|-----------|-------------|
| Stage timer | `clock`, `timer`, `eventNow` |
| Backstage / confidence | `clock`, `timer`, `eventNow`, `eventNext`, `message` |
| Schedule / lobby | `fetchRundown()` + `clock` + `timer` + `eventNow` |
| Multiview | Embed Ontime views in iframes, `clock` for header |

### Step 2: HTML Skeleton

```html
<!doctype html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title><!-- Descriptive title --></title>
  <style>
    :root {
      --bg: #050706; --panel: #0b0f0d; --text: #e9ece8; --muted: #8b938d;
      --green: #78c982; --red: #ff4147; --amber: #ffc15c; --blue: #7aa7f2;
      --font: "Trebuchet MS", "Avenir Next", Verdana, sans-serif;
    }
    * { box-sizing: border-box; }
    html, body { width: 100%; height: 100%; }
    body { margin: 0; overflow: hidden; font-family: var(--font); color: var(--text); background: var(--bg); }
  </style>
</head>
<body>
  <!-- HTML structure -->
  <script>
    // All JS here
  </script>
</body>
</html>
```

### Step 3: WebSocket Connection

```js
let reconnectTimeout;
let live = false;

connectSocket();

function connectSocket() {
  const ws = new WebSocket(getSocketUrl());
  ws.onopen = () => { live = true; clearTimeout(reconnectTimeout); updateConnection("live", true); };
  ws.onclose = () => { if (live) updateConnection("reconnecting", false); reconnectTimeout = setTimeout(connectSocket, 1000); };
  ws.onerror = () => { updateConnection(live ? "error" : "offline", false); };
  ws.onmessage = (event) => {
    try {
      const { tag, payload } = JSON.parse(event.data);
      if (tag === "runtime-data" || tag === "runtime-patch") { handleRuntimeUpdate(payload); }
      if (tag && tag.includes("refetch")) { fetchRundown(); loadProjectData(); }
    } catch (e) { console.warn("WS parse error", e); }
  };
}
```

### Step 4: URL Resolution Helpers (required in every view)

Centralises origin, WebSocket URL, cloud basePath, and auth token forwarding.

```js
function getServerConfig() {
  const origin = getLocalTestingOrigin() || window.location.origin;
  const wsOrigin = origin.replace(/^http:/, "ws:").replace(/^https:/, "wss:");
  return { origin, wsOrigin, basePath: getBasePath(), search: getServerSearch() };
}
function getSocketUrl()    { const c = getServerConfig(); return `${c.wsOrigin}${c.basePath}/ws${c.search}`; }
function getServerOrigin() { return getServerConfig().origin; }

// file:// fallback + ?server= override
function getLocalTestingOrigin() {
  const server = new URLSearchParams(window.location.search).get("server");
  if (server) { const n = server.includes("://") ? server : `http://${server}`; return n.replace(/\/$/, ""); }
  if (window.location.protocol === "file:") return "http://localhost:4001";
  return "";
}

// Cloud stage hash (getontime.no only)
function getStageHash() {
  if (!window.location.href.includes("getontime.no")) return "";
  const s = window.location.pathname.split("/").filter(Boolean)[0];
  return s ? `/${s}` : "";
}
function getBasePath() { return getStageHash() || ""; }

// Preserves auth tokens; strips ?server= override
function getServerSearch() {
  const p = new URLSearchParams(window.location.search); p.delete("server");
  const q = p.toString(); return q ? `?${q}` : "";
}
```

**Usage:**
```js
new WebSocket(getSocketUrl());
fetch(`${getServerOrigin()}${getBasePath()}/data/rundowns/current${getServerSearch()}`);
logo.src = `${getServerOrigin()}${getBasePath()}/user/logo/${encodePath(project.logo)}`;
```

### Step 5: Runtime Data & Merge Pattern

WebSocket sends `runtime-data` (full snapshot) and `runtime-patch` (partial). **Always merge — never replace.**

#### Runtime Fields

| Field | Contents |
|-------|----------|
| `clock` | Server wall-clock (ms from midnight) |
| `timer` | `{ current, duration, elapsed, playback, startedAt, expectedFinish, finishedAt, addedTime }` |
| `eventNow` | Currently loaded event object or `null` |
| `eventNext` | Next event object or `null` |
| `rundown` | `{ numEvents, selectedEventIndex, plannedStart, plannedEnd, offset, expectedEnd }` |
| `offset` | `{ absolute, relative, mode, expectedGroupEnd, expectedRundownEnd, expectedFlagStart }` |
| `message` | `{ timer: { text, visible, blink, blackout, secondarySource }, secondary }` |
| `auxtimer1/2/3` | `{ duration, current, playback, direction }` |
| `groupNow` | Active group or `null` |
| `eventFlag` | Targeted flag or `null` |

#### Event Object Fields

`id`, `type` (event/group/milestone), `title`, `cue`, `timeStart`, `timeEnd`, `duration`, `colour`, `note`, `skip`, `flag`, `parent`, `custom` (object of user-defined fields).

#### Merge Implementation

```js
let runtime = { clock: null, eventNow: null, eventNext: null, timer: null, rundown: { selectedEventIndex: 0 } };

function handleRuntimeUpdate(patch) {
  Object.keys(patch || {}).forEach((key) => {
    const v = patch[key];
    if (v && typeof v === "object" && !Array.isArray(v) && runtime[key] && typeof runtime[key] === "object") {
      runtime[key] = { ...runtime[key], ...v };
    } else { runtime[key] = v; }
  });
  render();
}
```

#### Clock Consistency Rule

Ontime's `payload.clock` is the authoritative display value. When a websocket `runtime-data` or `runtime-patch` message contains `clock`, render that value directly

```js
if ("clock" in payload) setText("clockTime", formatClockTime(payload.clock, true));
```

Do **not** increment the live server clock locally with `runtime.clock += 1000`, `serverClock += 1000`, or similar. That makes custom views drift ahead of Ontime, especially when a websocket clock update and a local interval tick happen close together.

Use local wall-clock time only as a fallback before live data arrives or after the websocket disconnects:

```js
function renderClock() {
  const value = live && typeof runtime.clock === "number"
    ? runtime.clock
    : timeOfDayMs(new Date());
  setText("clockTime", formatClockTime(value, true));
}

function timeOfDayMs(date) {
  return ((date.getHours() * 60 + date.getMinutes()) * 60 + date.getSeconds()) * 1000;
}
```

If a view calls `renderClock()` on an interval, the interval must only re-render the current `runtime.clock`; it must not mutate or advance it while `live === true`. Ontime will send the next authoritative clock value.

### Step 6: Utility Functions

```js
function formatClockTime(value, showSeconds) {
  if (value == null || Number.isNaN(Number(value))) return showSeconds ? "--:--:--" : "--:--";
  const t = Math.floor(Math.abs(value) / 1000);
  const h = Math.floor(t / 3600) % 24, m = Math.floor((t % 3600) / 60), s = t % 60;
  return showSeconds ? `${pad(h)}:${pad(m)}:${pad(s)}` : `${pad(h)}:${pad(m)}`;
}
function formatDuration(value) {
  if (value == null || Number.isNaN(Number(value))) return "--:--:--";
  const sign = value < 0 ? "-" : "", t = Math.floor(Math.abs(value) / 1000);
  return `${sign}${pad(Math.floor(t/3600))}:${pad(Math.floor((t%3600)/60))}:${pad(t%60)}`;
}
function pad(v) { return Math.floor(v).toString().padStart(2, "0"); }
function setText(id, v) { const el = document.getElementById(id); if (el) el.textContent = v; }
function updateConnection(text, isLive) {
  const el = document.getElementById("connectionStatus"); if (!el) return;
  el.textContent = text; el.classList.toggle("live", isLive);
}
function getProgress(timer) {
  if (!timer || !timer.duration || timer.duration <= 0) return 0;
  const elapsed = timer.elapsed == null ? timer.duration - timer.current : timer.elapsed;
  return Math.max(0, Math.min(100, (elapsed / timer.duration) * 100));
}
function encodePath(v) { return String(v).split("/").map(p => encodeURIComponent(p)).join("/"); }
```

### Step 7: Fetch Rundown (optional)

WebSocket only gives `eventNow`/`eventNext`. For the full schedule, fetch via HTTP.

**Endpoint**: `/data/rundowns/current` (v4, plural). Old `/data/rundown/current` returns 404 on v4 — try both for compat.

**Rundown shape**: `{ id, title, order[], flatOrder[], entries: { [id]: EventObject }, revision }`. Use `flatOrder` for correct display order. `entries` is an object keyed by ID, not an array.

**Refetch strategy**: Poll every 30s + refetch on websocket `"refetch"` tag.

```js
let rundownEntries = [];
fetchRundown();
setInterval(fetchRundown, 30000);

async function fetchRundown() {
  const o = getServerOrigin(); if (!o) return;
  try {
    const r = await fetchFirst([
      `${o}${getBasePath()}/data/rundowns/current${getServerSearch()}`,
      `${o}${getBasePath()}/data/rundown/current${getServerSearch()}`
    ]);
    if (!r) return;
    const d = await r.json();
    const all = Array.isArray(d.entries) ? d.entries
      : (d.flatOrder || d.order || []).map(id => d.entries && d.entries[id]).filter(Boolean);
    rundownEntries = all.filter(e => e && e.type !== "group");
    render();
  } catch (e) { console.warn("Rundown fetch failed", e); }
}
async function fetchFirst(urls) {
  for (const u of urls) { const r = await fetch(u, { cache: "no-store" }); if (r.ok) return r; }
  return null;
}
```

### Step 8: Project Data, Logo, Info & QR (optional)

#### HTTP Data Endpoints

| Endpoint | Description |
|----------|-------------|
| `GET /data/project` | Project title, description, info, url, logo filename, custom fields |
| `GET /data/rundowns/current` | Current rundown (see Step 7) |
| `GET /data/custom-fields` | Registered custom field definitions |
| `GET /data/settings` | App settings |
| `GET /data/view-settings` | View configuration |
| `GET /api/poll` | Runtime state snapshot |
| `GET /api/version` | Ontime version string |

#### Project Data Fields

```text
GET /data/project → { title, description, url, info, logo, custom }
```

- `project.title` — display title
- `project.info` — operator-managed info text (prefer over `description` for public panels)
- `project.url` — QR target URL
- `project.logo` — filename; load from `/user/logo/<filename>` (NOT `/ontime-logo.png` which is the app icon)

#### QR Code Rule

Ontime provides `project.url` but **not** a QR image. Generate a real scannable QR inline. Do not draw fake patterns or fetch from external services.

Recommended: inline a small MIT-licensed QR encoder (e.g., `qrcode-generator`) in `<script>`. Render to DOM/SVG/canvas. Show URL text fallback. Hide panel if `project.url` is empty.

```js
function buildProjectQr(url) {
  const grid = document.getElementById("qrGrid"); if (!grid) return;
  const clean = String(url || "").trim();
  setText("qrText", clean || "No project URL configured");
  grid.innerHTML = "";
  if (!clean || typeof qrcode !== "function") { grid.style.display = "none"; return; }
  grid.style.display = "grid";
  const qr = qrcode(0, "M"); qr.addData(clean); qr.make();
  const mc = qr.getModuleCount();
  grid.style.gridTemplateColumns = `repeat(${mc}, 1fr)`;
  for (let r = 0; r < mc; r++) for (let c = 0; c < mc; c++) {
    const cell = document.createElement("span");
    cell.className = qr.isDark(r, c) ? "qr-cell on" : "qr-cell";
    grid.appendChild(cell);
  }
}
function getProjectInfo(p) { return String(p && (p.info || p.description || "")).trim(); }
function getProjectQrUrl(p) { return String(p && p.url || "").trim(); }
```

#### Load Project Data

```js
async function loadProjectData() {
  const logo = document.getElementById("brandLogo"), mark = document.getElementById("brandMark");
  try {
    const r = await fetch(`${getServerOrigin()}${getBasePath()}/data/project${getServerSearch()}`, { cache: "no-store" });
    if (!r.ok) throw new Error();
    const p = await r.json();
    setText("projectTitle", p.title || "Ontime Project");
    setText("projectInfo", getProjectInfo(p));
    buildProjectQr(getProjectQrUrl(p));
    if (logo && mark && p.logo) {
      logo.onerror = () => { logo.remove(); mark.classList.add("fallback"); };
      logo.src = `${getServerOrigin()}${getBasePath()}/user/logo/${encodePath(p.logo)}`;
    } else if (mark) mark.classList.add("fallback");
  } catch (_) {
    setText("projectTitle", "Ontime Project"); setText("projectInfo", ""); buildProjectQr("");
    if (logo) logo.remove(); if (mark) mark.classList.add("fallback");
  }
}
```

#### HTTP Playback Control (reference)

All playback commands are `GET` requests returning `{"payload":"success"}`:

| Action | Endpoint |
|--------|----------|
| Start loaded | `GET /api/start` |
| Start by index/id/cue | `GET /api/start/index/<n>`, `/api/start/id/<id>`, `/api/start/cue/<cue>` |
| Start next/previous | `GET /api/start/next`, `/api/start/previous` |
| Pause | `GET /api/pause` |
| Stop | `GET /api/stop` |
| Load by index/id/cue | `GET /api/load/index/<n>`, `/api/load/id/<id>` |
| Load next/previous | `GET /api/load/next`, `/api/load/previous` |
| Reload | `GET /api/reload` |
| Roll mode | `GET /api/roll` |
| Add/remove time | `GET /api/addtime/add/<ms>`, `/api/addtime/remove/<ms>` |
| Change event field | `GET /api/change/<event-id>?title=new&cue=new` |
| Aux timer | `GET /api/auxtimer/<1\|2\|3>/start\|pause\|stop\|duration/<ms>\|direction/count-up\|count-down` |
| Message | `GET /api/message/secondary/<text>`, `/api/message/timer?blackout=true` |

## Best Practices

- CSS custom properties for theming; `clamp()` for responsive font sizes (7" to 90" screens).
- Dark backgrounds by default; `overflow: hidden` on body.
- Always show a connection status indicator.
- Display Ontime's live `payload.clock` exactly; never locally add seconds to it.
- Merge patches — never replace runtime state.
- Absolute fetch paths only — relative paths break under `/external/<name>/`.
- Poll rundown every 30s + refetch on WS signal.
- Prefer `eventNow.id` for highlight, not `selectedEventIndex` (breaks when groups are filtered).
- Use `project.info` for info panels; `project.url` as QR target (generate locally).
- Add fallback data so views render before live data arrives.
- Keep file under 4 MB.

## Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| Replacing state on patch | Merge nested objects: `{ ...current[key], ...value }` |
| `file://` origin is `"null"` | `getLocalTestingOrigin()` falls back to `localhost:4001` |
| External fonts/CDNs/images | Keep everything inline — no internet guaranteed after upload |
| No reconnection logic | Auto-reconnect on close with 1s interval |
| Ignoring `refetch` tag | Reload rundown + project data on any tag containing "refetch" |
| Times are ms from midnight | Divide by 1000 for seconds; `% 24` for hours |
| Clock is ahead by 1-2s | Do not run `runtime.clock += 1000` or `serverClock += 1000`; render `payload.clock` directly while live |
| Wrong logo URL | `/user/logo/<file>` = project logo; `/ontime-logo.png` = app icon |
| Wrong rundown endpoint | v4 uses `/data/rundowns/current` (plural); old singular returns 404 |
| Relative fetch after upload | Resolves to `/external/<name>/data/...` — use absolute paths |
| Event highlight mismatch | `selectedEventIndex` ≠ filtered list index; match by `eventNow.id` |
| Fake QR pattern | Generate real QR from `project.url` with inline encoder |
| `project.description` for info | Prefer `project.info`; use `description` only as fallback |
| HTTPS → `ws://` fails | `getServerConfig()` derives `wss://` from `https://` automatically |

## Debugging

```bash
curl -i http://localhost:4001/data/rundowns/current          # rundown
curl -i http://localhost:4001/data/rundown/current           # old endpoint (expect 404 on v4)
curl -sS http://localhost:4001/data/project | node -e '      # project fields
  let s="";process.stdin.on("data",d=>s+=d);process.stdin.on("end",()=>{
    const p=JSON.parse(s); console.log({title:p.title,info:p.info,url:p.url,logo:p.logo});
  })'
node -e "const fs=require('fs'),h=fs.readFileSync('index.html','utf8'); \
  new Function([...h.matchAll(/<script>([\s\S]*?)<\/script>/g)].map(m=>m[1]).join('\n')); \
  console.log('JS syntax ok');"
```
