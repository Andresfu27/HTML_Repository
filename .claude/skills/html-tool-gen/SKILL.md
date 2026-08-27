---
name: sparks-gen
description: "Use this skill whenever the user asks you to create, modify, or extend an HTML tool, widget, page, or interactive app that should follow their strict structural rules. Triggers include: \"create an HTML tool\", \"build a tool in HTML\", \"make a widget\", \"make an HTML page\", \"add a feature to this tool\", or any request to build or edit a self-contained HTML file with a config toolbar. Always use this skill for HTML tool generation or modification — it enforces absolute constraints the user requires and must never be skipped."
---

# HTML Tool Generation Skill

The user has strict, non-negotiable structural rules for every HTML tool you create or edit. Violating any rule makes the output non-functional.

This skill is split across files — view only the ones the current task needs.

---

## ABSOLUTE STRUCTURAL RULES (always apply, always active)

1. **Exactly one `<style>` block, exactly one `<script>` block.** `<style>` lives in `<head>`, no other `<style>` tag anywhere. `<script>` lives at the bottom of `<body>`, no other `<script>` tag anywhere.
2. **Zero inline styles.** No `style="..."` attribute on any element, ever. Dynamic styling → assign/toggle a CSS class via JS; define the class in `<style>`.
3. **Zero inline scripts.** No `onclick=`, `onchange=`, `oninput=`, `onsubmit=`, or any `on*=` attribute anywhere. All event listeners registered with `addEventListener` inside the single `<script>` block.
4. **No external JS or CSS beyond the two Google Fonts `<link>` tags.** No `<script src=...>`, no extra `<link rel="stylesheet">`, no `@import`.

These four rules apply identically whether you're building a new tool or editing an existing one — an edit that reintroduces an inline style or a second script block is still a violation.

**Save / sync / patch / force-save / private-save buttons — functional rules only.** `skeleton.md`, `collaborative.md`, and `private-config.md` fix the *behavior* of these buttons: when each one appears or hides, what request it sends and when, dirty-state logic, and the `postMessage` protocol — these are protected. Their *appearance* — label wording, icon, and colors (including the dirty-state highlight and `btnForceSave`'s warning color) — defaults to the Hinicio brand in `brand.md` but is always adaptable: follow the app's own branding if it has one, or let the user style them however they ask.

---

## ROUTING — view the file(s) that match the task

| Task | View |
|---|---|
| Building a new tool from scratch, or transforming something (spec, data, another format) into one | `skeleton.md` (full starter HTML/JS + toolbar behavior rules), then `brand.md` |
| Modifying, fixing, or adding a feature to a tool that already exists | `modify.md` (what to check depending on what the edit touches), plus `brand.md` if the edit is visual |
| Either of the above, when the request explicitly asks for **partial save** or **collaborative editing**, or you spot one of the proactive signals (large embedded assets, multi-user/shared/team framing) | also view `collaborative.md` — opt-in only, propose before building if the user didn't ask directly |
| The tool needs to know, display, or act on **who the current connected user is** (their email, presenter/owner role, attributing an entry to whoever made it) | also view `user-identity.md` — email is recovered from the URL the tool was loaded/embedded from, not from a dedicated identity field |
| The tool needs to store a value that must stay private to whoever is currently using it — an API key, token, or other credential that shouldn't be shared when the tool itself is shared — or you're about to add a field like that | also view `private-config.md` — a small, dedicated save path, independent of everything in `collaborative.md`; wire it in proactively when you spot a credential-like field, and say you did |
| User explicitly asks to check, audit, or verify an existing tool against the rules (not as part of building/editing it) | `self-check.md` — this checklist is **not** run automatically after every generation or edit; only run it on explicit request |
| The tool needs to **connect to / read from the Hinicio database** — Projects (list, metadata, assets, chunks) or People (list, profile, photos, a person's projects) | also view `hinicio-database.md` — read-only GET endpoints for both Projects and People (plus the related search/compare/chat POST calls for context), reverse-engineered from a working reference tool and the API's own reference doc. Covers reading only; saving/writing stays in `skeleton.md`'s `POST_CONFIG_*` mechanism |

Don't view files outside what the current task needs — `collaborative.md`, `user-identity.md`, `self-check.md`, and `hinicio-database.md` in particular are only relevant to a minority of requests.
