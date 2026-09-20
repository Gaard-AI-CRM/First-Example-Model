# Gaard Capital CRM (prototype)

A single-file, offline-capable CRM prototype for Gaard Capital, an OCIO / RIA firm.
It demonstrates five features against generated sample data: Smart Import with
duplicate detection, Email Logging with record auto-update, ROI by Outreach Type,
a Dashboard, and rules-driven Auto Follow-Ups with local email drafting.

## Run it

Open `index.html` in any modern browser. There is no build step, no package
install, and no server. The only network requests are the Google Font at page load.

To develop with a local server (avoids localStorage restrictions on `file://` in
some browsers):

    python3 -m http.server 8765

then open http://localhost:8765/index.html

View example page from Claude: https://claude.ai/artifact/5XiH97MJaVRuKwhNXoAThK

Or try this link: https://gaard-ai-crm.github.io/First-Example-Model/

## Where things live

Everything is in `index.html`, in this order. Each section has a header comment.

| Section | What it holds |
|---|---|
| 1. Config and sample data | `SAMPLE_DATA`: names, orgs, events, follow-up rules, templates, marketing metrics, example CSV. Edit this to change the demo. The seeded generator that turns it into records follows it. |
| 2. Data layer | `Store` (single state object, versioned, undo), `Persist` (one localStorage key, every access wrapped), memoized selectors `S.*` |
| 3. Intelligence | `findDuplicates`, `parseCSV`, `parseEmail`, `extractSignals`, `scheduleFollowUp`, `draftEmail`, `classifySegment`. All deterministic and local. Docblocks state what an LLM-backed version would need. |
| 4. Rules engine | Business-day math with US holidays, follow-up scheduler driven by the editable rule table, contact refresh after every logged entry |
| 5. Views | UI primitives, charts (hand-rolled SVG), `Actions` (every write), then one object per screen in `Views` |
| 6. Router | Hash routes, e.g. `#/contacts?stage=Proposal`, `#/contacts/<id>`, `#/followups?filter=overdue` |
| 7. Boot | `Store.init()`, first render, `window.GaardCRM` debug handle (`GaardCRM.stress(2000)` loads a 2,000 contact book) |

Design tokens are CSS custom properties on `:root` at the top of the file.

## Conventions

- Derived values (funnel numbers, follow-ups, AUM totals) are never stored; they are
  computed from records through `S.*` selectors that memoize on `Store.version`.
- Every write goes through `Store.commit(label, mutator)` so it is undoable and persisted.
- Zero network calls at runtime. Keep it that way unless the client opts in.
- Dates are ISO strings; follow-up math uses local time and business days.
- No em dashes anywhere in copy or comments.

## Marketing metrics adapter

Replace `getMarketingMetrics()` (Section 5, right after the dashboard view) with a
fetch that returns `{ source, period, metrics: { key: number } }`. The tile renders
whatever keys are present; keys ending in `_rate` show as percentages.

See `BRIEF.md` for the full original build brief.
