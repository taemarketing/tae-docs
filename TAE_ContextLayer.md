<!-- title: TAE ContextLayer -->

# TAE ContextLayer

*The Artist Evolution — updated August 13, 2026. Originally written June 2026; the strategic thesis below is unchanged, but Section 14 (Roadmap) has been updated to reflect real progress, and a status note has been added right after the executive summary.*

---

## The Ninety-Second Version — Executive Summary

ContextLayer is the infrastructure that encodes TAE's client knowledge — goals, audience, voice, positioning, history — into the AI, so every action it takes is informed by strategy rather than generic. Execution-only AI tools are proliferating and will keep getting cheaper. What they can't replicate is judgment — and judgment comes from a relationship that generic tools structurally cannot have. That gap is TAE's durable advantage.

**What it is, in four lines:**
- A retrieval-backed knowledge layer that loads the right client context into the AI at the right moment.
- A client-facing interface where one prompt edits the site, answers a data question, or drafts a deliverable.
- An operations layer where the AI monitors, flags, and drafts across TAE's whole roster — with human approval.
- A product TAE can sell in tiers, and eventually white-label to other agencies.

**How to read this document:** It's honest about what exists versus what is a build. Capabilities in Section 08 carry a build-status tag — Live Today, Near-Term, Requires Build, or Future Vision — so this reads as a roadmap, not a wishlist. Two sections are deliberately sober (Longevity: what happens if a vendor disappears; Security: what happens if someone tries to break in). The hard problems — data isolation, brand enforcement, cost at scale — are named, not hidden.

**The numbers, briefly:** Fixed infrastructure runs roughly $110–150/month for the whole agency regardless of client count. The real variable cost is AI tokens — on the order of $5–25/month for a typical active client, more for heavy users — controllable through model tiering, prompt caching, and per-tier prompt caps. The genuine scaling constraint is not infrastructure; it is TAE's human review capacity. Full model in Section 11.

---

## Status Update — July 24, 2026

Today's session closed out most of Phase 1 of the roadmap below (Section 14), plus a running start on part of Phase 2. Concretely, in the order the original roadmap listed them:

- ✅ **"All TAE repos on GitHub"** — done, and exceeded. The original Phase 1 named four repos (tae-portal, tae-discovery, tae-twin, tae-social-analyzer). As of today all seven live tools are on GitHub under `taemarketing`: those four plus F.R.I. Marketing, the Dono Dashboard, and ACS Dashboard — the three that were still manual-deploy-only got fixed today.
- ✅ **"Vercel connected to GitHub for auto-deploy on every push"** — same scope expansion: now true for all seven, not just the original four.
- ✅ **"TAE admin dashboard fully operational with client management"** — unchanged, already true.
- ✅ **"Discovery system live and generating leads"** — unchanged, already true.

That's Phase 1 complete. On Phase 2, one bullet — *"Establish the security foundation — scoped credentials, secrets store, per-client isolation"* — got a real head start, not from ContextLayer work directly but from an unrelated database audit that happened to land the groundwork:

- The shared Supabase database had row-level security completely disabled on 12 tables (not scoped-too-loosely — actually off, full public read/write). That's fixed — RLS is on everywhere it should be, verified against Supabase's own linter.
- Two admin database views and one admin function that bypassed access control entirely are locked down.
- The single flat database namespace finished moving to the `core` / `discovery` / `portal` structure this document's technical sections assume — all four phases done, including the highest-risk one: `companies`, `clients`, `partners`, `sessions`, and `discovery_links` (the tables touching the actual client login system) moved to `core` as one unit, verified against the real login query and both admin views before deploying. What's *not* done yet: each app still connects with credentials that can technically see every schema — the schemas exist and hold the right tables, but access isn't scoped by them yet. That's the natural next step toward this section's "scoped credentials" goal, not something ContextLayer work has touched directly.

None of this is the MCP server itself — Phase 2's actual centerpiece, "build the MCP server, the central hub connecting Claude to all TAE systems," has not been started. But the ground it would stand on is measurably more solid than it was this morning.

---

## Status Update — July 30, 2026

Phase 2's centerpiece is no longer unstarted: the MCP server itself now exists, deployed, with one real tool live.

- ✅ **New repo, `taemarketing/tae-contextlayer`** — private, same GitHub account as six of TAE's seven other tools, same git-push-to-deploy pattern.
- ✅ **First tool shipped: `get_client_context(company_name)`** — a hosted MCP tool (Next.js + `mcp-handler`, Streamable HTTP transport, stable v1 SDK line) that pulls a client's core company/contact record, latest Discovery interview intelligence (MIS scores, recommendations, insights), and recent Portal activity (tickets, reports) into one bundle in a single call — reaching across the `core`, `discovery`, and `portal` schemas live. Verified end-to-end against real data (tested against ACS's actual tickets and reports) using the official MCP Inspector.
- ✅ **Access model resolved deliberately broad, not narrow.** The server connects with the Supabase service-role key — the same full-access trust tier `tae-portal` already uses — because ContextLayer's job is cross-system analysis and goal-driven action, not a siloed per-client view. This was a direct course-correction mid-build: an earlier draft of this work tried to scope the server down to a single new table out of over-cautious habit, and that was the wrong call for what ContextLayer is actually for.
- 🟡 **Auth is a static bearer token for now** — good enough gating for a first version; full OAuth (per-client login) is real future work, not needed yet.
- **One real finding surfaced along the way, worth flagging on its own:** none of today's Discovery sessions are actually linked to a `core.companies` row via `clients.company_id` — every discovery prospect's `client_id` currently points to a client record with `company_id` still null. So `get_client_context`'s discovery lookup is wired correctly but will return empty for every real company today, until that linkage is either backfilled or created going forward. Not a ContextLayer bug — a pre-existing gap in how Discovery and the Portal connect.
- ✅ **Second tool shipped: `mark_as_client(company_name, contact_email?)`** — the fix for that finding, and the deliberate answer to "how do we know an interviewee became a client." TAE does not auto-detect conversions (a Basecamp project gets created per Discovery session automatically via a TAE Dashboard button, so project existence isn't a reliable signal, and conversion volume is low enough that automation isn't needed anyway). Instead this is a manual, one-time, explicitly-invoked action: it flips the company's `relationship_stage` to `active` and, if a contact email is given, links that person's full Discovery session history to the company by setting their `clients.company_id` — the two things that have to happen together for a client's lifetime to be traceable back to their original interview(s). Verified against a real conversion: Harvey Vogel.
- ✅ **Third tool shipped: `log_relationship_event(company_name, event_type, service?, reason?, note?, recorded_by)`** — the ongoing lifecycle-logging counterpart to `mark_as_client`. Appends to a new `contextlayer.relationship_events` table (never overwrites — a company's story is a sequence of events, not one status field): `engagement_started` / `renewed` / `upsold` / `paused` / `churned`, each with an optional reason (budget/results/fit/pricing/competitor/other) and always a `recorded_by`. `mark_as_client` now also logs the matching `converted` event on first activation. A second new table, `contextlayer.client_engagements`, tracks what service a company actually bought — distinct from what Discovery recommended — the pairing this whole outcome-tracking design exists for (see the 5–10 year discussion below capability 08). Both tables live in a new `contextlayer` schema, owned by this repo, exactly as the Supabase audit anticipated.
- ✅ **A home for every human-gated action: the ContextLayer section.** Originally a standalone `/admin/contextlayer` page reachable via a footer link; moved same day into an inline collapsible section on the main TAE Dashboard itself, positioned under Notifications — same header-with-chevron pattern as Analytics/Partners/Notifications, not a separate page anymore. Standing rule going forward: any ContextLayer capability that needs a person to pull the trigger gets a control here, not a scattered one-off UI. Calls the real deployed MCP server directly (via `@modelcontextprotocol/sdk`'s client, server-to-server) rather than reimplementing the logic in tae-portal — a dashboard click and a Claude-session prompt run the identical code path. Verified end-to-end in the browser, twice — once as the original standalone page, again after moving it inline, both against real data.
- ✅ **Vercel deployment done, verified live.** `tae-contextlayer` is deployed at `https://tae-contextlayer.vercel.app`, auto-deploying on push (confirmed running the latest commit at deploy time). Tested directly against production — all seven tools list correctly and `get_client_context` pulls real Supabase data through the live server. `tae-portal`'s **local** `.env.local` now points `CONTEXTLAYER_MCP_URL` at the real URL instead of `localhost:3457`, and the full chain (tae-portal → deployed tae-contextlayer → Supabase) was verified end-to-end locally.
- ✅ **RESOLVED (noticed 2026-08-04, exact fix date unrecorded) — tae-portal's production deployment does have the ContextLayer env vars.** Previously logged here as still open. Confirmed working, not just assumed: this session ran two dozen-plus live `curl` calls straight against `https://tae-portal-lilac.vercel.app/api/admin/contextlayer/add-cadence-task` and `.../cadence-tasks` in production, over multiple separate work sessions, every one of them succeeding with real Supabase data (44 real cadence-task rows as of this check) — that only works if `CONTEXTLAYER_MCP_URL` and `CONTEXTLAYER_BEARER_TOKEN` are actually set on the live deployment, not just locally. Whoever set them (or whatever redeploy picked them up) did so without this doc getting updated — worth remembering that "still open" items here need to be re-checked before being trusted, not just assumed still true.
- ✅ **RESOLVED 2026-08-01 — the `contextlayer` schema is exposed, and the actual root cause was found.** It was never a Supabase platform incident and never anything wrong in our migrations or grants (both were independently confirmed correct along the way). The real problem: Supabase's newer Settings → Integrations → Data API panel is disconnected from the actual live setting — its "Exposed schemas" checkboxes don't reflect or write to reality. Proof: querying the database directly (`select rolconfig from pg_roles where rolname = 'authenticator'`, via `supabase db query --linked`, a direct Postgres connection independent of that dashboard) showed `pgrst.db_schemas = public, graphql_public, discovery, portal, core` — `contextlayer` genuinely absent, and (retroactively explaining an earlier mystery) `core`/`discovery`/`portal` were never actually "checked" in that panel either, despite always working, because the panel was never a real window into this setting at all. Fixed directly with `ALTER ROLE authenticator SET pgrst.db_schemas = '...contextlayer'` plus `NOTIFY pgrst, 'reload config'` and a second `NOTIFY pgrst, 'reload schema'` (PostgREST caches schema recognition and table introspection separately — the first notify alone got a more specific "table not found" error, not full success). Verified with a real write through the live production server (`assign_account_owner` on Harvey Vogel) and a read-back via `get_client_context`, plus confirmed `core`/`discovery`/`portal` still work exactly as before. All five `contextlayer` tables and all seven tools are now fully live. Lesson for next time a Supabase dashboard setting doesn't seem to take: check the actual Postgres-level state directly before assuming it's a platform incident.
- ⬜ **Autopilot constraint, standing as of 2026-07-30:** nothing in any TAE tool gets a scheduled job or self-triggered call against the Anthropic API until TAE is fully launched and out of beta. Schema and manually-invoked MCP tools (everything shipped so far) don't touch the Anthropic API at all and are unaffected by this — it specifically blocks any future *automated* analysis (e.g. the self-improvement loop actually reviewing outcomes and rewriting Discovery's own rules on a cron). That stays a human-triggered, on-demand capability for now.
- ✅ **Fourth tool shipped: `mine_discovery_sessions(group_by, since?, until?, industry?, service?)`** — the descriptive-mining piece. One flexible tool rather than five narrow ones: buckets completed sessions by industry, service, month, year, entry mode, or referral partner, returning count, average MIS scores, average budget signal, urgency breakdown, and conversion rate per group — every average carries its sample size so a thin group never reads as more confident than it is. Verified against the real 15 completed sessions across all six groupings plus filters.
- ✅ **Partner-attribution bug found and fixed, same day.** Initial testing showed every session as "Direct (no partner)," which read like a Discovery data-capture gap — it wasn't. `sessions.partner_id` is a dead column (zero references anywhere in either `tae-discovery` or `tae-portal`); the real, actively-used path is `sessions.link_id → core.discovery_links.partner_slug → core.partners` (by slug, not id). Fixed in the mining tool and re-verified: Omni Strategy Group now correctly attributed for its 2 real referred sessions. Worth remembering: nothing about Discovery's own capture needed to change — this was entirely a bug in reading the wrong field.
- ✅ **Missing service labels added to the Discovery admin page** — `SERVICE_LABELS` in `tae-portal`'s `/admin/discovery` only covered 7 of the ~16 real service values in use (found via the mining tool); the missing 9 (`seo`, `campaign_strategy`, `ppc_google_ads`, etc.) now display properly instead of falling back to the raw key.
- ✅ **Industry-taxonomy mechanism shipped: `contextlayer.industry_aliases` + `add_industry_alias`.** Deliberately open-ended, not a fixed category list, per direction: `mine_discovery_sessions`' industry grouping resolves each session's raw industry text through this table if a mapping exists, falling back to the raw text otherwise — so a new industry nobody's seen before just stands alone as its own group until a person notices a real duplicate and folds it in. A control for this lives in the ContextLayer section alongside the other tools. Honest note from checking the real data before seeding anything: today's 13 distinct industry values have no actual duplicates yet (the two repeat-count pairs are identical text from the same company's repeat sessions) — so nothing was pre-mapped; the mechanism is built and ready for when real duplication actually shows up.
- ✅ **The living strategic brief itself shipped: `contextlayer.brief_entries` + `add_brief_entry`.** This is Section 02's actual encoded knowledge layer — Goals, Audience, Voice & Tone, Positioning, History, Strategic Priorities — as a real, queryable thing for the first time, not just a description in this doc. Append-only, same discipline as everything else here: nothing is edited or deleted. An entry can optionally supersede an earlier one (goals/priorities/positioning genuinely go stale over time; history/audience/voice entries mostly just accumulate). `get_client_context` now surfaces the current — non-superseded — brief as part of its bundle. Deliberately starts **empty** for every company: not backfilled from Discovery's `team_brief`, which is itself sometimes-inferred text (see the Tequila Trujillo `brief_audit` example earlier in this doc) rather than a confirmed fact. A control lives in the Console alongside the other three tools, with an optional field for the entry id being replaced. **A real regression was caught in testing, not shipped:** the first version of the `get_client_context` change let an unhandled error from the brief lookup break the *entire* tool response — fixed to degrade gracefully (an error note inside the bundle) before this went out.
- ✅ **Discovery's "never recite known information" hard rule shipped.** The existing "don't reference prior sessions" language only lived inside the conditional `seed_context` text generated for returning clients — meaning it only applied when a person had prior history *and* the system remembered to include it. Added an always-on version to `tae-discovery`'s main interview system prompt (`lib/prompts/system.ts`) Hard Rules section: never recite, reference, or reveal anything already known about the prospect — from seed context, prior sessions, or any other source — unless they state it themselves in that conversation. Applies to every interview now, not just returning-client sessions. Verified the prompt compiles and the app serves cleanly; a full live multi-turn behavioral test (actually running a seeded returning-client interview and confirming the model adheres) was not performed — lower priority than the prompt change itself given how low-risk a text addition to an existing rules section is.
- ✅ **Sixth tool shipped: `assign_account_owner(company_name, owner_name, recorded_by)`.** Which TAE employee owns each client relationship — the prerequisite gap-detection needs before it can route a question to the right person. New `contextlayer.account_owners` table, same append-only-history discipline as `relationship_events`: assigning a new owner ends the prior assignment (`ended_at`) rather than deleting or overwriting it, so ownership history stays on record. `get_client_context` now surfaces the current owner. Fifth Console form added. No employee directory exists, so this stays fully manual — a person just states who owns it.
- ⬜ **Still not started, unchanged:** per-app Postgres-role isolation for the three existing apps (tracked separately in the Supabase audit), a data-completeness/gap-detection capability for the living brief (flagging missing categories per client, now buildable given account ownership exists — plus, as a separate, costlier, manual-only step once out of beta, Claude-assisted "do we have this somewhere and just haven't recorded it" analysis, surfaced via a Console card with an inline answer box; this is capability 12 below getting concrete), and connecting GitHub/Basecamp/analytics as further tools.

### Status Update — July 31, 2026

- ✅ **The ContextLayer task-cadence registry shipped** — the item left "still not started" above. Two new tables (`contextlayer.cl_tasks`, `contextlayer.cl_task_runs`) and three new tools: `get_cadence_tasks` (read, returns each task with its 5 most recent runs), `add_cadence_task`, `log_cadence_task_run`. A task carries a cost-estimate *range* (low/high), not a single figure, matching the budget-signal pattern already used elsewhere in TAE's tools — actual token cost varies run to run. Each run is a receipt: who did it, when, what it actually cost, optional notes. Same append-only discipline as `relationship_events` and every other ContextLayer ledger: nothing is edited or deleted, a task's history is a sequence of receipts. Seeded with the first known task: Discovery System Self-Improvement Loop, quarterly, $5–$25 estimated. Console UI is a three-column board (Monthly / Quarterly / Yearly) inside the ContextLayer section, each column showing its running estimated-cost total so the collective cost of an entire cadence is visible before anything gets automated — the explicit design goal, stated up front: this registry is the on-ramp for the day these tasks connect to Claude Developer Console and run themselves, and knowing the cost envelope before that switch flips matters more once the list is too long to eyeball. `run_type` on a receipt defaults to `manual`; `automated` is reserved for that day and stays unused until the standing autopilot constraint (no scheduled/self-triggered API calls until TAE is out of beta) lifts.
- ✅ **A real mid-build course-correction, worth recording because it's a mistake, not just a decision.** The first version of this feature put its two tables in tae-portal's own `portal` schema with plain REST access — the wrong owner. Caught by reading `TAE_ContextLayer.md` itself (this document) before shipping the UI: every other human-gated ContextLayer action lives in the `contextlayer` schema and is reached exclusively through the deployed MCP server via `callContextLayerTool`, never a direct Supabase call from tae-portal. Corrected: the portal-schema tables were dropped, the real tables were recreated in `contextlayer` schema inside the `tae-contextlayer` repo (where they belong), and tae-portal's UI now calls the three tools above the same way it calls `mark_as_client` or `assign_account_owner`. Full technical writeup — including a second, unrelated finding this surfaced (a cross-repo Supabase migration-numbering collision, twice) — lives in `TAE_Supabase_Database_Audit.md`.

### Status Update — August 4, 2026

- ✅ **Cadence registry expanded from 3 buckets to 6, plus reframed as a living brainstorm, not a finished feature.** Added `daily`, `weekly`, and a `future` bucket (for ideas that are inherently event-driven — e.g. noticing a client posted "closed" on Facebook and propagating that everywhere it's needed — and can't usefully be periodic no matter how tight the schedule) alongside the original monthly/quarterly/yearly. The Console section's divider was renamed from "Recurring · cadence tasks" to **"Developing Task List"**, and its description now says outright that this is an internal checklist for deciding what's worth paying for and how often, and what would actually have to get built to make each idea real — not a shipped list of what ContextLayer already does. The standing "nothing here runs automatically yet" note got folded into that same description instead of sitting alone above it.
- ✅ **Every cadence task's description now carries an explicit "Build requirement" line** — either *buildable now* (with why: uses an existing MCP tool, is a pure manual checklist, needs no new data) or *blocked* (with exactly what's missing: a `last_login_at` field that doesn't exist yet, a review-platform connection, an uptime pinger, analytics data, a per-client competitor set). Several blocked ideas are noted as needing to live on a future **client/partner dashboard** rather than TAE's internal one, since they're inherently per-client (reviews, social, competitor tracking) rather than roster-wide.
- ✅ **One-off human-gated actions (Mark As Client, Log Relationship Event, Add Industry Alias, Add Brief Entry, Assign Account Owner) are now a single collapsible group**, same chevron pattern as the cadence buckets, instead of always being expanded under their own divider — the divider is gone; the group collapses as one unit.
- ✅ **19 new candidate tasks added across daily/weekly/future**, brainstormed from Section 08's 12 capabilities and "Expanding Possibility Space" list below, explicitly aimed at "what would a business owner love to see": daily lead digests, uptime pulses, new-review alerts, ticket-aging flags, traffic-anomaly checks, and social-mention checks; weekly business-pulse reports, posting confirmations, competitor pulses, ticket/response-time reports, budget-signal checks, and top-performer call-outs. **Deliverable & Deadline Tracker moved from monthly to weekly** — a deadline tracker checked once a month catches almost nothing before it's already late.
- ⚠️ **Real mistake, caught the same session, worth recording so it doesn't repeat:** adding `daily`/`weekly` to the Postgres check constraint on `cl_tasks.cadence` is only half the change. The `add_cadence_task` and `get_cadence_tasks` MCP tool definitions in `tae-contextlayer`'s route handler each hard-code their own Zod enum (`z.enum([...])`) for the `cadence` field, completely independent of the database constraint — so the first attempt to add a daily task failed at the *tool's own input-validation layer*, before the request ever reached Supabase. Both had to be updated and redeployed. **Lesson: any time a cadence/enum-like value gets a new option, the DB constraint and every MCP tool's Zod schema that mentions it are two separate edits, not one — grep for the old enum's literal values across the route file before considering the change done.**
- ✅ **Cadence list edited down after the brainstorm, same day.** Removed two daily tasks that didn't earn their place (New Lead & Inquiry Digest, Urgent Ticket Aging Flag). Added GEO — Generative Engine Optimization, how well a site gets found and cited by AI answer engines, not just traditional search — to the monthly Site Health & SEO Hygiene Check's scope. Moved Ticket & Response-Time Report from weekly to monthly and renamed it (dropped the "Weekly" prefix) since a weekly cadence was overkill for what it actually checks. Scrubbed "business owner" phrasing out of every task description per direct feedback — task copy now describes the check itself, not who'd supposedly love seeing it.
- ✅ **The Reports pill — ContextLayer reports now show up somewhere a person will actually see them.** New `contextlayer.cl_reports` table (append-only, same discipline as every other ContextLayer ledger) plus `get_cl_reports`/`add_cl_report` MCP tools. `tae-portal`'s admin feed gained a "Reports" pill alongside Support Tickets/Discovery System/OSG Discovery/Social Analyzer, merging in real `cl_reports` rows. A report with a `source_url` opens it in a new tab on click; without one, the card correctly shows no arrow and isn't clickable rather than implying a dead link works. Also added to the notification-category options (Reports is a category like any other, though — matching every other category today — nothing actually dispatches a notification for it yet; subscriptions are stored, not acted on).
- ✅ **Analytics section removed from the TAE Dashboard, to revisit later.** The component (`ClientActivityChart`) and its data-fetching are left in place, just unrendered — a comment marks why, so bringing it back isn't a rebuild.
- ✅ **New TAE Dashboard "Reports" section — the same drag-and-drop report builder used on client/partner dashboards, now usable for TAE's own reports.** Rather than duplicating the ~800-line CSV-parsing/Claude-chart-generation pipeline, a report created here is simply a `portal.report_tabs` row with both `company_id` and `partner_slug` null — "internal" — which the schema already allowed (both columns were already nullable; the API layer just never had a code path that didn't require one or the other). `/api/admin/reports` (GET and POST) now accepts `internal=true` to read/write that null/null slice. The existing per-client chart editor page is reused completely unmodified for internal reports via a reserved sentinel slug (`_tae-internal`) in its URL — that page already resolves everything by `reportId` alone, `slug` was only ever used for the "back" button's link, which now branches on the sentinel. Internal reports created this way also feed the Reports pill above, alongside `cl_reports` — same pill, same badge, sorted together by date. See "The Reporting Network" below for how this and `cl_reports` relate, and how any future ContextLayer task could target either.
- ⚠️ **Honest verification gap.** The new Reports section's create-and-generate flow (drag a CSV in, Claude builds a report) was never actually click-tested in a real browser, because doing so requires a logged-in TAE admin session (`tae_session` cookie), which this working session doesn't have — the admin dashboard *looks* reachable without one only because most of its reads go straight to Supabase client-side under permissive RLS, bypassing the app's own auth entirely. What *was* verified directly: `tsc` and a full `next build` both pass clean, and the underlying database assumption (`report_tabs` accepting a null/null row, and the exact query filter used by `internal=true`) was confirmed by inserting and querying a real row via the service-role key, then removing it. The create-CSV-upload-generate-redirect flow itself is unverified end-to-end — worth an actual admin doing one real test before relying on it.

### Status Update — August 13, 2026

- ✅ **Cadence buckets replaced entirely — time-based grouping gone, topical categories in.** Between August 4 and today the registry briefly grew to 8 time-based buckets (`holiday` and `sales_prospects` were added, each with its own starter tasks, following the same DB-constraint-plus-Zod-enum two-place pattern as before). Then, on reflection, all of it — daily/weekly/monthly/quarterly/yearly/holiday/sales_prospects/future — was replaced with **5 topical categories**, the fewest that still group every real task by what it's fundamentally about: `brand_web_presence`, `marketing_performance`, `client_sales_relationships`, `finance_cost`, `internal_ops_quality`. All 55 existing tasks were individually remapped (not a mechanical find-and-replace — each task got read and assigned by hand), via a two-migration drop-then-restore-constraint sequence so the remap could run as application-level `PATCH` calls rather than raw SQL. The `cadence` column/field name was kept as-is — renaming it would've touched every call site for no functional gain — but every MCP tool description now explains it holds a category, not a frequency.
- ✅ **Anonymous feedback shipped: thumbs up/down plus a suggested-frequency multiple-choice, per vote, on every task.** New `contextlayer.cl_task_votes` table and `add_task_vote` MCP tool. Explicitly anonymous by direct instruction — `voter_name` is nullable, no uniqueness constraint on `(task_id, voter_name)`, and nothing stops someone voting twice: this is internal reference for prioritization, not a real voting system, and multiple people share one TAE admin login anyway so session identity was never going to be a reliable signal. The **Suggest a New Task** box calls `add_cadence_task` directly — no approval/pending step, per direct instruction — and was moved twice on request: first to the bottom of the cadence section (below the 5 category buckets), then to the very bottom of the whole ContextLayer panel (below the One-off actions card).
- ✅ **F.R.I. Marketing is now a real client — the first live test case for the Website module.** Created in `core.companies` (`relationship_stage: active`, `domain: frimarketingllc.com`). New `website` value added to `portal.client_features.feature_key` (this repo's first migration in its reserved `100–199` block). A Website module card now appears in Manage Client's toggle grid and, when enabled, on both the client-facing dashboard and the internal Manage Client page — each links out to the client's real domain.
- ✅ **`get_site_analytics` — ContextLayer's first tool with a real external data source, not just internal Supabase data.** New `contextlayer.site_analytics_sources` table maps a company to its Vercel project (F.R.I. Marketing → `prj_c29DfP5wQwqcPTd9CSCHDi4ikGEl` is the only row so far). The tool looks up a company, finds its Vercel project through that table, and queries Vercel's real, officially-documented Web Analytics REST API (`GET /v1/query/web-analytics/visits/aggregate`) using a `VERCEL_API_TOKEN` env var the user set directly in Vercel (never handled or typed by the assistant — API tokens are handled by the person who holds them, same as every other credential in this project). Verified end-to-end against production, not just unit-tested: real numbers (9 visitors, 12 pageviews over 7 days, matching the site's actual first days live) flowed all the way from Vercel's API through the deployed MCP tool, through a `tae-portal` proxy route, into both the client dashboard's Website section and Manage Client's Website card, replacing what had been an honest "coming soon" placeholder.
- ✅ **Two proxy routes, two different trust levels, on purpose.** `/api/admin/contextlayer/site-analytics` takes an arbitrary `company_name` param — fine, because it's only reachable from the admin dashboard. `/api/portal/site-analytics` takes no params at all; it resolves the company from the caller's own session (`resolvePortalCompanyId`), the same pattern `/api/portal/reports` already used — so a client can only ever see their own site's analytics, never another client's, no matter what they pass.
- ✅ **Six cadence task descriptions updated now that the blocker they named is actually gone.** Underperforming Page/Content Audit, Daily Traffic Anomaly Check, Weekly Business Pulse Report, Top Performer of the Week, and Post-Holiday Performance Recap moved from *Blocked — needs analytics/traffic data connected* to *Buildable now*, each naming `get_site_analytics` as the real data source and noting the real remaining constraint: it only works for a company with a `site_analytics_sources` row, which today means only F.R.I. Marketing. Search Ranking & Traffic Correlation Review moved to *Partially buildable* — the traffic half is real now, but search-ranking data is a genuinely different source (Google Search Console, not Vercel) and still isn't connected.
- ⚠️ **A migration-tooling friction, not a collision — worth its own record since it's easy to mistake for the problem the reserved blocks already solved.** `supabase db push` refuses outright (not just warns) when the remote ledger has any version its local repo doesn't recognize, even one safely inside another repo's own reserved block. Hit twice today, in both directions — tae-portal's first `100_tp_...` push failed on tae-contextlayer's real `022`/`300`–`307`; tae-contextlayer's next push then failed on tae-portal's real `100`. Fixed the same way each time: a no-op placeholder file in the *other* repo's migrations folder for each unrecognized version, never touching the real migration. Full account in `TAE_Supabase_Database_Audit.md`, finding 5's sixth update.
- ✅ **Six more read-only Website-section tools shipped, one piece at a time by direct instruction — each built, deployed, and verified against F.R.I.'s real production site before moving to the next.** All reuse `site_analytics_sources`'s Vercel mapping, except uptime which only needs `core.companies.domain`:
  - `get_site_deployments` — recent Vercel deployments (state, commit, rollback-candidacy) via Vercel's Deployments API. Rollback itself deliberately not built — read-only only, per direct instruction to hold off on any write/action tool at that point.
  - `get_dependency_report` — pulls `package.json` from the latest deployment via Vercel's Deployment Files API, checks each dependency against the public npm registry. Degrades honestly for sites with no `package.json` (F.R.I.'s static `index.html` site, for one) rather than erroring opaquely.
  - `get_site_uptime` — a live HTTP request to the stored domain right now, not a historical monitor. The one tool in this batch that needs no Vercel mapping at all — works for any company with a domain.
  - `get_broken_links` — fetches the homepage, extracts every `<a href>` on it, checks each one's status. Deliberately shallow (homepage only, not a full crawl) and distinguishes internal from external links.
  - `get_sitemap_robots_status` — checks `robots.txt` and `sitemap.xml` existence, whether robots.txt declares the sitemap, whether it's accidentally blocking the whole site.
  - `get_dns_domain_status` — DNS records (A/AAAA/CNAME/MX/TXT/NS) via public DNS-over-HTTPS, plus Vercel's own domain-verification status when a project is mapped. Real finding surfaced immediately: F.R.I.'s DNS is still managed through Wix's nameservers even though the A record correctly points at Vercel — not broken, but a fact worth knowing.
  - `get_env_vars_redacted` — lists Vercel env var keys/type/target/`updatedAt` only. **Never** requests or forwards the actual `value` field, even for `type: "plain"` variables where Vercel's own list API would otherwise return the plaintext inline — redaction enforced at the tool layer, not the UI layer.
- ⏭️ **Cache purge investigated and explicitly skipped, not built.** No documented public Vercel REST API exists for purging the Data Cache or CDN cache — dashboard-only. Worse: on Hobby/Pro plans (TAE is Pro), purging the Data Cache is **team-wide**, not per-project — a "purge this client's cache" button would silently nuke every other TAE-hosted client's cache in the same team. The real, safely-scoped mechanism (`revalidateTag`/`revalidatePath` via a `/api/revalidate` route) is a code change to each client's own site, not a ContextLayer capability — noted as future per-site work, not built today.
- ✅ **Lightweight, admin-only Content management shipped: `get_site_content` / `update_site_content`.** Explicit scope from direct instruction: TAE Manage Client page only, no client-facing editing, not a generic HTML editor or WYSIWYG CMS. `site_analytics_sources` grew `github_owner`/`github_repo` columns (migration 309) rather than a new table — a site's Vercel project and GitHub repo are 1:1 in practice, same concern `get_site_analytics` already tracked. A curated field list (7 fields for F.R.I.: hero badge/headline/subtext, CTA headline/subtext, contact line, contact email) extracted via regexes anchored on stable HTML structure/class names, not on current text content, so editing one field's value doesn't break the next field's lookup. Reads/writes go straight through GitHub's Contents API to the site's real `index.html`; Vercel's existing auto-deploy-on-push does the rest — no new deploy mechanism invented. Verified end-to-end with a real (harmless, no-op) commit that round-tripped through the whole stack: `tae-portal` UI → proxy route → MCP tool → GitHub API → real commit `4c9eca5` on `taemarketing/fri-marketing`'s `main` branch. New `GITHUB_API_TOKEN` env var follows the same pattern `VERCEL_API_TOKEN` set: a fine-grained PAT, `Contents: Read and write` only, scoped to `fri-marketing` specifically today — onboarding a future client's repo means adding it to this same token's repository list, not minting a new token, a deliberate choice over a classic org-wide PAT to keep the blast radius smaller if it ever leaks. Content list UI made collapsible immediately after shipping — 7 fields at once was too much vertical space alongside the Website section's other widgets.
- ⏸️ **Content management UI parked the same day, not deleted — a real, working feature shelved on UX grounds, not a technical failure.** The raw-textarea field list worked end-to-end (see the verified commit above) but wasn't the right interaction: too much vertical space, and a preference for seeing the change rather than editing blind text. Two next-step directions were scoped but deliberately not built yet: (1) a **true in-page click-to-edit** flow — a short-lived signed token opens the live site itself with an embedded editor script, elements tagged `data-tae-field="..."` become directly editable, saves route through the same `update_site_content` path; real cost is that it requires injecting persistent editor code into the client's own deployed site plus a public token-gated save endpoint, since the whole point is editing on `frimarketingllc.com` itself, not a TAE-hosted page. (2) A **middle ground**: keep editing inside Manage Client, but add a toggleable full-screen/bottom-sheet preview (mobile-first — a sheet, not a side-by-side split, since side-by-side has nowhere to go on a phone) that dry-run-renders the real HTML with draft edits swapped in via the same field-matching logic `update_site_content` uses, minus the GitHub commit — so typing shows a real preview before saving anything. Neither built yet. UI gated behind `CONTENT_MANAGEMENT_UI_ENABLED = false` in `tae-portal/app/admin/clients/[slug]/page.tsx` — the tools, the GitHub token, and the field map are all still live; only the Manage Client rendering and its data fetch are switched off, so turning it back on later is a one-line flip once a direction is chosen.

### Status Update — later August 13, 2026: branding, holiday themes, and the field-map scaling fix

- ✅ **The field map moved from hardcoded code to data — the long-term fix a second client's site would have needed anyway.** `contextlayer.site_content_fields` (migration 310) replaces the `SITE_CONTENT_FIELDS` object that used to live in `tae-contextlayer`'s route file: one row per company per field (`field_key`, `label`, `category`, a stored regex `pattern` source string, `group_index`). Onboarding a new client's editable fields is now a config insert, not a code deploy. F.R.I.'s existing 7 content fields were migrated in as data; category (`content` vs `brand`) is new, so a future UI can group them.
- ✅ **Branding fields added for F.R.I.: `brand_accent_color` / `brand_accent_light`, extracted from the site's own `--orange`/`--orange-light` CSS custom properties.** Because F.R.I.'s color system already used CSS variables, brand editing needed no new mechanism at all — same `update_site_content` path, just two more field rows. Verified live: `get_site_content` correctly returns the real hex values (`#F97316`/`#FB923C`) pulled straight from the deployed CSS.
- ✅ **Append-only audit log for every content/brand write: `contextlayer.site_content_changes` (migration 311).** Same discipline as `relationship_events`/`cl_task_votes` — old value, new value, who, commit sha, timestamp, never edited or deleted. Does double duty: it's also the revert mechanism for holiday themes (below), since `old_value` is exactly what "put it back" needs — no hardcoded default to drift out of sync with whatever the real branding happens to be at revert time.
- ⏸️ **First holiday-theme design shipped, then superseded the same day — worth recording both, not just the final state.** `contextlayer.holiday_themes` (migration 312) originally seeded the top 5 retail-calendar holidays as `field_key → value` overrides (accent colors + a themed hero badge), with `get_holiday_themes`/`apply_holiday_theme`/`revert_holiday_theme` built on top of the same field-substitution mechanism content editing used. This worked (verified live, including two real bugs found and fixed via an actual apply→revert cycle — see below) but was the wrong shape for the real ask: swapping a handful of field values can never give a different header, different copy throughout, or popups — only a full alternate page can. Replaced same-day (migration 313) with the file-based system described next; `holiday_themes` and its three tools were dropped entirely, not left running alongside the replacement.
- ✅ **Holiday pages, take two: real, complete HTML files per holiday, not field overrides — closes out the "Holiday Site Theming" cadence task.** `holidays/<key>.html` are full pages committed in the client's own repo, authored collaboratively (a person + Claude, in a session like this one) ahead of the holiday via `save_holiday_page`, with no limit on what can differ from the normal page — header, copy, popups, anything. `contextlayer.holiday_pages` (migration 313) is now pure metadata — whether a slot actually has a designed page is a fact about the repo (`list_holiday_pages` checks via GitHub), not a database flag. `apply_holiday_page` swaps a designed page in as `index.html`, snapshotting the previous whole page into `contextlayer.site_page_snapshots` first; `revert_holiday_page` restores it exactly. Same append-only discipline as everywhere else, just at file granularity instead of per-field. **Manual only, both directions**, same standing reason as before — no cron, apply and revert are one-click actions for a person to run the morning of and the day after. Preview is trivial with this design (no computation, `get_holiday_page` just reads the real file), shown via a full-screen iframe overlay on Manage Client — mobile-first, a sheet rather than a side-by-side split. **The slot list itself went through two corrections the same day** (migrations 314, 315): the original 5 were a retail-shopping guess (Halloween, Black Friday, Christmas, New Year, Valentine's); corrected first to the 6 actual US core business holidays (New Year's Day, Memorial Day, Independence Day, Labor Day, Thanksgiving Day, Christmas Day), then corrected again to keep both sets rather than choose — the final 9 slots cover the full calendar year: New Year's Day, Valentine's Day, Memorial Day, Independence Day, Labor Day, Halloween, Black Friday/Holiday Season, Thanksgiving Day, Christmas Day. Verified end-to-end against F.R.I.'s real repo: all 9 slots correctly report "not designed yet" (no `holidays/*.html` files exist), which is the honest state until an actual page gets designed.
- ⚠️ **The two bugs found during the (now-superseded) field-override version are still real lessons, not wasted work.** (1) Committing several fields back-to-back to the same file hit a genuine GitHub Contents API 409 (read-after-write replication lag between rapid sequential commits, not a logic error) — the retry-with-backoff fix carried forward into `applyContentFieldUpdate`, which `update_site_content` still uses for ordinary content/brand edits. (2) A too-coarse "already reverted" check (looking for *any* revert row tied to a batch instead of tracking per-field completion) doesn't apply to the new file-based revert at all — reverting a whole page is one commit, not several, so that failure mode can't recur by construction. Worth noting as a small point in the file-swap design's favor: fewer moving parts per operation means fewer of these races to guard against in the first place.
- ⚠️ **Two real bugs found and fixed via a live end-to-end test on F.R.I.'s actual production site — not caught until an apply-then-revert cycle was actually run, not just unit-reasoned about.** (1) Committing 3 fields back-to-back to the same `index.html` hit a genuine GitHub Contents API 409 ("sha does not match") on the 2nd/3rd field — read-after-write replication lag between rapid sequential commits, not a logic error. Fixed with a one-retry-with-backoff loop that re-fetches the file and recomputes the diff against the freshest content before retrying the commit. (2) The resulting partial revert (2 of 3 fields succeeded) then made `revert_holiday_theme` refuse to retry at all, because its "already reverted" check looked for *any* revert row tied to the batch instead of tracking which specific fields were still pending — fixed to check per-field, so re-running revert only touches what's actually left. Both fixes verified with a second full apply→revert cycle (Halloween theme) that came back clean, including a genuine no-op case: Halloween's palette happens to match F.R.I.'s actual brand orange, so those 2 color fields correctly skipped committing anything at all (no-op detected before ever calling GitHub), while the one field that actually changed (`hero_badge`) reverted exactly.
- ✅ **`get_site_content`/`update_site_content` descriptions and the `applyContentFieldUpdate` internals now shared by three tools, not one** — `update_site_content` (manual, single field), `apply_holiday_theme`, and `revert_holiday_theme` all route through the same field-resolution, commit, and change-logging code, so the 409 fix and the audit-log write only had to be written once.
- **What's still explicitly not built:** any actual designed holiday page (all 9 slots are real, working, and confirmed empty — the next step is a real design session per holiday, not more infrastructure), a real logo image upload (F.R.I.'s wordmark is text, not an image — swapping to a real logo file needs a new binary-asset-commit path, base64 through the same GitHub API, not yet built), and font-family swapping.

### Status Update — even later August 13, 2026: the data-attribute convention, closing the loop from "recommended" to "built and proven"

- ✅ **The generic-editor convention flagged as "recommended but not applied" earlier the same day is now actually built: `tae-docs/TAE_Site_Field_Convention.md`.** A new markup convention (`data-tae-field="<key>"` attributes on editable elements) plus a starter taxonomy of standard key names (brand colors/logo/font, hero fields, contact info including `contact_hours` — which is what the already-logged "Holiday Hours & Closure Sweep" cadence task actually needs to function, footer/CTA, an announcement banner pair for a future "propagate a closed-today post everywhere" capability). Explicitly a starting list, not a claim of completeness — extending it later is "tag one more element, re-run the scanner," not a migration.
- ✅ **`site_content_fields` grew a second extraction mode: `selector_type` (`'regex'` | `'data_attr'`, migration 316).** F.R.I.'s original 8 hand-written-regex fields are untouched and still work exactly as before — this is additive, not a forced migration. `data_attr` fields are found by a real balanced-tag scanner (`extractByDataAttr`), not a stored regex: it locates `<TAG data-tae-field="key">`, then counts nested opens/closes of that same tag name to find the true matching close, so content containing other markup (a `<span>` inside a tagged `<h1>`, for instance) still resolves correctly — with the one stated limitation that a tag nested inside *itself* (a tagged `<div>` wrapping another `<div>`) resolves to the inner close, not the outer one.
- ✅ **`scan_site_fields` shipped — onboarding a new client's fields is now "run one tool," for any site built with the convention's attributes in place.** Reads the live HTML, finds every tagged element, inserts one `site_content_fields` row per key not already registered. Labels come from the standard taxonomy where the key matches it, otherwise a humanized fallback. Safe to re-run.
- ⚠️ **A real, if narrow, bug found and fixed via a live test — not the 409/revert bugs from earlier, a new one.** `applyContentFieldUpdate`'s replacement step used `html.replace(fullMatch, ...)` — a plain *string* search, blind to position. A verification field with a zero-width regex lookahead had a `fullMatch` of just `"<span>"`, which also occurred earlier in the same document (a nav-logo span) — the string replace silently edited that occurrence instead of the one the regex had actually, correctly, uniquely matched. Caught immediately because the live check (fetching the real deployed HTML after the write) showed the wrong element tagged, not because anything errored. **Fixed properly, not patched around the specific trigger:** replacement is now always position-based (`spliceReplacement`, using the match's real index), for both the regex and `data_attr` paths. Existing production fields never hit this — their `fullMatch` blocks are long, distinctive tag+class combinations, naturally unique — but `data_attr` fields are shorter and more generic by nature, making this a real risk going forward, not just a one-off caused by an unusual test pattern. The live damage (one wrongly-tagged nav-logo span) was found, fixed, and verified restored before moving on — nothing was left broken to discover later.
- ✅ **Full round-trip verified against F.R.I.'s real production site, both extraction modes side by side.** Tagged the footer's copyright line (`data-tae-field="footer_copyright_text"`), ran `scan_site_fields` (found and registered it), confirmed `get_site_content` returns both the new `data_attr` field and all 8 original `regex` fields correctly in the same response, then round-tripped a real value change through `update_site_content` on the new field and reverted it — confirmed via a fresh fetch of the live deployed page each time, not assumed from a 200 response.

### The Reporting Network — how any ContextLayer task can produce a report, for any audience (added 2026-08-04)

Two different things both got called "reports" today, and they're deliberately not the same thing — this section is the map so a future task doesn't conflate them.

**`cl_reports` (in `contextlayer`) — the log.** A lightweight, append-only record that something was produced: a title, a one-or-two-sentence summary, an optional full-text body, and an optional pointer (`source_url`) to the real thing elsewhere, or a pointer (`source_task_id`) back to the cadence task that produced it. Cheap to write, meant for anything ContextLayer concludes or finds — a Self-Improvement Loop findings summary, a Full Roster Trend Review's key takeaway, a Partner Performance Review's verdict. It shows up in the admin feed's Reports pill the moment it's logged.

**`report_tabs` (in `portal`) — the deliverable.** The real thing: a visual, chart-based report built from real data (CSV upload or PDF/screenshot AI recreation, via `/api/admin/reports/generate` or `/api/admin/reports/from-pdf`), rendered through `report_charts`/`report_datasets`. One pipeline, three possible owners, same code:
- **Client-facing** — `company_id` set. What a client sees at `/portal/reports/[id]`.
- **Partner-facing** — `partner_slug` set. What a partner sees at `/partners/[slug]/reports/[id]`.
- **TAE-internal** — both null. Lives only in TAE's own admin feed and the Reports section on the TAE Dashboard, at `/admin/clients/_tae-internal/reports/[id]`.

**The two connect, they don't compete.** A `cl_reports` row's `source_url` can point straight at a `report_tab`'s preview URL — the lightweight log entry becomes the feed-visible pointer to the full visual deliverable. A task that just needs to state a conclusion writes a `cl_reports` row alone. A task that needs to hand someone an actual chart-based document builds a `report_tab` (targeted at whichever of the three audiences fits) and, if it's worth flagging in the feed on its own, logs a `cl_reports` row pointing at it.

**What this means for any current or future cadence task.** Every task in the Developing Task List (Section 08 above; the cadence registry itself) already has an unambiguous, three-way-routable path to a real deliverable, without inventing anything new:
- Route to **TAE** → `add_cl_report` for a conclusion, or a `report_tab` with `internal: true` for a document. (Full Roster Trend Review, Full Cost/Margin Review, the Self-Improvement Loop.)
- Route to a **partner** → a `report_tab` with `partner_slug` set. (Partner Performance Review.)
- Route to a **client** → a `report_tab` with `company_id` set. (Report Quality Spot-Check's counterpart, if it ever produces a document instead of just a review note.)

**What's not built yet.** None of this is wired automatically — every path above is a real, working, MCP-callable or API-callable route today, but nothing *calls* it from inside a cadence task run. A task still has to be manually run, and its output manually logged or built, exactly like everything else in this registry, per the standing autopilot constraint (no scheduled/self-triggered calls until TAE is out of beta). What today's work adds is that the destination is no longer an open question — when a task does get automated, or when a person runs it manually and wants to record the result, the three routes above are exactly where it goes, not a fourth thing to invent.

**Update, 2026-08-13 — the network's input side started filling in too, not just the output side.** Everything above assumes a task already knows what it wants to say; `get_site_analytics` (see this date's status update above) is the first real *data source* a task can pull from before it has anything to report — real Vercel Web Analytics for any client with a `site_analytics_sources` row. It doesn't produce a report on its own; it's the thing a task like Weekly Business Pulse Report or Top Performer of the Week would call first, then hand the result to one of the three routing paths above. Worth naming as its own category going forward: **sources** (data a task reads) are a different thing from **destinations** (where a task's output goes), and today's the first day both sides of that pair are real for the same task.

**Update, later 2026-08-13 — a third category, distinct from either: actions.** Sources read; destinations report; `update_site_content` (and, structurally, the same shape any future write-capable site tool would take) *changes the live thing itself* — it doesn't produce a report about the site, it edits the site. Deliberately scoped narrow and admin-only for now (see this date's Content management entry above), but worth naming precisely so a future task that wants to *do* something rather than *observe* or *report* something has a clear category to fall into, not a fourth ad-hoc pattern invented per tool.

### 5–10 year view — why outcome data is the real asset (added 2026-07-30)

Industry/service/date breakdowns of Discovery sessions are useful, but any competitor with an intake form could build the same descriptive dashboard. What's actually hard to replicate: pairing a session's *initial signal* (MIS scores, service-fit recommendation, budget signal, industry) with what *actually happened* — did they convert, what did they buy, did the recommended service work, did they renew or churn, what was their real lifetime value. That pairing is what lets TAE's own judgment get calibrated by reality instead of staying a plausible-sounding model guess every time — the ContextLayer thesis (judgment compounds, generic tools can't copy it) turned into rows in a table. `client_engagements` and `relationship_events` are the first piece of that: the outcome half of the pair. The signal half already exists in `discovery.*`. Once both sides have enough history, a person can manually ask Claude to review the gap between prediction and outcome — the seed of the self-improvement loop (capability 12) growing past "was this interview good" into "was this recommendation right," and eventually past Discovery into the rest of the TAE ecosystem. Secondary, sooner-to-pay-off angles worth remembering: industry-vertical playbooks (useful in the very next sales call in a familiar vertical, no outcome data required), partner/referral quality scoring, and pricing intelligence from aggregated budget signals.

### Design principle — consent before record (internal, added 2026-07-30)

Borrowed directly from *The Cue* (`the-cue-card.md`'s "Consent Before Record" section): expression belongs to the person who produced it until they explicitly choose to make it permanent record, not the moment a platform captures it. Applied here as a standing rule for every TAE tool, ContextLayer included: **commit a new asserted fact to Supabase only when a real person has explicitly confirmed it — never by silent inference or automatic detection.** `mark_as_client` already follows this by construction (a manual, explicitly-invoked action); any future tool that writes a derived conclusion back as permanent fact — an analytics categorization, or the presence-based capture in capability 12 below — needs the same explicit confirm-before-commit step. This is internal only for now, not client-facing, and explicitly does not change Discovery's own interview-capture flow — Person-to-AI conversation is already TAE's most-honest data source by its own separate honesty framework; the consent question is about what happens to that data *afterward*, once a tool turns it into a fact other systems rely on.

---

## 01 — The Problem With Generic AI

Every major website platform — GoDaddy, Wix, Squarespace, Webflow — has already shipped or is actively building an AI that can edit your site, draft copy, swap images, add pages. More are arriving every quarter. They will all work. They will all get better. And none of them will know what you are actually trying to accomplish.

That is the ceiling of execution-only AI. It asks what you want changed. It does not know why it matters. It has no relationship with you, no record of your decisions, no understanding of your audience, no opinion about whether what you just asked for actually serves your goals. It is a capable tool pointed at nothing in particular.

Execution without judgment is a commodity. It always becomes a race to zero — the cheapest, fastest tool wins, and eventually it is free.

### The Question That Changes Everything

That single shift — from execution-only to strategy-informed — is the difference between a commodity tool and a compounding advantage. It is the core of what ContextLayer is, and the rest of this document is how it gets built.

> Generic AI asks what you want. TAE's AI already knows why it matters — because we told it.

---

## 02 — The Relationship Is the Strategy

TAE builds knowledge of every client through the relationship. Not just the website — the whole picture. Who their audience actually is. What makes them different from every competitor in the market. What conversion looks like for them — a phone call, a form, a booked appointment. What they have tried before. What did not work and why. Where they are trying to go.

That knowledge comes from everywhere. Kickoff conversations. Brand assets and guidelines. Campaign results. Analytics — what is actually converting, not what looks good. Tickets and requests over time. Strategy sessions. Market research. The observations TAE makes from working inside a client's world.

Until now, that knowledge lived in TAE's heads, in documents, in scattered notes. Every time a new project started, someone had to re-learn the client. Every time a campaign was briefed, the context had to be reconstructed. Every time Claude was brought into a session, the background had to be explained from scratch.

### ContextLayer Changes That Permanently

ContextLayer takes everything TAE knows about a client and encodes it into a living strategic brief — a permanent, always-accessible document that Claude reads before acting on anything for that client. The brief is not a one-time setup. It grows with the relationship. Every new decision, every campaign result, every strategic shift gets added. The longer TAE works with a client, the richer the brief gets, and the smarter every AI action becomes.

- **Goals** — Specific, measurable outcomes — what the client is trying to move. Every AI action is evaluated against these.
- **Audience** — Who they are actually trying to reach and what that audience responds to. The AI writes for this client's specific person.
- **Voice and tone** — The words they use, the words they avoid, the register they inhabit. The AI stays in theirs.
- **Positioning** — What makes them different, what claims they own, and what they should never say because it sounds like everyone else.
- **History** — What has been tried, what worked, what failed. The AI does not repeat mistakes it already knows about.
- **Strategic priorities** — What TAE and the client have agreed to focus on right now — tactical requests are evaluated against it.

> The brief is not a document the client reads. It is the context the AI lives in. Every prompt a client submits, every change Claude makes, every recommendation it surfaces — all of it runs through what TAE already knows about them.

### What the Brief Unlocks for the Client

A client with a fully encoded brief has something no generic AI tool can offer: an on-demand creative department that already knows them. Proposals written in their voice. Content calendars built around what has worked. Performance data answered in plain language. The brief is TAE's expertise made permanent — the client is the beneficiary every time they use it.

---

## 03 — What ContextLayer Actually Is

ContextLayer is the infrastructure that makes TAE's client knowledge permanent, actionable, and AI-powered. It is not a feature, a plugin, or an add-on. It is the layer that sits between what TAE knows and what the AI does.

### Claude the App vs. Claude via ContextLayer

**Without ContextLayer — Claude starts knowing nothing.** Every session begins from scratch. You paste in background. You explain the client. You set up the context. The moment the session ends, it is gone. The next session starts over. Claude is capable but blind — it executes what you tell it right now, with no memory of what came before and no strategic direction to aim toward.

**With ContextLayer — Claude starts with the strategy in hand.** Every session opens with the relevant client picture retrieved automatically — goals, audience, voice, history, priorities. Claude does not need to be briefed and does not need context pasted in. It already knows why the work matters and what it is working toward. Every team member's Claude starts this way. Every time.

| Without ContextLayer | With ContextLayer |
|---|---|
| Claude knows nothing about the client | Claude knows the full strategic picture |
| You paste context in every session | Context loads automatically, always current |
| AI executes the request | AI executes the request toward the stated goal |
| Generic output that sounds like everyone | Output filtered through this client's voice and positioning |
| Strategy lives in someone's head | Strategy lives in the system — permanent and shared |
| One person's Claude, one session at a time | Every team member's Claude, all clients, always |

> ContextLayer is not a smarter AI. It is a strategically informed one. The intelligence was always there. ContextLayer gives it something to be intelligent toward.

### How It Actually Works — A Sober Note

Claude does not hold every client's history in memory at once — no AI model can. ContextLayer works by retrieval (RAG: retrieval-augmented generation): client knowledge lives in a structured store, and the system loads the relevant slices into context for each task. Not everything, every time — the right things, at the right moment. The architecture is well-understood and buildable. But quality of retrieval is quality of system: good retrieval feels like Claude knows everything; poor retrieval feels like Claude forgot. That is the core engineering challenge.

---

## 04 — The Technology Underneath — MCP Explained

The technology matters less than what it enables — but understanding it clarifies why ContextLayer is a persistent system and not just a well-organized prompt.

ContextLayer is built on the Model Context Protocol — MCP. An open standard published by Anthropic that defines how AI models connect to external systems. Think of it as a universal adapter. Any system that speaks MCP can connect to any AI model that speaks MCP.

What this means practically: the strategic brief does not exist as a document someone pastes into Claude. It exists as a live connection. Claude does not read a snapshot of what TAE knew last month — it reads the current state of every connected system in real time. When a client's goals change, the brief reflects it. When a campaign result comes in, Claude can see it. The strategy is not static. It is always current.

### What MCP Connects Claude To

| System | What Claude Can Do |
|---|---|
| Client strategic briefs | Goals, audience, voice, positioning, history — the strategy layer |
| GitHub repos | Full code access to every TAE-built site — read, edit, deploy |
| Supabase database | Client data, tickets, discovery results, portal activity |
| Basecamp | Project status, timelines, tasks — operational context |
| Analytics platforms | What is actually converting — real performance data |
| Client CMS APIs | Content updates on client sites directly |
| Web search | Competitor and market monitoring in real time |
| Email platforms | Draft and send client communications |

MCP is an open protocol adopted by multiple AI companies — OpenAI, Google, and others are building MCP compatibility into their own models. This matters for resilience: the integration layer — how ContextLayer connects to TAE's systems — is portable. If TAE ever needed to move off Claude, the plumbing would survive the switch.

Honest caveat: portable plumbing is not a free swap. MCP standardizes the connections, not the model — and output quality, the thing TAE's edge depends on most, varies between providers. So the connections move easily; the model is swappable but would need re-tuning to preserve quality. Claude is the right engine today, and the architecture ensures TAE is never trapped if that changes.

### The Planned Stack

More specific than the table above, this is the actual planned build:

- **MCP server** — hosted on Vercel (serverless, HTTP/SSE transport)
- **GitHub** — the file store, read/write via the GitHub API
- **Supabase** — project memory, change log, auth
- **Upstash Redis** — file locking and rate limiting, so two people's Claude sessions can't collide on the same edit
- **Vercel API** — to trigger deployments directly
- **Subdomain** — `mcp.theartistevolution.com`

**Key distinction worth keeping straight:** a *local* MCP setup — where each person runs their own server pointing at Git — is a tool, one person at a time. A *hosted* MCP with shared memory, where everyone's Claude sees the same up-to-date picture, is the actual ContextLayer product. The plan is the second one.

---

## 05 — GitHub — The Memory That Compounds

Every strategic decision made for a client gets recorded. The copy rewrite that lifted conversions. The positioning pivot after a market shift. The page restructure that fixed the bounce rate. The headline that finally clicked. Every one of these decisions lives in GitHub — timestamped, attributed, and permanently accessible.

A repo — repository — is a regulated, recorded residence for a project's code. It is a folder with a built-in historian. Every change is snapshotted. You can return to any moment in the project's history instantly. GitHub is the platform that hosts repos and adds collaboration on top.

For ContextLayer, GitHub is where the relationship compounds. Generic AI tools start from zero with every client. TAE's system accumulates. The longer a client is with TAE, the more context Claude has — not just about what the site looks like today, but about every decision that brought it here. That context informs every future decision. The gap between TAE's AI and a generic tool widens every month.

### How GitHub Should Be Structured — and Its Current Status

For a business, GitHub should not be tied to any one individual's personal account. The right structure has three layers:

**Layer 1 — The Organization Account.** A GitHub Organization is the shared account that owns all repos. It represents the business — not any individual. All code lives here. Team members are invited in and out without affecting ownership. **Status: done.** `taemarketing` is set up as the org, and as of today all seven live tools' repos live under it (one exception, `tae-twin`, sits under a second org — `The-Artist-Evolution` — confirmed as an intentional, acceptable split rather than an oversight).

**Layer 2 — The Owner Account.** GitHub requires every organization to be owned by at least one personal account. This should be a dedicated business account — created with a TAE-owned email address not tied to any individual. Credentials stored in a shared password manager. **Status: not yet confirmed** — worth a direct check rather than assuming either way.

**Layer 3 — Individual Employee Accounts.** Each team member creates their own personal GitHub account. They are invited as collaborators on the repos they need. Access can be granted or revoked without affecting the organization or any other team member. **Status: not yet built out** — currently effectively single-operator.

This structure ensures the business owns its code permanently — independent of any individual's employment, email address, or account status.

> GitHub is institutional memory. Claude is intelligence. ContextLayer is the wire between them — so nothing learned is lost, and nothing needed is out of reach.

---

## 06 — The TAE Hosting Advantage

Strategy encoded at the AI level requires access at the code level. Where a client's site is hosted determines how deeply the strategy can be acted on.

When a client's site lives on TAE's infrastructure — GitHub repo under the TAE org, deployed via Vercel — Claude has complete access to everything. Not just the words on a page but the structure of every page, the logic behind every interaction, the design of every component. The strategic brief drives everything from copy to architecture.

When a client's site is on a closed platform — Squarespace, Wix — Claude can only touch what the platform exposes. The strategy is still encoded, but the ceiling on acting on it is set by a third party TAE has no control over.

### Connection Depth by Platform

| Platform | Strategy Access Level | What Claude Can Act On |
|---|---|---|
| TAE-built + TAE-hosted | Full | Copy, structure, design, architecture, deployment — everything |
| TAE-built on client's host | Full | Same — hosting location does not limit code access |
| WordPress (self-hosted) | High | Full code via GitHub + content via API |
| Webflow | Medium | Content and CMS updates only — structural strategy limited |
| Shopify | Medium | Products, content, theme sections via API |
| Squarespace / Wix | Low | Basic content only — platform is a ceiling on strategy execution |

Every capability ContextLayer enables — brand enforcement, discovery feeding the build, autonomous maintenance, proactive recommendations — requires code access. TAE hosting unlocks all of it. One pipeline for all clients: every TAE-hosted site deploys the same way, monitored the same way. No managing ten different platform logins and update processes. A change that takes days on a closed platform takes minutes when Claude has full code access and a direct deployment pipeline.

Every TAE-hosted site adds to ContextLayer's cross-client intelligence. The more sites on TAE's infrastructure, the smarter the recommendations become for every client.

> TAE hosting is not a power grab. It is a service upgrade. The client gets more capability, faster turnaround, and a strategic AI layer they would not have access to anywhere else.

### The Hosting Economics — A Different Model

Traditional web hosting works the way most clients are used to: each client pays separately for their own hosting account — GoDaddy, Bluehost, SiteGround — typically $10–30/month per site, managed independently, with separate logins, separate billing, and no shared intelligence between them.

TAE's model is different. All client sites live under a single Vercel account — one flat seat cost, unlimited projects. TAE pays the infrastructure bill. Clients pay TAE a hosting management fee as part of their retainer. The economics compound: a handful of clients covers the base infrastructure cost, and each additional site adds margin rather than a new hosting bill.

Honest clarification: $20/month is per team-member seat and covers unlimited projects, not unlimited usage — a high-traffic site can exceed the included bandwidth and incur overages. Still dramatically cheaper than per-client hosting, and it scales predictably.

| Tier | Cost | What It Adds |
|---|---|---|
| Pro | $20/mo per team member seat | Unlimited projects, 1TB bandwidth, preview deployments, analytics, password-protected previews, faster builds — covers everything TAE needs for the foreseeable future |
| Enterprise | Custom pricing | Guaranteed SLA uptime, SSO/advanced security, dedicated support, audit logs, IP allowlisting, priority build queue — relevant when clients require contractual uptime guarantees |

Pro is the right tier now and for a long time. The upgrade path to Enterprise exists when TAE is managing high-traffic sites where clients are asking for contractual uptime guarantees — that is a revenue conversation, not just an infrastructure one.

---

## 07 — The Conversational Interface

The client-facing layer of ContextLayer is a single text input. One place. One prompt. Whatever the client needs — a site change, a document, an answer, a deliverable — they type it in plain language and the system handles it. No dashboard to navigate, no tool to learn, no human to brief, no queue to wait in.

This interface operates in three distinct modes — all powered by the same strategic brief, all informed by everything TAE has encoded about that client.

**Mode 1 — Site Editing.** The Conversational CMS. The client types what they want changed on their site. The AI interprets it through the strategic brief — goals, audience, voice, positioning — and produces a change that serves the actual objective, not just the literal request. Direct publishing by default. Guardrails enforce the strategy automatically.

**Mode 2 — Data Querying.** "How many leads last month?" "Which page drives the most traffic?" "Best-performing social post this quarter?" Plain language in, real answer out — no dashboard, no report requested. Numbers come from actual database queries, not the AI's recall, which is the critical build detail: the model translates the question into a real query, runs it, and reports the result. No invented figures.

**Mode 3 — Document and Deliverable Generation.** One prompt produces a strong first draft of anything the client needs — proposal, content calendar, press release, pitch deck, RFP response, staff FAQ, service one-pager — written in their voice, built from TAE's strategic knowledge of them. Low-stakes internal documents are often ship-ready. High-stakes external pieces earn a human review. Either way: ten minutes, not two days.

> A small business that could never afford a full creative department, an analyst, and a strategist now has all three — available instantly, already briefed, already aligned to their goals. That is what the conversational interface delivers.

### Two Paths: In-Scope Publishes, Out-of-Scope Reviews

There is one rule that governs whether a change goes live instantly or routes to TAE review, and it is worth stating plainly because it is the safety model of the whole system:

**In-scope changes publish directly. Out-of-scope changes route to TAE review. Nothing else.**

TAE defines the scope once, at setup. Inside that boundary — editing copy on approved pages, swapping approved imagery, updating content — the client publishes directly, with no queue and no waiting, the same as any CMS. Outside that boundary — new pages, structural changes, anything the guardrails flag — the request becomes a proposal that lands in TAE's review queue instead of going live. The client experiences instant control for everyday work and a fast turnaround for everything else. The two paths are not in tension; the scope line is what separates them.

### TAE's Role: Guardrails, Not Gatekeeping

TAE's role is to define the rules at setup — and then let the system run. The guardrails are how the strategy gets enforced automatically at scale. They come in two kinds, and the distinction is important because it is the difference between a guarantee and a judgment call.

- **Brand guardrails — deterministic (hard block).** The objective, checkable rules — colors, fonts, spacing, logo usage, required and forbidden words — are enforced by a validation layer that runs after the AI and before anything publishes. This is code, not judgment: a change that violates these rules is blocked, not merely discouraged. This is where "the AI cannot publish off-brand" is literally true, because a deterministic check is doing the enforcing.
- **Brand guardrails — model-based (flag and route).** The subjective stuff — tone, voice, whether the copy feels on-strategy — cannot be guaranteed by a rule, because it is a matter of judgment. Here the AI flags likely drift and routes it for human review rather than promising perfection. Honest framing: deterministic rules are guaranteed; tone is assisted, not guaranteed.
- **Goal guardrails.** The AI evaluates every change against the stated conversion goal. A client asking for something that would undermine their own objective gets a better version suggested and, if it crosses a defined line, routed to review.
- **Infrastructure guardrails.** Navigation, page architecture, core layout can be locked. Clients update copy and imagery freely without touching anything structural.
- **Prompt limits.** TAE can cap how many prompts a client submits per billing cycle — creating a natural tier structure and preventing runaway usage.
- **Escalation triggers.** New pages, structural changes, anything outside defined scope routes to TAE review instead of publishing automatically. Everything in scope goes live without a second look.

> Clients feel like they have total control — because within their defined boundaries, they do. The strategy runs in the background. They never see it. They just see that every change looks right and works.

### When TAE Review Is Required — What Is a PR?

For out-of-scope requests, the system routes to a TAE review queue. This is handled through a Pull Request — PR. Despite the name, nothing to do with public relations. It means: Claude makes the proposed changes on a separate branch that does not affect the live site, then opens a formal proposal: here is what I did, review it, approve to go live.

A PR includes a summary of what changed, a side-by-side view of every edit, and a live Vercel preview URL — the actual site with changes applied, fully functional, before anything ships. TAE reviews, approves with one click, or asks Claude to adjust. Nothing touches the live site until a human says so.

---

## 08 — What Becomes Possible — 12 Capabilities

Each of these capabilities is an expression of the same idea: AI with strategy encoded is categorically different from AI without it. These are not incremental improvements. They are things that were either impossible or required significant human labor before.

Each is tagged with a build status so this reads as a roadmap, not a wishlist. Some of this exists today, some is near-term work, some requires a real build, and some is genuine future vision. Knowing which is which is the point of an internal document.

**01 — Strategy-Driven Site Changes** · *Near-term.* When a client submits a prompt, Claude does not just execute the literal request. It interprets the request against the strategic brief — goals, audience, positioning — and produces an output that serves the actual objective. A client asking for a new headline gets one written for their specific audience, in their specific voice, toward their specific goal.

**02 — Discovery Directly Feeds the Build** · *Near-term.* TAE's Discovery System captures who a client is — goals, audience, what makes them different. ContextLayer closes the loop: Claude reads the discovery answers and the client's existing site, then drafts what the updated site should say based on the client's own words. That draft lands in TAE's review queue. The client never saw the draft process. They just see that their vision became their site.

**03 — Brand Consistency Enforcement** · *Requires build.* Two layers. The objective rules — colors, fonts, spacing, logo usage, required and forbidden words — are enforced by a deterministic validation check that blocks non-conforming changes before they publish. The subjective rules — tone, voice, on-strategy feel — are flagged by the AI for human review. The hard rules are guaranteed because code enforces them; tone is assisted, not guaranteed. Building the deterministic check is the real work here.

**04 — The Living Client Brief** · *Requires build.* Every interaction — tickets, approved changes, campaign results, strategy conversations — feeds a continuously updated knowledge store the AI retrieves from. When a new project starts, Claude pulls the full relevant history. No briefing required. This depends on the retrieval architecture described earlier — it is the foundation most other capabilities sit on.

**05 — Autonomous Site Maintenance** · *Requires build.* Claude proactively flags broken links, outdated event dates, expired offers, and off-brand content before clients notice — without being asked. It runs on a schedule, detects drift, and opens a PR with the fix already written. TAE approves. Requires scheduled background jobs and the monitoring harness, but the building blocks all exist.

**06 — Cross-Site Operational Monitoring** · *Requires build.* Across all TAE-hosted sites, the system surfaces operational patterns: which client hasn't updated their homepage in six months, which sites failed a brand check, which have broken links or slow load times. Note this is TAE's own roster — operational housekeeping across sites TAE manages, not reaching into anyone else's data. Genuinely useful, and lower-risk than it sounds because it stays inside TAE's own infrastructure.

**07 — Competitive and Market Awareness** · *Future vision.* Claude monitors competitor sites and industry trends on a schedule, then surfaces insights to the relevant client's portal — filtered through what matters for their specific positioning. Real, but further out: reliable web monitoring and signal-versus-noise filtering is its own meaningful build.

**08 — Inter-Client Learning (Curated, Then Aggregated)** · *Future vision.* The most-scrutinized capability, so here is the honest version. Near-term, learning across clients is human-curated: TAE notices what worked for one client, generalizes the lesson with the specifics stripped out, and encodes it into a shared playbook the AI uses for everyone. A person is the privacy filter. Longer-term, an anonymized aggregate layer can extract patterns automatically — metrics and structures, never raw content — across enough clients that nothing traces back to one, ideally with client consent via a benchmarking opt-in. What TAE will never do is let one client's raw data surface in another's session. See the data-isolation note in the revenue and security sections.

**09 — Proactive Relationship Management** · *Near-term.* ContextLayer sees that a client hasn't logged in for 30 days, has an open ticket with no reply, and their last report was six weeks ago. It surfaces that as a relationship health alert before the client feels neglected. Claude drafts a check-in. TAE sends it in one click. Most of the underlying data already lives in the portal database.

**10 — The Internal Operator** · *Future vision.* Claude stops being a tool TAE prompts and becomes closer to a junior operator. It monitors all systems, takes the small authorized actions on its own, and queues the bigger ones for approval. The admin dashboard becomes a place to approve work rather than do it. This is the furthest-out vision — it depends on every capability above being reliable first, and on a hard-earned trust in the guardrails.

**11 — The On-Demand Creative Department** · *Near-term.* A client with a fully encoded strategic brief gets something no generic tool offers: an on-demand creative department that already knows them. One prompt produces a strong first draft of a proposal in their voice, a content calendar built around what has worked, a press release, an RFP response, a staff document, a service one-pager. Bulk marketing needs handled in a single session — design and positioning informed by TAE's expertise. High-stakes external pieces still earn a human review (see the Conversational Interface). The value is turning a two-day deliverable into a ten-minute on-strategy draft.

**12 — Closing Its Own Data Gaps** · *Future vision, built on pieces that already exist.* TAE Discovery already runs a self-improving lens framework — a growing set of automated checks that review real sessions, find quality problems, and edit their own rules in place. Today it's scoped narrowly: interview quality only. The planned direction is for that same framework to run across a client's *full* lifecycle — not just the interview, but onboarding, project delivery, and outcomes — asking a bigger question than "was this interview good?": did the MIS score and service-fit recommendation actually turn out to be right?

Answering that requires outcome data that mostly doesn't exist yet — did the client buy what was recommended, did the campaign work, did they renew. Rather than stall there, ContextLayer's job is to notice the gap and act on it. **Presence, not prompting, is the primary mechanism** — a principle borrowed deliberately from *The Cue*, a separate manifesto about exactly this failure mode: platforms that wait until after the fact and demand a form be filled out (*"How would you rate your recent experience?"*) are mining residue, not listening, and what they collect skews toward outliers because only the enraged or enchanted bother to answer. The same failure is possible here — a card that interrupts someone to ask "did that deal close?" is uncomfortably close to the exact pattern being warned against. So the design leads with the other thing *The Cue* names as one of the two channels where genuine, unperformed information actually surfaces: conversation with an AI, where there's no audience to perform for. ContextLayer's job is to catch the fact when someone mentions it in passing — in a Claude session, in whatever work is already happening — the same way a person might casually say "oh, they signed the ad package last week." Caught in context, confirmed before it's kept (consent and provenance apply here exactly as they would to anything else ContextLayer touches), it closes the gap without ever having to ask.

**The directed question is the fallback, not the front door** — for the real remainder that presence alone won't catch. When it's used, it should stay disciplined about what kind of thing it asks for: a specific fact a specific person already knows ("did the deal close"), not a reconstructed feeling ("how did that go," "were they happy") — the second kind is exactly the sentiment-extraction *The Cue* indicts, just aimed at TAE's own team instead of fans. The mechanism for it: a card in the TAE Dashboard's activity feed, the same pattern already proven three times over (tickets, discovery sessions, social analyses all already normalize into that feed) — a fourth `type`, a new tab, a detail page to answer it. Directed at a specific person when the system knows who'd know. If it's answerable by connecting a new system instead — invoicing, a CRM, an ad platform's API — that's the better fix regardless, since a system fact needs asking exactly once, ever.

Either way, the system tracks its own progress the same way *The Cue* argues trust should work generally: **a body of evidence, not a score.** Not a single "% complete" number, but a navigable, specific, examinable report — turned inward on TAE's own data instead of a client's — showing what's known, what's missing, and where it came from. A score compresses and hides exactly the thing that matters; a record you can read and judge for yourself doesn't.

None of this exists yet beyond the pieces it's built from: the lens framework (discovery-scoped today), the notification pipeline (used for other things today), and the activity feed's card pattern (used for three other things today). Closing the loop — full-lifecycle lenses, presence-based capture, the fallback card, a self-facing coverage report — is genuine future work. Full detail and the discovery-side half of this lives in `tae-discovery/docs/improvement-loop.md`.

### The Expanding Possibility Space

The 12 capabilities above are a starting point, not a ceiling. As ContextLayer deepens across TAE's client base, the surface area of what becomes possible grows continuously. This is a broader view of the direction — a backlog of intent, not a set of promises with delivery dates:

Automated bad review detection and response drafting across all platforms · Review velocity monitoring — alerts when positive reviews slow or negative ones spike · Fake or spam review identification and flagging · Reputation trend reporting delivered automatically to client portals · Taking a single social post and distributing it everywhere it is missing in the client ecosystem · Detecting inconsistencies in name, address, and phone number across all directories and fixing them · Keeping Google Business, Yelp, Facebook, and all profiles in sync automatically · Repurposing long-form content into short-form across every relevant channel · Generating social captions, email subject lines, and ad copy from a single brief · Widely-customizable intelligent scheduling — content, check-ins, reports, renewals · Audience-optimized post timing based on each client's actual engagement patterns · Proactive alerts when a client site goes down, slows significantly, or returns errors · Competitor publish alerts — notified the moment a competitor posts something significant · Deadline and milestone reminders tied to client project timelines · Auto-generating meta descriptions and alt text for every image on every client site · Detecting and fixing missing schema markup automatically · Core Web Vitals monitoring with performance degradation alerts · Identifying thin or duplicate content pages and drafting improvements · Broken internal and external link detection and repair before clients notice · Auto-generating FAQ sections from client ticket and conversation history · Correlating social performance with site traffic to surface what is actually driving results · Identifying underperforming pages relative to their traffic and diagnosing why · Spotting seasonal trends in a client's industry before they peak · Detecting when a client's search ranking drops and surfacing a probable cause · Monitoring competitor ad messaging and flagging positioning shifts · Finding operational efficiencies across TAE's client roster automatically · Auto-categorizing and routing incoming client tickets by type and urgency · Generating client-ready monthly reports in each client's preferred format without manual assembly · Summarizing all client activity into a weekly internal TAE briefing automatically · Identifying which clients haven't been contacted in a set number of days and drafting outreach · Surfacing which deliverables are overdue across the full client roster in one view · Predicting client churn risk based on engagement, ticket volume, and activity patterns · Identifying cross-sell and upsell opportunities based on gaps in a client's current ecosystem · Generating renewal talking points built from actual results achieved during the engagement · Drafting case study content from campaign results automatically · Flagging when a client's industry experiences a PR moment TAE should help them respond to · Inter-client pattern recognition — what works in one market informing strategy in another · Automated A/B copy testing recommendations based on page performance data · Client onboarding automation — brief generation, asset requests, kickoff prep · Contract and retainer renewal tracking with proactive TAE alerts · Building a searchable institutional knowledge base from all past TAE work and decisions

> This list is not exhaustive. It grows every time TAE adds a client, deepens a relationship, or connects a new system. The longer ContextLayer runs, the more it finds to do.

---

## 09 — The Permanent Advantage — Why Generic AI Can't Catch Up

The market will be flooded with AI that can execute. Build pages, draft copy, swap images, suggest updates. All of it will work. All of it will get cheaper. None of it will know the strategy — because strategy comes from a relationship, and generic tools do not have relationships. They have users.

Here is why the gap between TAE's AI and the generic market is not just a head start — it is a structural advantage that compounds over time.

- **The strategy accumulates.** Generic AI starts from zero with every session. TAE's system adds to itself — every interaction, campaign result, and strategic decision becomes context for the next one. Month one versus year three, the system knows more and acts more precisely at every stage. The gap between TAE's AI and a generic tool widens every month a client stays.
- **The brief drives the output.** This is the core of the advantage. Generic AI optimizes for "looks right"; TAE's AI optimizes for what the client is trying to accomplish, because the goal is encoded before they ever type a prompt. The output reflects it: a proposal is not a template filled in but a document shaped by years of understanding what makes this client win; a content calendar is built around what has actually worked for this audience. That specificity is the product of a relationship — and a template-driven tool that has none cannot replicate it.
- **The expertise is encoded.** GoDaddy AI has no opinion about whether a client's messaging positions them correctly, or whether the copy it just wrote is on-brand. TAE has spent years developing that judgment for specific clients, and ContextLayer makes it available to the AI automatically — without TAE having to be in the room for every execution. Generic tools know the website; TAE's AI knows the analytics behind it, the conversations that shaped it, and the decisions that informed every page.
- **The relationship is the product.** Generic tools sell software — the client signs up, uses it, and leaves when something cheaper arrives. TAE sells an evolving partnership where the AI gets smarter the longer it continues. The switching cost is not the tool; it is the accumulated strategic context that lives inside ContextLayer. That context belongs to the relationship. It cannot be exported to GoDaddy.

> The market will commoditize execution. TAE's moat is judgment — and judgment comes from a relationship the AI has been trained on for years. That is not a feature. That is not a product. That is a compounding advantage that gets harder to replicate every single month.

---

## 10 — The Revenue Architecture

ContextLayer is not just an operational tool — it is a product. The infrastructure being built has three natural monetization tiers.

| Tier | Offering | Who Pays For AI | TAE Charges For |
|---|---|---|---|
| 1 — Setup | Client-owned ContextLayer install | Client pays their own API costs | Setup fee + training + support retainer |
| 2 — Managed | TAE-managed, strategy-encoded AI | TAE (included in retainer) | Monthly fee for AI-assisted maintenance |
| 3 — Operator | Full AI operator for client's digital presence | TAE (included in retainer) | Premium retainer — highest tier |

### The Network Effect — And Its Hard Boundary

Each client TAE onboards makes the system more valuable: more strategic context, more generalized lessons, better recommendations. The system compounds — and so does the value TAE delivers.

But this is exactly where an honest design has to draw a hard line, because there is a real tension between "the system learns across clients" and "each client's data is isolated." Both are promised in this document, and they cannot both be true in their naive forms. Here is how TAE resolves it:

Client work happens in isolated contexts. When the AI works on Client A, it has access to Client A's data — not Client B's. Raw client data never crosses between sessions. Cross-client value flows through a separate, deliberately narrow channel: generalized lessons (near-term, human-curated) and anonymized aggregate patterns (longer-term, never raw content, ideally consented). The "shared intelligence" is a layer of de-identified strategy — not a single brain that can see everyone at once. That distinction is the difference between a defensible system and a confidentiality breach, and it is a design constraint, not an afterthought.

> The rule, stated once: Isolated by default. Shared only as anonymized, generalized strategy. One client's raw data never appears in another client's session. Ever.

### The White-Label Play — And the Competition

Once mature, TAE can sell the entire ContextLayer infrastructure — portal, discovery system, Conversational Interface, brand enforcement, strategic brief encoding — as a white-label product to other agencies.

GoHighLevel and similar platforms already sell white-label agency infrastructure — CRM, funnels, scheduling, generic AI assists. TAE does not out-feature them. TAE wins on the one axis they structurally cannot compete on: strategy-encoded output from a real agency-client relationship. That is the only axis worth claiming.

---

## 11 — Cost, Scaling & Margins

This section models what ContextLayer actually costs to run, how that cost behaves as TAE adds clients, and where the margin lives. The numbers are estimates built on current provider pricing and reasonable usage assumptions — directionally honest, meant to be revisited with real usage data, not treated as precise forecasts.

### Two Kinds of Cost

ContextLayer's cost splits cleanly in two. Fixed infrastructure is what TAE pays to keep the platform running at all — roughly flat regardless of client count. Variable cost is almost entirely AI tokens, and it scales with how much each client actually uses the system. The fixed cost amortizes as clients are added; the variable cost is controllable with deliberate engineering.

### Fixed Infrastructure (Whole Agency, Monthly)

| Line Item | Approx / mo | Notes |
|---|---|---|
| Vercel Pro | $20 / seat | Unlimited projects; ~$40 at two seats |
| Supabase | $25 | Database + retrieval store (pgvector) |
| Email (Resend) | $0–20 | Free tier covers low volume |
| Password manager | ~$20 | Shared team vault for credentials |
| Domains + Cloudflare DNS | ~$5 | DNS free; registration amortized |
| GitHub Organization | $0–16 | Free, or Team at $4/seat |
| **Estimated total** | **~$110–150** | Flat regardless of client count |

### Variable Cost — AI Tokens Per Client

The AI is the real cost driver, and it is usage-based. The model is tiered deliberately: a fast, inexpensive model handles most small work, a mid model handles the bulk of generation, and the most capable model is reserved for high-value drafting. Repeated context — the client brief loaded on most calls — is cached, cutting the cost of reading it by roughly 90 percent. Figures below are per active client, per month.

| Client Profile | What Drives It | Est. / mo |
|---|---|---|
| Light | Occasional edits, a few queries | $3–8 |
| Typical active | Regular edits, monitoring, some drafts | $10–25 |
| Heavy / power user | Frequent top-model drafting, high volume | $40–80 |

Reference pricing behind these estimates (per million tokens): the workhorse mid model is about $3 in / $15 out, the top model about $5 in / $25 out, the fast model about $1 in / $5 out. Cached context reads at roughly a tenth of input price, and non-urgent batch work runs at half price. These rates move over time; the levers below matter more than any single figure.

### Unit Economics — How It Behaves at Scale

| Clients | Fixed | AI (~$15 avg) | Total / mo | Per Client |
|---|---|---|---|---|
| 4 | $130 | $60 | ~$190 | ~$48 |
| 10 | $130 | $150 | ~$280 | ~$28 |
| 25 | $150 | $375 | ~$525 | ~$21 |

The shape is the point: fixed cost spreads across more clients, so all-in cost per client falls as TAE grows and trends toward just that client's token usage. A ContextLayer or hosting fee of even $100–300/month per client — folded into the retainer — produces gross margins in the 80–90 percent range, widening with scale.

### The Levers That Control Variable Cost

- **Model tiering.** Route each task to the cheapest model that does it well — fast model for classification and queries, mid model for most generation, top model only for high-value drafting. The single biggest cost lever.
- **Prompt caching.** The client brief and other repeated context are cached, so the AI does not pay full price to re-read them on every call. Reads drop to roughly a tenth of input cost.
- **Per-tier prompt caps.** The prompt limits in the Conversational Interface double as a cost control: they cap each client's monthly token spend and turn usage into a predictable, tier-priced number.
- **Batch and off-peak work.** Non-urgent autonomous work — overnight monitoring, bulk maintenance — runs on the batch path at half price, since it is not latency-sensitive.
- **Retrieval discipline.** Retrieve the right slices, not everything. Tighter retrieval is both better quality and lower cost — fewer tokens loaded per call.

### The Real Scaling Constraint Is Not Infrastructure

Worth stating plainly, because it changes how TAE should plan: the binding constraint at scale is not server cost or token cost — both stay modest and predictable. It is human review capacity. Every out-of-scope change, every high-stakes deliverable, every escalation routes to a person. The more clients on the system, the more approvals per day TAE must make. That is a staffing question, not an infrastructure one — and the right constraint to design around, because it is also where TAE's judgment adds the value clients pay for.

> The cost of running ContextLayer is small and shrinks per client as TAE grows. The cost that matters is the human time to review what it produces — and that time is the product.

---

## 12 — Longevity — Every Ingredient, Every Fallback

What if every ingredient disappears — GitHub, Vercel, Anthropic, Supabase, all of it? The goal is that no single vendor failure loses data or takes longer than hours to recover from. Each ingredient below has tested fallbacks and a protection protocol.

### GitHub
Git is not GitHub — it is an open protocol GitHub is built on. Every repo pulled to any machine is a complete independent copy of the full history. Disappearance costs an afternoon, not any work.

| Fallback Option | What It Requires |
|---|---|
| GitLab | Free, same interface, self-hostable if needed |
| Bitbucket | Atlassian-backed, enterprise-grade alternative |
| Gitea (self-hosted) | Runs on a $5 server TAE owns — no third party at all |
| Local machines | Every pulled repo IS a full backup — `git pull` is an insurance policy |

Protection: regular `git pull` across all team machines. Every machine with a current clone is a complete independent backup.

### Vercel
The sites are files — not locked inside Vercel. Any TAE-built site redeploys elsewhere with the same codebase in under an hour.

| Fallback Option | What It Requires |
|---|---|
| Netlify | Direct competitor, same workflow, one-click import from GitHub |
| Cloudflare Pages | More resilient infrastructure, free tier, global CDN |
| Render / Railway | Full app hosting including databases |
| DigitalOcean / Hetzner | $10/month server runs everything TAE has today |
| Coolify (self-hosted) | Open source Vercel — runs on your own hardware, zero platform dependency |

Protection: maintain Docker configuration files for every project. Infrastructure as code means the platform is always swappable.

### Claude / Anthropic API
All AI calls route through one abstraction layer. Switching providers means changing that file — not rebuilding. Quality re-tuning required, but the plumbing survives.

| Fallback Option | What It Requires |
|---|---|
| OpenAI GPT-4o | API-compatible, MCP-capable, comparable capability |
| Google Gemini | Strong multimodal capabilities, growing MCP support |
| Mistral | European provider, strong privacy stance, API-compatible |
| Meta Llama (self-hosted) | Open source, runs on private hardware, zero external dependency — the nuclear fallback |

Protection: abstract all AI calls through a single interface layer. Test the swap to at least one alternative provider annually.

### Supabase
Supabase is PostgreSQL. Standard SQL, exportable at any time — not proprietary, not locked.

| Fallback Option | What It Requires |
|---|---|
| Neon | Serverless PostgreSQL, same connection format, generous free tier |
| Railway | Full-stack hosting including PostgreSQL |
| Self-hosted PostgreSQL | $10/month server, complete ownership, no third party |

Protection: automated daily SQL dumps exported to a location TAE controls. Maximum data loss capped at 24 hours regardless of what happens to Supabase. **Status as of this update: not yet built** — the current Supabase project has no automated backup export in place. Genuinely open item, not just a hypothetical in this section.

### Basecamp, Email, and DNS
Basecamp is the loosest dependency — a data source for context, not a load-bearing wall. Fallbacks: Linear, Notion, Plane. One MCP connector update, a few hours of work.

Email sending (Resend) is separate from email receiving (domain MX records). If Resend disappeared, sending would pause until a replacement was configured. Fallbacks: Postmark, SendGrid, Amazon SES. Switching takes hours.

Domain registrars are the deepest layer. Protection: register TAE domains across at least two different registrars, use Cloudflare for DNS on all domains, and keep physical documentation of all domain ownership.

### The Seven Protection Protocols

These are the concrete standing practices that ensure TAE can recover from any ingredient disappearing — individually or simultaneously.

1. **Daily automated database backups** — All Supabase data exported as SQL to a location TAE controls. Stored in at least two geographic locations. Maximum data loss capped at 24 hours. *(Not yet implemented — see Supabase section above.)*
2. **Every repo cloned locally on multiple machines** — Git pull as a standing habit. Every pulled repo is a complete independent backup of the full history.
3. **Docker configs for every project** — Every TAE site and tool has a configuration file defining exactly how to run it. Any site can be spun up on any server in under an hour.
4. **One abstraction layer for all AI calls** — Every call to Claude routes through a single interface in the codebase, so the provider can be swapped at one integration point rather than rewired throughout. The swap still requires re-tuning and re-testing for quality — tested against at least one alternative annually so the path stays proven.
5. **Open formats for everything** — No proprietary file formats. No locked exports. Every piece of data TAE holds can be read and moved without the tool that created it.
6. **Domain diversity and Cloudflare DNS** — TAE domains spread across multiple registrars. All DNS on Cloudflare. Physical documentation of all domain ownership kept offline.
7. **Annual fire drill** — Once a year, simulate a key provider going down and actually execute the recovery steps. Not a planning exercise — a real test. Recovery from any single provider should take hours, not weeks.

> The goal is not to prevent any ingredient from disappearing. It is to make recovery from any disappearance a matter of hours — and to ensure zero data is ever lost regardless of what goes down. Every recipe survives when you own the ingredients.

---

## 13 — Security — Protecting What We Centralize

The longevity section answered "what if a vendor disappears." This section answers a different and equally important question: what if someone tries to break in? The two are separate threat models, and a system that centralizes this much deserves both.

The honest premise: centralization is the source of ContextLayer's power and the source of its risk. Putting every client's site code, credentials, and strategic data under one roof is what makes the AI capable across the whole roster — and it also makes that roof a target. A compromise of TAE would not be a compromise of one client; it could be a compromise of all of them. That is not a reason to avoid the architecture. It is a reason to take the security of it as seriously as the capability.

### The Five Threats and the Response to Each

- **Client data bleed.** The risk that one client's data surfaces in another client's AI session. Response: per-client scoped credentials and isolated contexts. The AI working on Client A is technically scoped to Client A's systems — it cannot reach Client B's data because the access does not exist in that context, not merely because it was told not to. Cross-client value flows only through the de-identified aggregate layer described in the revenue section.
- **Prompt injection.** The risk that a malicious prompt — from a client, or someone impersonating one — tricks the AI into acting outside its scope (exfiltrating data, editing files it should not touch). Response: scope is enforced by the system, not by instructions to the model. The AI's available tools and reachable files are constrained by allowlists at the infrastructure level. A prompt cannot grant access the credentials do not already permit. Out-of-scope actions route to human review rather than executing.
- **Credential compromise.** The risk that the keys to client systems leak. Response: secrets live in a managed secrets store, never in code or in chat, scoped to the narrowest necessary permission, and rotated on a schedule. No single credential unlocks everything. The principle is least privilege — every key opens the smallest possible door.
- **TAE as a breach target.** The risk that TAE itself is the way in. Response: multi-factor authentication on every account, the GitHub organization structure described earlier (no single personal account owns everything), a shared password manager rather than reused or shared passwords, and access that can be revoked per person without disturbing the whole. The fewer single points of human failure, the smaller the target.
- **Autonomous action gone wrong.** The risk that the AI, acting on a schedule without a human present, does something harmful. Response: autonomous actions are limited to a defined, low-stakes allowlist; everything consequential is staged for human approval via the PR flow; and every action is logged and reversible through GitHub's history. The operator vision earns more autonomy only as the guardrails prove themselves — trust is granted incrementally, not assumed.

### The Standing Security Practices

- Least privilege everywhere — every credential scoped to the narrowest necessary access
- Secrets in a managed store, never in code or chat, rotated on a schedule
- Per-client isolation — scoped contexts, no shared raw data across client sessions
- Scope enforced at the infrastructure level via allowlists, not by prompting the model
- Multi-factor authentication on every account that touches client systems
- Human approval required for any consequential or out-of-scope AI action
- Full audit trail — every AI action logged, attributed, and reversible through GitHub
- Annual security review paired with the longevity fire drill

**Status as of this update:** the Supabase side of "least privilege" and "per-client isolation" got real, verified progress today — 12 tables that had zero access control (not scoped-too-loosely; actually no policy layer at all) now have row-level security enabled, two admin views and one admin function that bypassed access control entirely are locked down, and the flat namespace has started splitting into `core`/`discovery`/`portal`. That is groundwork for this section's principles, not this section's principles being fully realized — actual per-client scoped credentials for a live MCP server don't exist yet because the MCP server doesn't exist yet.

> Capability and security are not a trade-off here — they are the same discipline. The same isolation that protects a client's data is what makes the cross-client intelligence honest. Build the guardrails first, and the capability is safe to grow into.

---

## 14 — The Roadmap — What Gets Built in What Order

The infrastructure being built right now is not separate from ContextLayer — it is ContextLayer's foundation. Every piece laid today is a load-bearing wall of the larger system.

### Phase 1 — Now
- ✅ All TAE repos on GitHub — tae-portal, tae-discovery, tae-twin, tae-social-analyzer, *and, as of today, F.R.I. Marketing, the Dono Dashboard, and ACS Dashboard too*
- ✅ Vercel connected to GitHub for auto-deploy on every push — *all seven, not just the original four*
- ✅ TAE admin dashboard fully operational with client management
- ✅ Discovery system live and generating leads

**Phase 1 is complete as of July 24, 2026.**

### Phase 2 — Near Term
- 🟡 Build the MCP server — the central hub connecting Claude to all TAE systems *(started — `taemarketing/tae-contextlayer` exists with one live tool, `get_client_context`, reaching across `core`/`discovery`/`portal`; deployed to GitHub, Vercel import still pending. More tools get added the same way going forward.)*
- ⬜ Stand up the retrieval store and prove retrieval quality on real client data *(not started — today's first tool queries existing operational tables directly, not a retrieval/vector store)*
- 🟡 Establish the security foundation — scoped credentials, secrets store, per-client isolation *(partial: Supabase RLS gaps closed, all four schema-namespacing phases complete including `core` — but apps still connect with unscoped credentials that can see every schema, and ContextLayer itself deliberately also uses full-access service-role credentials, by design, not narrowed — per-app Postgres-role isolation remains a distinct, separate, not-yet-started task)*
- ⬜ Connect GitHub repos and Basecamp to the MCP server *(not started — Supabase is the first system connected; GitHub/Basecamp/analytics are the next tools to add)*
- ⬜ Add team members to GitHub org and Vercel for collaborative editing *(not started)*

### Phase 3 — Medium Term
- Build and encode the first client strategic briefs
- Launch the Conversational Interface for the first client (editing, querying, drafting)
- Build the deterministic brand-enforcement check (hard block) plus model-based flagging
- Build the living client brief on top of the retrieval store
- Stand up the data-query path — language to real query to result, no invented numbers

### Phase 4 — Long Term
- Cross-site operational monitoring across TAE's own roster
- Inter-client learning — human-curated playbook first, anonymized aggregates later
- Competitive and market awareness feeds filtered by client strategy
- Autonomous operator mode, expanded incrementally as guardrails prove out
- White-label packaging — competing on strategy-encoding, not feature parity

### Operating & Maintaining It — The Work That Never Ends

Building ContextLayer is a project; running it is a practice. The roadmap above gets it stood up — but a strategy-encoded system only stays valuable if the strategy stays current and the machinery stays healthy. These are the recurring responsibilities, not one-time tasks:

- **Keep the briefs current.** A strategic brief that goes stale quietly degrades every output built on it. As goals shift, campaigns conclude, and positioning evolves, the brief has to be updated — partly automatic, partly a deliberate human review on a cadence.
- **Keep retrieval sharp.** Retrieval quality is the system's quality. As the knowledge store grows, what gets retrieved for a given task needs periodic tuning so the AI keeps pulling the right context, not just more of it.
- **Review what the AI produces.** High-stakes deliverables and out-of-scope changes need human eyes. This is ongoing labor by design — and, per the cost section, the real scaling constraint. Staff for it deliberately.
- **Rotate secrets, review access.** Credentials get rotated on schedule, team access is granted and revoked as people come and go, and the security practices in Section 13 get a standing review — paired with the longevity fire drill.
- **Re-test the escape hatches.** Once a year, actually exercise the fallbacks: prove the provider swap still works, prove a backup restores, prove recovery is hours not weeks. A fallback plan untested is a fallback plan unproven.
- **Watch the cost line.** Token spend per client drifts as usage grows. Monitor it, keep model tiering and caching honest, and adjust per-tier prompt caps before a heavy user quietly erodes margin.

---

> The market will build tools that execute. TAE is building something that understands. That is not a head start. That is a different category entirely — and it gets more defensible every month.

*The Artist Evolution — ContextLayer — originally June 2026, updated July 24, 2026.*
