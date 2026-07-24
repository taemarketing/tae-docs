# TAE Docs

Reference documentation that spans more than one TAE tool — so it doesn't
belong inside any single tool's own repo. Tool-specific docs (like
tae-discovery's interview logic) stay in that tool's own `docs/` folder;
this repo is for things that describe the whole system.

- **TAE_Supabase_Database_Audit.md** — every table in the shared Supabase
  database, which tool owns what, security findings and fix status.
- **TAE_Tools_Setup_Reference.md** — every TAE tool, what it does, how it
  deploys, where its data comes from.

Both are meant to stay current, not be one-time snapshots — update them
directly rather than starting the investigation over next time something
changes.
