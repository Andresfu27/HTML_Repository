# Private Per-User Config (Opt-In)

A dedicated save path for values that must never be shared when the tool itself is shared — typically an API key, token, or other credential. The backend stores this in a separate per-user table, so the same tool URL loaded by two different people never shows or sends the other person's data.

This is a **small, standalone mechanism** — it exists to carry a handful of sensitive fields, not a general second config. Don't grow it into a patch/sync system: it never chunks, diffs, or merges. Every save is a full, plain POST of just the private fields.

---

## When to wire this in

- **The user explicitly asks for it** — "store this API key per-user", "don't share this with other users of the tool", "keep this private". Build it directly.
- **You're adding a field that's clearly a secret** — an API key, access token, password, or similar credential — to a tool's config, and nothing says otherwise. Default to routing it through this mechanism rather than the regular (shared) config, and say in your reply that you did this and why, so the user can redirect you if they actually wanted it shared. Once you've made this call for a tool, keep applying the same convention to any further private fields added to that tool.

This is **independent of `collaborative.md`**: whether a tool's regular config uses plain bulk-save or collaborative/patch mode has no bearing on this mechanism, and vice versa. Never route a private field through `btnSavePatch`/`checkSync()`/`btnForceSave`, and never let this mechanism's presence change anything about those.

---

## New protected constant

`POST_PRIVATE_CONFIG_URL` joins the protected-lines list, but **only in tools that use this mechanism** — unlike `POST_SYNC_URL`, it is not declared in every tool. Once added to a tool, treat it as protected the same way: don't rename, reorder, or remove it.

```js
const POST_PRIVATE_CONFIG_URL = '@@URLPOSTPRIVATECONFIG@@';
```

Gate everything on this exactly like `POST_CONFIG_URL` gates `btnSendConfig`: only create `btnSavePrivateConfig` (and only fetch) when the placeholder has been replaced.

Reuse the existing `POST_CONFIG_AUTH` / `POST_CONFIG_UID` constants for the private request's auth header and uid field — don't declare separate ones unless the user says the private endpoint needs different credentials.

## Static injection

Same pattern as the regular `_INJECTED_CONFIG` block, with its own markers and its own JSON slot so the backend can template in one user's private data at serve time without touching the shared config block:

```js
/* @@STATIC_CONFIG_PRIVATE_START@@
const _INJECTED_PRIVATE_CONFIG = @@STATIC_CONFIG_PRIVATE_JSON@@;
@@STATIC_CONFIG_PRIVATE_END@@ */
```

On `DOMContentLoaded`, alongside the existing `_INJECTED_CONFIG` check: if `_INJECTED_PRIVATE_CONFIG` exists and is non-empty, call `applyPrivateConfig(_INJECTED_PRIVATE_CONFIG)`.

## `buildPrivateConfigJSON()` / `applyPrivateConfig()`

Mirror `buildConfigJSON()` / `applyConfig()`, but for private fields only:

```js
function buildPrivateConfigJSON() {
  return {
    // Only private, per-user values go here (e.g. apiKey) — never a value
    // that also appears in buildConfigJSON().
  };
}

function applyPrivateConfig(cfg) {
  if (!cfg || typeof cfg !== 'object' || !Object.keys(cfg).length) return;
  // document.getElementById('apiKey').value = cfg.apiKey ?? '';
}
```

**A value lives in exactly one of `buildConfigJSON()` / `buildPrivateConfigJSON()`, never both.** This is what keeps private data out of Export JSON and out of the regular save payload — `btnExportConfig` and `sendConfig()`/`savePatch()`/`btnForceSave` only ever call `buildConfigJSON()`, so anything absent from it is already safe. Likewise, `applyConfig()` (used by Import and by `_INJECTED_CONFIG`) must never write into a private field — only `applyPrivateConfig()` does.

### Marking a field private in the HTML

Two separate things, both needed:

- **For the code**: `data-private="true"` on the field itself, by default.

```html
<input id="apiKey" type="password" data-private="true">
```

This is a marker for you and for anyone reading the file later, not something the code needs to query — since `buildConfigJSON()`/`buildPrivateConfigJSON()` are hand-written per tool, the real guarantee is the one-list-or-the-other rule above. Use `data-private="true"` unless the tool's structure makes another convention clearer (e.g. a wrapper section for a whole block of private fields) — adapt case by case, but stay consistent within one tool once you've picked something.

- **For the user**: every private field also needs a visible lock indicator, so it's obvious at a glance — without knowing anything about `data-private` — which fields stay on this device/account and which get shared with everyone who opens the tool. This is a functional requirement (some clear marker must be there), not just decoration; the exact treatment is a style choice, same as everywhere else. Default: prefix the field's label with the same `🔒` used on `btnSavePrivateConfig`.

```html
<label for="apiKey">🔒 API Key</label>
<input id="apiKey" type="password" data-private="true">
```

If several private fields sit together in one wrapper/section, a single `🔒 Private — not shared` heading on the section covers all of them; you don't need to repeat the lock on every field inside it. Either way, don't rely on placeholder text, color alone, or a tooltip as the only signal — the lock (or equivalent icon) needs to be part of the visible label/heading text itself.

## Private dirty tracking — fully independent

Its own baseline, its own function, never touching the public tracker's variables or classes. Unlike the public Save button (always visible, just highlighted when dirty), this one **stays hidden until there's actually something private to save** — it appears on the first change and disappears again once saved, so it never sits there unexplained for someone who has no private fields to fill in:

```js
let _lastSavedPrivateSnapshot = null;

function updatePrivateSaveButtonDirtyState() {
  const btn = document.getElementById('btnSavePrivateConfig');
  if (!btn || _lastSavedPrivateSnapshot === null) return;
  const dirty = JSON.stringify(buildPrivateConfigJSON()) !== _lastSavedPrivateSnapshot;
  btn.classList.toggle('dirty', dirty);
  btn.classList.toggle('hidden', !dirty);
}

document.body.addEventListener('input', updatePrivateSaveButtonDirtyState);
document.body.addEventListener('change', updatePrivateSaveButtonDirtyState);
```

Set the baseline once on `DOMContentLoaded` (after `applyPrivateConfig` runs) and again after every successful private save — same shape as the public baseline, just never shared with it.

This tracker does **not** post `DASHBOARD_STATUS` messages. That protocol is the public config's contract with the parent dashboard; don't fold private-save state into it.

## `btnSavePrivateConfig` — dedicated button, plain full save

- **Only rendered if `POST_PRIVATE_CONFIG_URL` is set.** If the placeholder isn't replaced, don't create it — not hidden, not present.
- **Created hidden, and stays hidden until a private field is actually dirty.** This is the one deliberate difference from `btnSendConfig`'s always-visible rule — a save button for data the user hasn't touched (and, for most users, will never fill in) is just confusing clutter in the toolbar. Reveal it on the clean→dirty transition, hide it again once the save succeeds.
- Fully separate from `btnSendConfig` / `btnSyncCheck` / `btnSavePatch` / `btnForceSave` — all of those can coexist with it on screen at once; none of the "only one save mode visible" rules from `collaborative.md` apply to it.
- Default label `🔒 Save private data` (style, not function — same split as the regular Save button: adapt the wording/icon to the app's branding or the user's request, but keep it constant across every state; no "Sending…" swap).
- Always a plain full `POST` of `buildPrivateConfigJSON()` — never a patch, never a sync-check, regardless of what mode the tool's regular config is in.
- Own feedback pills, `savedPrivatePill` / `errPrivatePill` — kept separate from `savedPill`/`errPill` so a public save and a private save in flight at the same time never overwrite each other's feedback. Each save succeeds or fails entirely on its own.

```js
async function sendPrivateConfig() {
  const btn = document.getElementById('btnSavePrivateConfig');
  btn.disabled = true;
  try {
    const headers = { 'Content-Type': 'application/json' };
    if (POST_CONFIG_AUTH && POST_CONFIG_AUTH !== '@@AUTHORIZATION@@') {
      headers['Authorization'] = POST_CONFIG_AUTH;
    }
    const res = await fetch(POST_PRIVATE_CONFIG_URL, {
      method: 'POST',
      headers: headers,
      body: JSON.stringify({
        config: buildPrivateConfigJSON(),
        uid:    POST_CONFIG_UID
      })
    });
    if (!res.ok) throw new Error('HTTP ' + res.status);
    const pill = document.getElementById('savedPrivatePill');
    pill.textContent = 'Saved ✓';
    pill.classList.add('show');
    setTimeout(function () { pill.classList.remove('show'); }, 3000);
    _lastSavedPrivateSnapshot = JSON.stringify(buildPrivateConfigJSON());
    btn.classList.remove('dirty');
    btn.classList.add('hidden');
  } catch (err) {
    const pill = document.getElementById('errPrivatePill');
    pill.textContent = 'Error: ' + err.message;
    pill.classList.add('show');
    setTimeout(function () { pill.classList.remove('show'); }, 4000);
  } finally {
    btn.disabled = false;
    btn.textContent = '🔒 Save private data';
  }
}

if (POST_PRIVATE_CONFIG_URL && POST_PRIVATE_CONFIG_URL !== '@@URLPOSTPRIVATECONFIG@@') {
  const btn = document.createElement('button');
  btn.id = 'btnSavePrivateConfig';
  btn.textContent = '🔒 Save private data';
  btn.classList.add('hidden'); // revealed by updatePrivateSaveButtonDirtyState() once something private changes
  btn.addEventListener('click', sendPrivateConfig);
  document.getElementById('configToolbar').appendChild(btn);
}
```

Add `#savedPrivatePill` / `#errPrivatePill` spans to the toolbar markup next to the existing `savedPill`/`errPill`, and wire the `DOMContentLoaded` handler to also apply/baseline the private config:

```js
document.addEventListener('DOMContentLoaded', function () {
  if (typeof _INJECTED_CONFIG !== 'undefined') applyConfig(_INJECTED_CONFIG);
  _lastSavedSnapshot = JSON.stringify(buildConfigJSON());
  if (typeof _INJECTED_PRIVATE_CONFIG !== 'undefined') applyPrivateConfig(_INJECTED_PRIVATE_CONFIG);
  _lastSavedPrivateSnapshot = JSON.stringify(buildPrivateConfigJSON());
});
```

## Never persisted client-side

Private values must never touch `localStorage` — inside an iframe or not. The existing "no localStorage inside an iframe" rule for the regular config is a convenience trade-off; for private data there's no exception either way, since anything written to `localStorage` sits in the browser indefinitely. Keep private values in memory only, for the lifetime of the page.

## Export / Import stay public-only

`btnExportConfig` and `btnImportConfig` are untouched by this mechanism — they call `buildConfigJSON()` / `applyConfig()` exactly as before. Never add a private-data export or import path; that would hand a credential to whoever opens the downloaded file or has file access, defeating the entire point of keeping it server-side and per-user.
