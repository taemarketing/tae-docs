<!-- title: TAE Tools — Full Setup Reference -->

# TAE Tools — Full Setup Reference

*A plain-language map of every tool, where it lives, and how it stays updated — so we can organize, fix, or move things later without guessing or breaking what already works. Last updated July 23, 2026.*

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
| TAE Twin | ContextLayer groundwork | Git push → auto | Not yet reviewed in depth |

Every tool above is now on the same simple pattern — save a change, `git push`, it's live within a minute or two.

---

## Each tool, in detail

### TAE Dashboard
- **Folder:** `tae-portal`
- **What it does:** The main portal your clients and partners log into — tickets, reports, published files, notifications.
- **Updates:** Save a change, `git push`, Vercel deploys it automatically. This is the pattern every tool below should ideally match.
- **Data:** Reads and writes the shared Supabase database — full detail in the [Supabase Database Audit](https://claude.ai/code/artifact/ad0fc3af-4fa3-40c3-926b-efdfe7de5f34).

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
- **What it does:** Your internal dashboard, refreshed roughly monthly, pulling in data from Basecamp plus a few smaller integrations (Noko time tracking, "friction" notes, Claude session data).
- **Updates:** Was manual-only (no git history at all) before today. **Fixed July 23, 2026:** now has a real git repo at `github.com/taemarketing/tae-dono-dashboard`, connected to its Vercel project — `git push` now deploys automatically, same as TAE Dashboard and TAE Discovery.
- **Data:** Not connected to the shared Supabase database at all — it pulls from Basecamp and a couple of other small services, each with its own folder inside `tae-dono-dashboard` (`basecamp-sync`, `noko-sync`, `friction-sync`, `claude-sync`).
- **Known loose end (not urgent):** the Noko integration has its access code written directly into two files instead of stored more safely. It already works and nothing needs to change today — just worth fixing properly next time someone's in that code, rather than on its own.

### ACS Dashboard
- **Folder:** `acs-dashboard`
- **What it does:** Not yet reviewed in detail — flagged here because it had the exact same setup gap the Dono Dashboard had.
- **Updates:** **Fixed July 23, 2026** — now at `github.com/taemarketing/acs-dashboard`, connected to its Vercel project. `git push` deploys it automatically.

### TAE Twin
- **Folder:** `tae-twin`
- **What it does:** Groundwork for [ContextLayer](../TAE_ContextLayer_v6.pdf) — not reviewed in depth for this document.
- **Worth flagging:** its GitHub home is a *different* organization (`The-Artist-Evolution`) than every other tool above (`taemarketing`). Confirmed 2026-07-24 this is fine as-is — not an accident worth fixing, just worth knowing before assuming "the GitHub org" is one single place.

### Apex site — removed
Was a one-off test of building a site from a PSD design with Claude Code — never a real tool, never deployed anywhere. Confirmed no longer needed and moved to Trash on 2026-07-24 (not permanently deleted — still recoverable from Trash for a while if that changes). No longer listed above.

---

## Things worth knowing before changing anything

- **Two different GitHub homes exist:** most tools live under `github.com/taemarketing`; `tae-twin` lives under `github.com/The-Artist-Evolution`. If something's missing from where you expect, check the other one.
- **The Vercel connection available in Claude sessions can only see one project** (`tae-social-analyzer`) even though the others share the same Vercel team. It's a working connection, just scoped narrowly — so I can't check on tae-portal, tae-discovery, the Dono Dashboard, ACS Dashboard, or TAE Twin's live status through that tool the way I can for Social Analyzer. The Vercel dashboard itself always has the full picture.
- **Every tool now has a git repo, a GitHub backup, and auto-deploy set up.** (Apex site, previously the one exception, was a test project that's no longer needed and has been removed entirely — see above.)
- **The shared database** (used by TAE Dashboard, TAE Discovery, and Social Analyzer) has its own detailed writeup, including some security gaps that are confirmed but not yet fixed: [TAE Tools — Supabase Database Audit](https://claude.ai/code/artifact/ad0fc3af-4fa3-40c3-926b-efdfe7de5f34) and the follow-up findings doc in memory.

---

## Keeping this current

This is meant to stay accurate, not be a one-time snapshot. Worth re-checking after any deploy-setup change, a new tool gets added, or before a bigger reorganization — ask to have this file updated rather than starting the investigation over.
