# SahuliatAI — Setup & Configuration

This is the application package (Next.js 16 + Supabase). This README covers **how to install, configure, run, and deploy it**.

> **What the product is, the architecture, the agents, and the data model** live in the **[top-level README](../README.md)**.
> For a longer, click-by-click walkthrough (where to find each key in each dashboard), see **[`SETUP.md`](./SETUP.md)**.

---

## Prerequisites

| Tool | Version | Install |
|------|---------|---------|
| Node.js | ≥ 20 (24 LTS recommended) | <https://nodejs.org/> |
| pnpm | ≥ 9 | `npm install -g pnpm` |
| Supabase CLI | latest | `brew install supabase/tap/supabase` |
| `psql` | latest | `brew install libpq && brew link --force libpq` *(used by `db:seed`)* |
| Vercel CLI | latest *(only to deploy from CLI)* | `pnpm add -g vercel` |

---

## Quickstart

### Option A — guided script (recommended)

The bundled script checks prerequisites, creates `.env.local`, installs deps, generates VAPID keys, and optionally links Supabase + pushes migrations + seeds:

```bash
cd product
bash scripts/setup.sh     # or: pnpm setup
```

Fill `.env.local` when prompted (see [Configuration](#configuration)), then:

```bash
pnpm dev                  # http://localhost:3010
```

Re-running `scripts/setup.sh` is safe — every step skips if already done.

### Option B — manual

```bash
cd product
pnpm install
cp .env.example .env.local   # then fill in the values below
pnpm vapid:generate          # paste the keys into .env.local
pnpm db:link && pnpm db:push && pnpm db:seed && pnpm db:seed:auth
pnpm dev                     # http://localhost:3010
```

---

## Configuration

All variables live in **`.env.local`** for local dev (gitignored). Copy from [`.env.example`](./.env.example), which is the authoritative, commented template. There are three parallel files, each a **full** set of vars for its environment:

| File | Used by | Notes |
|------|---------|-------|
| `.env.local` | `pnpm dev` | `NEXT_PUBLIC_APP_URL=http://localhost:3010` |
| `.env.preview` | `pnpm deploy:preview` | Vercel preview |
| `.env.prod` | `pnpm deploy:prod` | Vercel production |

Every DB and deploy script loads `.env.local` first, falling back to `.env.prod`.

### Required variables

| Variable | Where to get it |
|----------|-----------------|
| `NEXT_PUBLIC_SUPABASE_URL` | Supabase → Project Settings → API |
| `NEXT_PUBLIC_SUPABASE_ANON_KEY` | Supabase → Project Settings → API |
| `SUPABASE_SERVICE_ROLE_KEY` | Supabase → Project Settings → API *(keep secret)* |
| `SUPABASE_PROJECT_REF` | Supabase → Project Settings → General → Reference ID |
| `DATABASE_URL` | Supabase → Database → Connection string → URI → **Transaction pooler** |
| `GOOGLE_GEMINI_API_KEY` | <https://aistudio.google.com/apikey> |
| `GOOGLE_MAPS_SERVER_KEY` | Google Cloud → Credentials *(server key)* |
| `NEXT_PUBLIC_GOOGLE_MAPS_BROWSER_KEY` | Google Cloud → Credentials *(browser key, referer-restricted)* |
| `VAPID_PUBLIC_KEY` / `VAPID_PRIVATE_KEY` / `NEXT_PUBLIC_VAPID_PUBLIC_KEY` | `pnpm vapid:generate` |
| `REMINDERS_FIRE_SECRET` | `openssl rand -hex 32` |
| `NEXT_PUBLIC_APP_URL` | `http://localhost:3010` (local) / your Vercel URL (prod) |

### Optional variables

| Variable(s) | Effect when omitted |
|-------------|---------------------|
| `GEMINI_MODEL` | Defaults to `gemini-2.5-flash`. |
| `NEXT_PUBLIC_USE_GOOGLE_APIS` | `true` by default; set `false` to use PostGIS + Haversine + static-sector fallbacks (no Google calls). |
| `SUPABASE_ACCESS_TOKEN` | CLI prompts `supabase login` interactively instead. |
| `WHATSAPP_PHONE_NUMBER_ID`, `WHATSAPP_ACCESS_TOKEN` | `notify_provider` falls back to SMS → mock. |
| `TWILIO_ACCOUNT_SID`, `TWILIO_AUTH_TOKEN`, `TWILIO_FROM_NUMBER` | `notify_provider` falls back to mock. |
| `VERCEL_TOKEN`, `VERCEL_ORG_ID`, `VERCEL_PROJECT_ID` | Only needed for headless CLI deploys. |
| `TWA_SHA256_FINGERPRINTS` | Android TWA `assetlinks.json` is generated empty. |

> **Graceful degradation:** the app runs without Gemini, Google Maps, WhatsApp, or Twilio keys — each falls back to a deterministic mock so the full flow stays demonstrable. Only **Supabase is mandatory**.

---

## Database setup

```bash
pnpm db:link          # link the Supabase CLI to your project
pnpm db:push          # apply migrations (extensions, tables, RLS, RPCs, pg_cron)
pnpm db:seed          # categories + ~30 demo providers (via psql + DATABASE_URL)
pnpm db:seed:auth     # demo Auth users (see below)
pnpm db:types         # regenerate lib/supabase/database.types.ts
```

Extensions used (enabled by the first migration — verify under **Database → Extensions**): `postgis`, `pg_cron`, `pg_net`, `btree_gist`, `pgcrypto`.

### Wire up `pg_cron` reminders

Supabase's hosted Postgres can't reach `localhost`, and reminder config lives in a table (not `alter database`). After migrations, run in the Supabase SQL editor:

```sql
insert into public.app_config(key, value) values
  ('reminders_fire_url',    'http://localhost:3010/api/reminders/fire'),
  ('reminders_fire_secret', 'YOUR_REMINDERS_FIRE_SECRET')
on conflict (key) do update set value = excluded.value, updated_at = now();
```

For local testing, either tunnel with `ngrok http 3010` (and use that URL above) or fire the queue manually: `select public.drain_due_reminders();`. For production, set `reminders_fire_url` to your deployed URL.

---

## Run

```bash
pnpm dev                  # http://localhost:3010 (Turbopack)
```

### Demo accounts (seeded by `pnpm db:seed:auth`)

| Role | Email | Password |
|------|-------|----------|
| Customer (Ayesha) | `ayesha@example.com` | `Demo!1234` |
| Provider (Ali AC Services) | `ali@example.com` | `Demo!1234` |
| Provider (Bright Tutors) | `tutor@example.com` | `Demo!1234` |

Open two windows (customer + provider) to watch the realtime two-phase booking flow.

---

## Deploy (Vercel)

**Recommended — Git integration.** Connect the repo in the Vercel dashboard once; every push gets a preview URL, `main` goes to production. No CLI or token needed.

**CLI deploys** use `scripts/deploy.sh`, which validates env vars, runs `typecheck` + `build`, pushes DB migrations, syncs every env var to Vercel, and deploys:

```bash
pnpm deploy:preview       # uses .env.preview (fallback .env.local)
pnpm deploy:prod          # uses .env.prod (fallback .env.local)
```

Teammates without DB rights: `SKIP_DB_PUSH=1 pnpm deploy:preview`.

**After the first production deploy**, update three things (the script prints these):
1. `app_config.reminders_fire_url` → your Vercel URL.
2. Supabase → Auth → URL Configuration → Site URL + redirect URLs.
3. Google Maps browser key → add `https://<your-vercel-url>/*` to allowed referers.

`vercel.json` (committed) pins `regions: ["sin1"]`, per-route `maxDuration` for the agent/reminder routes, and SW/manifest cache headers.

---

## Scripts reference

| Command | Purpose |
|---------|---------|
| `pnpm dev` | Dev server on :3010 (Turbopack) |
| `pnpm build` / `pnpm start` | Production build / serve |
| `pnpm typecheck` | `tsc --noEmit` |
| `pnpm lint` | ESLint |
| `pnpm test` / `test:unit` / `test:integration` | Vitest |
| `pnpm db:link` / `db:push` / `db:reset` | Link CLI / apply migrations / push + reseed |
| `pnpm db:seed` / `db:seed:auth` | Seed providers / demo Auth users |
| `pnpm db:types` | Regenerate Supabase TS types |
| `pnpm db:verify` / `db:diff` / `db:status` | Read-only schema check / diff / pending check |
| `pnpm vapid:generate` | Generate Web Push VAPID keys |
| `pnpm pwa:check <url>` | Validate PWA/APK asset readiness |
| `pnpm env:pull` | Pull Vercel env into `.env.local` |
| `pnpm deploy:preview` / `deploy:prod` | Deploy to Vercel |

---

## Troubleshooting

| Symptom | Fix |
|---------|-----|
| `pnpm build` missing module | `pnpm install` again; ensure Node ≥ 20. |
| Auth redirect loop | Check Supabase → Auth → URL Configuration (Site URL + redirects). |
| Push not delivering | Verify `VAPID_*`, accept the browser prompt; install the PWA on iOS. |
| `pg_cron` not firing | Confirm the `app_config` rows exist; tunnel localhost with ngrok. |
| Gemini errors | App falls back to deterministic mocks; set `GOOGLE_GEMINI_API_KEY` and restart. |
| Map tiles missing | Set `NEXT_PUBLIC_GOOGLE_MAPS_BROWSER_KEY`, or `NEXT_PUBLIC_USE_GOOGLE_APIS=false`. |
| `db:seed` fails | Ensure `psql` is installed and `DATABASE_URL` uses the **Transaction pooler** URI. |

For anything not covered here, see **[`SETUP.md`](./SETUP.md)**.
