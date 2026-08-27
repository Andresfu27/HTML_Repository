# Building a New Tool From Scratch

View this file when the user asks you to build/create a new HTML tool, or to transform something (a spec, a spreadsheet, another format) into one. Also view `brand.md` for colors/fonts/logo. If the request signals collaborative editing or partial save (see `collaborative.md`'s signal list), view that file too — otherwise skip it. If the tool needs to store a credential or other value that must stay private per-user (or you're about to add a field like that), view `private-config.md` too.

The absolute structural rules in `SKILL.md` (single style/script block, zero inline styles/scripts, no external resources beyond Google Fonts) apply to everything below — this file doesn't restate them.

## Script block rules

- Keep `CONFIG_IS_PUBLIC` (commented out), `POST_CONFIG_URL`, `POST_SYNC_URL`, `POST_CONFIG_AUTH`, `POST_CONFIG_UID`, `sendConfig()`, and its `addEventListener` **exactly as in the skeleton** — do not rename, reorder, remove, uncomment, or add/change any characters on those 5 lines.
- `POST_SYNC_URL` is present in **every** tool you generate, whether or not that tool uses collaborative/partial-save mode — it costs nothing to declare and stays inert (`'@@URLSYNCCONFIG@@'`, never fetched) unless that mode is explicitly wired in.
- **Replace only `buildConfigJSON()`** — its returned object must include every user-configurable value the tool exposes.
- **On `DOMContentLoaded`**: check if `_INJECTED_CONFIG` exists and is a non-empty object; if so, call `applyConfig(_INJECTED_CONFIG)` to pre-populate every field. Injected values always override hardcoded defaults.
- All tool logic goes **after** the `sendConfig` / `addEventListener` block, inside the same single `<script>` tag.

### `CONFIG_IS_PUBLIC` — the single flag for "tool is on a public URL" behavior
- The skeleton contains the line `// const CONFIG_IS_PUBLIC = true;`, commented out, one of the 5 protected lines above.
- **Never uncomment it, edit it, move it, or set its value.** The backend is responsible for injecting/activating this variable at deploy time — leave the line exactly as it appears in the skeleton, always commented out, with no exceptions.
- Whenever a request asks for behavior conditioned on the tool being public — hiding an element, disabling a feature, showing a different message, restricting an action — that behavior must be driven by reading this variable, not by creating or activating one yourself.
- Check it defensively: `typeof CONFIG_IS_PUBLIC !== 'undefined' && CONFIG_IS_PUBLIC === true`.
- **Never declare a second variable with this name or a similar one** (e.g. `isPublic`, `IS_PUBLIC`, `configIsPublic`) to represent the same concept.

## Config toolbar rules

### Export to JSON (`btnExportConfig`)
- Calls `buildConfigJSON()`, downloads result as `config.json`.
- Uses a hidden `<a>` with a blob URL — no `window.open`, no new tab.

### Import from JSON (`btnImportConfig`)
- Triggers a hidden `<input type="file" accept=".json">`.
- Reads file, parses JSON, calls `applyConfig(cfg)` — same behaviour as `DOMContentLoaded` injection.

### Save button (`btnSendConfig`) — regular (bulk) save mode

This is the **default** behavior: the whole config is sent in one request every time the user saves. Use this unless the user has explicitly asked for partial-save or collaborative editing for this specific tool — see `collaborative.md` for that alternative. Never mix the two: a tool is in one mode or the other, never both.

- **Only rendered if `POST_CONFIG_URL` is set** (i.e. its value ≠ `'@@URLPOSTCONFIG@@'`).
- If the placeholder is not replaced → do not create the button at all (not hidden, not present).
- **Labeled `💾 Save` by default** (Hinicio convention, per `brand.md`). This is a style choice, not a functional one — adapt the wording and/or icon if the user asks, or if the tool is being built to match another app's branding or language (e.g. "Publicar", "Update", a different icon). The functional requirement underneath, regardless of which label is chosen: it must stay the same string in every state (idle, sending, after success/error) — don't swap in a "Sending…" or "Saved!" label; use the `dirty` class, a spinner, or the feedback pills to communicate state instead.
- While the request is in flight the button may be `disabled` but its label must not change.
- In this mode, `btnSendConfig` is a normal, always-visible button — it is only ever hidden by mode selection (see `collaborative.md`), never by any other condition.

### Dirty state tracking
- Track a baseline snapshot (`_lastSavedSnapshot`, a `JSON.stringify` of `buildConfigJSON()`) representing the last-known-saved state.
- Set the baseline: once on `DOMContentLoaded` (after any injected/imported config is applied), and again immediately after a successful `sendConfig()` call.
- Any time the tool's data changes (field input/change events, or programmatically after `applyConfig()` runs from an Import), recompute `JSON.stringify(buildConfigJSON())` and compare it to the baseline.
- If they differ → the tool is "dirty": add a `dirty` CSS class to `btnSendConfig`. If they match → remove the `dirty` class.
- The `dirty` class must give the Save button a visible highlight so the user notices unsaved changes at a glance — style it in the single `<style>` block, never inline. This is a functional requirement (some highlight must exist); the treatment itself is a style choice — a glow/border using `--hinicio-cyan` by default (per `brand.md`), or the tool's own accent color if it uses different branding, or whatever the user asks for.
- Dirty tracking only matters when `btnSendConfig` exists; if `POST_CONFIG_URL` isn't set, skip this (no button to highlight).
- Native `input`/`change` events don't fire for values set programmatically (e.g. inside `applyConfig`), so explicitly recheck dirty state right after every `applyConfig()` call, not just via event listeners.

### Notifying the parent window of save status
- Whenever the tool transitions into the dirty state (the `dirty` class is added to `btnSendConfig`), post this exact message to the parent window:
```js
window.parent.postMessage({ type: "DASHBOARD_STATUS", status: "unsaved" }, "*");
```
- After a successful `sendConfig()` call (same point where `_lastSavedSnapshot` is updated and the `dirty` class removed), post this exact message:
```js
window.parent.postMessage({ type: "DASHBOARD_STATUS", status: "saved" }, "*");
```
- Only fire the `"unsaved"` message on the transition from clean → dirty, not on every subsequent input while already dirty.
- The message shape must be sent exactly as written above — do not rename keys, change the type string, or alter the target origin `"*"`.
- This applies regardless of whether the tool is running inside an iframe or not.

### Feedback pills (`savedPill`, `errPill`)
- Always present in the DOM; used by `sendConfig()` for success/error feedback.

### No localStorage inside an iframe
- Detect with `window.self !== window.top` → stored in `IS_IFRAME`.
- Inside iframe: never read from or write to `localStorage`. All state in memory only.
- Outside iframe: `localStorage` may be used for convenience.

### Import/Export JSON hidden inside an iframe
- If `IS_IFRAME` is `true`, hide `btnExportConfig`, `btnImportConfig`, and `importFileInput` via a `hidden` CSS class (`.hidden { display: none; }`), never via `.style.display`.
- `btnSendConfig` is unaffected by iframe state — its visibility depends only on `POST_CONFIG_URL` being set.

---

## The skeleton

Use this exact skeleton as the starting point for every new HTML tool:

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Tool Name</title>
<link href="https://fonts.googleapis.com/css2?family=Jost:wght@400;500;600;700&family=Mulish:wght@400;600;700&display=swap" rel="stylesheet">
<style>
/* All styles here */
.hidden { display: none; }
#btnSendConfig.dirty {
  box-shadow: 0 0 0 3px var(--hinicio-cyan-bright);
  border-color: var(--hinicio-cyan);
}
</style>
</head>
<body>

<!-- Config toolbar -->
<div id="configToolbar">
  <button id="btnExportConfig">⬇ Export JSON</button>
  <button id="btnImportConfig">⬆ Import JSON</button>
  <input id="importFileInput" type="file" accept=".json">
  <!-- btnSendConfig is injected by JS only when POST_CONFIG_URL is set -->
  <span id="savedPill"></span>
  <span id="errPill"></span>
</div>

<!-- Tool UI goes here -->

<script>
// ── STATIC CONFIG INJECTION ──
/* @@STATIC_CONFIG_START@@
const _INJECTED_CONFIG = @@STATIC_CONFIG_JSON@@;
@@STATIC_CONFIG_END@@ */

// const CONFIG_IS_PUBLIC = true;
const POST_CONFIG_URL = '@@URLPOSTCONFIG@@';
const POST_SYNC_URL = '@@URLSYNCCONFIG@@';
const POST_CONFIG_AUTH = '@@AUTHORIZATION@@';
const POST_CONFIG_UID = '@@UID@@';

const IS_IFRAME = window.self !== window.top;

if (IS_IFRAME) {
  document.getElementById('btnExportConfig').classList.add('hidden');
  document.getElementById('btnImportConfig').classList.add('hidden');
  document.getElementById('importFileInput').classList.add('hidden');
}

function buildConfigJSON() {
  return {
    // All user-configurable values go here
  };
}

function applyConfig(cfg) {
  if (!cfg || typeof cfg !== 'object' || !Object.keys(cfg).length) return;
  // Populate each field from cfg, e.g.:
  // document.getElementById('myField').value = cfg.myField ?? '';
}

// ── Dirty state tracking ──
let _lastSavedSnapshot = null;

function updateSaveButtonDirtyState() {
  const btn = document.getElementById('btnSendConfig');
  if (!btn || _lastSavedSnapshot === null) return;
  const dirty = JSON.stringify(buildConfigJSON()) !== _lastSavedSnapshot;
  const wasDirty = btn.classList.contains('dirty');
  btn.classList.toggle('dirty', dirty);
  if (dirty && !wasDirty) {
    window.parent.postMessage({
      type: "DASHBOARD_STATUS",
      status: "unsaved"
    }, "*");
  }
}

document.body.addEventListener('input', updateSaveButtonDirtyState);
document.body.addEventListener('change', updateSaveButtonDirtyState);

// ── Export ──
document.getElementById('btnExportConfig').addEventListener('click', function () {
  const json = JSON.stringify(buildConfigJSON(), null, 2);
  const a = document.createElement('a');
  a.href = URL.createObjectURL(new Blob([json], { type: 'application/json' }));
  a.download = 'config.json';
  a.click();
  URL.revokeObjectURL(a.href);
});

// ── Import ──
document.getElementById('btnImportConfig').addEventListener('click', function () {
  document.getElementById('importFileInput').click();
});
document.getElementById('importFileInput').addEventListener('change', function (e) {
  const file = e.target.files[0];
  if (!file) return;
  const reader = new FileReader();
  reader.addEventListener('load', function (ev) {
    try {
      applyConfig(JSON.parse(ev.target.result));
      updateSaveButtonDirtyState();
    } catch (err) {
      const pill = document.getElementById('errPill');
      pill.textContent = 'Import error: ' + err.message;
      pill.classList.add('show');
      setTimeout(function () { pill.classList.remove('show'); }, 4000);
    }
  });
  reader.readAsText(file);
  e.target.value = '';
});

// ── Send Config (only wired when URL placeholder is replaced) ──
async function sendConfig() {
  const btn = document.getElementById('btnSendConfig');
  btn.disabled = true;
  try {
    const headers = { 'Content-Type': 'application/json' };
    if (POST_CONFIG_AUTH && POST_CONFIG_AUTH !== '@@AUTHORIZATION@@') {
      headers['Authorization'] = POST_CONFIG_AUTH;
    }
    const res = await fetch(POST_CONFIG_URL, {
      method: 'POST',
      headers: headers,
      body: JSON.stringify({
        html:   null,
        config: buildConfigJSON(),
        uid:    POST_CONFIG_UID
      })
    });
    if (!res.ok) throw new Error('HTTP ' + res.status);
    const pill = document.getElementById('savedPill');
    pill.textContent = 'Saved ✓';
    pill.classList.add('show');
    setTimeout(function () { pill.classList.remove('show'); }, 3000);
    _lastSavedSnapshot = JSON.stringify(buildConfigJSON());
    btn.classList.remove('dirty');
    window.parent.postMessage({
      type: "DASHBOARD_STATUS",
      status: "saved"
    }, "*");
  } catch (err) {
    const pill = document.getElementById('errPill');
    pill.textContent = 'Error: ' + err.message;
    pill.classList.add('show');
    setTimeout(function () { pill.classList.remove('show'); }, 4000);
  } finally {
    btn.disabled = false;
    btn.textContent = '💾 Save';
  }
}

if (POST_CONFIG_URL && POST_CONFIG_URL !== '@@URLPOSTCONFIG@@') {
  const btn = document.createElement('button');
  btn.id = 'btnSendConfig';
  btn.textContent = '💾 Save';
  btn.addEventListener('click', sendConfig);
  document.getElementById('configToolbar').appendChild(btn);
  // If this tool uses collaborative/partial-save mode (see collaborative.md),
  // add btn.classList.add('hidden') here instead of leaving it visible
  // (unless the server version is "0" — in which case it stays visible instead),
  // and create btnSyncCheck / btnSavePatch / btnForceSave alongside it.
}

// ── TOOL LOGIC BELOW ──
document.addEventListener('DOMContentLoaded', function () {
  if (typeof _INJECTED_CONFIG !== 'undefined') applyConfig(_INJECTED_CONFIG);
  _lastSavedSnapshot = JSON.stringify(buildConfigJSON());
});
</script>
</body>
</html>
```
