<!-- title: TAE Site Field Convention -->

# TAE Site Field Convention

*The Artist Evolution — added August 13, 2026. The build-time convention for making a new TAE-hosted client site work with ContextLayer's content/brand editing and holiday pages from day one, instead of needing bespoke reverse-engineered regex per client the way F.R.I. Marketing's site does.*

---

## The problem this solves

`get_site_content`/`update_site_content` need a way to find a specific piece of copy or a brand value inside a site's real HTML. For F.R.I. Marketing (the first client onboarded), that mapping was hand-written: a person read the site's actual markup and wrote a regex per field, anchored on that site's specific tag/class structure (`contextlayer.site_content_fields`, one row per field, `selector_type = 'regex'`). That works, but it means every new client's site needs the same manual reverse-engineering before ContextLayer can touch it.

The fix: tag editable elements at build time, so extraction is generic instead of bespoke.

## The convention

Every element a future TAE tool might need to read or write gets a `data-tae-field="<key>"` attribute directly on it, using the standard key names below:

```html
<div class="hero-badge" data-tae-field="hero_badge">Est. 1997 · Bentonville, AR</div>
<h1 data-tae-field="hero_headline">Your <span class="accent">Retail</span><br>Unfair Advantage</h1>
<a href="mailto:you@example.com" data-tae-field="contact_email">Get in Touch</a>
```

Once a site is deployed with these attributes, `scan_site_fields` (a ContextLayer MCP tool) reads the live HTML, finds every `data-tae-field` occurrence, and inserts one `site_content_fields` row per key automatically — `selector_type = 'data_attr'`, no regex written by hand. Onboarding a new client's editable fields becomes running one tool, not writing code.

**One real limitation, stated plainly:** extraction finds the tagged element's closing tag by counting nested opens/closes of the *same tag name* — so a tagged `<div data-tae-field="...">` containing another `<div>` inside it won't extract cleanly (it'll stop at the first `</div>`, which is the inner one, not the outer one). Tag something with an element type unlikely to nest inside itself — `<h1>`, `<p>`, `<span>`, `<a>` — not a `<div>` wrapping other divs, when in doubt.

## The starter taxonomy

Not exhaustive by design — extending it later is "tag one more element, re-run the scanner," not a migration. These are the field keys worth using consistently across every future site, so cross-client tooling (anything that reads the same key from multiple clients) has a stable name to reach for.

### Brand
| Key | What it is |
|---|---|
| `brand_primary_color` | Main brand color (hex) |
| `brand_secondary_color` | Secondary brand color (hex), if the palette has one |
| `brand_accent_color` | Accent/highlight color (hex) — what F.R.I. calls its orange |
| `brand_accent_light` | Lighter variant of the accent, typically used on hover |
| `brand_logo` | Logo image — **needs a binary-asset upload path that doesn't exist yet** (see TAE_ContextLayer.md's "what's still not built"); text-based wordmarks don't need this |
| `brand_font_family` | Primary font, if it's not hardcoded per-element |

### Hero / Header
| Key | What it is |
|---|---|
| `hero_badge` | Small eyebrow text above the main headline |
| `hero_headline` | The main headline — may contain simple inline HTML (`<span>`, `<br>`) |
| `hero_subtext` | Supporting paragraph under the headline |
| `hero_cta_primary_label` / `hero_cta_primary_link` | Primary hero button text and destination |
| `hero_cta_secondary_label` / `hero_cta_secondary_link` | Secondary hero button, if there is one |
| `nav_logo_text` | Text-based wordmark in the nav, if not an image |

### Contact
| Key | What it is |
|---|---|
| `contact_email` | Special-cased already: matched by scanning every `mailto:` link's address, not a single tagged element, since a real contact email usually appears more than once on a page |
| `contact_phone` | Phone number |
| `contact_address` | Street address |
| `contact_hours` | Business hours — **this is the field the already-logged "Holiday Hours & Closure Sweep" cadence task needs to actually function**; tag it even if the current site doesn't display hours yet |

### Content sections
Named per the site's real structure, same pattern F.R.I. uses (`cta_headline`, `cta_subtext`, etc.) — there's no fixed universal list here since every site's sections differ. The convention is the naming *pattern* (`<section>_headline`, `<section>_subtext`, `<section>_body`), not a fixed set of section names.

### Footer / Closing CTA
| Key | What it is |
|---|---|
| `cta_headline` | Bottom-of-page call-to-action headline |
| `cta_subtext` | Supporting text under it |
| `footer_copyright_text` | Copyright line |

### Announcement banner (future capability, tag now if building the element anyway)
| Key | What it is |
|---|---|
| `announcement_banner_text` | The message shown in a site-wide banner |
| `announcement_banner_active` | Whether the banner is currently shown — this is the field a future "client posts closed on Facebook → propagate everywhere" capability would eventually write to |

## What this does *not* change

- `site_content_fields` rows with `selector_type = 'regex'` (F.R.I.'s original 8 fields) keep working exactly as before — this convention is additive, not a forced migration of existing fields.
- Holiday pages (`holidays/<key>.html`) are unrelated to this — they're full alternate files, not field extraction, so tagging doesn't apply to them.
- This only helps sites TAE controls the build of. It doesn't retroactively make a pre-existing, un-tagged site (or a client-built site TAE didn't create) work with generic extraction — those still need the old bespoke-regex approach if ContextLayer support is wanted for them.
