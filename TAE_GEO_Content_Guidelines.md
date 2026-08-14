<!-- title: TAE GEO Content Guidelines -->

# TAE GEO Content Guidelines

*The Artist Evolution — added August 13, 2026. A writing pattern for future TAE-hosted client sites, not a tool — see `TAE_ContextLayer.md` Section 15, "Content-structure guidelines" for how this fits into the broader SEO/GEO checklist. This is copy discipline applied at build time, same as the site field convention is a code discipline applied at build time.*

## Why this exists

Search engines rank pages. AI answer engines quote sentences. A page can be well-optimized for one and still be a bad source for the other — the difference is whether a single sentence, lifted out of context, is still true, specific, and useful on its own. This doc is the checklist for that second thing when writing copy for a new client site.

## The six rules

**1. Lead with the answer, then explain.** The first sentence of any section should be the actual fact or claim, not a lead-in to it. An AI answer engine quoting a page grabs a short span of text — if the real answer is buried in sentence three, the quote either misses it or grabs something less useful.

**2. Phrase headings as real questions where it's natural.** People type questions into AI assistants close to how they'd ask a person. A heading like "How long has F.R.I. been in business?" is more likely to match a real query than "Our History" — and it forces the section under it to actually answer that question directly, which reinforces rule 1.

**3. Structure comparisons and lists as real lists, not prose.** Numbered steps, bullet points, and tables extract cleanly. A paragraph that buries three service offerings in a run-on sentence doesn't. If a section is naturally "here are the N things we do," format it that way — don't make an AI (or a person) parse it out of prose.

**4. Use concrete specifics over vague superlatives.** "Family owned since 1997" is a fact a model can quote and a reader can verify. "Trusted for decades" is not — it's unfalsifiable, and answer engines tend to under-trust unverifiable claims relative to specific ones. Real numbers, real dates, real named results beat adjectives every time.

**5. Make each sentence self-contained.** A sentence that only makes sense with the sentence before it ("This includes vendor setup, too." — includes *what*?) is hard to quote in isolation. Write claims so a single sentence, lifted out and shown alone, still reads correctly.

**6. Keep NAP and hours consistent everywhere they appear.** Name, address, phone, and hours should read identically in the visible page copy, the JSON-LD `LocalBusiness` structured data (see checklist item 6), and anywhere else they're mentioned. Inconsistency between these is a trust signal search engines and answer engines both penalize — and it's an easy thing to drift on a page that gets edited piecemeal over time.

## One structural constraint worth knowing, not a copy rule

If a future client site leans on client-side JavaScript to render its actual content (a React/Next app doing client-side data fetching, for instance), some AI crawlers will see a blank page — they don't all execute JavaScript the way a browser does. F.R.I.'s site is plain static HTML, so this isn't a live risk today, but it's worth flagging before a future client site gets built on a stack that renders content client-side only.

## Worked example, using F.R.I.'s own copy

**Before** (what a "why us" paragraph often looks like, written for a human skimming, not for extraction):

> We've been around for a long time and know the business inside and out. Our team has deep relationships and really understands what it takes to succeed at retail, which is why clients trust us with their most important launches.

**After** (same claim, restructured against the six rules):

> **How long has F.R.I. Marketing been in business?**
> F.R.I. Marketing has represented brands at Walmart and Sam's Club since 1997 — 29 years. The team is made up of former Walmart associates, not outside consultants, so buyer relationships and internal processes are things they've worked from the inside, not studied from the outside.

This is close to how the live site's actual "Why F.R.I." section already reads (`hero_badge`/`hero_subtext` fields, live-editable via ContextLayer) — it's a real example already applying most of these rules, not a hypothetical.

## When to use this doc

Reference this when writing (or reviewing) copy for a new client site's build, and when auditing an existing client site's copy as part of the "Site Health & SEO Hygiene Check" cadence task (`brand_web_presence` category, `contextlayer.cl_tasks`). Not a tool, not something ContextLayer enforces automatically — a discipline applied by whoever's writing the copy, same as it's applied here.
