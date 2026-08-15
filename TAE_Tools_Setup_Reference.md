<!-- title: TAE Tools — Full Setup Reference -->

# TAE Tools — Full Setup Reference

*A plain-language map of every tool, where it lives, and how it stays updated — so we can organize, fix, or move things later without guessing or breaking what already works. Last updated August 15, 2026.*

---

## How to read this, in plain English

A few words show up over and over below. Here's what they actually mean, once, so the rest of this doc doesn't need to keep explaining itself:

- **Repo (repository)** — the folder of code for one tool, tracked by a system called *git* so every change is recorded and can be undone. If a tool "has a repo," that history exists. If it doesn't, the only copy of its code is whatever's sitting on this computer right now.
- **GitHub** — where those repos are stored online, as a backup and as the trigger for deploys (below).
- **Vercel** — the hosting service that actually runs each tool and gives it a live web address.
- **Deploy** — publishing a change so it's live on the real website, not just sitting on this computer.
- **"Git push → auto-deploy"** — the good pattern: save a change, run one command, and Vercel updates the live site by itself within a minute or two.
- **"Manual deploy"** — the fragile pattern: editing the code does *nothing* to the live site until someone specifically runs a deploy command from that exact folder. Easy to forget, easy to lose track of what's actually live.
- **Database (Supabase)** — where the shared, structured data lives (client records, tickets, reports, etc.) for several tools at once. Covered in its own document — link below — this one only says which tools use it.
- **Connector** — a way of linking an outside account (like Figma or Supabase) directly into a Claude session, so it can pull real data instead of guessing. Turning one "on" in claude.ai's Settings → Connectors doesn't always mean it's usable right away — see below.

---

## The tools, at a glance

| Tool | What it's for | Updates via | Data source |
|---|---|---|---|
| TAE Dashboard | Client + partner portal | Git push → auto | Shared database |
| TAE Discovery | Adaptive client interview | Git push → auto | Shared database |
| Social Analyzer | Social media analysis + lead dashboard | Git push → auto | Shared database |
| F.R.I. Marketing site | Client-facing marketing site | Git push → auto | None |
| TAE Dono Dashboard | Internal monthly reporting dashboard | Git push → auto | Basecamp + a few other services (not the shared database) |
| ACS Dashboard | Client-specific dashboard | Git push → auto | Unconfirmed — not yet reviewed |
| TAE Twin | Marketing site | Git push → auto | None |
| TAE ContextLayer | Hosted MCP server for Claude | Git push → auto | Shared database, full access |

Every tool above is now on the same simple pattern — save a change, `git push`, it's live within a minute or two.

---

## Each tool, in detail

### TAE Dashboard
- **Folder:** `tae-portal`
- **What it does:** The main portal your clients and partners log into — tickets, reports, published files, notifications.
- **Updates:** Save a change, `git push`, Vercel deploys it automatically. This is the pattern every tool below should ideally match.
- **Data:** Reads and writes the shared Supabase database — full detail in the [Supabase Database Audit](https://claude.ai/code/artifact/ad0fc3af-4fa3-40c3-926b-efdfe7de5f34).
- **Database changes:** **New as of 2026-07-27** — the Supabase CLI is now linked to this project (`supabase link`), so schema changes go through `supabase/migrations/*.sql` + `supabase db push` instead of hand-editing the dashboard. See the Supabase audit doc's finding 5 for why that matters.

### TAE Discovery
- **Folder:** `tae-discovery`
- **What it does:** The adaptive interview tool prospects go through before becoming clients.
- **Updates:** Same as TAE Dashboard — git push, auto-deploys.
- **Data:** Owns most of the shared database's core tables (companies, clients, sessions).

### Social Analyzer
- **Folder:** `tae-social-analyzer`
- **What it does:** Analyzes a prospect's social presence; results show up in an internal Lead Dashboard.
- **Updates:** Git push, auto-deploys — confirmed working correctly.
- **Data:** Writes to one table (`analyses`) in the shared database.

### F.R.I. Marketing site
- **Folder:** `fri-marketing-site`
- **What it does:** The client-facing marketing site for F.R.I., live at `fri-marketing.vercel.app`.
- **Updates:** **Fixed July 23, 2026** — now at `github.com/taemarketing/fri-marketing`, connected to its Vercel project. `git push` deploys it automatically.
- **Data:** None. It used to connect to the shared database (a leftover from an earlier 3-design comparison version); that connection was fully removed on July 23, 2026 — verified directly against the live site.
- **Note:** The *old* version of this site — the 3-design comparison page — lived in a folder called `fri-marketing-redesign`. That folder still exists on disk but is no longer live anywhere; safe to ignore or archive.

### TAE Dono Dashboard
- **Folder:** `tae-dono-dashboard` (renamed from `clients-dashboard` on 2026-07-24 to match the live product name — also fixed a leftover `package.json` name field that wrongly said "acs-dashboard", and a hardcoded path in `basecamp-sync/patch.js` that referenced the old folder name)
- **What it does:** Your internal dashboard, refreshed roughly monthly, pulling in data from Basecamp plus a few smaller integrations (Noko time tracking, "friction" notes, Claude session data, and a scan of the `~/Desktop/Clients` folder for file-output counts).
- **Updates:** Was manual-only (no git history at all) before July 23, 2026. **Fixed July 23, 2026:** now has a real git repo at `github.com/taemarketing/tae-dono-dashboard`, connected to its Vercel project — `git push` now deploys automatically, same as TAE Dashboard and TAE Discovery. Confirmed working end-to-end during the August 2026 update (verified via the Vercel API that a pushed commit reached `READY` state).
- **Data:** Not connected to the shared Supabase database at all — it pulls from Basecamp and a couple of other small services, each with its own folder inside `tae-dono-dashboard` (`basecamp-sync`, `noko-sync`, `friction-sync`, `claude-sync`).
- **Noko API — v1 token is read-only (found 2026-08-15):** the hardcoded v1 API token (`entries.json?token=...`, still in `noko-sync/sync.cjs` and `basecamp-sync/`) works fine for reads but silently rejects writes — a POST to create an entry returns a 422 with an empty body regardless of payload (confirmed via a control test: even `{}` gets the identical empty 422, while a bad token correctly 401s — that pattern means a permission block, not a validation error). **Fix that works:** use the newer **v2 API** (`https://api.nokotime.com/v2/entries`) authenticated with a **Personal Access Token** instead (Noko → Connected Apps & API → Personal Access Tokens → generate one; shown once). The v1 token is still fine for the existing read-only sync scripts — no need to touch those.
- **Real bugs found and fixed in `basecamp-sync/sync.js` (2026-08-15):**
  - Basecamp's `updated_since` filter on `/projects/recordings.json` is **silently ignored** — confirmed by requesting comments from the last 24 hours and getting results going back 15+ pages / several months. Every "incremental" sync was secretly re-pulling the full account history. Fixed with client-side early-termination (stop paginating once a page's oldest record predates the last sync), same pattern already used for Noko's `user_ids` filter (also ignored by that API).
  - `dailyActivity`/`dailyComments`/`dailyUploads` were **never seeded from the previously saved file** in incremental mode — unlike the per-project stats, which are. Every sync run silently overwrote the daily-activity history with only that run's narrow fetch window, so running the sync twice in a row erased real data. Fixed by seeding from `existingData.dailyActivity` before merging in new dates.
  - `scanClaudeSessions()` dropped any session file with a >4-hour timestamp span entirely instead of capping it at 4 hours, undercounting daily Claude activity on heavy work days. Fixed to cap instead of drop.
  - `fetchAll()` had no request timeout, so a single stalled HTTP request could hang the whole sync indefinitely with zero output. Added a 20s timeout, 3 retries, and per-page progress logging.
- **Noko/Basecamp data gaps are real, not sync bugs:** twice now (April/May and July/August 2026), Donovan's own Noko time entries genuinely stopped being logged for weeks at a time even though Basecamp activity continued normally. Both times it was confirmed via direct API pagination (not just the sync script) that no entries existed for that user in that window. The fix each time was estimating hours by distributing a rolling-average monthly total across active clients, weighted by Basecamp comment volume per client, then logging the result as real Noko entries (v2 API) so the record stays consistent going forward — not just patching the dashboard's local data file.
- **Data syncs are now on an automated monthly cron (set up 2026-08-15):** a `launchd` job (`com.tae.dono-dashboard-sync`, plist at `~/Library/LaunchAgents/`) runs `tae-dono-dashboard/scripts/sync-and-deploy.sh` on Donovan's own Mac at 7:00 AM on the 1st of every month. It chains all three read-only syncs (Noko, Basecamp, Claude), then commits and `git push`es if anything changed — Vercel auto-deploy takes it from there. Logs land in `scripts/logs/` (last 12 runs kept). Deliberately does **not** automate the Noko-hours backfill for data gaps — that stays a manual, reviewed step, since it involves a judgment call about how to distribute estimated hours across clients.
  - **Two prerequisites this needed, both now fixed:** (1) Basecamp's OAuth access token expires every 14 days and previously required clicking through a browser re-auth flow — a hard blocker for anything unattended. Fixed by adding token-refresh logic using the `refresh_token` that was already being saved but never used (`refreshTokens()` in `basecamp-sync/sync.js`, checked via a `savedAt` timestamp added to `tokens.json`). (2) macOS blocks background/launchd-spawned processes from reading `~/Desktop` by default ("Operation not permitted") even though the same script works fine run manually from Terminal — fixed by granting Full Disk Access to `/bin/bash` and `node` in System Settings → Privacy & Security. Both were one-time fixes; nothing further needed monthly.
  - **Known limitation:** if the Mac is asleep or off at 7 AM on the 1st, launchd runs the job the next time it wakes rather than skipping it — so the sync can land a few hours late some months, but won't be silently missed.
- **2021–2023 file counts were silently hardcoded to zero (found and fixed 2026-08-15):** `src/data.ts`'s `monthlyOutput` array hand-coded the entire 2021-04 through 2023-09 range as all-zero, with a comment claiming "no folder file breakdown available" for that period. That was wrong — the real Clients-folder scan does have data for those months (2021: 4 files, 2022: 35 files, etc.), it was just being discarded, which showed up on the dashboard as 2021/2022 = 0 files with `NaN%`/`+Infinity%` year-over-year comparisons. Fixed by generating that range dynamically from the real scan (`basecamp-data.json`'s `monthlyFolderFiles`) instead of hardcoding it, falling back to zero only for months genuinely absent from the scan. Also excludes scan dates before 2021-01 (1980, 2004, 2013, 2014, 2019, 2020 all show up in the raw scan) — those are almost certainly corrupted/reset file timestamps predating the business, not real work, so they're filtered out rather than included.
- **The rest of `monthlyOutput` (Oct 2023–May 2026) was ALSO stale, not just the zeroed block above (found and fixed 2026-08-15, same day, second pass):** those months held real-looking hardcoded numbers, but they were a frozen snapshot from whenever someone last generated the array by hand — never updated as more files got added/modified in the Clients folder since. Confirmed by an independent recount: 2024 was hardcoded at 881 files vs. 1,991 in a fresh scan, less than half. Fixed by replacing the entire array with a fully dynamic build off the live scan for every month from business start (2021-04) through the last completed month — no literals left to go stale, self-corrects on every future sync.
- **Local Clients-folder file timestamps for 2021–2023 are unreliable and can't be trusted even after the fix above:** both the modification date AND the separate macOS creation date for that era's files (spot-checked on Car-Mart, which has 1,491 tracked files) show the same wrong pattern — the majority dated 2024, not when the work actually happened. Two independent timestamps agreeing rules out a fluke; it means the files were physically copied/restored onto this Mac in 2024, resetting both dates. Donovan confirmed this lines up with a computer crash and restore around that time — nothing to do with any TAE tooling. **Takeaway: file mtime/btime is not a trustworthy proxy for "when work happened" for anything from 2021–2023 on this Mac**, even though it's fine for 2024 onward. Basecamp's server-side upload timestamps are the only reliable signal for that period (see next item) — don't try to fix pre-2024 volume from local file scans again.
- **Account-wide monthly rollups were silently losing all history on every incremental sync run (found and fixed 2026-08-15, third pass same day):** the exact same seeding gap fixed for `dailyActivity` earlier in the day also affected `monthlyUploads`, `monthlyComments`, and `monthlyActiveProjects` — none of them were seeded from the previously saved file in incremental mode, only `dailyActivity` got that fix at first. Two incremental runs back-to-back left all three completely empty (confirmed: 0 entries, down from 63 real months in the original July 1 full-sync baseline). This combined with the timestamp problem above to make 2021–2022 look like almost nothing happened — Basecamp's historical upload data was the one reliable signal for that era, and it was exactly what this bug was erasing. Fixed by adding the same seeding pattern to all three, restoring the original full-sync baseline's 63 months of real history, and re-running incrementally to merge in new months without erasing anything. Total files (Apr '21–Jul '26) moved from ~2,962 (undercounted, stale array only) → 4,974 (live scan only, but missing pre-2024 Basecamp signal) → **26,357** (both fixes combined) — with 2022 correctly landing as the peak year (5,581 files), matching the known Car-Mart-heavy period instead of flattening it to near-zero.
- **Process lesson from the above (2026-08-15):** a month-over-month or year-over-year swing of this magnitude (files dropping by tens of thousands, a "best year" flipping to near-zero) is itself the signal that something's wrong — it should be caught and investigated *before* presenting the number as fixed, not after Donovan pushes back a second or third time. When a "fix" produces a dramatically different total, sanity-check it against an independent source (a fresh `find`/count, a different API, the user's own memory of the period) before calling it done.

### ACS Dashboard
- **Folder:** `acs-dashboard`
- **What it does:** Not yet reviewed in detail — flagged here because it had the exact same setup gap the Dono Dashboard had.
- **Updates:** **Fixed July 23, 2026** — now at `github.com/taemarketing/acs-dashboard`, connected to its Vercel project. `git push` deploys it automatically.

### TAE Twin
- **Folder:** `tae-twin`
- **What it does:** The public marketing site — plain HTML/CSS, not related to ContextLayer. (Corrected 2026-07-30: previously described here as "ContextLayer groundwork," which wasn't accurate — the actual ContextLayer server is its own separate repo, see below.)
- **Worth flagging:** its GitHub home is a *different* organization (`The-Artist-Evolution`) than every other tool above (`taemarketing`). Confirmed 2026-07-24 this is fine as-is — not an accident worth fixing, just worth knowing before assuming "the GitHub org" is one single place.

### TAE ContextLayer
- **Folder:** `tae-contextlayer`
- **What it does:** The hosted MCP server — the connector between Claude and everything TAE knows about a client. Full strategic brief and complete tool-by-tool build history in `TAE_ContextLayer.md` (this repo) — this entry is a summary, not the full record. As of 2026-08-13, exposes **31 tools** across three groups:
  - **7 client/relationship tools** — `get_client_context` (pulls a client's core record, account owner, living strategic brief, latest Discovery intelligence, and Portal activity into one bundle), `mark_as_client`, `log_relationship_event`, `mine_discovery_sessions`, `add_industry_alias`, `add_brief_entry`, `assign_account_owner`.
  - **6 cadence/reporting tools** — `get_cadence_tasks`, `add_cadence_task`, `log_cadence_task_run` (the internal recurring-task registry, now organized into 5 topical categories rather than time buckets), `add_task_vote` (anonymous internal thumbs up/down), `get_cl_reports`, `add_cl_report` (append-only log of real ContextLayer-produced findings).
  - **18 website/site-management tools** — `get_site_analytics`, `get_site_deployments`, `get_dependency_report`, `get_site_uptime`, `get_broken_links`, `get_sitemap_robots_status`, `get_dns_domain_status`, `get_env_vars_redacted`, `get_site_content`/`update_site_content`, `generate_sitemap_robots`, `generate_llms_txt`, `scan_site_fields`, and 5 file-based Holiday Pages tools (`list_holiday_pages`, `get_holiday_page`, `save_holiday_page`, `apply_holiday_page`, `revert_holiday_page`). Most read real external infrastructure (Vercel, GitHub, DNS) rather than generated content; the two `generate_*` tools are the exception — `generate_sitemap_robots` mechanically crawls, `generate_llms_txt` takes authored content as input instead (see `TAE_ContextLayer.md`'s August 13 status update for why those two differ).

  A control panel for the client/relationship and cadence tools lives directly on the TAE Dashboard's main admin page — a collapsible "ContextLayer" section under Notifications, same pattern as Analytics/Partners/Notifications. The website-management tools surface separately, in Manage Client's own Website section (currently enabled for F.R.I. Marketing only).
- **Updates:** `github.com/taemarketing/tae-contextlayer` (private), git push → auto. **Live as of 2026-07-30** at `https://tae-contextlayer.vercel.app` — verified directly against production (all tools list correctly, `get_client_context` pulls real Supabase data through the live server).
- **Data:** Connects to the shared Supabase database with the **service-role key** — full access, by design, not scoped to one schema — because its job is cross-system analysis, not a narrow client silo. See the ContextLayer doc's July 30 status update for why. Owns its own `contextlayer` schema (`client_engagements`, `relationship_events`, `industry_aliases`, `brief_entries`, `account_owners`, `cl_tasks`, `cl_task_runs`) via its own linked Supabase CLI project, same discipline as tae-portal's migrations — now on its own reserved migration-number block, `300–399` (see `TAE_Supabase_Database_Audit.md` finding 5's fifth update).
- **Both previously-open items here are resolved — this section had gone stale before either got marked fixed:**
  1. **tae-portal's production env vars** — confirmed set and working, not just assumed: this session ran dozens of live `curl` calls straight against `https://tae-portal-lilac.vercel.app/api/admin/contextlayer/...` in production, every one succeeding with real Supabase data. `CONTEXTLAYER_MCP_URL`/`CONTEXTLAYER_BEARER_TOKEN` are live on the real deployment.
  2. **The `contextlayer` schema exposure** — resolved 2026-08-01 (see `TAE_ContextLayer.md`'s July 30 status update). Root cause was never a stuck Supabase toggle — the Settings → Integrations → Data API panel doesn't reflect the real setting at all; fixed directly at the Postgres role level (`ALTER ROLE authenticator SET pgrst.db_schemas`). All ten tools work end-to-end against production now.

### Apex site — removed
Was a one-off test of building a site from a PSD design with Claude Code — never a real tool, never deployed anywhere. Confirmed no longer needed and moved to Trash on 2026-07-24 (not permanently deleted — still recoverable from Trash for a while if that changes). No longer listed above.

---

## Things worth knowing before changing anything

- **Two different GitHub homes exist:** most tools live under `github.com/taemarketing`; `tae-twin` lives under `github.com/The-Artist-Evolution`. If something's missing from where you expect, check the other one.
- **The Vercel connection available in Claude sessions can only see one project** (`tae-social-analyzer`) even though the others share the same Vercel team. It's a working connection, just scoped narrowly — so I can't check on tae-portal, tae-discovery, the Dono Dashboard, ACS Dashboard, or TAE Twin's live status through that tool the way I can for Social Analyzer. The Vercel dashboard itself always has the full picture.
- **Figma and Supabase connectors confirmed live in Claude sessions, 2026-08-15** — verified with real calls, not just a "Connected" checkmark: Figma's `whoami` returned the real account (handle TAE, info's team), Supabase's `list_projects` returned the real `TAE Tools` project. Worth knowing: after connecting one in claude.ai's Settings → Connectors, it can take up to ~15 minutes before it's actually usable as a tool in a session — no reconnect or fresh session needed, just a bit of a wait.
- **Every tool now has a git repo, a GitHub backup, and auto-deploy set up.** (Apex site, previously the one exception, was a test project that's no longer needed and has been removed entirely — see above.)
- **The shared database** (used by TAE Dashboard, TAE Discovery, and Social Analyzer) has its own detailed writeup, including some security gaps that are confirmed but not yet fixed: [TAE Tools — Supabase Database Audit](https://claude.ai/code/artifact/ad0fc3af-4fa3-40c3-926b-efdfe7de5f34) and the follow-up findings doc in memory.

---

## Where this file itself lives

This document and the Supabase Database Audit both live in their own small repo — `github.com/taemarketing/tae-docs` — rather than inside any single tool's repo, since both describe the whole system rather than one piece of it. Tool-specific docs (like TAE Discovery's own interview logic) live inside that tool's own repo instead, under a `docs/` folder.

## Keeping this current

This is meant to stay accurate, not be a one-time snapshot. Worth re-checking after any deploy-setup change, a new tool gets added, or before a bigger reorganization — edit the file directly in `tae-docs` and push, rather than starting the investigation over.
