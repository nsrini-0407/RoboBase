# RoboBase

Parts inventory for a non-profit that runs several FTC teams out of one shared shop.

- **Org admins** add parts from the AndyMark / goBILDA / REV / Studica / Swyft catalog. Each part gets a shop
  part number (1001, 1002, …) to print on its bin label. Admins set low-stock thresholds, restock, correct
  counts, and see who took what.
- **Students** sign in with their team's account, type the part number and quantity they're taking (or returning),
  and that's it.
- When a part drops to its threshold, everyone on the org's alert list gets **one email** with a reorder link.
  It re-arms after the part is restocked above the threshold.

Stack: Next.js 16 (App Router, server actions) on Vercel · Supabase (Postgres, Auth, RLS) · Resend for email.

## How it fits together

| Table | Purpose |
|---|---|
| `organizations` | One per shop. Holds alert email list and the part-number counter. |
| `teams` | FTC teams in the org. Each gets a join code. |
| `profiles` | One per user: `role` = `admin` or `team`, plus `team_id`. |
| `invite_codes` | 8-char codes: one admin code per org, one per team. Only admins can read them. |
| `catalog_parts` | Global vendor catalog, filled by the import scripts. |
| `inventory_items` | What the org actually stocks, with `part_number`, `quantity`, `threshold`. |
| `transactions` | Append-only log of every take / return / restock / adjustment. |
| `low_stock_alerts` | Queue of threshold crossings; emailed right after the action, retried daily by cron. |

Quantities can **only** change through the `record_movement` and `set_item_quantity` database functions, so every
change is logged. Row-level security keeps each org's data separate. Students can see their shop's inventory and their own
team's history, but not other teams' logs or any join codes.

## Setup

1. **Supabase**: create a project and run `supabase/migrations/20260926000000_init.sql` (SQL editor, or `supabase db push`).
   - Auth → URL Configuration: set **Site URL** to your Vercel URL and add `https://YOUR-APP.vercel.app/auth/callback`
     (and `http://localhost:3000/auth/callback`) to **Redirect URLs**.
   - Supabase's built-in email sender is rate-limited (a few emails/hour). Before onboarding students, set up
     custom SMTP (Auth → SMTP; Resend works) or turn off "Confirm email".
2. **Env**: `cp .env.example .env.local` and fill it in.
3. **Catalog**: `npm install && npm run catalog:sync` (takes ~15 minutes, mostly goBILDA's ~2,400 product pages).
   Re-run whenever you want fresh prices / new parts.
   - **Studica** blocks automated access, so export or assemble a CSV with `name,sku,url,price,image_url` columns
     and run `npm run catalog:import-csv -- Studica ./studica.csv`. Admins can also add any part by hand
     ("Add parts → Custom part").
4. **Run**: `npm run dev`, then open http://localhost:3000.
5. **Deploy**: import the repo in Vercel, add the same env vars (with `NEXT_PUBLIC_SITE_URL` = production URL), deploy.
   `vercel.json` schedules a daily retry of any low-stock emails that failed.

## First run

1. Sign up → **Create organization**. You're the admin; your email is the first alert recipient.
2. **Teams** → add your 6 teams. Each shows a join code.
3. **Inventory → Add parts** → search the catalog, enter qty on hand + alert threshold + bin location.
4. **Inventory → Print bin labels** and stick them on the bins.
5. Students sign up and enter their team's code. Shared login per team (e.g. a shop tablet) works fine; so do individual accounts.
   An admin can also log parts for any team from the **Take parts** page (kiosk mode).
6. **Settings** → add more alert emails, send a test email, share the admin code with other mentors.

## Scripts

| Command | |
|---|---|
| `npm run dev` | Local dev server |
| `npm run catalog:sync [-- AndyMark goBILDA REV Swyft] [--dry-run] [--limit=N]` | Import vendor catalogs |
| `npm run catalog:import-csv -- <Vendor> <file.csv>` | Import a CSV into the catalog |
| `npm run typecheck` / `npm run lint` | Checks |
