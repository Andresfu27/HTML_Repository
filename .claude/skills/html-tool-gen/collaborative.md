# Collaborative / Partial-Save Mode (Opt-In)

Only view this file when either is true:
- The user explicitly asks for **partial save** (avoid sending the whole config on every save, only the diff) or **collaborative editing** (multiple people can have the tool open at once, and edits from other sessions shouldn't silently clobber each other).
- You identified one of the signals below and the user accepted the proposal to switch.

Build a tool in regular bulk-save mode (see `skeleton.md` / `modify.md`) by default. Both needs above are served by the same mechanism — there is no lighter "partial save only" variant. If neither is requested, don't view this file at all: no sync button, no patch button, no force button, no jsonpatch library, no baseline tracking beyond `_lastSavedSnapshot`.

When this mode is active, it **replaces** the regular Save button as the thing the user interacts with — `btnSendConfig` still exists (per the protected-lines rule) but stays permanently `hidden`, surfacing only as a last-resort "Force my changes" fallback in two situations: after a specific conflict-resolution choice, or after a broken patch save (both described below). Never show both a visible `btnSendConfig` and the collaborative buttons on the same tool — except the never-saved-yet (`"0"`) state below, which deliberately shows `btnSendConfig` alone instead of the patch/sync pair, precisely because there is nothing yet to patch or sync against.

## Proactively propose this mode — even if the user didn't ask

Don't wait for the user to know this mode exists. While planning or building a tool, watch for either signal, independent of whether the user used words like "partial save" or "collaborative":

- **High data volume** — the tool's config will realistically carry embedded images, PDFs, other base64 blobs, large tables/arrays, or anything else likely to make a single save payload large. A rough gut-check: if a saved config could plausibly run into the hundreds of KB or beyond, or clearly contains binary-ish embedded assets, this signal applies.
- **Simultaneous multi-user editing** — the request implies more than one person will have this tool open and editing at once: shared dashboards, team trackers, review/approval tools, anything described as "our," "the team's," or "shared," or any explicit mention that multiple people will use the same instance concurrently.

When either signal is present:
1. **Still build the tool.** Don't block on this — proceed with the regular bulk-save mode as the default, unless the user has already asked for the other mode.
2. **Flag it explicitly**, naming which signal triggered the suggestion (e.g. "this config will likely carry embedded images, which could make every save slow" or "since multiple people will edit this at once, concurrent saves could silently overwrite each other's changes").
3. **Give a concrete effort estimate for converting**, scaled to the tool's actual complexity — not a fixed number. As a rough guide: a simple tool with a handful of fields is a small, mechanical change (bundling the patch library, adding the three buttons, wiring the sync/save/merge/conflict calls — no new data modeling). A tool with many distinct editable entities (e.g. a list of named items a user can add/edit/remove, like nodes in a diagram) needs proportionally more work, mainly in building out the human-readable path-description dictionary and entity lookups so merge/conflict messages make sense. Say which of these your tool is closer to and why.
4. **Also flag the backend dependency**: this mode only works if the backend actually implements the `POST_SYNC_URL` sync-check endpoint and the PATCH contract (`missing_ops`, `conflicting_paths`, `server_version`) described below — mention this so the user isn't surprised if their backend doesn't support it yet.
5. **Only convert if the user explicitly agrees.** If they decline, don't ask again for the same tool in the same conversation, and proceed with (or leave as-is) the regular bulk-save version. Never switch modes unilaterally because you detected one of these signals — the proposal is the whole intervention; the decision is the user's.

## Why this exists

Sending the whole config on every save is simple but has two costs: it's wasteful for large configs, and if two people have the tool open, the second save silently overwrites whatever the first person changed — even in fields neither of them touched. Patch-based saving fixes both: only the diff travels over the wire, and the backend can tell the client precisely which fields (if any) actually collided with someone else's changes, instead of a blunt "stale, reload everything."

## Required library

Bundle `fast-json-patch` (the browser build) inline in the single `<script>` block, above the tool logic. It provides `jsonpatch.compare(a, b)`, `jsonpatch.applyPatch(doc, ops, validate, mutate)`, and `jsonpatch.getValueByPointer(doc, pointer)`. Note the mutate default: `applyPatch`'s 4th argument controls whether it mutates the document you pass in — when you don't intend to mutate a shared object, always pass a `deepClone()` of it in as the document, since relying on the 4th argument is easy to get wrong.

## New config constant

`POST_SYNC_URL` (already declared as a protected line in every tool, per the script-block rules) becomes live in this mode: it's the endpoint the sync-check call POSTs to. It is **independent** of `POST_CONFIG_URL` — a backend could support save without sync-check, or vice versa — so gate each button on its own URL being configured, never on both together.

## Server version tracking, and the never-saved (`"0"`) state

Track the server's version marker — a **string**, e.g. `__last_server_update_time__` from the injected config — in a variable such as `lastServerUpdateTime`. It travels outgoing as `last_update_from_tool` and comes back incoming as `server_version` (see `checkSync()` / `savePatch()` below).

**A version of the string `"0"` means the server document is empty — nothing has ever been saved.** There is nothing yet to diff or merge against, so this is a deliberate, narrow exception to the "never show both save UIs" rule above:
- Show the regular bulk `btnSendConfig` instead of the patch/sync pair — unhide it rather than leaving it in its normal collaborative-mode hidden state.
- Keep `btnSyncCheck` and `btnSavePatch` hidden and unwired (no click listener attached) — don't offer patch-based saving or sync-checking when there's no prior server state to compare to.

Compute this once, synchronously, right where the injected config is available (before `DOMContentLoaded`, since `_INJECTED_CONFIG` is already a plain literal by the time the script runs):
```js
const hasNoServerVersionYet = typeof _INJECTED_CONFIG !== 'undefined'
  && _INJECTED_CONFIG
  && _INJECTED_CONFIG.__last_server_update_time__ === '0';
```
Gate button creation/wiring on this flag:
```js
if (POST_CONFIG_URL && POST_CONFIG_URL !== '@@URLPOSTCONFIG@@') {
  const btn = document.createElement('button');
  btn.id = 'btnSendConfig';
  btn.textContent = '💾 Save';
  btn.classList.toggle('hidden', !hasNoServerVersionYet); // shown only in the never-saved state; hidden otherwise, per collaborative mode
  btn.addEventListener('click', sendConfig);
  document.getElementById('configToolbar').appendChild(btn);
  if (!hasNoServerVersionYet) {
    document.getElementById('btnSavePatch').classList.remove('hidden');
    document.getElementById('btnSavePatch').addEventListener('click', savePatch);
  }
}
if (!hasNoServerVersionYet && POST_SYNC_URL && POST_SYNC_URL !== '@@URLSYNCCONFIG@@') {
  document.getElementById('btnSyncCheck').classList.remove('hidden');
  document.getElementById('btnSyncCheck').addEventListener('click', checkSync);
}
```
Once the first save happens (via `sendConfig()` in this state), the server's version stops being `"0"`, and a subsequent load proceeds through the normal collaborative-mode path described in the rest of this chapter.

## Two extra buttons, one repurposed

| Button | Purpose | Visibility |
|---|---|---|
| `btnSyncCheck` | Ask the backend "would my pending changes merge cleanly?" without saving anything | Shown only if `POST_SYNC_URL` is set **and** the server version isn't `"0"` |
| `btnSavePatch` | Send only the diff (PATCH) | Shown only if `POST_CONFIG_URL` is set **and** the server version isn't `"0"` |
| `btnForceSave` | Full-document overwrite (POST), same request shape as `sendConfig()` | Hidden by default; revealed after the user picks "keep my local changes" on a conflict, or after a broken `savePatch()` response (see below) |
| `btnSendConfig` | Regular save button | Created (per protected-lines rule); hidden in normal collaborative mode, but **shown instead of the patch/sync pair** when the server version is `"0"` |

Label `btnSavePatch` "💾 Save changes" and `btnForceSave` "💾 Force my changes" by default (floppy-disk icon on both, matching `btnSendConfig`'s icon but with distinct text so the user can tell which action they're taking). The exact wording/icon is a style choice — adapt it to the app's own branding or language, or to whatever the user asks for; the functional requirement is only that each button's label stays distinguishable from the others.

`btnForceSave` also functionally needs a treatment that reads as unusual/overriding, distinct from the regular save color — by default a warning color (not the navy/cyan pair), reusing the palette's remaining variables or a single additional semantic `--*-warning` custom property if none fit, following the same "fall back before introducing a new color" rule as the rest of the palette. If the tool uses different branding, use that brand's own warning/danger color instead, or whatever color the user specifies — the requirement is the visual distinction from the regular save button, not this specific hex value.

## Two baselines, tracked separately

- `_lastSavedSnapshot` — unchanged from the regular-mode rule: last state confirmed written to the server via a full POST. Drives `btnSendConfig`'s dirty class (even though the button is hidden outside the `"0"` state, keep tracking it — `btnForceSave` reuses the same dirty-clearing logic on save success).
- `lastSyncedConfig` — new in this mode: a **deep clone** of the config as of the last successful sync/merge/patch-save, used purely to compute the outgoing patch (`jsonpatch.compare(lastSyncedConfig, currentConfig)`). Never conflate the two baselines — they answer different questions ("has anything changed since my last real save" vs. "what do I need to send to catch the server up").
- Add a second dirty tracker, `updateSavePatchDirtyState()`, that highlights `btnSavePatch` (same visual language as `btnSendConfig.dirty`) whenever `currentConfig` differs from `lastSyncedConfig`. Call it everywhere `updateSaveButtonDirtyState()` is called, plus after any merge or conflict-resolution outcome that moves `lastSyncedConfig`.

## Never send or receive the full document

Every request in this mode carries only a patch (an array of RFC 6902 ops) plus a small version marker (`last_update_from_tool` outgoing, `server_version` incoming) — never the whole config, in either direction. This is the entire point of the mode; if a step below seems to need the full document, that's a sign to rethink it rather than fetching one. (The one deliberate exception is `btnForceSave`'s full POST, and the `"0"` state's `sendConfig()` — both are explicit, narrow overrides, not the everyday path.)

## Headers and spinner — every request this mode makes, no exceptions

`checkSync()`, `savePatch()`, and `btnForceSave`'s full POST are three separate `fetch()` calls, and each one independently needs both of the following. Neither is optional on any one of them — a request built without the header, or a button that skips the spinner, is a bug even if the rest of the flow is correct.

**Authorization header.** Every request built in this mode follows the exact same conditional header pattern already used by `sendConfig()` in the regular-mode skeleton — copy it into `checkSync()`, `savePatch()`, and the shared full-POST helper that `btnForceSave` calls, not just into whichever one you write first:
```js
const headers = { 'Content-Type': 'application/json' };
if (POST_CONFIG_AUTH && POST_CONFIG_AUTH !== '@@AUTHORIZATION@@') {
  headers['Authorization'] = POST_CONFIG_AUTH;
}
```
It's easy to write `checkSync()` first, get it working, then copy-paste its `fetch()` shape into `savePatch()` and the force-save path while trimming the header block along with it since it "looked like sync-specific setup" — it isn't. Treat this block as protected in the same sense as the `POST_CONFIG_*` constants: every outgoing request in this file carries it.

**Spinner, always, regardless of how fast the response comes back.** Every button that triggers a request in this mode (`btnSyncCheck`, `btnSavePatch`, `btnForceSave`) needs a spinner element inside it that shows for the full duration of that request — not just on slow requests, not skipped for a "quick" dry-run check. Markup:
```html
<button id="btnSyncCheck" class="hidden" title="Check for server changes">
  <span class="btnSpinner hidden"></span><span class="btnLabel">&#8635;</span>
</button>
```
```css
.btnSpinner {
  width: 11px; height: 11px; border: 2px solid rgba(255,255,255,0.35); border-top-color: #fff;
  border-radius: 50%; display: inline-block; animation: btnSpin 0.7s linear infinite;
}
@keyframes btnSpin { to { transform: rotate(360deg); } }
```
And in the handler, unconditionally:
```js
const spinner = btn.querySelector('.btnSpinner');
btn.disabled = true;
spinner.classList.remove('hidden');
try {
  // ...request...
} finally {
  btn.disabled = false;
  spinner.classList.add('hidden');
}
```
This applies even to `checkSync()`, which is a lightweight dry-run and will often resolve almost instantly — resist the temptation to treat it as too quick to need one. A user who clicks and sees nothing happen for even a few hundred milliseconds will click again, and a consistent spinner is cheaper than explaining why some buttons in the toolbar show one and others don't.

## `checkSync()` — ask before saving, without saving

Wired to `btnSyncCheck`. POSTs `{ patch: jsonpatch.compare(lastSyncedConfig, currentConfig), uid: POST_CONFIG_UID, last_update_from_tool: lastServerUpdateTime }` to `POST_SYNC_URL`, with the headers block above (Authorization included whenever `POST_CONFIG_AUTH` is set) and the spinner shown for the request's full duration. This call must never write anything server-side — it's a dry run. Handle three outcomes:

1. **`is_updated: true`** (HTTP 200) — nothing pending, show a brief "Up to date" confirmation.
2. **200 with `missing_ops`** — the server has changes on fields you never touched; safe to merge. Call `showMergeModal(missingOps, serverVersion)`.
3. **HTTP 409 with `conflicting_paths`** — some of your pending changes collide with changes made elsewhere. Call `showConflictModal(conflictingPaths)`.

## `savePatch()` — the everyday save

Wired to `btnSavePatch`. Same patch-construction as `checkSync()`, same headers block (don't drop the Authorization check just because this is a different function), same spinner treatment, but PATCHes `POST_CONFIG_URL`. On success, update `lastServerUpdateTime` from the response's `server_version`, advance `lastSyncedConfig` to the current config, and clear `btnSavePatch`'s dirty class. If the response also includes non-empty `missing_ops` (the save succeeded but the server had moved further ahead on unrelated fields), immediately offer the same merge flow as `checkSync()` — don't require a second explicit sync check for that.

**On failure, distinguish the response status carefully — don't treat every non-2xx the same way:**
- **`409`** — a genuine conflict; hand off to the conflict modal flow as usual (see below). This is an expected, resolvable outcome, not a broken save path.
- **Anything else that isn't `200` or `409`** (500, 502, 401, a network failure, etc.) — the normal save path itself is broken in a way the UI can't resolve on its own. Reveal `btnForceSave` immediately, right alongside the error feedback, so the user always has a way to persist their work instead of being stuck retrying a save that keeps failing the same way:
```js
if (!res.ok) {
  let serverMsg = '';
  try {
    const errData = await res.json();
    if (errData && errData.msg) serverMsg = errData.msg;
  } catch (e) { /* body missing or not JSON, fine */ }
  // !res.ok already excludes 200, so checking for 409 alone is enough here.
  if (res.status !== 409) {
    document.getElementById('btnForceSave').classList.remove('hidden');
  }
  throw new Error(res.status + (serverMsg ? ': ' + serverMsg : ''));
}
```
- Once revealed this way, don't auto-hide `btnForceSave` again — only an explicit conflict-resolution flow (the `_forcedPaths`-driven show/hide described in the Conflict modal section below) should re-hide it, based on its own state.

## Merge modal — applying someone else's non-conflicting changes

List the affected fields in the user's terms, not raw JSON Pointers — see "Human-readable path descriptions" below. On confirm:

1. Compute `merged = jsonpatch.applyPatch(deepClone(currentConfig), missingOps).newDocument` (what the user should see) and, separately, `newBaseline = jsonpatch.applyPatch(deepClone(lastSyncedConfig), missingOps).newDocument` (the new sync baseline).
2. **Snapshot both for comparison before calling any config-apply function.** If your `applyConfig()`-equivalent assigns nested objects by reference rather than copying them (common when re-rendering from the merged object), any post-render normalization can silently mutate `merged` after the fact, corrupting a same-object comparison done afterward. Compute the "did this merge account for everything, or do I still have other pending edits?" check (`JSON.stringify(merged) === JSON.stringify(newBaseline)`) *before* handing `merged` off to be rendered.
3. Render `merged` into the tool's live state.
4. Set `lastSyncedConfig = newBaseline` — **not** a snapshot of the post-render live state. Advancing the baseline by only the merged-in ops (applied to the *old* baseline) preserves any other unrelated local edits as still-diffable; naively resetting the baseline to match the newly-rendered document would make those other edits invisible to the next patch computation, silently dropping them from the next save.
5. Only if step 2's comparison came back equal (nothing else was pending) — trust `serverVersion` and write it to `lastServerUpdateTime`, and post the `"saved"` parent-window message as if fully caught up. If something else was still pending, leave the version alone; otherwise the tool would report a version the server hasn't actually confirmed for the fields still outstanding.

## Conflict modal — fields that collided

Never fetch the server's actual value for a conflicting field — that would mean pulling the full document. Instead, offer exactly two choices:

- **"Load server version"** — revert *only* the conflicting fields to whatever `lastSyncedConfig` already holds for them (the value as of the user's last successful sync — the most honest available substitute for "what's on the server now" without fetching it). Do not touch any other pending local edit, and do not move `lastSyncedConfig` itself — a pure revert-to-baseline is already consistent with the baseline, so nothing needs to move there. After reverting, immediately re-run `checkSync()`: the revert only resolved the paths that were actually conflicting, and other pending edits (if any) still deserve their own check.
- **"Keep my local changes"** — record the conflicting paths into a running list (`_forcedPaths`), and reveal `btnForceSave`. Nothing is sent yet; this only registers intent. If the user later resolves a different conflict by choosing "load server version" on some of the same paths, remove those paths from `_forcedPaths` and re-hide `btnForceSave` if the list becomes empty.

`btnForceSave`, when clicked, does a full POST (identical request shape to `sendConfig()` — same headers block, same spinner treatment) — the only point in the everyday flow (outside the `"0"` state) where the whole document travels. Since `btnSendConfig` and `btnForceSave` both do a full POST with the same idle-label-swap-and-restore behavior, it's worth writing one shared helper (taking the triggering button and its idle label as arguments) that both `sendConfig()` and `forceSaveConfig()` call, rather than duplicating the header/spinner/request logic twice. On success, treat the server's version the same way a full save always does: reset it to `1` (a full POST is a complete overwrite, not a patch chained off a prior version — mirror that locally rather than trusting any version field in the response), clear `_forcedPaths`, and hide `btnForceSave` again.

## Human-readable path descriptions

Users should never see a raw JSON Pointer like `/currentDiagram/nodes/3/x`. Build:

- A small, reusable dictionary mapping property names to display labels (e.g. `label` → "Label", `color` → "Color") — reused by every place that needs to describe a path, not redefined per call site.
- A lookup that identifies the entity a path touches by what the user actually named it (a node's own label, an item's own name) rather than its array index — fall back to an index-based description (`"Item #3"`) only when no name is available or the referenced item can't be resolved (e.g. it was deleted).
- Compose the two: `"<Entity 'Name'> – <Property label>"`, or just the entity description when there's no specific property (the whole entity changed) or the property isn't in the dictionary.
