---
name: sites
description: Hosting, secrets, custom domains, deployment recovery, JSON records (a table API) and files (uploads, public CDN urls, private downloads) for the sites you are building, from sites on cookiejar.lol: one key per person, one site per project. Use when a site needs to save or read data, take uploads, serve images or documents, or when you would otherwise reach for a database, Postgres, Supabase, KV, Blob storage, or an ORM. sites is also the host: it builds and serves the project from its source (Next.js server-rendered or static). No Vercel, no other host.
metadata:
  priority: 8
  docs:
    - "https://cookiejar.lol/agent/"
  pathPatterns:
    - 'lib/hub.*'
    - 'src/lib/hub.*'
    - 'lib/storage/**'
    - 'lib/db.*'
    - 'src/lib/db.*'
  importPatterns:
    - "@/lib/hub"
    - "@vercel/blob"
    - "@vercel/postgres"
    - "@neondatabase/serverless"
    - "@supabase/supabase-js"
    - "@upstash/redis"
    - "@prisma/client"
    - "drizzle-orm"
    - "mongoose"
retrieval:
  aliases:
    - hosting
    - secrets
    - custom domains
    - rollback
    - database
    - table
    - records
    - file storage
    - uploads
    - blob storage
  intents:
    - add storage
    - save data
    - set up database
    - store uploads
    - serve images
    - persist form submissions
  entities:
    - hub
    - HUB_URL
    - HUB_TOKEN
---

<!-- Generated from docs/agent.md; edit the source and run npm run docs:generate. -->

# Cookiejar: complete customer-agent manual

Cookiejar (“sites”) hosts the projects you build and gives each one JSON records, file storage,
deployments, custom domains and secured env variables. One account key per person, one site per project.
The canonical manual is [cookiejar.lol/agent/](https://cookiejar.lol/agent/).
The installable [sites skill](https://raw.githubusercontent.com/rkz211/cookiejar/main/skills/sites/SKILL.md)
is generated from this same manual. You do not need a second reference to build or operate a site.

## Step 1 — Confirm setup and locate the key

For a new user, ask together: “Cookiejar is a closed-beta backend with records, files and hosting
for sites I build. Do you want me to install its skill and get set up? Where is your key file
(usually `sites-key-<name>.txt` in Downloads)?” Wait for their answers before setup or building.
If they already authorized setup, do not ask again. If they decline, stop. If they have no key,
ask them to request one from their Cookiejar contact; do not invent credentials or another backend.

### Joining the beta without a key

With the human's request to sign up, `POST /waitlist` accepts `{name,email,sitesPerMonth?,features?,agent?}`
without authentication and returns 201 with `{ok,onList,already,message}`. The public API base is
`https://lnh25l18k1.execute-api.us-west-2.amazonaws.com`. Name and valid email are required; name is
clipped to 120 characters, email to 254, features to 2000 and agent to 60. `sitesPerMonth` accepts
`1-2`, `3-10`, `11-50`, `50+` or `exploring`. A repeat email updates answers and preserves invitation
status; blank optional answers preserve earlier values. The limit is 20 requests per source IP
per hour (429). Joining the list does not create a key or authorize a build.

## Step 2 — Configure your environment

The key file contains `HUB_URL` (the API base URL) and `HUB_KEY`; lines beginning with `#` are comments.
Use its values, not the marketing site's URL as an API endpoint. Put them in your own persistent
agent/shell environment with the user's setup authorization. Never echo the key or copy it into
project files, `.env`, source archives, commits, browser code or logs. Keep the original key file
somewhere safe outside projects. If you have no persistent environment, read it at the start of
each session and retain the values only for the session. Never ask the human to paste credentials
into public messages. Shell examples below use Bash, curl, Node.js and zip.

```bash
test -n "$HUB_URL" && test -n "$HUB_KEY" && echo 'Environment ready'
```

`HUB_URL` has no trailing slash in these examples. Your account key is for agent operations.
Cookiejar supplies `HUB_URL` and a per-deployment `HUB_TOKEN` to the deployed server automatically.
`.sites/site` holds only a site id and may be committed. For local server development, set
`HUB_SITE=$(cat .sites/site)` in your process environment alongside your key; the client below
uses `HUB_KEY` plus `X-Site` locally. Reading secured env variables back (`GET /env/secured/reveal`) takes
a site token, so run that part with the site's `HUB_TOKEN` rather than the account key.

## Step 3 — Install once

Claude Code:

```bash
claude plugin marketplace add rkz211/cookiejar
claude plugin install sites@cookiejar
```

Other agents can fetch the [generated SKILL.md](https://raw.githubusercontent.com/rkz211/cookiejar/main/skills/sites/SKILL.md)
and follow it. Save that URL and the key-file location in agent memory if available, never the key.
Tell the human setup is complete; restart the agent if needed to pick up environment changes.
An already installed agent should proceed to the requested project rather than repeat onboarding.

## Step 4 — Choose storage and create or resume a site

| Need | Use |
|---|---|
| Structured data: posts, signups, settings, orders | Records under `/c/{collection}/{key}` |
| Images, PDFs, downloads a page links to | Public files, stable CDN URLs |
| Reports, invoices, private attachments | Private files, fresh signed download per read |
| Next.js server code or a static build | Cookiejar hosting |
| Server settings and API keys, kept encrypted | Secured env variables (`/env/secured`) |
| Ciphertext encrypted with a key you hold outside Cookiejar | Sealed secured env variables (`/env/secured`) |

Records are a simple table API. There are no relational joins, full-text search or multi-record
transactions. If those are essential, explain the requirement and ask before introducing another
backend. For a Cookiejar project, do not silently add a database, ORM, blob provider or another host.

Check `GET /sites` and an existing `.sites/site` before creating anything. Reuse the site's id for
that project. Account routes use the key alone; site-scoped routes use the key plus `X-Site`.
Do not send `X-Site` to account routes, because it selects the site-scoped router.

```bash
# From the project root; do this only if the project has no site yet.
set -e
HUB_URL=${HUB_URL%/}
BODY=$(node -e 'process.stdout.write(JSON.stringify({name:require("node:path").basename(process.cwd())}))')
SITE=$(curl --fail-with-body -sS -X POST "$HUB_URL/sites" \
  -H "Authorization: Bearer $HUB_KEY" -H 'content-type: application/json' -d "$BODY")
mkdir -p .sites
printf '%s' "$SITE" | node -pe 'JSON.parse(require("node:fs").readFileSync(0,"utf8")).siteId' > .sites/site
unset SITE # create also returns a manual site token; do not print or persist it
S=$(cat .sites/site)
H=(-H "Authorization: Bearer $HUB_KEY" -H "X-Site: $S")
curl --fail-with-body -sS "$HUB_URL/me" "${H[@]}"
```

All API paths below are relative to `HUB_URL`. Never expose a key or a site token to visitors.
Your application must authenticate/authorize its own users before server-side record operations,
private downloads or presigned uploads. A site's token can access the whole site; it is not a
per-visitor permission system.

## Account and site operations

| Method | Path | Request and result |
|---|---|---|
| GET | `/` | Public service discovery; use this manual for customer workflows |
| GET | `/key` | Account metadata for the key/session |
| POST | `/sites` | `{ "name": "My project" }` → 201 with `siteId`, `token`, `siteUrl`; ignore the token for the account-key workflow |
| GET | `/sites` | `{sites:[...]}` owned by this account |
| GET | `/sites/{siteId}` | Site details, URLs, domains, `buildId`, `modifiedAt`, disable/removal state |
| PATCH | `/sites/{siteId}` | `{ "name": "New name" }`; changes the name only |
| POST | `/sites/{siteId}/rotate` | Replaces the manual site token and re-enables a disabled site; new token returned once |
| DELETE | `/sites/{siteId}` | 202; disables access immediately and queues edge removal, retains data |
| GET | `/me` | Site details plus `publicFilesUrl` and `siteUrl` |
| PATCH | `/me` | `{ "name": "New name" }`, site-scoped rename |

After disabling, poll `GET /sites/{siteId}` with the account key until `edgeRemoval` is `complete`.
Rotation does not resume cancelled builds. Deployed server versions have separate retained tokens
for rollback and in-flight requests; rotating a manual token does not revoke those versions.
Disable the site to refuse every site token. Report a leaked account key to the Cookiejar operator
for replacement; do not print the credential in the report.

For changes to what is served, compare `buildId` (the active deployment id) and `modifiedAt`.
Record writes do not imply a new build. Fetch `GET /deploy/{buildId}` for deployment details.

The human portal is [cookiejar.lol/portal/](https://cookiejar.lol/portal/).
Its sign-in protocol is `POST /login/challenge {accountId}` → `{nonce}`, then
`POST /login {accountId, nonce, proof}` → `{session}`. The proof is
`hex(HMAC_SHA256(hex(SHA256(key)), nonce))`. Sessions last 12 hours and follow account-key routing,
including `X-Site`; key rotation ends them. Agents normally use their provided key directly.

## Records: shapes, limits and pagination

| Method | Path | Result |
|---|---|---|
| GET | `/c` | `{collections:[{collection,createdAt,updatedAt}]}` |
| GET | `/c/{collection}` | `{collection,items:[record],cursor}` |
| POST | `/c/{collection}` | Body is the JSON data; 201 with a generated time-sortable key |
| DELETE | `/c/{collection}` | Deletes all records and collection metadata; `{deleted: number}` |
| GET | `/c/{collection}/{key}` | One record, or 404 |
| PUT | `/c/{collection}/{key}` | Creates or replaces the complete JSON data; returns the record |
| DELETE | `/c/{collection}/{key}` | `{collection,key,deleted:true}`; 404 if absent |

A record is `{collection,key,data,createdAt,updatedAt}`. The body can be any JSON, including an
array or scalar, up to **358400 bytes (350 KiB)** of UTF-8 serialized JSON. Put larger content in
files and store a path or URL in a record. A PUT replaces data, not a field-level merge; it keeps
`createdAt` and updates `updatedAt`. There is no schema; keep shapes stable and version them when needed.

Collections match `[A-Za-z0-9_-]{1,64}`; keys match `[A-Za-z0-9_.:@~-]{1,256}`.
Use `POST` for generated keys or slugify user input. List query parameters are `prefix`, `limit`
(default 50, 1–100), `cursor` and `order=asc` or `order=desc`. Ordering is by key, not by a data field.
Pass the opaque `cursor` back unchanged, URL-encoded, until it is null. Prefix keys such as
`user:123:...` or `2026-10-09:...` to query groups. Collection deletion is not a transaction.

## Files: upload, read, list and delete

Here `{tier}` means `public`, `private` or `site`.

| Method | Path | Request and result |
|---|---|---|
| POST | `/files/{tier}` | `{path,contentType?,cacheControl?}` → `{access,path,method,uploadUrl,headers,expiresIn,url}` |
| GET | `/files/{tier}` | `{access,prefix,files:[{path,size,lastModified,url}],cursor}` |
| GET | `/files/{tier}/{path}` | `{access,path,url,expiresIn,size,contentType,lastModified}`; 404 if absent |
| DELETE | `/files/{tier}/{path}` | `{access,path,deleted:true}`; deleting an absent file is harmless |

Upload is two requests: ask for an upload URL, then PUT the raw bytes to it using exactly the
returned `headers`. Upload URLs expire after **900 seconds**. Do not send your Cookiejar bearer
token to that storage URL. Default content type is `application/octet-stream`. A public URL is
returned when the URL is minted, but the file exists only after the PUT succeeds. Private upload
and listing URLs are null; request file info for a fresh private download URL (**3600 seconds**).
`GET /files/{tier}/{path}?redirect=1` redirects with HTTP 302. Store the private path, never a signed URL.

Paths are 1–512 characters, relative, slash-separated, with nonempty segments containing only
letters, digits, `. _ ( ) ! * ' -`. No `.` or `..` segment or leading slash. File list queries use
`prefix`, `limit` (default 100, 1–1000) and opaque `cursor`; null means the end. File info sizes are bytes.
The file-signing handler imposes no application byte cap and offers no multipart API: uploads are
single storage PUTs. Do not confuse record limits or static deployment artifact limits with file uploads.
Apply your application's own upload size/type and access policy before handing URLs to browsers.

Public files are separate from the hosted web root and cached for five minutes by default.
Use content-hashed paths plus `cacheControl: "public, max-age=31536000, immutable"` for immutable assets.
The `site` tier serves the static web root. Once a managed static release is active, reads follow
that release and direct site-tier uploads/deletes return **409**; create another deployment.
Public/private file operations remain available.

## Deploy source and host it

| Method | Path | Request and result |
|---|---|---|
| POST | `/deploy` | `{kind:"server"}` (default) or `{kind:"static",outputDir:"dist"}` → 201, `deployId`, `source` upload details |
| POST | `/deploy/{deployId}/start` | Starts the uploaded source build; returns deployment state |
| GET | `/deploy/{deployId}` | State, timestamps, `build`, last 40 `log` lines, `error`, `url`, artifact state |
| GET | `/deploys` | `{deploys:[...],keep}`; entries include `active` and artifact state |
| POST | `/deploy/{deployId}/activate` | Activate a retained successful static or server deployment without rebuilding |
| GET | `/source` | Download metadata and signed `url` for the active deployment's uploaded source ZIP |
| GET | `/deploy/{deployId}/source` | Download metadata and signed `url` for that retained source ZIP |

ZIP the project contents with `package.json` at the top, including its lockfile. Exclude credentials,
`.env*`, dependencies, build output and repository metadata. Use a clean source tree and review archive
contents before uploading. Source is exactly what you upload; Cookiejar cannot remove a secret you include.

```bash
# Bash, from an already configured project; outputs only status and public URLs.
set -e
S=$(cat .sites/site)
H=(-H "Authorization: Bearer $HUB_KEY" -H "X-Site: $S")
D=$(curl --fail-with-body -sS -X POST "$HUB_URL/deploy" "${H[@]}" \
  -H 'content-type: application/json' -d '{"kind":"server"}')
DID=$(printf '%s' "$D" | node -pe 'JSON.parse(require("node:fs").readFileSync(0,"utf8")).deployId')
UP=$(printf '%s' "$D" | node -pe 'JSON.parse(require("node:fs").readFileSync(0,"utf8")).source.uploadUrl')
TMP=$(mktemp -d)
trap 'rm -rf "$TMP"' EXIT
zip -qr "$TMP/source.zip" . -x 'node_modules/*' '.next/*' '.open-next/*' '.git/*' '.sites/*' 'out/*' 'dist/*' '.env*' '*/.env*'
curl --fail-with-body -sS -o /dev/null -X PUT "$UP" -H 'content-type: application/zip' --data-binary @"$TMP/source.zip"
unset D UP
curl --fail-with-body -sS -X POST "$HUB_URL/deploy/$DID/start" "${H[@]}" >/dev/null
# Bounded polling. A timeout means inspect this same deployment, not blindly create another.
for attempt in $(seq 1 120); do
  R=$(curl --fail-with-body -sS "$HUB_URL/deploy/$DID" "${H[@]}")
  STATUS=$(printf '%s' "$R" | node -pe 'JSON.parse(require("node:fs").readFileSync(0,"utf8")).status')
  case "$STATUS" in
    live|failed|superseded) break ;;
  esac
  sleep 15
done
printf '%s' "$R" | node -pe 'const d=JSON.parse(require("node:fs").readFileSync(0,"utf8")); JSON.stringify({status:d.status,url:d.url,error:d.error,log:d.log},null,2)'
test "$STATUS" = live
```

Source upload and source download URLs expire after 900 seconds. Start before the source is uploaded
returns 409. States are `created` → `starting` → `building` → `placing` → `live`, or terminal `failed`
or `superseded`. Stop polling on **all three terminal states**. `superseded` means a later accepted
start or activation controls publication; it is not a successful deployment. `live` is historical
success; use `active` in `/deploys` or the site's `buildId` to find what serves now. Inspect the existing
deployment after a network error before retrying a mutation. Repeated start for a non-created deploy
returns its current state rather than starting a second build.

A static build runs `npm ci` then `npm run build`. `outputDir` defaults to `out` and must be a relative
folder without dot segments. Static artifacts require a root `index.html`, at most 50 MiB compressed,
100 MiB extracted, 20 MiB per file and 5000 ZIP entries. Symlinks, traversal and duplicate paths are
rejected. Releases are immutable and published as a whole; files use `max-age=0, must-revalidate`.
Ship `404.html`; a missing path otherwise may return a bare 403. `/about` and `/about/` resolve to
`/about/index.html`. Edge propagation still takes time after publication.

A server build packages Next.js through OpenNext. Route handlers, server actions and server components
run on the server. Use `images: { unoptimized: true }` in Next config; there is no image optimizer,
shared cache, ISR or revalidation queue. Render dynamically or build static pages; do not depend on ISR.
Build time varies; inspect state rather than assuming completion from elapsed time. Both kinds are
hosted at `https://<siteId>.cookiejar.lol/`. Poll `GET /edge` for certificate readiness (`live: true`).

### Versions, rollback and source recovery

By default the five most recent successful versions retain artifacts; `/deploys.keep` is authoritative. The list returns up to 50 recent deployments, without pagination.
The active version is protected from pruning. Artifact states are `retained`, `pruned` or `none`.
Activate only a successfully published retained version. Static and server rollback both work;
an old static deployment without a retained release must be rebuilt first (409).
Pruned source or activation returns **410 Gone**, not a retryable build error. Source recovery for a
manual file-only site returns 409; use `GET /files/site` instead. Download the ZIP from the returned
signed URL, unpack into a clean directory and install dependencies to resume local work. A server
rollback restores that version's deployment-time environment; current record/file/secret data is
not rolled back. Edge propagation is asynchronous; verify the public URL after activation.

## Environment variables (`/env`)

`GET /env` returns `{env, note}`. `PUT /env` takes the variable map directly and **replaces the entire
map**, not a patch. Read it, merge desired changes locally, then PUT the full map. Omitted and null
values are removed; `{}` clears it. Other values are converted to strings. Names match
`[A-Z_][A-Z0-9_]*`. The size limit is **3500 bytes**, computed as the UTF-8 name and value bytes plus
two bytes per entry. `HUB_URL`, `HUB_TOKEN`, `HUB_ORIGIN_SECRET`, `AWS_REGION` and `NODE_ENV` are reserved
and rejected even if null. Variables apply to the server on its next deploy, not immediately and
not as build-time variables. Values are readable as supplied. For values kept encrypted, use secured env variables (`/env/secured`).

## Secured env variables (`/env/secured`)

Secured env variables are environment values a site's server reads at runtime, kept encrypted
because they could be anything: settings, API keys, tokens. Add them through the API below, or by
hand in the portal: open the site, **env** tab, **Secured** section. Both write the same store.
Unlike plain `/env` variables, they need no redeploy. `/secrets` is the older path to the same
store and still works.

| Method | Path | Meaning |
|---|---|---|
| PUT | `/env/secured` | `{NAME:"value"}` stores values; `{NAME:{sealed:"ciphertext"}}` stores sealed values; null deletes |
| GET | `/env/secured` | `{secrets:[{name,sealed,version,createdAt,updatedAt}],note}`; names and versions |
| GET | `/env/secured/reveal` | `{values:{...},sealed:{...},note}`; optional comma-separated `names` |
| DELETE | `/env/secured/{NAME}` | `{name,deleted:true}`; 404 if absent |

Values are encrypted per site and name. The site's server reads them with its `HUB_TOKEN`, for
example `await hub.secured(['PAYMENT_KEY'])`; manually issued site tokens and retained deployment
tokens read them too. Account keys and portal sessions write and list them, and get **403** on reveal.

Sealed values are ciphertext you encrypt with a key you hold; Cookiejar stores and returns it
unchanged, and your application decrypts it where it runs.

Names match `[A-Z_][A-Z0-9_]*`, at most 80 characters. Values are nonempty UTF-8 strings up to
**4096 bytes**; sealed strings are base64/base64url up to **16384 characters**. A site holds up to
**100 secured variables**. PUT updates only the names given (unlike `/env`), in order; replacing a name increments its
version. A running server sees a new value on its next read, with no redeploy. `secret.put`,
`secret.delete` and `secret.reveal` events in the change feed carry names, not values. A 503 means
the store is unavailable; try again.

## Password protection

`PUT /password {"password":"your chosen password"}` protects all the site's hosts. Passwords must
be 1–200 characters with no newline. `DELETE /password` removes protection. Responses include
`passwordProtected` and `published`; allow about a minute for edge propagation. Visitors see one
password field and their browser remembers access for 30 days. Tools may send HTTP Basic with any
username and that password to the **hosted site**. This does not replace the bearer token for the
Cookiejar API, or your application's own per-user authorization. Verify every host after changing it.

## Edge and custom domains

| Method | Path | Request and result |
|---|---|---|
| POST | `/edge` | Attach the default host, idempotently; optional `{domains:[...]}` for already configured hosts |
| GET | `/edge` | Edge/certificate state, domains and `live` |
| POST | `/edge/invalidate` | 202, flush hosted-site cache after a manual site-tier publish |
| POST | `/edge/domains` | `{domain:"www.example.com"}` → 202 with `domain` state and DNS instructions |
| GET | `/edge/domains` | `{default,domains:[...]}` with `dns`, `tenant`, `gateway`, `live`, `error`, `manual` |
| DELETE | `/edge/domains/{host}` | Remove attachment and managed DNS; `{removed,dnsKept}` |
| POST | `/edge/domains/{host}/move` | `{to:"targetSiteId"}` → 202; move between sites owned by the same account |

Custom domains work for **both static and Next.js server sites**. Add a hostname you own (no scheme
or path), then inspect the returned `domain.manual` array. Cookiejar writes DNS automatically when
the zone is in its connected registrar account; otherwise create the listed records at your DNS
provider. Poll `/edge/domains`: a server site may also need a certificate-validation CNAME, so check
updated `manual` instructions while it progresses. `live: true` means attachment is ready; `error`
and `nextTryAt` explain pending work. Do not equate a 202 response with live traffic.

Apex domains need ALIAS/flattening support; managed registrar zones support ALIAS. One hostname belongs
to one site. A duplicate attachment on another site returns 409; use the explicit move operation on
the source, then poll the target until live. Names under `cookiejar.lol` are assigned by Cookiejar
and cannot be added as custom domains. There is currently no customer domain-search or domain-purchase
endpoint. Register a desired domain through your registrar, then attach it here.

For a manually published static site with no managed release, upload web-root files to the `site`
tier, call `POST /edge`, wait for `live`, then `POST /edge/invalidate`. Managed deployments invalidate
their own hosted-site cache. This operation does not flush the separate public-files CDN.

## Change feed and ongoing operation

`GET /changes?since=<cursor>&limit=100` returns `{changes,cursor,more,note}`. Each event has `id`,
`at`, `type` and details. The default limit is 100, max 200. Without `since` it returns the most recent
window in chronological order; it is not a full-history export. With `since`, results are oldest
first after that cursor. Save the cursor; continue with it while `more` is true, and on later wakes.
An empty response preserves the cursor (or null for a new empty feed).

Events include record put/delete, collection delete, file put/delete, deployment live/failed/activated,
domain live/moved, env changes, rename and secret put/delete/reveal. `file.put` records minting an upload
URL, not proof that bytes arrived; check file info after upload. Logging is best effort. Password,
site rotation/disable and every intermediate deployment state are not covered; use the corresponding
status routes. Use the feed to pick up new leads and changes when asked or when your agent wakes.

## Errors and recovery

API errors are JSON `{error:"message"}` and may include additional fields. Preserve the HTTP status
and useful error text; leave bearer tokens, signed URLs, secured variable values and user data out of reports.
Storage uploads and hosted sites may return non-JSON errors, so check their HTTP status too.

| Status | Meaning and action |
|---|---|
| 400 | Invalid JSON, name, path, cursor or parameters; correct the request |
| 401 | Missing/invalid/expired credentials; check the environment or renew the portal session |
| 403 | Disabled account/site, or an operation this credential does not do (for example reveal with an account key: use the site token) |
| 404 | Missing route, record, file, deployment or owned site; verify identifiers |
| 405 | Unsupported method; use the method in this manual |
| 409 | Conflict: missing source, competing operation, managed static-file write, domain already owned, or ineligible activation; resolve the stated cause |
| 410 | Source/artifacts gone or pruned; choose a retained deployment or rebuild from local source |
| 413 | Record, environment or secret limit exceeded; reduce the payload or use files |
| 429 | Waitlist rate limit or platform throttling; wait the stated interval and honor Retry-After if provided |
| 500 / 503 | Internal error/unavailable service; inspect current state and retry reads with bounded backoff, report persistent errors |

Do not blindly retry unchanged 4xx requests. For a transient conflict, reconcile state before a
bounded retry. A lost mutation response does not prove failure; for generated-record creation,
site creation or deployment creation, inspect existing results before creating duplicates.
On problems, report the method, redacted path, status and error to the human/Cookiejar contact.

## The client (copy this file, no dependency)

`lib/hub.ts` (import only from server code; keep this module out of client components):

```ts
// Server-side only. In production Cookiejar supplies HUB_TOKEN.
// Locally use your own process environment, never a project .env file.
const URL = process.env.HUB_URL?.replace(/\/$/, '')!;
const TOKEN = process.env.HUB_TOKEN || process.env.HUB_KEY;
const SITE = process.env.HUB_TOKEN ? undefined : process.env.HUB_SITE;
if (!URL || !TOKEN || (!process.env.HUB_TOKEN && !SITE)) {
  throw new Error('Set HUB_URL plus HUB_TOKEN, or HUB_KEY and HUB_SITE for local development.');
}

export class HubError extends Error {
  constructor(public status: number, message: string) { super(message); }
}

async function call<T>(method: string, path: string, body?: unknown, query?: Record<string, string | number | undefined>): Promise<T> {
  const qs = query
    ? '?' + Object.entries(query).filter(([, v]) => v !== undefined && v !== '').map(([k, v]) => `${k}=${encodeURIComponent(String(v))}`).join('&')
    : '';
  const res = await fetch(`${URL}${path}${qs}`, {
    method,
    headers: { authorization: `Bearer ${TOKEN}`, ...(SITE ? { 'X-Site': SITE } : {}), ...(body !== undefined ? { 'content-type': 'application/json' } : {}) },
    body: body !== undefined ? JSON.stringify(body) : undefined,
    cache: 'no-store',
  });
  if (!res.ok) {
    const err = await res.json().catch(() => ({ error: res.statusText }));
    throw new HubError(res.status, (err as { error?: string }).error ?? res.statusText);
  }
  return res.status === 204 ? (undefined as T) : (res.json() as Promise<T>);
}

export type Rec<T = unknown> = { collection: string; key: string; data: T; createdAt: string; updatedAt: string };
export type Page<T> = { items: Rec<T>[]; cursor: string | null };

const enc = encodeURIComponent;

export const hub = {
  /** Which site this token is: {siteId, name, calls, publicFilesUrl}. */
  me: () => call<{ siteId: string; name: string; calls: number; publicFilesUrl: string }>('GET', '/me'),
  /** The site's secured env variables (values decrypted; sealed ones as stored). Server-side only; call once at startup, keep in memory. */
  secured: (names?: string[]) => call<{ values: Record<string, string>; sealed: Record<string, string> }>('GET', '/env/secured/reveal', undefined, { names: names?.join(',') }),

  // ---- records --------------------------------------------------------------
  get: <T = unknown>(collection: string, key: string) =>
    call<Rec<T>>('GET', `/c/${enc(collection)}/${enc(key)}`),
  /** Read or null instead of throwing on 404. */
  find: async <T = unknown>(collection: string, key: string) => {
    try { return await hub.get<T>(collection, key); } catch (e) { if (e instanceof HubError && e.status === 404) return null; throw e; }
  },
  put: <T = unknown>(collection: string, key: string, data: T) =>
    call<Rec<T>>('PUT', `/c/${enc(collection)}/${enc(key)}`, data),
  /** Create with a generated, time-sortable key. */
  create: <T = unknown>(collection: string, data: T) =>
    call<Rec<T>>('POST', `/c/${enc(collection)}`, data),
  delete: (collection: string, key: string) =>
    call<{ deleted: true }>('DELETE', `/c/${enc(collection)}/${enc(key)}`),
  list: <T = unknown>(collection: string, opts: { prefix?: string; limit?: number; cursor?: string; order?: 'asc' | 'desc' } = {}) =>
    call<Page<T>>('GET', `/c/${enc(collection)}`, undefined, opts),
  collections: () => call<{ collections: { collection: string; updatedAt: string }[] }>('GET', '/c'),

  // ---- files ------------------------------------------------------------------
  /** Presigned PUT. Send the bytes to uploadUrl with exactly `headers`. Public files become live at `url` after the PUT succeeds. */
  uploadUrl: (access: 'public' | 'private', path: string, contentType: string, cacheControl?: string) =>
    call<{ uploadUrl: string; headers: Record<string, string>; url: string | null; path: string; expiresIn: number }>(
      'POST', `/files/${access}`, { path, contentType, cacheControl }),
  /** Upload from the server in one step. */
  upload: async (access: 'public' | 'private', path: string, contentType: string, bytes: Blob | ArrayBuffer | Uint8Array, cacheControl?: string) => {
    const u = await hub.uploadUrl(access, path, contentType, cacheControl);
    const put = await fetch(u.uploadUrl, { method: 'PUT', headers: u.headers, body: bytes as BodyInit });
    if (!put.ok) throw new HubError(put.status, `Upload failed: ${put.statusText}`);
    return u;
  },
  /** Info + url. Private urls last one hour; fetch a fresh one per request. */
  file: (access: 'public' | 'private', path: string) =>
    call<{ url: string; size: number | null; contentType: string | null; lastModified: string | null; expiresIn: number | null }>(
      'GET', `/files/${access}/${path.split('/').map(enc).join('/')}`),
  deleteFile: (access: 'public' | 'private', path: string) =>
    call<{ deleted: true }>('DELETE', `/files/${access}/${path.split('/').map(enc).join('/')}`),
  files: (access: 'public' | 'private', opts: { prefix?: string; limit?: number; cursor?: string } = {}) =>
    call<{ files: { path: string; size: number; lastModified: string | null; url: string | null }[]; cursor: string | null }>(
      'GET', `/files/${access}`, undefined, opts),
};
```

## Patterns (Next.js on sites)

### A form that saves submissions (server action)

```ts
'use server';
import { hub } from '@/lib/hub';

// Used directly as <form action={submitSignup}>, so it returns void (React 19
// rejects a returning action there). Return a value only with useActionState.
export async function submitSignup(formData: FormData): Promise<void> {
  const email = String(formData.get('email') ?? '').trim();
  if (!email) return;
  await hub.create('signups', { email, at: new Date().toISOString() });
}
```

### A page that reads records (server component)

```tsx
import { hub } from '@/lib/hub';

export default async function Posts() {
  const { items } = await hub.list<{ title: string; body: string }>('posts', { order: 'desc', limit: 20 });
  return <ul>{items.map((p) => <li key={p.key}>{p.data.title}</li>)}</ul>;
}
```

### Pagination

```ts
let cursor: string | undefined;
do {
  const page = await hub.list('orders', { limit: 100, cursor });
  // ...use page.items
  cursor = page.cursor ?? undefined;
} while (cursor);
```

### Browser upload straight to storage (no bytes through your server)

Route handler mints the url; the browser PUTs the file to it.

```ts
// app/api/upload/route.ts
import { hub } from '@/lib/hub';
export async function POST(req: Request) {
  // Authenticate and authorize the visitor before minting an upload URL.
  const { name, type } = await req.json();
  if (typeof name !== 'string' || typeof type !== 'string') {
    return Response.json({ error: 'name and type are required' }, { status: 400 });
  }
  const safe = name.replace(/[^A-Za-z0-9._-]/g, '_').slice(0, 160) || 'file';
  return Response.json(await hub.uploadUrl('public', `uploads/${Date.now()}-${safe}`, type));
}
```

```ts
// client
const signed = await fetch('/api/upload', { method: 'POST', headers: { 'content-type': 'application/json' }, body: JSON.stringify({ name: file.name, type: file.type }) });
if (!signed.ok) throw new Error(`Upload authorization failed: ${signed.status}`);
const meta = await signed.json();
const uploaded = await fetch(meta.uploadUrl, { method: 'PUT', headers: meta.headers, body: file });
if (!uploaded.ok) throw new Error(`Upload failed: ${uploaded.status}`);
// meta.url is the public CDN url; save it in a record if the page needs it later
```

### Serving a private file

```ts
// app/api/report/[id]/route.ts
import { hub, HubError } from '@/lib/hub';
export async function GET(_: Request, { params }: { params: Promise<{ id: string }> }) {
  const { id } = await params; // params is a Promise in Next 15+
  // authorize the caller first, then:
  try {
    const f = await hub.file('private', `reports/${id}.pdf`);
    return Response.redirect(f.url, 302);
  } catch (e) {
    if (e instanceof HubError && e.status === 404) return new Response('Not found', { status: 404 });
    throw e;
  }
}
```

## Working rules

- Keep all Cookiejar credentials in agent/server environments; never ship them to a browser.
- Preserve each project's `.sites/site`; do not create duplicate sites for a resumed project.
- Use the actual limits, states and permissions above; never promise a planned endpoint.
- Prefer records and files over adding a new infrastructure dependency. Explain requirements that exceed this API.
- Verify a deployment, domain, upload or rollback before telling the human it is complete.

## If you are a dot

Also follow https://www.cookiejar.lol/for/dot.html. It covers the dot bridge you reach Cookiejar through.

Current dot bridge: **version 0.5.0** (2026-10-23), source at https://github.com/rkz211/dot-bridge. Your deployed bridge reports its version as `serverInfo.version` in its MCP `initialize` response. If it is lower than 0.5.0, or it is the older `dot-api-bridge`, update it: clone https://github.com/rkz211/dot-bridge.git, run `npm test`, `npm run check` and `npm run build`, and redeploy to the same private Site (its SETUP.md, "Update an existing bridge"). Saved services carry over.

The bridge passes every Cookiejar route in this manual through as written: send them with `bridge_api_read` and `bridge_api_write`, `serviceId: "cookiejar"` and your `siteId`.
