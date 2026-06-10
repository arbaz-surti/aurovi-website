# Aurovi — Company Website

Marketing website for **Aurovi**, an AI-automation consultancy. This repo is the public
site plus the lead-capture pipeline behind its forms.

## What this folder is

A **static HTML site** originally exported from **Google Stitch**. There is **no build
system / package.json** — pages are plain HTML styled with **Tailwind via CDN** and a bit
of vanilla JS. The deployable site root is the **`stitch/`** directory (not the repo root).

## Layout

- `stitch/index.html` — the main landing page. Contains the **"Claim Your Spot" contact
  form** (`#journey-form`), the modal logic, custom dropdowns, and the FAQ accordion.
- `stitch/case-studies/` — case-study pages (currently **untracked / not yet deployed**).
- `brand/` — brand assets (logos, colors).
- `case-studies/` — Markdown **source** for case studies (working files, not the live site).
- `docs/superpowers/specs/` — design specs/notes.
- `stitch/*.jpg|*.png|*.jpeg` — images referenced by the pages.

## Lead capture (the main functional feature)

All form submissions become rows in a single **Supabase** (Postgres) table, `public.leads`,
reviewed in the Supabase **Table Editor** (spreadsheet-style, no SQL needed). Two ingestion
paths, by design:

- **Website contact form** (`#journey-form`): inserts **directly into Supabase first**
  (browser → REST, public `anon` key, **INSERT-only RLS**) so a lead is captured even if
  n8n is down, **then** fires a **best-effort** call to an n8n webhook for a thank-you email.
  Success UI is gated on the Supabase insert; the n8n call never blocks/breaks the UX.
- **Lead magnets** (built in **Tally**): Tally → n8n webhook (**W1**) → n8n inserts the
  lead (`service_role`) and emails the free resource. n8n is in the request path because
  delivery is the point.

### Schema model — hybrid
A few **common columns** every form shares + one **`responses` JSONB** that absorbs each
form's specific answers. **New lead magnets need no schema change** — their answers just go
into `responses`. Common columns: `full_name`, `email`, `source`, `source_name`, `status`,
`budget`, `service`, `magnet_delivered`, `thank_you_sent`, `notes`, `created_at`.

### Attribution — deterministic, no classifier
Each form sets `source` (stable slug) + `source_name` (human label):
- `website-contact` / "Website Contact Form"
- `lm-sop-engine` / "10-Minute SOP Engine"
- `lm-lead-recovery` / "7-Day Lead Recovery Sequence"

Filter by `source` / `source_name` in the Table Editor to slice leads by channel.

### Supabase
- Project ref: `yzzftkgfxjbipylckxbj` (URL `https://yzzftkgfxjbipylckxbj.supabase.co`).
- The **anon key is public by design** and lives in `stitch/index.html` — safe ONLY because
  of the INSERT-only RLS policy (anon can write, cannot read).
- **Never** put the `service_role` key in this repo or any client code — it belongs only in
  n8n's credential store.

## Conventions / gotchas

- The contact form has a **honeypot** field `hp_check` (off-screen). If filled, the submit
  is treated as a bot: it shows success but saves nothing.
- The contact form combines First/Last name → `full_name`; `first_name`, `last_name`,
  `company`, `message` are stored inside `responses`. `budget`/`service` are promoted to
  real columns (key sales filters).
- `N8N_THANKYOU_WEBHOOK_URL` in `stitch/index.html` is currently `''` (no-op). Set it once
  the n8n W2 thank-you workflow exists.
- **Adding a lead magnet:** create the Tally form → point its webhook at the n8n W1 webhook
  → set `source`/`source_name` → map answers into `responses`. No code/schema change here.

## Deploy

- GitHub: `github.com/arbaz-surti/aurovi-website`, branch `main`. **Pushes to `main`
  auto-deploy** via Vercel's Git integration (connected 2026-06-10).
- Host: **Vercel** (project `aurovi`, team `arbaz-surtis-projects`), live at
  **https://aurovi.ai/**.
- ⚠️ Project **Root Directory MUST = `stitch`** (set in project settings). The site lives
  in that subfolder; with root unset, Git builds serve the repo root and the homepage 404s.
- `stitch/vercel.json` sets static security headers (nosniff, SAMEORIGIN, referrer/
  permissions policy).
- Deployment Protection is ON → `*.vercel.app` deployment URLs require login; the public
  `aurovi.ai` domain is unaffected.
- No favicon (harmless `/favicon.ico` 404 in the console).

## Status / roadmap

- ✅ Contact-form capture built + verified (real headless-browser submission inserts to
  Supabase; error path + honeypot confirmed).
- ⏳ n8n **W2** (contact thank-you email) and **W1** (Tally lead magnets → insert + freebie).
- ⏳ Commit/deploy `stitch/case-studies/` if those pages should be live.
- 🔮 Deferred: a nicer pipeline UI (NocoDB or a custom Next.js dashboard), Supabase Auth /
  customer portal, optional AI lead-scoring of the `message` text in n8n.
