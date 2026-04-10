# Apex Geotech AR Dashboard — Operations Guide

## Overview

A live accounts receivable dashboard connected to Supabase (PostgreSQL), fed by Microsoft Fabric. Hosted on GitHub Pages.

| Component | Location |
|---|---|
| **Live Dashboard** | https://waylintaylorvick.github.io/ar-dashboard-/ |
| **Supabase Project** | https://supabase.com/dashboard/project/oirrixsqzuufgqbxgucs |
| **GitHub Repo** | https://github.com/WaylinTaylorVick/ar-dashboard- |
| **Fabric Notebook** | ApexDWH → sales.vw_ARInvoices |

---

## Updating the Dashboard (Most Common Task)

When Claude produces a new `apex_ar_live.html` file, deploy it like this:

### Prerequisites (one-time setup)
1. Install Scoop: https://scoop.sh
2. Install git: `scoop install git`
3. Log into GitHub CLI: `git config --global credential.helper manager-core`

### Deploy steps

Open VS Code terminal (`Ctrl + backtick`) and run:

```powershell
# 1. Copy the new file into the dashboard folder (rename it to index.html)
copy "C:\Users\taylo\Downloads\apex_ar_live.html" "C:\Users\taylo\dashboard\index.html"

# 2. Navigate to the dashboard folder
cd C:\Users\taylo\dashboard

# 3. Stage, commit, and push
git add index.html
git commit -m "Update dashboard"
git push
```

GitHub Pages redeploys automatically within ~60 seconds. Verify at:
https://waylintaylorvick.github.io/ar-dashboard-/

---

## Architecture

```
Microsoft Fabric (ApexDWH)
    └── sales.vw_ARInvoices
            │  (Fabric notebook runs nightly at 2 AM)
            ▼
    Supabase (PostgreSQL)
            │  ar_invoices table (150k+ rows)
            │  ar_notes table
            │
            ├── Live views (vw_*)     ← age against CURRENT_DATE
            ├── Historical views (v_*) ← age against as_of_week
            ├── Materialized views (mv_*) ← pre-computed, fast
            │
            ▼
    GitHub Pages
        index.html (single-file dashboard)
            └── Supabase Auth (email/password login)
```

---

## Data Flow

**Nightly (2 AM, automatic):**
1. Fabric notebook reads `sales.vw_ARInvoices`
2. Upserts rows into `ar_invoices` using the Supabase service role key
3. Calls `refresh_materialized_views()` to rebuild trend/billing/collections data

**Monday 7:30 AM (automatic):**
- Supabase cron sends the weekly collections digest email (requires Resend API key — see Email Setup below)

---

## Adding Users

1. Go to: https://supabase.com/dashboard/project/oirrixsqzuufgqbxgucs/auth/users
2. Click **Invite user** → enter email → Send invite
3. User receives an email with a link → they set their password → can log in immediately

> **Important:** The Site URL in Supabase Auth must be set to the GitHub Pages URL, not the edge function URL, or invite links will break.
> Set at: Authentication → URL Configuration → Site URL = `https://waylintaylorvick.github.io/ar-dashboard-/`

---

## Supabase Key Reference

| Key | Where to find | Used for |
|---|---|---|
| **Anon key** | Project Settings → API | Dashboard JS (read-only after login) |
| **Service role key** | Project Settings → API | Fabric notebook ETL, edge functions |

> ⚠️ Never commit the service role key to GitHub. Keep it in the Fabric notebook only.

---

## Fabric Notebook Configuration

The notebook cell that pushes data should have:

```python
SUPABASE_KEY = 'your_service_role_key'   # NOT the publishable/anon key
SUPABASE_URL = 'https://oirrixsqzuufgqbxgucs.supabase.co'
FABRIC_WAREHOUSE = 'ApexDWH'
FABRIC_VIEW      = 'sales.vw_ARInvoices'
BATCH_SIZE       = 500
TRUNCATE_FIRST   = True
```

The **last cell** of the notebook should call:

```python
import requests
response = requests.post(
    f"{SUPABASE_URL}/rest/v1/rpc/refresh_materialized_views",
    headers={
        "apikey": SUPABASE_KEY,
        "Authorization": f"Bearer {SUPABASE_KEY}",
        "Content-Type": "application/json"
    },
    json={}
)
print(f"Materialized views refreshed: {response.status_code}")
```

---

## Email Digest Setup (Optional)

The Monday digest email requires a free Resend account.

1. Sign up at https://resend.com
2. Create an API key
3. Go to Supabase → Edge Functions → Manage Secrets
4. Add these two secrets:

| Secret Name | Value |
|---|---|
| `RESEND_API_KEY` | Your Resend API key (starts with `re_`) |
| `COLLECTIONS_EMAIL_TO` | Recipient email(s), comma-separated |

Test immediately by clicking **Send Digest** in the dashboard header.

---

## Database Views Reference

### Live views (use for current AR data)
| View | Purpose |
|---|---|
| `vw_ar_summary` | Total AR by location, aged against today |
| `vw_customer_summary` | Per-customer AR buckets, aged against today |
| `vw_ar_invoices_bucketed` | All open invoices with live bucket assignment |

### Materialized views (pre-computed weekly)
| View | Purpose |
|---|---|
| `mv_weekly_trend` | 13-week stacked bucket trend |
| `mv_customer_trend` | 13-week trend per customer |
| `mv_payment_behavior` | Avg DTP, trend (improving/stable/deteriorating) per customer |
| `mv_billings_monthly` | Monthly billings by location/customer/salesperson |
| `mv_collections_monthly` | Monthly collections with DTP and on-time stats |

### Compatibility views (thin wrappers — dashboard queries these)
`v_weekly_trend`, `v_customer_trend`, `v_payment_behavior`,
`v_billings_monthly`, `v_collections_monthly`

### Other views
| View | Purpose |
|---|---|
| `v_near_boundary` | Open invoices within 7 days of crossing to next bucket |
| `v_payment_behavior` | Customer payment history and trend |
| `v_wow_movements` | Week-over-week invoice movements (Aged/Paid/New/Improved) |
| `v_historical_aging` | Historical aging snapshots using DAX-parity balance logic |

---

## Refreshing Materialized Views Manually

If the dashboard data looks stale, run this in the Supabase SQL editor:

```sql
SELECT refresh_materialized_views();
```

Or trigger via the dashboard Refresh button (↻) which calls the `refresh-ar-data` edge function.

---

## Edge Functions

| Function | URL | Purpose |
|---|---|---|
| `dashboard` | `/functions/v1/dashboard` | Serves dashboard HTML (backup — GitHub Pages is primary) |
| `refresh-ar-data` | `/functions/v1/refresh-ar-data` | Touches refreshed_at, triggers on Refresh button click |
| `send-collections-email` | `/functions/v1/send-collections-email` | Sends Monday digest, triggers on Send Digest button |
| `load-ar-csv` | `/functions/v1/load-ar-csv` | Accepts CSV upload (kept as backup to Fabric pipeline) |

---

## Troubleshooting

**Dashboard shows stale data**
→ Run `SELECT refresh_materialized_views();` in Supabase SQL editor
→ Or click ↻ Refresh in the dashboard

**Fabric notebook ran but no new data in Supabase**
→ Check the notebook is using the **service role key**, not the publishable/anon key
→ The anon key cannot write to `ar_invoices` (RLS blocks it)

**User can't log in / invite link goes to wrong place**
→ Check Authentication → URL Configuration → Site URL = GitHub Pages URL

**`BOOT_ERROR` when opening auth pages**
→ One of the edge functions is crashing — check Edge Function logs in Supabase dashboard

**GitHub push asks for credentials repeatedly**
→ Run: `git config --global credential.helper manager-core`
→ Then push again — a browser window will open to authenticate with GitHub

**Dashboard shows `Error: column does not exist`**
→ A REST query is referencing a wrong column name — check the browser console for the full error and which view/table it's hitting

---

## Cron Jobs (Automatic, No Action Needed)

| Job | Schedule | What it does |
|---|---|---|
| `refresh-ar-weekly` | Monday 7:00 AM ET | Refreshes timestamps + materialized views |
| `send-collections-email-weekly` | Monday 7:30 AM ET | Sends digest email (if Resend configured) |

View/edit at: Supabase → Database → Extensions → pg_cron

---

## Dashboard Tabs

| Tab | Data Source | Notes |
|---|---|---|
| Overview | `vw_ar_summary`, `mv_weekly_trend` | KPIs, location chart, 13-week trend |
| Aged AR Review | `vw_customer_summary` | Executive view, 60+ exposure |
| Collection Actions | `vw_customer_summary`, `v_near_boundary` | Priority queue, near-boundary alerts |
| Customer Detail | `vw_customer_summary` | Filterable/sortable customer table |
| Salesperson | `vw_customer_summary` | Aggregated from customer data |
| AR Movements | `v_wow_movements` | Week-over-week invoice movements |
| 💰 Billings | `mv_billings_monthly` | AR-derived billings, not audited revenue |
| 📈 Collections | `mv_collections_monthly` | Collections history, DTP, on-time analysis |
| 📝 Notes | `ar_notes` | Team collection notes, pinnable/resolvable |

---

## Contact / Handoff Notes

- Built with Claude (Anthropic) — to continue development, start a new Claude conversation and reference this README
- All infrastructure is in Supabase project `oirrixsqzuufgqbxgucs`
- Source data comes from Microsoft Fabric warehouse `ApexDWH`
- Dashboard is a single HTML file (`index.html`) — no build process, no dependencies to install
