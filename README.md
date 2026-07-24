# TAE Docs

Reference documentation that spans more than one TAE tool — so it doesn't
belong inside any single tool's own repo. Tool-specific docs (like
tae-discovery's interview logic) stay in that tool's own `docs/` folder;
this repo is for things that describe the whole system.

- **TAE_Supabase_Database_Audit.md** — every table in the shared Supabase
  database, which tool owns what, security findings and fix status.
- **TAE_Tools_Setup_Reference.md** — every TAE tool, what it does, how it
  deploys, where its data comes from.
- **TAE_ContextLayer.md** — the full strategic vision for ContextLayer
  (planned hosted MCP server), converted from the original PDF explainer.
  Cross-tool by nature — it's the plan for something that will eventually
  connect to everything TAE has.
- **TAE_Discovery_System_Manual.md** — how to generate discovery links, what
  each interview produces, and how to manage entries. Named for the
  Discovery System but genuinely spans two tools — the interview engine
  lives in tae-discovery, but Section 4 (managing entries, the link
  generator) is entirely about the TAE Portal's admin UI. That's why it's
  here rather than filed inside tae-discovery alone.

All four are meant to stay current, not be one-time snapshots — update them
directly rather than starting the investigation over next time something
changes. Each has a matching card in the TAE Toolbox (bottom of the TAE
Dashboard admin page) linking straight to its file here on GitHub, so the
card always shows whatever's actually committed.
