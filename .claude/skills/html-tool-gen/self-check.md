# Compliance Self-Check (on request only)

**Do not run this automatically after every generation or edit.** It costs a full re-read of the output plus reasoning over every item below, which is expensive to repeat on every tool. Only view and run this file when the user explicitly asks for it — e.g. "check this tool against the rules," "audit this file," "did you follow the structural rules?", "verify this is compliant."

When asked, view the tool's current HTML and check it against every item:

- [ ] Exactly one `<style>` tag inside `<head>`
- [ ] Exactly one `<script>` tag at the bottom of `<body>`
- [ ] Zero `style="..."` attributes
- [ ] Zero `on*="..."` attributes
- [ ] No extra `<link>` or `<script src=...>` tags beyond Google Fonts
- [ ] If a logo is included, `HINICIO_LOGO` starts with the literal `data:image/png;base64,` prefix immediately followed by the base64 payload (no missing prefix, no gap, no truncation)
- [ ] Import/Export JSON buttons are hidden (via `.hidden` class) whenever `IS_IFRAME` is `true`
- [ ] `btnSendConfig`, if present, keeps the same label/icon across every state (idle, sending, success, error) — no "Sending…" swap. The label/icon itself (default `💾 Save`) is a style choice and may legitimately differ if the user asked for different wording or branding
- [ ] `btnSendConfig`, if present, gains a `dirty` class (with visible highlight styling) whenever current data differs from the last-saved snapshot, and loses it once saved
- [ ] `POST_SYNC_URL` line is present, untouched, in every tool — even ones that don't use collaborative/partial-save mode
- [ ] Exactly one save mode is active per tool, never both: **either** `btnSendConfig` is a normal, visible, always-available button (regular bulk-save tools) **or** the tool is in collaborative/partial-save mode, in which case `btnSendConfig` is created but permanently `hidden` and `btnSyncCheck` / `btnSavePatch` / `btnForceSave` are the buttons the user actually sees — **except** the server-version-`"0"` state, which is a deliberate, narrow override of this rule, not a violation of it
- [ ] If collaborative/partial-save mode was not requested for this tool, `btnSyncCheck`, `btnSavePatch`, and `btnForceSave` do not exist in the DOM at all — not hidden, not present, no dead code referencing them
- [ ] If in collaborative/partial-save mode: when the server version is the string `"0"` (nothing has ever been saved), `btnSendConfig` is shown instead of the patch/sync buttons — `btnSavePatch` and `btnSyncCheck` stay hidden and unwired, since there's nothing yet to diff or sync against
- [ ] If in collaborative/partial-save mode: `btnForceSave` is revealed whenever a `savePatch()` response status is anything other than `200` or `409` — not only after an explicit "keep my local changes" conflict choice — so the user is never left unable to save
- [ ] The `// const CONFIG_IS_PUBLIC = true;` line is untouched — still present, still commented out, never assigned a value
- [ ] Any "when public / on public URL" behavior reads `CONFIG_IS_PUBLIC` defensively (`typeof ... !== 'undefined'`) rather than declaring or activating it — no duplicate/renamed variable created for the same concept
- [ ] On the clean→dirty transition, `window.parent.postMessage({ type: "DASHBOARD_STATUS", status: "unsaved" }, "*")` is sent exactly as specified (no renamed keys, no different origin)
- [ ] On successful save, `window.parent.postMessage({ type: "DASHBOARD_STATUS", status: "saved" }, "*")` is sent exactly as specified

If the tool uses private per-user config (`private-config.md`):
- [ ] `btnSavePrivateConfig` only exists when `POST_PRIVATE_CONFIG_URL` is set, starts hidden, and only becomes visible once a private field is dirty (hides again after a successful save) — it never sits visible with nothing to save
- [ ] It only sends a plain full POST — never a patch, never wired to `checkSync()`/`btnSyncCheck`/`btnSavePatch`/`btnForceSave`
- [ ] Its dirty tracking (`_lastSavedPrivateSnapshot`, `updatePrivateSaveButtonDirtyState()`) and feedback (`savedPrivatePill`/`errPrivatePill`) are fully separate from the public config's — no shared variables, classes, or pills
- [ ] No value appears in both `buildConfigJSON()` and `buildPrivateConfigJSON()` — every private field lives only in the latter
- [ ] Every private field (or the section grouping them) has a visible lock indicator (default `🔒`) in its label/heading text — not just the `data-private` attribute, not color/placeholder/tooltip alone
- [ ] `btnExportConfig`/`btnImportConfig` and `applyConfig()` never read or write a private field — only `applyPrivateConfig()` and `_INJECTED_PRIVATE_CONFIG` do
- [ ] No private field's value is ever written to `localStorage`, iframe or not
- [ ] The private-save flow never posts a `DASHBOARD_STATUS` message

Report back only the items that failed (with a one-line fix), plus a pass/fail summary. Don't re-print the whole checklist in the response.
