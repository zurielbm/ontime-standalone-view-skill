# Ontime Custom View Documentation

## Purpose

Use this as the custom-view-specific guide for building a standalone `index.html` that can be uploaded to Ontime. Keep API details in the focused resource docs listed below.

## Resource Map

- `websockets.md`: WebSocket endpoint, request tags, broadcast behavior, and message/control examples.
- `http.md`: HTTP endpoints, including `/data/project`, `/data/rundowns/current`, `/api/poll`, and playback/control APIs.
- `runtime-data.md`: Runtime payload fields such as `clock`, `timer`, `eventNow`, `eventNext`, `rundown`, `offset`, messages, and aux timers.
- `project-data.md`: Rundown resource shape and mutation endpoints.
- `event-data.md`: Event object fields, timer types, end actions, custom fields, and time semantics.

## Upload Requirements

Ontime custom views are uploaded as one HTML file. Put all HTML, CSS, JavaScript, fonts, images, and fallback assets in that file because external imports are not available in custom views.

Typical local URL after upload:

```text
http://localhost:4001/external/<view-name>/
```

## Required Data Sources

Custom schedule/timer views usually need both data channels:

- WebSocket `/ws` for live runtime state. See `websockets.md` and `runtime-data.md`.
- HTTP `/data/rundowns/current` for the full ordered schedule. See `http.md` and `project-data.md`.

For the Ontime server tested with this view, use `/data/rundowns/current` first. The older singular endpoint `/data/rundown/current` returned `404 Unhandled request` on that server, even though it appears in some older documentation.

Project logos come from `/data/project` and `/user/logo/<logo-filename>`. The app icon at `/ontime-logo.png` is not the project logo.

## URL Rules

Do not use relative API paths from an uploaded custom view. A fetch like this:

```js
fetch("data/rundowns/current");
```

resolves under `/external/<view-name>/`, not the Ontime API root.

Build URLs from a shared server config instead:

```js
function getServerConfig() {
  const origin = getLocalTestingOrigin() || window.location.origin;
  const wsOrigin = origin.replace(/^http:/, "ws:").replace(/^https:/, "wss:");

  return {
    origin,
    wsOrigin,
    basePath: getBasePath(),
    search: getServerSearch()
  };
}

function getSocketUrl() {
  const { wsOrigin, basePath, search } = getServerConfig();
  return `${wsOrigin}${basePath}/ws${search}`;
}
```

Use `wss://` when the page is served over HTTPS. Browsers block secure pages from connecting to insecure `ws://` sockets.

## Local, Cloud, And Token Handling

For direct file testing, `window.location.origin` is `file://`, so provide a server override:

```text
file:///path/to/index.html?server=http://localhost:4001
```

Recommended helper:

```js
function getLocalTestingOrigin() {
  const params = new URLSearchParams(window.location.search);
  const server = params.get("server");
  if (server) return (server.includes("://") ? server : `http://${server}`).replace(/\/$/, "");
  if (window.location.protocol === "file:") return "http://localhost:4001";
  return "";
}
```

Ontime Cloud URLs can include a stage hash before `/ws` and `/data`. Preserve that first path segment for `getontime.no` URLs:

```js
function getBasePath() {
  if (!window.location.href.includes("getontime.no")) return "";
  const stageHash = window.location.pathname.split("/").filter(Boolean)[0];
  return stageHash ? `/${stageHash}` : "";
}
```

If the view is loaded with share/auth query parameters, pass them through to WebSocket and HTTP requests. Remove only local-only parameters such as `server`:

```js
function getServerSearch() {
  const params = new URLSearchParams(window.location.search);
  params.delete("server");
  const query = params.toString();
  return query ? `?${query}` : "";
}
```

## Runtime Handling

WebSocket runtime messages may arrive as full snapshots or partial patches. Keep a local cache and merge nested objects so a small timer patch does not erase previously received runtime fields.

```js
function mergeRuntime(current, patch) {
  const next = { ...current };
  Object.keys(patch || {}).forEach((key) => {
    const value = patch[key];
    if (value && typeof value === "object" && !Array.isArray(value) && current[key] && typeof current[key] === "object") {
      next[key] = { ...current[key], ...value };
    } else {
      next[key] = value;
    }
  });
  return next;
}
```

Refetch the rundown when a WebSocket refetch tag is received, and consider a slow polling fallback such as every 30 seconds for signage views.

## Schedule Rendering

The current rundown response is keyed by entry ID. Render in `flatOrder` when available, falling back to `order`:

```js
const entries = (rundown.flatOrder || rundown.order || [])
  .map((id) => rundown.entries && rundown.entries[id])
  .filter(Boolean);
```

Filter `type === "group"` only if the design should hide group headers. Prefer `runtime.eventNow.id` for current-event highlighting; `rundown.selectedEventIndex` can drift from displayed indexes when groups or skipped entries are filtered.

## Minimal Build Checklist

1. Create one self-contained `index.html`.
2. Connect to `getSocketUrl()` and handle runtime snapshot/patch tags.
3. Cache merged runtime state locally.
4. Fetch `/data/rundowns/current` with absolute URLs built from `getServerConfig()`.
5. Convert `flatOrder` plus `entries` into the displayed schedule list.
6. Render `eventNow`, `eventNext`, timer values, clock, and schedule entries using the field definitions in `runtime-data.md` and `event-data.md`.
7. Add reconnect logic for the WebSocket and a slow HTTP rundown refetch fallback.
8. Include fallback/demo data only for the pre-connection empty state.

## Useful Debug Commands

```bash
curl -i http://localhost:4001/data/rundowns/current
curl -i http://localhost:4001/data/rundown/current
```

```bash
node -e "const fs=require('fs'); const html=fs.readFileSync('index.html','utf8'); const scripts=[...html.matchAll(/<script>([\s\S]*?)<\/script>/g)].map(m=>m[1]).join('\n'); new Function(scripts); console.log('script syntax ok');"
```

## Verified Local Findings

These findings came from the local server used while building the view:

- Server: `http://localhost:4001`
- `GET /data/rundowns/current`: `200 OK`
- `GET /data/rundown/current`: `404 Unhandled request`
- Current rundown shape: `{ id, title, order, flatOrder, entries, revision }`
- `entries` shape: object keyed by entry ID
