# Identifying the Connected User (Email)

View this file when the user asks the tool to know, display, or act on who the current viewer/editor is — e.g. "show the logged-in user's email," "tag each entry with who edited it," "only let the presenter do X," "figure out who's connected."

There's no `POST_CONFIG_*` field that carries the viewer's identity directly. The working pattern (seen in tools generated for this project) is to **recover the email from the URL the tool itself was loaded or embedded from**, since the embedding platform appends it there — not from any dedicated identity API.

## The detection function

Add this near the top of the `<script>` block, after the protected `POST_CONFIG_*` lines and before any tool logic:

```js
/* ── Role/identity detection ──
   The email identifying the current user comes from the URL this tool's own
   script was loaded from, e.g. .../xxxxxx/script/?email=aaa.aaa@aaa.com
   — read as an explicit "email" query parameter first, falling back to a
   loose regex match only if that param isn't present. */
function extractEmailFromScriptUrl() {
  const candidates = [];
  try {
    if (document.currentScript && document.currentScript.src) candidates.push(document.currentScript.src);
  } catch (e) { /* currentScript may be unavailable for inline scripts in some browsers */ }
  candidates.push(window.location.href);
  try { if (document.referrer) candidates.push(document.referrer); } catch (e) { /* ignore */ }

  for (const url of candidates) {
    if (!url) continue;
    try {
      const parsed = new URL(url, window.location.href);
      const paramEmail = parsed.searchParams.get('email');
      if (paramEmail) return paramEmail.trim().toLowerCase();
    } catch (e) { /* not a parseable absolute URL, fall through to regex below */ }
  }
  for (const url of candidates) {
    if (!url) continue;
    const m = String(url).match(/[a-zA-Z0-9_.+-]+@[a-zA-Z0-9-]+\.[a-zA-Z0-9-.]+/);
    if (m) return m[0].toLowerCase();
  }
  return null;
}
function extractEmail(str) {
  if (!str) return null;
  const m = String(str).match(/[a-zA-Z0-9_.+-]+@[a-zA-Z0-9-]+\.[a-zA-Z0-9-.]+/);
  return m ? m[0].toLowerCase() : null;
}
const detectedEmail = extractEmailFromScriptUrl() || extractEmail(POST_CONFIG_URL) || extractEmail(POST_CONFIG_UID) || null;
let myEmail = detectedEmail;
```

## Why it's built this way

- **Multiple candidate URLs, checked in order.** `document.currentScript.src` is most reliable when the tool is loaded as an external script; `window.location.href` covers the case where the tool itself is the top-level page or the query string was forwarded there; `document.referrer` catches iframe embeds where the parent passed the email along in its own URL. Checking all three (rather than picking one) makes the tool work across the different ways it can be embedded.
- **Explicit `?email=` param checked before the regex fallback.** An exact query param is unambiguous; the regex is a looser safety net for cases where the email ended up somewhere in the URL without a clean param name (e.g. baked into a path segment).
- **Regex runs only if the param lookup fails**, and only after every candidate URL has been tried for the param — don't regex-match the first candidate and stop; check the param on every candidate first, then fall back to regex on every candidate.
- **`POST_CONFIG_URL` / `POST_CONFIG_UID` as a last resort**, since some backends embed the user's email in the save endpoint or UID rather than the page URL. Reuse the existing protected `POST_CONFIG_*` constants — don't add new ones for this.
- **Normalize to lowercase and trim.** Email comparisons elsewhere in the tool (e.g. checking if the current user matches a stored "owner" or "presenter" field) should compare lowercase-to-lowercase, since the same address can arrive with different casing depending on where it was typed.
- **Returns `null` when nothing is found** — never fabricate or default to a placeholder email. Downstream code must handle the no-identity case explicitly (e.g. treat the user as anonymous, or hide identity-gated features) rather than assuming detection always succeeds.

## Using the detected email

- Treat `detectedEmail` as the source of truth and `myEmail` as the mutable working copy — some tools let the user manually claim/override an identity later in the session (e.g. a "claim presenter role" button); re-running detection or accepting a manual override should update `myEmail`, not `detectedEmail`, so the original detection result stays available for comparison or a "reset" action.
- If the tool needs a stable per-person key for storing responses, presence, or attribution (so reloads or repeated visits land on the same record instead of creating a new one each time), prefer `myEmail` over a randomly generated session ID when it's available, and fall back to a generated ID only when no email was detected.
- If the tool distinguishes roles (e.g. one specific person is "the presenter/owner" and everyone else is "audience/viewer"), compare `myEmail.toLowerCase()` against the stored owner email field — don't compare case-sensitively, and don't assume `myEmail` is non-null before comparing.
- Log the detected value once during boot (`console.log` is fine) so embedding issues are debuggable, but never surface the raw detection mechanics (which candidate matched, which regex fired) in user-facing UI — only the resulting email or role.
