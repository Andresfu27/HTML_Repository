# Connecting to the Hinicio Database (Projects & People — Read Endpoints)

View this file when the user asks a tool to connect to, query, or read from **the Hinicio database** (also referred to as OMEGA / ANDREA's backend) — e.g. "pull the list of projects", "show this project's metadata/assets/chunks", "add a project picker", "look up a person", "who's on this project's team", "let this tool read from Hinicio's API". This file covers **read (GET) access only**, across both the Projects and People parts of the API. It does not cover writing/saving data back (that's the unrelated `POST_CONFIG_*` / save-toolbar mechanism in `skeleton.md`) — write endpoints exist on this API too and are noted briefly below for context, but building a populate/validate form against them is a separate task from what this file covers.

The patterns below are reverse-engineered from a working reference tool (an internal Project Dashboard) plus the API's own reference doc — follow them exactly rather than inventing a different request shape.

---

## Base URL

The API lives at `/api/` on the same origin the tool's save endpoint was injected with. Never hardcode a domain — derive it the same way the reference tool does, so the tool keeps working wherever it's deployed/embedded:

```js
function resolveApiOrigin() {
  if (POST_CONFIG_URL && POST_CONFIG_URL !== '@@URLPOSTCONFIG@@') {
    try { return new URL(POST_CONFIG_URL).origin; } catch (e) { /* fall through */ }
  }
  const raw = window.location.origin;
  if (raw && raw !== 'null') return raw;
  return window.location.protocol + '//' + window.location.host;
}
let API_BASE = resolveApiOrigin() + '/api/';
```

This reuses the protected `POST_CONFIG_URL` constant from `skeleton.md` — don't declare a second URL placeholder for the database. If the tool also lets the user override the base (e.g. for testing against a different environment), keep that override in `buildConfigJSON()`/`applyConfig()` as `apiBase`, same as the reference tool.

## Auth header

Every request — Projects or People, GET or otherwise — carries the same header, built from the existing protected `POST_CONFIG_AUTH` constant. Don't add a separate auth constant for database calls, and don't try to build the four-part value yourself — it's injected as one opaque string:

```
Authorization: User :{email}, Token :{api_token}, Sub :{tenant_subdomain}, Admin : {true|false}
```

```js
function authHeaders() {
  const h = { 'Content-Type': 'application/json' };
  if (POST_CONFIG_AUTH && POST_CONFIG_AUTH !== '@@AUTHORIZATION@@') h['Authorization'] = POST_CONFIG_AUTH;
  return h;
}
```

**`Sub` matters and fails silently.** It routes the request to the correct tenant/client database. A missing or wrong `Sub` doesn't 401 — it falls back to the server's default tenant and returns a normal-looking response for the *wrong* client's data. If a tool built against this API ever returns plausible-but-unexpected projects or people, check that the injected `POST_CONFIG_AUTH` actually carries a `Sub` before assuming it's a data bug.

### Error mapping

Surface a helpful message instead of raw server text — the backend returns auth failures as HTTP 401 with a `code:` in the body, and a down chat/embedding backend as 502:

```js
function mapApiError(err) {
  const msg = err.message || 'Unknown error';
  if (err.status === 401) {
    if (/code:\s*3\b/i.test(msg)) return 'Authorization token mismatch — check the Token : value, no stray whitespace.';
    if (/code:\s*[12]\b/i.test(msg)) return 'User not recognized — check the exact email in User :.';
    if (/code:\s*4\b/i.test(msg)) return 'Session token expired — regenerate it.';
    return 'Unauthorized — check the Authorization header.';
  }
  if (err.status === 502) return 'Chat/embedding backend (Ollama) is unavailable right now — try again in a moment.';
  return msg;
}
```

## The `apiCall` helper

Wrap every JSON request through one helper so error handling stays consistent:

```js
async function apiCall(method, path, body) {
  const opts = { method, headers: authHeaders() };
  if (body !== undefined) opts.body = JSON.stringify(body);
  const res = await fetch(API_BASE + path, opts);
  let data;
  try { data = await res.json(); } catch (e) { data = {}; }
  if (!res.ok) {
    const err = new Error(data.error || 'HTTP ' + res.status);
    err.status = res.status;
    throw err;
  }
  return data;
}
```

`path` is relative and has no leading slash (e.g. `'projects/'`, not `'/projects/'`) — it's concatenated directly onto `API_BASE`.

---

## Projects

### GET `projects/` — list all projects

```js
const data = await apiCall('GET', 'projects/');
const projects = data.projects || [];
```

Each item: `id`, `project_code` (unique, human-readable, e.g. `PBEEIB26006-06` — treat as immutable/read-only), `status` (`draft` / `active` / `closed`), `metadata`, `created_at`/`updated_at`. This list is intentionally light for populating a picker — load the detail or metadata endpoint below for anything more.

### GET `projects/{project_code}/` — full project detail

```js
const p = await apiCall('GET', 'projects/' + code + '/');
```

Returns **metadata, documents (with chunks), and assets together** in one call — there's no separate "get chunks" endpoint; they're nested here:

| Field | Description |
|---|---|
| `id`, `project_code`, `status` | Same as the list entry |
| `metadata` | Nested jsonb object — groups include `team_leading`, `legal_entity`, `confidentiality`, client/title text fields, `location {country, region, city_or_site}`, `schedule {start_date, end_date}`, `display_period`, `classification {solution_vertical, type_of_service, product_or_sector, pda_workstream, development_stage}`, `commercial {currency, price, ...}`, `reference_content {about_project, context, critical_challenges[], scope_of_work[], impacts_added_value[], project_scale, hinicio_role, deliverables[]}` |
| `metadata_template` | Free-form, not currently validated against anything — treat as advisory only |
| `people` | **Sibling column, not nested inside `metadata`.** A jsonb dict keyed by email — see "Project ↔ person link" below |
| `documents` | Array of `{ id, title, source_filename, chunks: [{ id, chunk_index, content }] }` |
| `assets` | Array of asset objects — `id`, `mime_type`, `original_filename`/`filename`, optional `content_base64` (inline copy — build a data URI directly when present), optional `description` |

### GET `projects/{project_code}/metadata/` — lighter detail

Same basic record as above (`id`, `project_code`, `status`, `metadata`, `people`, `created_at`, `updated_at`) but **without** the nested `documents`/`chunks`/`assets`. Prefer this over the full detail endpoint whenever a tool only needs metadata or the team list and not the document/asset payload — it's a materially smaller response.

```js
const p = await apiCall('GET', 'projects/' + code + '/metadata/');
```

### GET `projects/{project_code}/assets/{asset_id}/serve/` — fetch a binary asset

Use this when the asset object has no `content_base64`. The API reference describes it as returning raw file bytes suitable for a plain `<img src>` — but in practice, if the asset is behind the same tenant-scoped auth as everything else, a bare `<img src=...>` can't carry the `Authorization`/`Sub` header and will 401 silently as a broken image with no visible error. Match the reference dashboard's approach and always fetch it as an authenticated blob instead:

```js
async function loadAssetAsBlob(projectCode, assetId) {
  const url = API_BASE + 'projects/' + projectCode + '/assets/' + assetId + '/serve/';
  const res = await fetch(url, { headers: authHeaders() });
  if (!res.ok) throw new Error('HTTP ' + res.status);
  const blob = await res.blob();
  return URL.createObjectURL(blob); // assign this to img.src
}

async function resolveAssetUrl(p, a) {
  if (a.content_base64) return 'data:' + (a.mime_type || 'application/octet-stream') + ';base64,' + a.content_base64;
  return loadAssetAsBlob(p.project_code, a.id);
}
```

If a tool is later confirmed to run against a deployment where this endpoint is genuinely public/unauthenticated, a direct `<img src>` is fine — but default to the blob-fetch pattern unless told otherwise.

### Project ↔ person link

`people` is **not a foreign key** — it's a jsonb dict on `core.projects.people`, keyed by the person's email:

```json
{
  "saul.villamizar@hinicio.com": { "role": "Consultant", "hours_charged": 120 }
}
```

Read it straight off `projects/{code}/` or `projects/{code}/metadata/` — no extra call needed. There's no server-side check that an email here actually exists in `people.people`, or that a listed person hasn't since been deleted — a team-member picker built against this data should resolve/validate emails against `GET /people/` (below) rather than trust the dict blindly.

### Other Projects endpoints (context only — not GET)

| Method | Path | Purpose |
|---|---|---|
| POST | `/projects/` | Create — `{project_code, status?, metadata?, metadata_template?}` |
| DELETE | `/projects/{project_code}/` | Deletes project, cascades documents/assets/chunks and their files |
| PUT/PATCH | `/projects/{project_code}/metadata/` | `{metadata?, people?}` — each merges into the existing jsonb (not a replace) independently |
| POST | `/projects/{project_code}/status/` | `{status}` — must be exactly `draft`/`active`/`closed` |
| POST | `/projects/compare/` | `{project_codes: [...]}` → status + metadata for several projects at once |
| GET / POST | `/projects/{project_code}/documents/` | List, or register a document record |
| POST | `/projects/{project_code}/documents/extract/` | Upload `.docx`/`.pptx`/`.pdf` — auto-extracts chunks + image assets |
| DELETE | `/projects/{project_code}/documents/{document_id}/` | Removes a document, its chunks, its file |
| POST | `/projects/{project_code}/assets/` | Upload an image (multipart, optional description); deduped by content hash |
| DELETE | `/projects/{project_code}/assets/{asset_id}/` | Removes one asset |

---

## People

### GET `people/` — list everyone

```js
const data = await apiCall('GET', 'people/');
const people = data.people || [];
```

Light rows only: `email` (the identifier — there's no separate surrogate id), `full_name`, `status` (`active`/`inactive`/`external`), `created_at`/`last_updated`. This is the right call for a "pick a person" control or for validating an email a project's `people` dict references — it's cheap precisely because it skips every jsonb field below.

### GET `people/{email}/` — full detail

```js
const person = await apiCall('GET', 'people/' + encodeURIComponent(email) + '/');
```

Adds the jsonb fields, plus their photo assets:

| Field | Shape |
|---|---|
| `contact_info` | `{ phone, location, secondary_emails: [] }` |
| `nationality` | `{ nationality }` or `{ nationalities: [...] }` for dual |
| `short_bio` | Per-language: `{ en: "...", es: "..." }` |
| `professional_experience` | Array — `role, employer, start/end date, location, description` (mixes work-history and project engagements) |
| `sectors_of_expertise` | Flat tag array, e.g. `["Hydrogen", "LCA & Carbon Footprint"]` |
| `education` | Array — `degree, institution, start/end date, location` |
| `language` | Array — one entry per language, each with CEFR ratings for `listening / reading / spoken_production / spoken_interaction / writing` |
| `skills` | Flat string array |
| `certifications` | Array — `name, issuer, date, expiry` |
| (assets) | Their photo objects — same shape as project assets: `id, kind, original_filename, mime_type, file_size, description` |

Only `status` is validated server-side (must be exactly `active`/`inactive`/`external`); every other field accepts whatever shape is sent, so a form built against this is the only thing keeping data like `language` entries consistent.

### GET `people/{email}/assets/{asset_id}/serve/` — fetch a photo

Same authenticated-blob caveat as the project asset serve endpoint above — reuse `loadAssetAsBlob`, just against the `people/` path instead of `projects/`:

```js
async function loadPersonAssetAsBlob(email, assetId) {
  const url = API_BASE + 'people/' + encodeURIComponent(email) + '/assets/' + assetId + '/serve/';
  const res = await fetch(url, { headers: authHeaders() });
  if (!res.ok) throw new Error('HTTP ' + res.status);
  const blob = await res.blob();
  return URL.createObjectURL(blob);
}
```

Person assets have no embedding/searchable column (unlike project assets) — `description` here is just a plain caption.

### GET `people/{email}/projects/` — reverse lookup

Every project that lists this email in its `people` dict, with that entry's `role`/`hours_charged` plus the project's client/title/scope — the other direction of the Project ↔ person link. Use this for a person's profile page rather than scanning every project's `people` dict client-side.

```js
const projects = await apiCall('GET', 'people/' + encodeURIComponent(email) + '/projects/');
```

### Other People endpoints (context only — not GET)

| Method | Path | Purpose |
|---|---|---|
| POST | `/people/` | Create — `{email, full_name, status?, ...}` — only `email`/`full_name` required |
| DELETE | `/people/{email}/` | Cascades their photo assets, removes files from disk |
| PUT/PATCH | `/people/{email}/update/` | Any subset of the create fields — **replaces** the field sent rather than merging (unlike project metadata's jsonb merge) — resending `skills` overwrites the old array, it doesn't append |
| POST | `/people/{email}/assets/` | Upload a photo (multipart, optional `kind`/`description`); deduped by content hash |
| DELETE | `/people/{email}/assets/{asset_id}/` | Remove one photo |

---

## Searching chunks/assets and cross-project queries (POST, for context)

Not plain GETs, and explicitly **not** part of the populate/validate People/Projects work — this is the RAG search layer used by the chatbot and MCP tools — but still the other way the reference dashboard reads project data:

- **POST `chunks/search/`** — `{ query, mode: 'lexical' | 'hybrid', limit, project_codes?: [...] }` (omit `project_codes` to search all projects) → `{ results: [...], mode }`; each result carries `project_code`, `document_title`, `chunk_index`, `content`, plus `rank` (lexical) or `rrf_score`/`lexical_rank`/`vector_rank` (hybrid).
- **POST `assets/search/`** — same shape, for images; results carry `project_code`, `document_title`, `description`/`filename`, `asset_id` (fetch via the project asset serve endpoint above).
- **POST `chatbot/internal/rag/`** — `{ model, messages: [{role, content}, ...], top_k, image_top_k, project_codes?: [...] }` → `{ content, sources: [{chunk_id, project_code, excerpt}], images: [{url, description, document_title}] }`.

## Loading states and errors

Every read is asynchronous against a real backend — always show a loading state while it's in flight and a clear error message (via `mapApiError`) on failure, exactly like the reference tool's `setMain('<div class="empty-state"><span class="spinner"></span>...')` / `catch` pattern. Never leave the UI silently blank while a request is pending or after it fails.
