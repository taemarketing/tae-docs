<!-- title: TAE Discovery System Manual -->

# Discovery System Manual

*The Artist Evolution — v1.0.5, June 2026. Internal reference. How the system works, what it produces, and how to use it. Converted from PDF to Markdown July 24, 2026 — content unchanged, so it stays easy to find and edit going forward instead of needing to be regenerated as a new PDF each time something changes.*

The TAE Discovery System is an AI-powered interview platform that replaces static intake forms with an adaptive conversation. It profiles prospects and clients, scores their marketing maturity, and delivers two outputs: a team intelligence brief for TAE and a branded report for the client. Everything runs on Claude Sonnet 4.6.

---

## Section 1 — Starting an Interview

### The Three Link Tracks

Every interview is started from a discovery link. Links are generated in the TAE Portal and come in three tracks. The track determines question depth, tone, and what context Claude has before the interview begins.

**Cold Link**
- For: new prospects with no prior relationship with TAE
- Questions: 5–8 · Covers Goals 1–10
- Tone: professional, welcoming, from scratch
- TAE name rule: must write "The Artist Evolution (TAE)" on first reference
- Use: embed on the TAE website, include in cold outreach emails

**Warm Link**
- For: referrals, prospects TAE is actively courting, or pre-call primers
- Questions: 12–15 · Covers Goals 1–20
- Tone: engaged, slightly collegial — assumes some context
- Extra: unlocks deeper strategy questions not asked on cold track
- Use: send directly before a scheduled sales call or to a warm referral

**Hot Link**
- For: existing contracted TAE clients — check-in sessions only
- Questions: 4–6 · Focused on what's changed, what's working, what's next
- Tone: collegial, peer-to-peer — no re-establishing basics
- Context: Claude reads the Client Profile and prior session data before asking anything
- Output: delta report + upsell signal flags for TAE
- Use: send quarterly or before a renewal/upsell conversation

### How to Generate a Link

1. Go to TAE Portal (`tae-portal-lilac.vercel.app/admin`) — Discovery System section
2. Select track: Cold, Warm, or Hot
3. Fill in Context Notes (optional on all tracks) and expand sections as needed
4. Add an internal label (e.g. "Major Wool Records – June Outreach")
5. Choose output: Link or Embed code
6. Click Generate — URL appears instantly, click Copy
7. Dashboard tracks: Clicks, Starts, and Completions per link

### Context Notes (All Tracks)

Every link — Cold, Warm, and Hot — includes a free-form Context Notes field. These notes are stored on the link and injected into Claude's seed context the moment the prospect submits their email — before the first question is asked.

- **Cold example:** link placed on homepage — prospect likely unfamiliar with TAE, keep intro framing broad, music industry focus
- **Warm example:** referred by Marcus Webb, interested in brand + website, Nashville music venue, ~$8k budget signal
- **Hot example:** returning client, Soul Kitchen Records, rebranding mid-year, adding video services
- Claude uses this context silently — it never reads it back or acknowledges it in conversation
- Combined with Basecamp Client Profile data when available — stacks intelligently

### Interview Directives

Warm and Hot links support an optional Interview Directives section with two fields Claude treats as hard instructions — not suggestions.

- **Goals:** specific things Claude should find out (e.g. "Determine if they have existing SVG brand assets")
- **Rules:** behavioral constraints (e.g. "Always ask about current website before wrapping up")
- Directives are collapsed by default — click the section header to expand
- Use sparingly — they override Claude's adaptive logic for those specific points

### Page Copy Customization

Every link has an optional copy layer that overrides the default interview landing page text. This lets TAE send a link that feels personally crafted — not a generic intake form.

- Editable fields: Headline, Subheadline, CTA Button text, Footer Note
- Leave any field blank to keep the default — only override what you need
- Live preview updates in real time as you type — right side of the generator card
- Preview mirrors the exact interview page design at reduced scale
- Customizations are stored on the link — prospects see them when the link is opened

### Embed Mode

The Link Generator supports two output formats. Toggle between them before clicking Generate.

- **Link:** a standard URL — use for email, DMs, or any direct share
- **Embed:** an iframe snippet — paste into a website or landing page to embed the interview inline

---

## Section 2 — The Interview Experience

### What the Prospect Sees

- Landing page: email address field + Terms of Service checkbox — both required before starting
- One question at a time — no long forms, no page breaks
- Question formats vary: free text, single choice, multi-choice, scale, tag cloud
- Multi-choice and tag cloud questions include an "Other" option — toggling it opens a text field for context not covered by the listed options
- Single choice and scale questions do not have an Other option — they route Claude's logic and need a clean signal
- Progress is tracked — prospects can see roughly where they are
- On completion: animated loading screen (20–40 seconds) then full report
- Report page: Marketing Intelligence Score, recommendations, next step CTA
- Optional asset upload at bottom of report (logos, brand files, photos, notes)

### Mid-Interview File Upload

During the interview, Claude may request a document upload if the prospect mentions an existing report, agency deliverable, competitive analysis, or anything else worth reviewing as part of the analysis.

- Upload panel appears inline — below the current question
- Accepts PDF, TXT, and CSV files up to 10 MB
- Text is extracted from the file and appended to Claude's context before the next question
- Claude uses the content to inform all scores, insights, and recommendations — without quoting it verbatim
- Uploading is optional, clearly labeled, can be skipped
- Only one upload is requested per interview
- The file is saved to TAE's secure storage and visible in the Assets tab of the dashboard entry
- A banner appears in the Team Brief tab when a document informed the analysis

### What Claude Does Between Each Answer

1. Receives the full transcript so far plus all instructions
2. Evaluates which goals haven't been covered yet
3. Generates the single sharpest question to cover the next most important gap
4. Decides when the interview is complete based on goal coverage, not a fixed question count

### 20 Discovery Goals

**Cold Track (Goals 1–10):** Identity, Business Stage, Development Priority, Core Pain, Digital Presence, Current Marketing Setup, Agency History, Budget Signal, Urgency Trigger, Campaign Interest.

**Warm Track adds (Goals 11–20):** Marketing Objective, Target Audience Depth, Competitive Position, Brand Foundation, Brand Perception, Core Services & Benefits, Proof Points, Past Marketing History, Tone & Brand Voice, Success Metrics.

---

## Section 3 — What Gets Generated

### The Two Outputs

Every completed interview produces two documents simultaneously. They are generated sequentially by Claude immediately after the final answer is submitted — the Team Brief first, then the Client Report, which inherits the MIS scores from the brief to ensure consistency.

**Output 1: Team Brief (TAE Internal — Never Share)**
- Marketing Intelligence Score: 5 dimensions, 0–100 each, max 500 total
  - Brand Presence, Digital Visibility, Content & Creative, Lead Generation, Campaign Intelligence
- Score rationale: one-line explanation per dimension
- Service Fit: each of TAE's 7 service lines rated High / Medium / Low with rationale
  - Retail Growth Strategy, Digital Advertising, Brand Activation, Influencer & Content, Audience & Demand Generation, Marketing Execution & Analytics, Retail Support
- Budget Signal: inferred range (low–high), confidence level, evidence quote
- Urgency Flag: None / Soft / Hard + trigger description
- Key Insights: 3–5 strategic bullets about this prospect
- Red Flags: 0–3 concerns to address before or during the sales call
- Recommended Opening: 2–3 suggested conversation starters for the TAE team

**Infinity Blue Partner Referral (Conditional)**

When the interview surfaces clear signals of Amazon/Walmart marketplace activity, DTC ecommerce, retail expansion, or international growth, the Team Brief includes an additional section flagging potential alignment with Infinity Blue — TAE's ecommerce and marketplace partner.

- Only appears when signals are explicit — never inferred speculatively
- Lists specific Infinity Blue service lines with one-sentence rationale per signal
- Services: Marketplace Management, Marketplace Advertising, DTC Ecommerce, Retail Expansion, International Growth
- TAE team uses this section to determine whether to involve Infinity Blue in the conversation

**Output 2: Client Report (Share-Ready)**
- MIS scores with progress bars — same dimensions, client-facing language
- "What Your Scores Reveal" — narrative paragraph contextualizing the scores
- Priority Recommendations: top 3–4 services with headline, why now, and outcome framing
- "What's Possible" — forward-looking vision paragraph
- Next Step CTA — directs the prospect to schedule a call with TAE

**Editing Outputs**
- The Team Brief is editable — the Client Report is not (the prospect has already seen it)
- "Edit Brief" button sits at the top of the Team Brief tab — only visible on that tab
- Edit MIS scores (0–100), rationale text, bullet lists, budget range, urgency level
- Changes save to the database — edits persist across sessions
- PDF downloads reflect the current saved state — always edit before downloading

---

## Section 4 — Managing Entries: TAE Portal — Discovery Reports

Discovery reports are accessed through the TAE Portal. The standalone Discovery Dashboard (`tae-discovery.vercel.app/admin`) is no longer the primary interface — all reports now route to the unified TAE admin.

**Access at:** `tae-portal-lilac.vercel.app/admin`

### Activity Feed
- Main dashboard shows a unified feed of all activity: discovery leads, support tickets, social analyses
- Filter by Discovery System to see only completed interview sessions
- New entries appear with a green indicator dot
- Click any entry to open the full report detail page

### Discovery Leads Page
- Full list of completed sessions at: `tae-portal-lilac.vercel.app/admin/discovery`
- Shows business name, urgency, industry, date, MIS score, budget, and top service fits
- Click any row to open the full entry detail

### Entry Detail Page
- Team Brief tab: full intelligence brief with MIS scores, service fit, budget, urgency, insights, red flags, opening questions, and Infinity Blue referral signals when applicable
- Client Report tab: the share-ready report with score bars, recommendations, CTA
- Transcript tab: every question and answer from the interview, in order
- Assets tab: files uploaded by the prospect — both mid-interview uploads and end-of-interview uploads
  - Mid-interview uploads show note: "Uploaded during interview for analysis"
  - Files with a Download button are saved to TAE's secure storage and can be retrieved
  - A green banner appears in the Team Brief tab when a mid-interview document informed the analysis
- Edit Brief button: sits inside the Team Brief tab — unlocks all fields for inline editing, save or cancel
- Create Basecamp Project button: one click creates the project, uploads Client Profile, posts session summary
- Archive button: removes from main list without deleting
- Downloads: Team Brief PDF (internal), Client Report PDF (share-ready), Transcript TXT — each appears on its own tab

### Link Generator

The Link Generator lives in the Discovery System section of the TAE Portal main admin page. Scroll down and click "Discovery System" to expand it.

- Select track: Cold / Warm / Hot
- Context Notes (all tracks): free-form field passed to Claude as seed context
- Interview Directives (collapsible): Goals and Rules fields — Claude treats these as hard instructions
- Page Copy (collapsible): headline, subheadline, CTA button, footer note — preview updates live
- Output mode: Link or Embed — toggle before generating
- Internal label: for admin tracking only, never shown to the prospect
- Generate: creates a unique URL with a `?lid=` parameter
- Link history table: all generated links with Clicks / Starts / Completions stats
- Copy URL button per row for quick sharing

---

## Section 5 — Project Management: Basecamp Integration

The Discovery System connects to TAE's Basecamp account and can create projects, post session summaries, and upload files — all from within the TAE Portal.

### What Happens When "Create Basecamp Project" Is Clicked

1. Searches Basecamp for an existing project matching the business name
2. Creates a new project if none found — named exactly as the business name
3. Generates a Client Profile document from the interview data
4. Uploads Client Profile to the project's Docs & Files vault
5. Posts a session summary to the project's Message Board
6. Saves the project ID and vault ID to the company record in the database
7. Button switches to "Open in Basecamp →" on success

### Client Profile Document (Auto-Generated)
- Contact name, business, industry, entry mode
- Full MIS scores with rationale per dimension
- Service fit ratings with rationale
- Budget signal range, confidence level, and evidence quote
- Urgency level and trigger
- Key insights and red flags

### Cost Per Interview

The system runs on Claude Sonnet 4.6 via the Anthropic API. Pricing: $3.00 per million input tokens / $15.00 per million output tokens.

- Cold interview (5–8 questions): ~$0.15 per completed session
- Warm interview (12–15 questions): ~$0.28 per completed session
- Blended average across all tracks: ~$0.20 per completed session
- File upload with PDF extraction: adds negligible cost — extraction is local, not billed

### Infrastructure Costs (Monthly Estimates)
- Supabase (database + file storage): Free tier covers early volume; Pro tier $25/month for production
- Vercel (hosting): Free tier sufficient at current scale; Pro $20/month recommended before client-facing launch
- Anthropic API: ~$10/month at 50 interviews
- **Total at 50 interviews/month: ~$55/month — approximately $1.10 per qualified lead profile**

---

*The Artist Evolution — Discovery System Manual — v1.0.5, June 2026.*
