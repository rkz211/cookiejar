---
name: sites
description: Hosting, JSON records (a table API) and files (uploads, public CDN urls, private downloads) for the sites you are building, from sites on cookiejar.lol: one key per person, one site per project. Use when a site needs to save or read data, take uploads, serve images or documents, or when you would otherwise reach for a database, Postgres, Supabase, KV, Blob storage, or an ORM. sites is also the host: it builds and serves the project from its source (Next.js server-rendered or static). No Vercel, no other host.
metadata:
  priority: 8
  docs:
    - "https://cookiejar.lol/contract/README.md"
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

# sites

You are wiring a site into **sites** (the hub): one backend that already exists, with a JSON records API, a
files API and hosting. You do **not** provision a database, a bucket, a host or an ORM. Your key plus a site id
does everything from your side; the deployed server gets its own token.

## Decide

| The site needs | Use |
|---|---|
| Save/read structured data (posts, signups, settings, orders, comments) | **Records**: `/c/{collection}/{key}` |
| Images, PDFs, video, downloads that pages link to | **Public files**: `/files/public` → stable CDN url |
| Uploads only the owner should see (reports, invoices, raw form attachments) | **Private files**: `/files/private` → 1-hour presigned url per read |
| Relational joins, full-text search, transactions across many rows | Not sites. Say so and stop; ask before adding a database. |

Records are any JSON up to 350 KB. Collections and keys are plain identifiers. There is no schema; keep
a stable shape per collection and version it in a field if it changes.

## Provision and host (you do it yourself; sites IS the host)

There is no Vercel and no other host. A site's files are served by the sites edge; a site's data lives in
the sites API. Your own environment carries `HUB_URL` and `HUB_KEY`. Check first:

```bash
test -n "$HUB_URL" && test -n "$HUB_KEY" && echo ok
```

If either is missing, stop and ask for the key file. Do not invent a fallback store or a fallback host.

**1. Create the site.** One site per project. Name it after the project. Your key is the umbrella over every
site it creates: with the header `X-Site: <siteId>` it can do anything on that site. So a project holds only its
**site id**, which is not a secret; no token ever lands in the project.

```bash
SITE=$(curl -sf -X POST "$HUB_URL/sites"   -H "Authorization: Bearer $HUB_KEY" -H "content-type: application/json"   -d "{\"name\":\"$(basename "$PWD")\"}")
mkdir -p .sites && printf '%s' "$SITE" | node -pe 'JSON.parse(require("fs").readFileSync(0,"utf8")).siteId' > .sites/site
echo "site: $(cat .sites/site) -> $(printf '%s' "$SITE" | node -pe 'JSON.parse(require("fs").readFileSync(0,"utf8")).siteUrl')"
```

(The response also carries a per-site `token`; you do not need it. Ignore it, never store it.)

From here every call is `-H "Authorization: Bearer $HUB_KEY" -H "X-Site: $(cat .sites/site)"`. A project that
already has `.sites/site` does not get a second site.

**2. Deploy it: sites builds and hosts the project from its source.** Both kinds go through the same
three calls. `server` (default) is any Next.js app: server actions, route handlers, server components all
work. `static` is a project whose `npm run build` writes a folder of files (pass `outputDir`, default `out`).

```bash
S=$(cat .sites/site); H=(-H "Authorization: Bearer $HUB_KEY" -H "X-Site: $S")
# a) a deploy + a place to put the source. Zip the project root (package.json at the top; skip node_modules, .next, .git, .sites)
D=$(curl -sf -X POST "$HUB_URL/deploy" "${H[@]}" -H "content-type: application/json" -d '{"kind":"server"}')
DID=$(printf '%s' "$D" | node -pe 'JSON.parse(require("fs").readFileSync(0,"utf8")).deployId')
UP=$(printf '%s' "$D"  | node -pe 'JSON.parse(require("fs").readFileSync(0,"utf8")).source.uploadUrl')
zip -qr /tmp/source.zip . -x 'node_modules/*' '.next/*' '.open-next/*' '.git/*' '.sites/*' 'out/*'
curl -sf -o /dev/null -X PUT "$UP" -H "content-type: application/zip" --data-binary @/tmp/source.zip
# b) build + place
curl -sf -X POST "$HUB_URL/deploy/$DID/start" "${H[@]}" >/dev/null
# c) wait: created -> building -> placing -> live (or failed, with the build log tail in the response)
until R=$(curl -sf "$HUB_URL/deploy/$DID" "${H[@]}"); printf '%s' "$R" | grep -qE '"status":"(live|failed)"'; do sleep 15; done
printf '%s' "$R" | node -pe 'const j=JSON.parse(require("fs").readFileSync(0,"utf8")); j.status+" "+(j.url||"")+(j.error?" "+j.error:"")+(j.status==="failed"?"\n"+j.log.join("\n"):"")'
```

A server deploy takes about three minutes the first time (the build), a static one about ninety seconds. The
site is then at `https://<siteId>.cookiejar.lol/`; the first time, allow a few more minutes for its
certificate (`GET /edge` shows `live: true` when the domain is active). Re-deploy by repeating a) to c).
**Versions, rollback, clone.** The 5 most recent successful deploys keep their artifacts (`GET /deploys`
shows `artifacts: retained` and which one is `active`). Roll back or forward with
`POST /deploy/{deployId}/activate` (static and server alike; seconds, no rebuild). Clone a site back to a
machine with `GET /source` (the live deploy) or `GET /deploy/{deployId}/source`: the reply's `url` is a
short-lived download of the project zip exactly as it was uploaded; unzip it, `npm install`, and you have
the source. Older deploys are `pruned` and can be neither activated nor cloned. Rename a site with
`PATCH /sites/{siteId} {"name": "..."}` (key) or `PATCH /me` (key + `X-Site`); nothing else changes.

**Environment variables for the site's server**: `PUT /env` (key + `X-Site`) with `{"KEY":"value"}` (≤ 3.5 KB total), applied at
the next deploy. `HUB_URL` and `HUB_TOKEN` are set for you: the deployed server authenticates to sites with
its own server token, so the client below works unchanged in production.

**Next.js on sites**: set `images: { unoptimized: true }` in `next.config` (no image optimizer), and do not
rely on ISR/revalidation (there is no cache or queue; pages render on request or at build). Everything
else is standard.

**Static sites**: include a `404.html` at the root of the build; without it a missing path answers a bare 403.
Directory urls resolve to `index.html` (`/about` and `/about/` both serve `/about/index.html`).

**Has the site changed?** `GET /sites/{siteId}` (key) or `GET /me` (key + `X-Site`) returns `buildId`, the deploy
serving right now, and `modifiedAt`, the last time what the site serves changed (a publish or activation, or a
site-tier file write/delete). Keep the pair you saw last; a different `buildId` or a later `modifiedAt` means
something changed. `GET /deploy/{buildId}` has that build's log and status.

**A custom domain** (static or Next.js alike): `POST /edge/domains` with `{"domain":"www.example.com"}` (key +
`X-Site`). If the zone is in the hub's Porkbun account the DNS record is written for you and nothing else is
needed; otherwise the 202 body's `manual[]` lists the exact records (an ALIAS/CNAME to the edge, and for a
server site a validation CNAME) for the human to create. Poll `GET /edge/domains` until the entry shows
`live: true`; a static site is live minutes after DNS resolves, a server site a few minutes more (its
certificate + gateway domain are issued automatically). `error` says what it is waiting on. Remove with
`DELETE /edge/domains/{host}`. Apex names (`example.com`) work on Porkbun zones (ALIAS record).

**Credentials, and where they live:**

| Credential | Lives in | Never goes to |
|---|---|---|
| `HUB_KEY` (your key) | your own shell environment | any project, any file, any commit, any browser |
| `.sites/site` (the site id) | the project; not a secret, may be committed | — |
| the site's server token | the deployed server's environment as `HUB_TOKEN`, set by sites at deploy | anywhere else; you never see it |

## The client (copy this file, no dependency)

`lib/hub.ts`:

```ts
// The hub client. Server-side only: it reads HUB_TOKEN.
const URL = process.env.HUB_URL!;
const TOKEN = process.env.HUB_TOKEN!;

export class HubError extends Error {
  constructor(public status: number, message: string) { super(message); }
}

async function call<T>(method: string, path: string, body?: unknown, query?: Record<string, string | number | undefined>): Promise<T> {
  const qs = query
    ? '?' + Object.entries(query).filter(([, v]) => v !== undefined && v !== '').map(([k, v]) => `${k}=${encodeURIComponent(String(v))}`).join('&')
    : '';
  const res = await fetch(`${URL}${path}${qs}`, {
    method,
    headers: { authorization: `Bearer ${TOKEN}`, ...(body !== undefined ? { 'content-type': 'application/json' } : {}) },
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
  /** Presigned PUT. Send the bytes to uploadUrl with exactly `headers`. Public files are live at `url`. */
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
import { revalidatePath } from 'next/cache';
import { hub } from '@/lib/hub';

// Used directly as <form action={submitSignup}>, so it returns void (React 19
// rejects a returning action there). Return a value only with useActionState.
export async function submitSignup(formData: FormData): Promise<void> {
  const email = String(formData.get('email') ?? '').trim();
  if (!email) return;
  await hub.create('signups', { email, at: new Date().toISOString() });
  revalidatePath('/'); // the page that lists signups
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
  const { name, type } = await req.json();
  const safe = name.replace(/[^A-Za-z0-9._-]/g, '_');
  return Response.json(await hub.uploadUrl('public', `uploads/${Date.now()}-${safe}`, type));
}
```

```ts
// client
const meta = await fetch('/api/upload', { method: 'POST', body: JSON.stringify({ name: file.name, type: file.type }) }).then((r) => r.json());
await fetch(meta.uploadUrl, { method: 'PUT', headers: meta.headers, body: file });
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

## Rules

- **Nothing secret in page code.** You act with your key + `X-Site`; the deployed server gets its own `HUB_TOKEN` from sites. Only server code (route handlers, server actions, server components) calls sites.
- **Keys are identifiers**, not sentences: `[A-Za-z0-9_.:@~-]`, ≤ 256 chars. Slugify user input before using it as a key; or use `create()` and let the hub generate one.
- **Prefix keys to query by group**: `2026-09-21:...` keys list by day with `prefix: '2026-09-21'`; `user:<id>:...` lists one user's rows.
- **One record ≤ 350 KB.** Larger content is a file; store its url in the record.
- **Public file urls are cached 5 minutes** at the edge. Immutable assets: upload with `cacheControl: 'public, max-age=31536000, immutable'` and a content-hashed path.
- **Private urls expire in one hour.** Never store one; store the path and fetch a fresh url on each request.
- **Errors** are `{error}` with a real status: 400 bad input, 401 bad token, 403 site disabled, 404 missing, 413 too big. Surface the message; do not retry 4xx.
- **Do not install** `@vercel/blob`, `@vercel/postgres`, `@neondatabase/serverless`, `@supabase/*`, `@upstash/redis`, `@prisma/client` or `drizzle-orm`, and do not deploy to Vercel, Netlify or any other host, for a sites project. If a requirement truly needs them (joins, search, transactions), say so and ask first.
- **A leaked key** (`HUB_KEY`): stop and tell the human at once; only the admin can replace it. Site tokens are minted per deploy and never leave the server; redeploying rotates them.

## Reference

`GET $HUB_URL/` returns the full route list. Full contract: https://cookiejar.lol/contract/README.md
