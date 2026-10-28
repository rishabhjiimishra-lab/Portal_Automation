# Hybrid job publishing — setup

This is your original React/Vite website with its existing layout, styling and filters preserved. New jobs are read from Supabase when the `VITE_` variables are set. Without them, the original hardcoded list remains visible for local development only. In production with Supabase configured, **only published rows** are displayed.

## 1. Supabase

Create/open a Supabase project. In SQL Editor, run `supabase/schema.sql`. The public anon/publishable key has **SELECT-only** access to `status='published'` records. All changes use a server-side service role key. **Never put the service role key in Vercel, Vite or frontend code.**

## 2. Migrate old jobs

Set `SUPABASE_URL` and `SUPABASE_SERVICE_ROLE_KEY` in your local shell, then run `python automation/import_legacy.py` once. Existing jobs are imported **pending** to avoid publishing old/unverified jobs. Open Supabase Table Editor → `public.jobs`, verify official application links and vacancy status, and set selected records' `status` to `published`. If you want the previous site to show all jobs immediately, verify and approve them before deploying the new frontend. The existing `src/data.js` remains intact as an offline fallback when Supabase isn't configured.

## 3. Configure company sources

Edit `automation/boards.json`, e.g.:

```json
{
  "greenhouse": [{"token":"YOUR_REAL_EMPLOYER_BOARD_TOKEN", "company":"Employer Name", "auto_publish":true}],
  "lever": [{"token":"YOUR_REAL_EMPLOYER_BOARD_TOKEN", "company":"Employer Name", "auto_publish":false}]
}
```

Use **real board tokens verified from official employer job boards**, not invented tokens. `auto_publish: true` means official allowlisted board postings with sufficient metadata publish automatically; incomplete postings and all boards with `auto_publish: false` go to `pending`. Verify that the employer actually owns the board before enabling automatic publishing. The collector does not crawl arbitrary pages, invent salaries, or import government jobs from third-party listings. Government jobs should be added after checking official government notifications.

## 4. GitHub Actions

Repository → Settings → Secrets and variables → Actions → New repository secret:

- `SUPABASE_URL`: Supabase project URL
- `SUPABASE_SERVICE_ROLE_KEY`: secret service-role key

Actions → **Sync job postings** → Run workflow for first test. Scheduled workflow runs about every six hours (GitHub cron can be delayed). Check Actions logs. If a source errors, it won't trigger closure checks. A job disappears from the official board on two *successful nonempty* syncs before its status changes to `closed`. You can manually approve `pending`, reject bad jobs, or close them in Supabase Table Editor. A previously rejected listing stays rejected on re-sync.

## 5. Vercel frontend

Vercel project → Settings → Environment Variables:

- `VITE_SUPABASE_URL`: project URL
- `VITE_SUPABASE_ANON_KEY`: public anon/publishable key

Redeploy once to bake these values into the Vite bundle. Future jobs update from Supabase without rebuilding. The app reloads data every five minutes while the page is open; new visits fetch immediately. Run locally with `npm ci`, `npm run dev`; validate with `npm run build`.

## 6. Publishing/review workflow

In Supabase Table Editor filter `status = pending`; check employer, source URL, role, and location. To approve set `status = published`; to reject set `status = rejected`. Public users can only see published jobs because of row-level security. No public admin endpoint or secret key is shipped.

## Important limitations

- The site currently has no pagination UI; the API loader fetches up to 10,000 records and can be optimized later.
- Auto-classification is conservative; government category is never inferred from ATS sources.
- Posting date for new records is **time first collected**, not necessarily the employer's publication date.
- Closing jobs based on absence can be imperfect if a provider returns partial but successful data. Review closures periodically.
- SQL and Python migration must be run by you against your Supabase project. No live credentials were used or database modified here.
- If any credentials from the original `.env.local` or `.git` history have been shared publicly, rotate them. These sensitive folders are excluded from this deliverable.
