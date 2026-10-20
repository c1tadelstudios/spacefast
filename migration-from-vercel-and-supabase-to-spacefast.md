# Migration from Vercel and Supabase to Spacefast

A step-by-step account of moving a production **Next.js (App Router)** app off **Vercel** (hosting) and **Supabase** (Postgres, Auth, Storage) onto **[Spacefast](https://spacefast.com)** — code, database, sign-in, files and domain. Written from a real migration, including the parts that went wrong, so you don't repeat them.

| Piece | Before | After |
|---|---|---|
| Hosting | Vercel, GitHub integration, auto-deploys `main` | A Spacefast **Space** connected to the same GitHub repo |
| Database | Supabase Postgres 17 with row-level security (RLS) | The Space database — **MariaDB 11.8**, reached in code as `env.DB` |
| Sign-in | Supabase Auth (email one-time codes) | The app's own **Google OAuth** + a signed session cookie |
| Files | Supabase Storage buckets | The Space's storage binding, `env.STORAGE` |
| Domain | `app.example.com` → Vercel | `app.example.com` → Spacefast |

**How long it takes.** Hosting alone is an afternoon. The database and auth are the real work: for an app with ~12 tables and ~5,000 lines touching Supabase, plan on several days. The approach below keeps most UI code unchanged.

**Contents**

1. [Before you start](#1-before-you-start)
2. [Get the app building on Spacefast](#2-get-the-app-building-on-spacefast)
3. [Back up Supabase](#3-back-up-supabase)
4. [Reach the Space database from Next.js](#4-reach-the-space-database-from-nextjs)
5. [Convert the schema from Postgres to MariaDB](#5-convert-the-schema-from-postgres-to-mariadb)
6. [Load your data](#6-load-your-data)
7. [Replace Supabase Auth with Google sign-in](#7-replace-supabase-auth-with-google-sign-in)
8. [Replace row-level security](#8-replace-row-level-security)
9. [Move files to Space storage](#9-move-files-to-space-storage)
10. [Test on a branch preview](#10-test-on-a-branch-preview)
11. [Cut over](#11-cut-over)
12. [Move your custom domain](#12-move-your-custom-domain)
13. [Clean up](#13-clean-up)
14. [Gotchas, all in one place](#14-gotchas-all-in-one-place)

---

## 1. Before you start

- **Next.js 16.2 or newer.** Spacefast builds server-rendered Next.js automatically (with a pinned OpenNext) from 16.2 up. Older versions need a static export.
- **Work on a branch.** Every pushed branch gets a private preview on Spacefast; only the production branch goes live. Keep Vercel and Supabase serving users until you cut over.
- **Tools.** The Spacefast CLI (`npm i spacefast`, command `sf`), Docker (for `pg_dump` and a local MariaDB), and GitHub push access from your terminal (`gh auth login` — GitHub no longer accepts account passwords for `git push`).
- **Write things down as you go.** Several values (secrets, the database password) can't be read back later.

## 2. Get the app building on Spacefast

1. Create a Space and connect your GitHub repo from its **Builds** page. Install the Spacefast GitHub App on the account or organization that owns the repo.
2. Copy your environment variables into the Space. Skip Vercel-only ones (`VERCEL_*`, Vercel KV/Blob, etc.).
   - A variable saved as **secret is write-only** — nobody can read it back, including the dashboard. Keep your own copy.
   - **`NEXT_PUBLIC_*` values are baked in at build time.** After changing one, rebuild. With the CLI: `sf builds retry <latest build id> --yes` reruns the build with current settings.
3. A new Space is private. Making it public applies to the **live** version only; branch previews stay private.

At this point your Supabase-backed app should run on `<space>.space.fast` unchanged — a useful checkpoint before touching the data layer.

> **Agent/CLI access:** to let a coding agent manage the Space, mint a **handoff link** at `my.spacefast.com/start` (single use, 15 minutes). **Switch to the right team first** — a link minted under another team only grants that team.

## 3. Back up Supabase

Take two backups before changing anything.

**A. Through the API** (no database password needed). With the service role key:
- page through every table via PostgREST (`/rest/v1/<table>?select=*` with a `Range` header),
- list users via `/auth/v1/admin/users`,
- download every Storage object via `/storage/v1/object/list/<bucket>` and `/storage/v1/object/<bucket>/<path>`.

**B. A real `pg_dump`** (needs the database password — if you don't have it, reset it in Supabase; your app's API keys are unaffected). No local Postgres tools? Use Docker:

```sh
docker run --rm -e PGPASSWORD="$DB_PASSWORD" -v "$PWD":/out postgres:17 \
  pg_dump "host=<pooler-host> port=5432 dbname=postgres user=postgres.<project-ref> sslmode=require" \
  --schema=public --schema-only --no-owner -f /out/public-schema.sql
```

Repeat with `--data-only` for the data, `--table=auth.users --table=auth.identities` for accounts, and `--table=storage.buckets` for bucket settings. Save storage RLS policies from `pg_policies` too.

- Use the **Session pooler** host from Supabase → Connect. In our case the working host was `aws-1-<region>.pooler.supabase.com`; `aws-0-…` answered "tenant not found". The direct `db.<ref>.supabase.co` host is IPv6-only.
- **Stop after one failed password.** Repeated failures can get your IP blocked by the pooler.
- **Free Supabase projects pause after about a week idle,** and a paused project's hostname stops resolving — it looks deleted. Restore it from the dashboard before backing up.
- **Your repo's migrations may not match production.** Ours had tables created by hand and a migration that was never applied. Convert from the live dump, not from the repo's SQL files.

## 4. Reach the Space database from Next.js

The database is exposed to your server code as a D1-style binding on the OpenNext Cloudflare context. No extra package or config file was needed:

```ts
// lib/db.ts (server-only)
export function platformEnv(): Record<string, unknown> {
  const ctx = (globalThis as any)[Symbol.for('__cloudflare-context__')];
  return ctx?.env ?? {};
}

const { results } = await (platformEnv().DB as any)
  .prepare('select 1 as ok, version() as v')
  .all();
// → [{ ok: 1, v: '11.8.9-MariaDB-log' }]
```

- The API is D1-shaped — `prepare(sql).bind(...params).all()` with `?` placeholders — **but the engine is MariaDB**, not SQLite. Write MySQL-dialect SQL.
- There is **no connection string**, by design. Admin access is the dashboard's **db console** (phpMyAdmin).
- Mark routes that query it `export const dynamic = 'force-dynamic'`.
- Read your other settings from the same `env` object (fall back to `process.env`).

**Tip — deploy a probe first.** Before rewriting anything, ship a tiny route that reports whether `env.DB` exists, runs `select 1`, lists binding names, and lists the methods on other bindings (`Object.getOwnPropertyNames(Object.getPrototypeOf(env.STORAGE))`). It answered three questions for us in one request — and caught a wrong assumption about storage (section 9).

## 5. Convert the schema from Postgres to MariaDB

| Postgres | MariaDB |
|---|---|
| `uuid DEFAULT gen_random_uuid()` | `CHAR(36) DEFAULT (UUID())` — but generate ids in the app so writes can return the row |
| `timestamptz DEFAULT now()` | `DATETIME(3) DEFAULT CURRENT_TIMESTAMP(3)`, stored as UTC |
| `jsonb` | `JSON` |
| `boolean` | `BOOLEAN` — comes back as `0`/`1`; convert to `true`/`false` |
| `numeric` | `DECIMAL(p,s)` — may come back as a string; convert to number |
| `CHECK (x = ANY (ARRAY[...]))` | `CHECK (x IN (...))` |
| foreign keys to `auth.users` | foreign keys to your own `users` table |
| `INSERT … ON CONFLICT (…) DO UPDATE` | `INSERT … ON DUPLICATE KEY UPDATE` |
| `CREATE POLICY …` (RLS) | nothing — enforce in your server code (section 8) |

- **`ON DUPLICATE KEY` fires on any unique key**, not just the one you meant. If a client-supplied `id` reaches an upsert, it can overwrite someone else's row. Strip ids from user upserts.
- Add a `users` table to replace `auth.users` — id, email (unique), name, avatar, provider subject id, an admin flag, an archived flag.
- **Validate against the same engine locally** before importing: `docker run -e MARIADB_ROOT_PASSWORD=x -p 127.0.0.1:33306:3306 mariadb:11.8`, load your schema, then your data.
- Keep `CREATE DATABASE` and `USE` statements out of your files (see section 6).

## 6. Load your data

Spacefast has no import command. Use the dashboard: your Space → **Database → Open db console** (a single-use phpMyAdmin sign-in).

1. **Select the existing database first.** Each Space has exactly one, pre-created. Ours was named with a number — the same as the console user; run `SHOW DATABASES;` in the SQL tab if you're not sure.
   - Skipping this gives `#1046 No database selected`.
   - Trying to create one gives `#1044 Access denied … to database`.
   - In the SQL tab a numeric name needs backticks: ``USE `123456789`;``
2. **Import** the schema file, then the data file.

To build the data file, convert your backup JSON into MariaDB `INSERT`s:
- format timestamps as `YYYY-MM-DD HH:MM:SS.fff` in UTC,
- serialize JSON columns, write booleans as `1`/`0`,
- wrap everything in one transaction.

If you script this in Python 3.9, note `datetime.fromisoformat` rejects Postgres's 1–5 digit fractional seconds — pad them to 6.

This is also the moment to decide what comes over. We moved a single account's data and started fresh for everyone else.

## 7. Replace Supabase Auth with Google sign-in

Spacefast has built-in app sign-in (Google, Gravatar, Spacefast accounts) for its Zero runtime, but **a Next.js app receives no identity from the platform** — our probe saw only Spacefast's own access cookie, no user headers. So the app does its own OAuth. It's about 150 lines.

**Google Cloud Console**
1. **OAuth consent screen:** External; while in **Testing**, add each person who may sign in as a test user.
2. **Credentials → Create OAuth client ID → Web application.** (A "Desktop" client has no redirect URIs — we made that mistake once.)
3. Add one **authorized redirect URI per hostname**, including your branch preview: `https://<space>--br-<branch>.space.fast/api/auth/google/callback`, `https://<space>.space.fast/…`, and your custom domain.
4. Put `GOOGLE_CLIENT_ID` and `GOOGLE_CLIENT_SECRET` (secret) in the Space's variables.

**The flow**

```
/login  ──▶  /api/auth/google              set a one-time state cookie, redirect to Google
        ◀──  /api/auth/google/callback     check state, exchange the code server-side,
                                           read the ID token's verified email,
                                           look the email up in your users table,
                                           set a signed session cookie, redirect to /
```

- Build redirect URIs from the host the visitor used — Spacefast forwards it as `x-forwarded-host`. Allow-list your hostnames.
- An ID token obtained directly from Google's token endpoint over TLS can be trusted without separately verifying its signature; still check `aud` equals your client id and `email_verified` is true.
- **Invite-only** is just "the email must already be in `users`". Inviting someone = inserting their email.
- Sign the session cookie with HMAC-SHA256 via **Web Crypto** (`crypto.subtle`) so it works in middleware too. Store `{ userId, email, isAdmin, expiry }`.
- In `middleware.ts`, verify the cookie, **delete any client-sent identity headers**, then set `x-user-id` / `x-is-admin` for your API routes.
- **Let app assets through the gate.** Exempt `/manifest.webmanifest`, `favicon`/`icon`/`apple-icon` files and your home-screen icons from the sign-in redirect — otherwise browsers get your login page instead of the manifest and "Add to Home Screen" loses its name and icon. Also note Next.js only serves `favicon.ico`, `icon.*` and `apple-icon.*` from `app/`; other icon files belong in `public/`.

## 8. Replace row-level security

Supabase apps often query the database **from the browser** and rely on RLS to keep users in their own rows. With no browser-reachable database, you have two choices: rewrite every component to call purpose-built API routes, or keep the call shape and move the rules server-side. We did the second, and the large components barely changed.

1. Write a small query builder with the same chained API your code already uses — `from(t).select(cols).eq(k, v).in(k, list).order(k).limit(n).single()/.maybeSingle()`, plus `insert/update/upsert/delete`. It only **describes** a query.
2. **In the browser**, the builder POSTs that description to one endpoint, e.g. `/api/db`. **On the server**, the same builder runs directly.
3. The endpoint is your new RLS:
   - only table and column names from an allow-list ever reach SQL; every value is a bound parameter;
   - for tables with a `user_id`, force `user_id = <signed-in user>` on reads, updates and deletes, and set it on inserts;
   - block tables like `users` entirely; let admins through where your app needs it;
   - refuse `update`/`delete` with no filter.
4. Return results shaped like Supabase's (`{ data, error, count }`), including `single()`'s "no rows" error, so existing error handling keeps working.
5. Test it against the local MariaDB with your real data: type conversion, every filter you use, upserts on composite keys, cascading deletes, and the ownership rules (you can't read or overwrite another user's rows).

While you're in there, **audit API routes that take a `userId` from the request.** We found two that let any signed-in user read other people's data.

## 9. Move files to Space storage

The storage binding is **not** the R2-style `put/get/list` you might expect. It mirrors Spacefast's storage SDK:

```ts
const storage = platformEnv().STORAGE as {
  upload(file: Blob, opts?: { public?: boolean }): Promise<{ id: string; contentType: string; size: number; url: string }>;
  get(id: string): Promise<Response | null>;
  delete(id: string): Promise<void>;
};

const { id } = await storage.upload(new Blob([bytes], { type: file.type }));  // private by default
```

- **The platform assigns each object a 32-character hex id; you can't choose a path.** Supabase's `bucket/<user>/<file>` scheme doesn't carry over. Store each returned id in your database (e.g. a `photos` table with `user_id` and `storage_id`) and check ownership there before serving.
- Serve files through an authenticated route that calls `get(id)` and streams the `Response` body back.
- **Limit: 5 MiB per file.** Phone photos are often larger. Shrink in the browser before upload — draw to a canvas at most 2400px on the long edge and export JPEG — so users never see the limit.
- Re-upload old files through your new endpoint rather than copying ids around.

## 10. Test on a branch preview

- Push your branch; Spacefast builds a **private preview** at `https://<space>--br-<branch>.space.fast`. Open it in a browser signed in to Spacefast.
- Run the full Next.js build locally first (`next build`) — it catches invalid route exports and type errors faster than a remote build.
- Check sign-in, your imported data, writes, file upload, and the admin paths.

## 11. Cut over

1. Remove your probe route.
2. **Disconnect Vercel from the repo first** (Vercel → Settings → Git), or pause its deployments. Vercel still watches `main`: merging the Spacefast code there also deploys it to Vercel, where it can't work (there's no `env.DB`).
3. Merge to `main` and push. The live version switches when the build finishes (a couple of minutes); the old version keeps serving until then.
4. Remove the old Supabase variables from the Space.

## 12. Move your custom domain

Moving only the app's subdomain (e.g. `app.example.com`) keeps your marketing site wherever it is.

1. Add the domain to the Space: dashboard **Domains → Add domain**, or `sf domains add app.example.com --space <space>`. It lists the exact records to create — copy the values from **your** output:
   - two **A** records for the hostname (routing),
   - a **TXT** record on the hostname (ownership verification).
2. At your DNS host:
   - **Delete the old CNAME** to Vercel. A name can't hold a CNAME alongside A or TXT records, and Spacefast's ownership check fails while it exists (`provider_claim_rejected … Remove any CNAME at this hostname`).
   - Add each A record as its **own entry**; many DNS panels take one IP per record.
   - Don't "edit the CNAME to an IP" — that publishes an invalid CNAME, not an A record.
   - Publish **exactly one** verification TXT record. A second `atomic-domain-…` value on the same name fails verification with `duplicate_verification_records`.
   - Leave MX/email records and other hosts' records alone.
3. Add `https://app.example.com/api/auth/google/callback` to the Google OAuth client.
4. Remove the domain from the old Vercel project.
5. Wait. Spacefast re-checks every ~30 seconds for 10 minutes, then every 15 minutes, and issues HTTPS once DNS points at it. Manual checks (`sf domains check`) are capped at 6 per hour — don't spend them all at once. A failing check never takes a live site down.
6. Optionally make the domain the Space's primary address.

## 13. Clean up

- Delete the Vercel project once the domain is off it.
- Keep Supabase paused (or exported and deleted) after a quiet week, and keep your backups.
- Revoke any agent/API keys you no longer need.
- Remove the Supabase packages (`@supabase/ssr`, `@supabase/supabase-js`) — the type checker will find any stragglers.

## 14. Gotchas, all in one place

| Symptom | Cause | Fix |
|---|---|---|
| New setting has no effect | `NEXT_PUBLIC_*` is baked at build | Rebuild (`sf builds retry … --yes`) |
| Supabase host "doesn't exist" (NXDOMAIN) | Free project auto-paused | Restore it in the dashboard |
| `pg_dump`: tenant not found | Wrong pooler host | Use the Session pooler host from Supabase → Connect (`aws-1-…` for us) |
| `#1046 No database selected` | Imported without selecting the Space's database | Click it in phpMyAdmin's sidebar first |
| `#1044 Access denied … to database` | Tried `CREATE DATABASE` | Use the one pre-made database |
| Upsert overwrote the wrong row | `ON DUPLICATE KEY` matches any unique key | Never let client ids into upserts |
| No logged-in user in a Next.js route | Spacefast passes no app identity to Next.js | Run your own OAuth + session cookie |
| Google: redirect URI mismatch | Desktop client, or a hostname not registered | Web application client; one URI per hostname |
| Storage `put is not a function` | Binding is `upload/get/delete`, ids assigned by the platform | Store returned ids in your database |
| Upload rejected | 5 MiB per-file limit | Shrink images in the browser |
| Vercel site broke after merging | Vercel still deploys `main` | Disconnect Vercel's Git integration before merging |
| "Add to Home Screen" lost its icon/name | New auth middleware redirects `/manifest.webmanifest` and icons to the login page | Exempt the manifest and icon files from the sign-in check |
| Domain stuck pending | Old CNAME still present | Delete it; add A + TXT records |
| Domain verification `failed`: `duplicate_verification_records` | Two `atomic-domain-…` TXT values on the hostname | Keep only the value Spacefast lists |
| Agent key sees the wrong Space | Handoff minted under another team | Switch teams, then mint at `/start` |

---

*Written from a real migration of a Next.js 16 coaching app in October 2026. Spacefast is in research preview; details may change — check [spacefast.com/docs](https://spacefast.com/docs) for the current behavior.*
