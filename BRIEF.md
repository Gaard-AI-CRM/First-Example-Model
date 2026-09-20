# Build the Gaard Capital AI CRM

You are building a working, single-file, offline-capable CRM prototype for Gaard Capital. Read this entire brief before writing any code. Everything you need (client background, financial domain knowledge, feature specs, data model, design direction, and delivery requirements) is here. Do not ask clarifying questions; where something is unspecified, make the choice a careful senior engineer would make and note it in a short ASSUMPTIONS section at the end of your final message.

---

## 1. Who the client is

**Gaard Capital** is an OCIO (Outsourced Chief Investment Officer) firm registered as an RIA (Registered Investment Advisor). Four service lines:

1. Investment policy design (writing the Investment Policy Statement, or IPS, for a client)
2. Discretionary portfolio management (Gaard makes trades on the client's behalf under an agreed mandate)
3. Rebalancing and portfolio design (asset allocation, drift bands, periodic rebalancing back to target)
4. Investment reporting (performance and holdings reporting to boards, committees, and individuals)

Client base: **institutions** (endowments, foundations, nonprofits, family offices), **individuals** (high net worth households), and **401(k) plans** (plan sponsors, meaning the employer, plus their retirement committees).

**The problem.** Gaard runs their business development off a master spreadsheet with ~2,000 rows of contacts that grows by 30 to 100 contacts per week. It is edited by hand. Email correspondence is pasted into cells. Duplicates, stale rows, and human error are constant. They have tried expensive off-the-shelf CRMs (think Salesforce, Redtail, Wealthbox) and abandoned them because they were too complicated and the team stopped using them.

**What they want.** A custom, simple CRM that does five things extremely well and nothing else. It must:

- Migrate the spreadsheet into a real structured data model
- Be easy to use without any fear of "breaking" it
- Work **offline**. This is a hard requirement. Gaard does not want an AI with unrestricted access to their full client list because of data leak and liability risk. All "AI" behavior in this prototype must run locally in the browser with no network calls. See Section 4.
- Be shareable as a link so mentors (Darren Wesemann) and the Gaard contact (Jon Clifford) can click through it

This prototype is a **framework demo** for approval before full development begins. It is not the production system. Judge every decision by: does this make the demo clearer and more convincing?

---

## 2. Financial domain knowledge you must apply

Use this vocabulary correctly in the UI, sample data, and code comments. Anyone reviewing this works in wealth management and will notice mistakes.

**AUM (Assets Under Management).** The dollar value of assets a firm manages. For a prospect, "pipeline AUM" or "prospective AUM" is the estimated investable assets that would come under management if they onboard. Display in $M with one decimal for institutional (e.g. $42.5M), $K or $M for individuals. Never show raw unformatted numbers.

**Pipeline stages** (use exactly these, in this order):

1. `Lead` - contact captured, no conversation yet
2. `Contacted` - at least one outbound touch sent
3. `Engaged` - they replied or attended something
4. `Discovery` - a discovery call or meeting is scheduled or completed
5. `Proposal` - Gaard has sent an IPS draft, fee proposal, or engagement letter
6. `Onboarded` - signed advisory agreement, assets transferring or transferred (this is a **client**)
7. `Lost` - explicitly declined or went dark past the follow-up window
8. `Not a Fit` - disqualified (too small, wrong geography, conflict, etc.)

A "next step" means moving from any stage to a later non-terminal stage. "Converted" means reached `Onboarded`.

**Contact types / client segments:** `Institution`, `Individual`, `401(k) Plan`. Institutions and 401(k) plans belong to an **Organization** (the endowment, the company sponsoring the plan). Individuals may or may not have an org (a family office would; a retired doctor would not).

**Roles at institutions you will see in sample data:** CFO, Executive Director, Board Chair, Investment Committee Chair, Treasurer, Controller, Director of Finance, Plan Administrator, HR Director (for 401k), Trustee.

**Outreach channels** (the four the client named, use exactly these):
- `Cold Email`
- `Cold Call`
- `In-Person Event` (conferences, local nonprofit finance roundtables, chamber events)
- `Virtual Event` (webinars Gaard hosts or attends)

Plus one more: `Referral`, because in reality it is the highest-converting channel for RIAs and the demo should show it. Keep the four named channels prominent.

**Typical sales cycle.** Institutional OCIO deals take 6 to 18 months and often require an RFP and a board vote. Individuals take 1 to 3 months. 401(k) plan changes often happen at plan-year boundaries. Sample data should reflect this: institutional deals sit in `Discovery` and `Proposal` for a long time; individual deals move faster.

**Compliance framing (light touch, no lecturing).** RIAs are subject to SEC books-and-records rules, so logging correspondence is not just nice, it is a compliance asset. Mention this in one tooltip or one line on the Email Logging view. Do not build a compliance module.

**Fee context** (for sample data realism only): OCIO fees are typically 0.25% to 0.75% of AUM annually for institutions, higher for small individual accounts. You can show "Est. annual revenue" as a derived field = pipeline AUM × an assumed fee rate stored in settings. Keep the fee rate editable in the sample data config.

---

## 3. The five features (the entire scope)

Build these five, deeply. Do not add features beyond these. Every feature must be fully functional against the sample data, not a mockup with dead buttons.

### Feature 1: Smart Import

**Intent:** Paste or upload new contacts, the system catches duplicate contacts and duplicate organizations, and auto-sorts everything into the right place.

**Requirements:**
- Input via (a) a textarea where the user pastes CSV or tab-separated rows straight out of a spreadsheet, and (b) a file picker for `.csv`. Support a header row with flexible column names (map `email`, `e-mail`, `Email Address` all to email; same tolerance for name, org, phone, title, channel, AUM).
- **Duplicate contact detection**, local and deterministic:
  - Exact match on normalized email = definite duplicate
  - Fuzzy match on name + org (normalize case, strip punctuation, handle "Bob"/"Robert", "Jon"/"Jonathan", trailing "Jr.", etc. via a small nickname table; use a Jaro-Winkler or Levenshtein ratio with a stated threshold) = probable duplicate
- **Duplicate organization detection:** normalize legal suffixes (Inc, LLC, Foundation, Endowment, Fund, Trust, "The"), strip punctuation, compare with a similarity ratio. "Wasatch Community Foundation" and "The Wasatch Community Foundation, Inc." must be flagged as the same org.
- Show a **review screen before commit**: three groups (New, Probable Duplicate, Definite Duplicate). For each duplicate, show the existing record side by side with the incoming row and let the user choose Merge (field-by-field, incoming fills blanks only unless overridden), Skip, or Create Anyway. Nothing is written until the user clicks Commit Import.
- **Auto-sort** on commit: assign segment (Institution / Individual / 401(k) Plan) from org keywords and title keywords; attach to existing org or create a new one; set stage to `Lead`; set source channel from the row if present else prompt once for a default channel for the batch; stamp `created_at` and `import_batch_id`.
- After commit show a summary: X created, Y merged, Z skipped, with a link to the batch.

### Feature 2: Email Logging

**Intent:** All email correspondence is attached to the contact with the date. The contact record auto-updates with the latest correspondence.

**Requirements:**
- Each contact has a chronological correspondence thread. Each entry: `date`, `direction` (Outbound / Inbound), `subject`, `body`, `channel` (Email, Call Note, Meeting Note), `logged_by`.
- Log an email by (a) a form, and (b) a **"paste raw email" box** that parses `From:`, `To:`, `Date:`, `Subject:` headers and the body from a pasted email (Gmail and Outlook plaintext formats). Parsing is local, regex-based.
- **Auto-update the contact record** when a new entry is logged, deterministically:
  - `last_contacted_at` = date of latest outbound
  - `last_response_at` = date of latest inbound
  - `last_touch_summary` = first 140 chars of latest entry, cleaned
  - If an inbound entry is logged and stage is `Contacted`, auto-advance to `Engaged` and show a small toast explaining why
  - Scan body for signals with a local keyword ruleset and surface them as suggested field updates the user can accept or dismiss: new title ("I've moved into the CFO role"), new phone, new AUM mention ("our endowment is about $30 million"), a scheduling signal ("let's set up a call"), an explicit decline ("we've decided to stay with our current advisor"). Never auto-write these without a click; show them as chips on the contact page.
- One line of copy on this view noting that a complete correspondence log supports RIA books-and-records obligations.

### Feature 3: ROI by Outreach Type

**Intent:** Track which channel each contact came from and see how many from each channel moved to next steps or onboarded.

**Requirements:**
- Every contact has `source_channel` (one of the five channels) and optionally `source_event_id` if it was an event.
- A funnel table with one row per channel: Contacts Sourced, Reached Next Step (advanced past `Lead` at least once), Discovery Calls, Proposals, Onboarded, Conversion Rate (Onboarded ÷ Sourced), Pipeline AUM, Won AUM, and Est. Annual Revenue Won.
- A stacked horizontal bar or funnel visual per channel. Use a real charting library from a CDN (Chart.js or ECharts) or clean hand-rolled SVG. No placeholder images.
- Date-range filter (last 30 / 90 / 365 days / all) and segment filter (Institution / Individual / 401(k) / all).
- Events get their own sub-table: event name, date, type (in-person or virtual), attendees logged, contacts sourced, how many advanced, how many onboarded, AUM won.
- Every number must be computed from the contact and event data at render time. No hardcoded totals.

### Feature 4: Dashboard

**Intent:** Shows outreach activity, follow-ups due, pipeline AUM, event attendance and results. Must integrate with existing marketing metrics.

**Requirements:**
- Default landing view. Tiles in this order: Follow-ups Due Today, Follow-ups Overdue, Emails Going Out This Week (the client asked for this phrase specifically), Pipeline AUM by Stage, Outreach Activity (touches per week, last 12 weeks, split outbound vs inbound), Upcoming Events, Recent Event Results, Channel Conversion snapshot (pulls from Feature 3).
- Pipeline AUM by Stage as a horizontal bar with stage labels and $ totals; clicking a bar filters the contact list to that stage.
- **Marketing metrics integration.** Gaard's existing marketing dashboard is unspecified. Build a clearly labeled adapter: a `marketingMetrics` block in the sample data config with the shape `{ source: "Manual | HubSpot | Mailchimp | Google Analytics", period: "YYYY-MM", metrics: { website_visits, newsletter_sends, newsletter_open_rate, newsletter_click_rate, webinar_registrations, linkedin_followers } }` and a tile row that renders whatever keys are present. Add a code comment above the adapter that says exactly which function to replace with a fetch call when a real source is connected. The demo should make it obvious that plugging in their real feed is a one-function change.
- Every tile is clickable and drills into the relevant filtered view.

### Feature 5: Auto Follow-Ups

**Intent:** The system tells you who to follow up with and when, based on their last response. Ability to draft generic outreach emails for each follow-up.

**Requirements:**
- A deterministic **follow-up rules engine**, stored as an editable table in the sample data config, not buried in code. Default rules:
  - Stage `Contacted`, no inbound reply: follow up at +5 business days, then +10, then +21, then mark `Lost` (surface as a suggestion, do not auto-mark)
  - Stage `Engaged`, last touch inbound: follow up within 2 business days
  - Stage `Discovery`: follow up 3 business days after last touch
  - Stage `Proposal`: follow up 7 business days after proposal sent, Institution; 3 for Individual
  - Stage `Onboarded`: quarterly check-in (90 days)
  - Any contact whose last touch was inbound and unanswered: overdue immediately
- Compute `next_follow_up_at` and `follow_up_reason` for every contact from the rules plus their thread. Show a Follow-Ups queue sorted by overdue first, then due today, then upcoming, with the reason in plain English ("No reply to your 9/8 email, second touch due").
- **Email drafting, fully local.** A template engine with merge fields (`{{first_name}}`, `{{org}}`, `{{last_topic}}`, `{{days_since}}`, `{{service_line}}`, `{{sender_name}}`) and a template per (stage × touch number × segment). Templates live in the sample data config and are editable in a Settings view. Clicking Draft opens the filled template in an editable box with Copy to Clipboard and Mark as Sent (which logs an outbound entry via Feature 2 and recomputes the follow-up). Include 8 to 10 well-written templates that sound like a real OCIO, not generic SaaS copy. Reference the four service lines naturally.
- Snooze (1 / 3 / 7 days) and Dismiss with reason.

---

## 4. Offline and "AI" constraints

- **Zero network calls at runtime** except loading CDN libraries at page load. No API keys. No fetch to any AI provider. Every "smart" behavior (dedupe, parsing, signal extraction, follow-up scheduling, drafting) is deterministic local logic. This is a feature, not a limitation: state it in the UI footer as "Runs entirely in your browser. No client data leaves this device."
- Architect for a future opt-in AI layer: put all local intelligence behind a small `Intelligence` module with functions like `findDuplicates()`, `parseEmail()`, `extractSignals()`, `scheduleFollowUp()`, `draftEmail()`. Each has a docblock saying what a future LLM-backed version would do differently and what data it would need to see. Do not implement the LLM version.
- Persist state to `localStorage` under one namespaced key, with a Reset to Sample Data button in Settings that is guarded by a confirm. Wrap every storage read and write in try/catch and render correctly if storage is unavailable.

---

## 5. Data model

Implement as plain JS objects. Keep it small.

```
Organization { id, name, normalized_name, segment, website, city, state, notes, created_at }
Contact {
  id, org_id (nullable), first_name, last_name, title, email, phone,
  segment, stage, stage_history: [{stage, at}],
  source_channel, source_event_id (nullable),
  pipeline_aum, won_aum (nullable),
  last_contacted_at, last_response_at, last_touch_summary,
  next_follow_up_at, follow_up_reason, follow_up_touch_number, snoozed_until,
  tags: [], import_batch_id, created_at, updated_at
}
Correspondence { id, contact_id, date, direction, channel, subject, body, logged_by, signals: [] }
Event { id, name, date, type, location, attendees_logged, notes }
ImportBatch { id, at, created, merged, skipped, default_channel }
Settings { sender_name, firm_name, assumed_fee_rate, followUpRules: [], templates: [], marketingMetrics: {} }
```

---

## 6. Sample data

- **All sample data lives in one clearly marked block at the top of the file** (`const SAMPLE_DATA = {...}`) so a non-engineer can edit names, numbers, or add rows without touching logic. Put a comment above it: "Edit this block to change the demo data."
- Generate the sample data with a seeded deterministic generator (so the demo is identical every load) but keep the generator's inputs (name lists, org lists, AUM ranges) in that same top block. Alternatively, hand-write it. Either way, editing must be obvious.
- Volume: ~120 contacts, ~45 organizations, ~6 events (3 in-person, 3 virtual), ~400 correspondence entries spread across the last 14 months, 3 import batches.
- Distribution that tells a story: Referral converts best, In-Person Events second, Cold Email sources the most contacts but converts worst, Cold Call in between, Virtual Events cheap and mid. Roughly 12 to 15 `Onboarded`. Institutions have larger AUM and longer stage histories. Make sure there are at least 6 overdue follow-ups and 4 due today relative to the real current date (compute dates relative to `Date.now()` so the demo never goes stale).
- Use Utah and Mountain West geography (Salt Lake City, Park City, Provo, Ogden, Boise, Denver). Org names should sound real and local: community foundations, universities, hospital systems, credit unions, regional manufacturers with 401(k) plans, family offices. Use fictional names; no real organizations or people.
- Include 3 planted duplicate pairs in the sample data that Smart Import would catch if re-imported, and a sample CSV string in the config that the user can load into Smart Import with one click ("Load example import") which contains 2 exact dupes, 3 fuzzy dupes, 1 duplicate org with different suffix, and 6 genuinely new contacts.

---

## 7. Design direction

The reviewers are finance people and a mentor evaluating craft. The bar is "looks like a product a $500M RIA would pay for", not a hackathon demo.

- Restrained, dense, professional. Think Bloomberg-adjacent calm rather than consumer SaaS. Muted neutral surface, one accent color, semantic colors reserved for status (overdue, due, won). No gradients, no glassmorphism, no emoji in the UI.
- Typography: a single high-quality sans from Google Fonts (Inter or IBM Plex Sans) plus tabular figures for every number. Right-align numeric columns.
- Left sidebar nav with the five features plus Contacts and Settings. Top bar with global search (name, org, email) that is fast against 2,000 rows.
- Tables are the primary surface: sortable columns, sticky header, row hover, keyboard navigable, virtualized or paginated so 2,000 rows are fast.
- **Light mode only, with dark chrome.** The content surface is light: off-white page background, white cards and tables, near-black text. The structural elements are dark: a charcoal or navy sidebar and top bar with light text, dark primary buttons, dark table headers, dark KPI tile headers with the big number in white, and dark chart marks (ink-toned bars and lines, one accent for the highlighted series). The contrast between the dark frame and the light working area is the look. No theme toggle and no `prefers-color-scheme` switching; define all colors as CSS custom properties on `:root` so a dark mode can be added later without touching components.
- Desktop first. This will be reviewed on laptop and desktop screens, so design for 1280px to 1920px wide and use the width: multi-column dashboard grid, wide tables with many visible columns, side-by-side compare panes in Smart Import, a split contact view (record on the left, correspondence thread on the right). It should degrade gracefully down to about 1024px, but do not spend effort on phone layouts.
- Empty states, loading states, and confirms for anything destructive. Toasts for every write with an Undo where cheap.
- Accessibility: proper labels, focus rings, contrast at least 4.5:1, charts have a text alternative table.

---

## 8. Technical requirements

- **One self-contained `index.html`** file. Vanilla JS with ES modules inline, or Preact/React via CDN if you judge it clearly worth it. CSS inline. Only external loads: one Google Font, optionally one charting library from cdnjs.cloudflare.com or cdn.jsdelivr.net/npm/. Under 2 MB total.
- Clean module structure inside the file, in this order: config and sample data, data layer (store, persistence, derived indexes), Intelligence module, rules engine, views, router, boot. Comment each section header.
- Derived values (funnel numbers, follow-ups, AUM totals) are computed from source records on every render via memoized selectors. Never store derived totals.
- Dates: store ISO strings, compute in local time, business-day math for follow-ups (skip weekends; a small US holiday list for the current year is a bonus).
- Performance target: any view renders in under 100 ms with 2,000 contacts. Test this by temporarily bumping the generator to 2,000 and profiling.
- No console errors or warnings. No dead buttons. Every control does something.

---

## 9. Delivery and verification

1. Write `index.html` and open it in the browser. Walk through every feature yourself: run the example import and confirm the dupe groups are correct; paste a raw email and confirm headers parse and the contact record updates; check that the ROI table sums match the contact counts; click every dashboard tile; draft, edit, and Mark as Sent a follow-up and confirm it moves out of the queue. Fix anything broken before moving on.
2. Test at 1440px wide and at 1024px wide. Confirm the dark sidebar, top bar, and tile headers hold at least 4.5:1 contrast against their light text.
3. **Publish it as a Claude Artifact** so it has a shareable link. Load the `artifact-design` skill first, follow its page contract, then publish. The title should be "Gaard Capital CRM". Give me the link.
4. Final message, in this order: the artifact link, a 10-line "how to demo this" walkthrough for the person presenting it, a list of what is intentionally out of scope for the prototype, and the ASSUMPTIONS section.

Do not use em dashes anywhere in the UI copy, templates, comments, or your messages.
