# Modifying or Extending an Existing Tool

View this file when the user asks you to change, fix, or add a feature to an HTML tool that already exists (uploaded, pasted, or built earlier in this conversation) — as opposed to building one from scratch. Don't regenerate the skeleton or rewrite working parts; edit only what the request touches, and read the file first so any edit lands against its real current content.

The absolute structural rules in `SKILL.md` (single style/script block, zero inline styles/scripts, no external resources beyond Google Fonts) apply to your edit exactly as they would to a new build — an edit that reintroduces an inline `style=` or a second `<script>` block is still a violation, even if the rest of the file was already compliant. Preserve the existing single `<style>`/`<script>` blocks; add rules/logic inside them, don't create new ones.

## Before editing, identify what the change touches

- **Pure UI/logic addition** (new field, new chart, new section, new computation) — usually just needs: adding to `buildConfigJSON()` and `applyConfig()` if the new data should persist/export, and following the zero-inline-style/script rule for any new markup. No toolbar changes needed.
- **Touches the save/export/import toolbar** — re-read the relevant rules before editing:
  - Save button behavior, dirty-state tracking, and the `postMessage` calls: see `skeleton.md`'s "Config toolbar rules" section (Save button, Dirty state tracking, Notifying the parent window). Only the behavior is protected here — label wording/icon and highlight color are style, not function, and can be changed if asked (or to match another brand) without it counting as a rule violation.
  - Don't change `POST_CONFIG_URL`, `POST_SYNC_URL`, `POST_CONFIG_AUTH`, `POST_CONFIG_UID`, `sendConfig()`, or its `addEventListener` unless the request specifically requires it — these are protected lines.
  - Don't touch the commented-out `// const CONFIG_IS_PUBLIC = true;` line. If the request wants public-URL-conditioned behavior, gate it on reading that variable defensively — see `skeleton.md`'s `CONFIG_IS_PUBLIC` section.
  - If the tool is in collaborative/partial-save mode, the same split applies to `btnSyncCheck` / `btnSavePatch` / `btnForceSave`: visibility logic, request shapes, and baseline tracking in `collaborative.md` are protected; their labels and colors (including `btnForceSave`'s warning color) are style and can be adapted.
- **Touches branding/visuals** (colors, fonts, logo) — see `brand.md`. Keep the existing Hinicio palette/fonts unless the user explicitly asks to change them for this tool.
- **Needs to identify the connected user** (their email, a presenter/owner role, attributing an entry to whoever made it) — see `user-identity.md` for the detection pattern; don't invent a different identity source.
- **Adds collaborative editing or partial save to a tool that doesn't have it** — this is a mode conversion, not a small edit. Propose it per `collaborative.md`'s "Proactively propose this mode" section if you spot the signals, but only build it out if the user agrees; view `collaborative.md` in full before starting.
- **Tool is already in collaborative/partial-save mode and the request touches sync/save/merge/conflict behavior** — view `collaborative.md` before editing; its baselines (`_lastSavedSnapshot` vs `lastSyncedConfig`) and button-visibility rules are easy to break with a partial understanding.
- **Adds a field that must stay private per-user, or touches an existing private-save setup** — view `private-config.md`. Unlike collaborative mode this doesn't need the user's sign-off first: if you're adding something credential-like, wire it into the private mechanism and say you did. If the tool already has `btnSavePrivateConfig`, keep it fully separate from any edits to the regular save/collaborative flow — the two never share state.

## After editing

Don't run the full compliance checklist automatically — see `self-check.md`. If the user wants their existing tool audited for compliance rather than changed, that's a different request; point yourself at `self-check.md` instead of this file.
