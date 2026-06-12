---
name: ontime-standalone-view
description: Creates minimal standalone Ontime custom views as single-file index.html uploads. Use for stage timers, backstage or confidence monitors, schedules, lobby displays, multiviews, and other browser views that use Ontime data.
---

# Ontime Standalone View

Create a self-contained `index.html` that implements exactly the Ontime view the user requested.

## Primary Rule: Minimal Output

Generate the smallest complete implementation that satisfies the request.

- Add only UI, data, styles, state, helpers, and error handling required by requested behavior.
- Do not add speculative features, reusable architecture, configuration systems, generic components, demo data, decorative panels, controls, animations, comments, or compatibility code.
- Do not include a feature merely because this skill documents it.
- Do not preemptively support schedule data, project data, logos, QR codes, messages, auxiliary timers, playback controls, multiple views, or connection indicators.
- Do not create abstractions for logic used once unless they make the code shorter or prevent a real correctness problem.
- Prefer direct DOM updates and a small state object over frameworks, classes, component systems, or generalized render pipelines.
- Prefer a few explicit CSS declarations over a large theme or design system.
- Use fallback values only where the requested interface would otherwise be broken before data arrives.
- Stop adding code when every user-visible requirement and required Ontime integration is implemented.

Before finishing, inspect every substantial block and ask: **Which explicit requirement needs this?** Remove it if there is no concrete answer.

## Output Contract

1. Create a new directory containing one `index.html`, unless the user specifies a path.
2. Keep CSS in `<style>` and JavaScript in `<script>`.
3. Use system fonts and no external runtime dependencies.
4. Keep the file below Ontime's 4 MB upload limit.
5. Support the environment the user requests. If none is specified, support Ontime hosting plus local `file://` testing with `localhost:4001`.

Views uploaded to Ontime are served from `/external/<name>/`. Any required HTTP request must use an absolute server URL, not a relative path.

## Build Workflow

### 1. Extract Requirements

List the visible elements and interactions requested by the user. Map each one to the minimum Ontime data it needs.

| Requested feature | Minimum data |
|---|---|
| Server clock | `clock` |
| Main timer | `timer` |
| Current event | `eventNow` |
| Next event | `eventNext` |
| Timer message | `message` |
| Auxiliary timer | requested `auxtimer1`, `auxtimer2`, or `auxtimer3` |
| Full schedule | current rundown HTTP endpoint |
| Project title, info, URL, or logo | project HTTP endpoint |
| Playback action | only the corresponding API endpoint |
| Multiview iframe | no runtime socket unless the surrounding UI needs live data |

Do not subscribe to, fetch, store, or render data outside this mapping.

### 2. Select Only Necessary Integration

- **Static iframe layout:** HTML and CSS only.
- **Live runtime field:** one WebSocket connection and state containing only used fields.
- **Full schedule:** WebSocket only if live highlighting or clock/timer data is requested; otherwise HTTP alone may be enough.
- **Project metadata:** fetch only `/data/project`.
- **Playback control:** implement only requested commands.

Read references only as needed:

- Core custom-view URL, runtime, and validation rules: [resources/CUSTOM_VIEW_DOCUMENTATION.md](resources/CUSTOM_VIEW_DOCUMENTATION.md)
- Runtime field shapes: [resources/runtime-data.md](resources/runtime-data.md)
- Event fields and timer behavior: [resources/event-data.md](resources/event-data.md)
- Rundown/project data: [resources/project-data.md](resources/project-data.md)
- HTTP commands: [resources/http.md](resources/http.md)
- WebSocket commands and messages: [resources/websockets.md](resources/websockets.md)

Do not load unrelated references.

### 3. Implement From a Bare Page

Start with only:

```html
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>View name</title>
  <style>
    html, body { height: 100%; }
    body { margin: 0; }
  </style>
</head>
<body>
  <!-- Requested UI only -->
  <script>
    // Required integration only
  </script>
</body>
</html>
```

Add declarations only when the requested design or behavior needs them. The skeleton is a starting point, not a mandatory block to preserve.

### 4. Preserve Required Correctness

Apply these rules only when the corresponding integration exists:

- Derive `wss://` from HTTPS and `ws://` from HTTP.
- Preserve query parameters used for Ontime cloud authentication, excluding a local `server` override.
- For `file://`, use `http://localhost:4001` unless the user requested another local server.
- Account for the Ontime cloud stage path when using `getontime.no`.
- Merge `runtime-patch` into existing runtime state; do not replace the full state.
- Treat websocket `clock` values as authoritative. Never increment the server clock locally.
- Reconnect a required WebSocket after closure.
- Use absolute HTTP paths under the resolved server origin/base path.

Implement these rules with the fewest helpers practical for the selected features. A view with no HTTP fetch does not need HTTP URL helpers. A view with no WebSocket does not need socket logic.

### 5. Keep Rendering Narrow

- Cache or query only DOM elements that are updated.
- Format only values displayed by the view.
- Use one render function only if several fields must update together; otherwise update the affected element directly.
- Add intervals only for behavior that truly needs an interval.
- Avoid polling when websocket updates or a one-time fetch satisfy the requirement.
- Add a connection status element only when the user asks for one or loss of connection would make the display misleading.

### 6. Validate and Prune

Verify:

1. Every requested element and behavior is present.
2. The JavaScript parses.
3. Required Ontime URLs work in the target environment.
4. Runtime patches preserve previously received fields.
5. No external dependency is required.
6. The page contains no unused selectors, functions, variables, state fields, endpoints, fallback panels, or copied reference code.

Use this syntax check when applicable:

```bash
node -e "const fs=require('fs'),h=fs.readFileSync('index.html','utf8'); new Function([...h.matchAll(/<script>([\s\S]*?)<\/script>/g)].map(m=>m[1]).join('\n')); console.log('JS syntax ok')"
```

## Feature Gates

Include the following only when explicitly required:

| Code or behavior | Include when |
|---|---|
| Rundown fetch and parsing | A full or partial schedule is displayed |
| Rundown polling | The requested schedule must refresh without a relevant websocket refetch signal |
| Project fetch | Project metadata is displayed |
| Logo URL encoding | A project logo is displayed |
| QR encoder | A scannable QR code is requested |
| Playback endpoints | Interactive playback controls are requested |
| Message handling | A message is displayed |
| Auxiliary timer handling | An auxiliary timer is displayed or controlled |
| Progress calculation | A progress bar or percentage is displayed |
| Clock fallback interval | A continuously moving fallback clock is required |
| Connection UI | Connection state is requested or operationally necessary |
| Legacy endpoint fallback | The user must support an older Ontime version |

When a feature is gated out, omit all of its markup, CSS, JavaScript, state, and helpers.

## Avoid

- Copying a comprehensive template into every view.
- Adding all runtime fields to state by default.
- Fetching both rundown and project data "just in case."
- Shipping a QR library when plain URL text was requested.
- Adding a 30-second poll to a view that does not display a rundown.
- Adding elaborate reconnection UI to a simple timer.
- Building generic theme systems for a fixed design.
- Adding mobile, touch, keyboard, accessibility, or browser compatibility behavior not needed by the requested use case. Preserve basic semantic HTML, but do not invent product requirements.
- Retaining unused code after requirements change.
