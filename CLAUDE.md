# Property Timeline — Project Context

## What this is
A single-file static site (`index.html`) tracking a remortgage + onward property
purchase, hosted on GitHub Pages at https://perezkabagambe.github.io/property-timeline/
No build step, no dependencies, no framework — plain HTML/CSS/JS in one file.

## The real-world plan this app tracks
- Keeping the existing flat in Southend, remortgaging it (not selling) to fund the
  deposit on an onward purchase.
- Mortgage offer ported from an earlier, fallen-through purchase.
- Travel to Uganda 3 Sept – 1 Nov 2026 — no viewings, signings, exchange, or
  completion can happen in that window.
- Target onward purchase completion: end of November 2026.
- Key milestones (all editable in the app, not hardcoded truth): ported offer
  expiry (~1 Sept), remortgage offer expiry (~30 Sept, renewable), depart/return
  dates, target completion date.

## Architecture

**Single IIFE in a `<script>` tag.** Three independent data domains, each with its
own load/persist/render cycle:
- `tasks` + `milestones` — the Gantt chart (finance/search/legal/adhoc lanes)
- `properties` — viewing tracker cards (schools, status, photos, video)
- `todos` — simple checklist

**Persistence:** `window.storage.get/set(key, value, shared=false)`. This API only
exists natively inside Claude.ai's artifact preview — on real hosting (GitHub
Pages) it's shimmed on top of `localStorage` near the top of the script. Don't
remove that shim; without it, nothing persists outside of Google Drive sync.

**Google Drive sync (optional):** OAuth2 implicit flow via Google Identity
Services (`google.accounts.oauth2`), scope `drive.file` (app can only see files
it creates). On sign-in it looks for `property-timeline-data.json` in Drive,
merges `{tasks, milestones, properties, todos}` from it, and every local change
fire-and-forgets a push back to Drive. `GOOGLE_CLIENT_ID` near the top of the
script must have an Authorized JavaScript origin of exactly
`https://perezkabagambe.github.io` in Google Cloud Console — no path, no
trailing slash.

**Milestone stacking:** the "Key Dates" row uses a greedy bin-packing algorithm
(`computeMilestoneLevels`) so any number of close/same-date milestones stack
vertically instead of overlapping. Row height grows dynamically.

**Schools lookup — automated, backed by an embedded DfE dataset.** `window.SCHOOLS_DB`
(a ~1.3MB blob near the top of `index.html`, in its own `<script>` tag before the
main one) holds England's state-funded schools: ~14.8k primary (name, postcode,
lat/lon, national rank, KS2 rating) and ~3.3k secondary (name, postcode, lat/lon,
national rank, Progress 8, Attainment 8). Rows are plain arrays, not objects
(`meta.primaryCols` / `meta.secondaryCols` give the column order) — this was a
deliberate size optimization, don't refactor to keyed objects. Source data: DfE's
`compare-school-performance.service.gov.uk` (Open Government Licence) + GIAS for
postcodes; coordinates via `postcodes.io`. Regenerating this blob means re-running
the fetch/geocode pipeline described in chat history, not hand-editing it.
Full source citations and methodology live in `UK_School_Performance_Database.xlsx`
(the standalone deliverable this was built from — not deployed, just a reference).

Each property has a `postcode` field (own form input, separate from the free-text
`address`). On add/edit, `ensurePropertySchools` geocodes it via `postcodes.io`
(free, no key, CORS-friendly), computes Haversine distance against every row in
`SCHOOLS_DB`, and caches the nearest 5 primary + 5 secondary onto
`p.nearestSchools`. `backfillAllPropertySchools` runs the same thing for existing
properties on load and after a Drive sync merge — it tries to extract a postcode
from `address` via regex if `p.postcode` is blank. Two sortable tables (primary
above secondary) render in a `.prop-schools-panel` beside the card, inside a
`.prop-row` flex container — replaced the old `.props-grid` card-only layout.

This replaced an earlier, deliberately-manual Snobe.co.uk-based design (Snobe's ToS
prohibits automated scraping of their rankings). The DfE data above is a different,
openly-licensed source, so that restriction doesn't apply here — but the Snobe ToS
restriction itself still stands if anyone ever proposes pulling from Snobe directly.

**Property statuses:** Viewing Scheduled / Viewing Complete / Offer Made /
Offer Rejected / Offer Accepted — a dropdown per property, shown as a colored
badge.

## Deployment
- Deployed file must be named `index.html` at the repo root (GitHub Pages
  serves it automatically).
- No CI/build — edits are committed straight to the file that's served.
- Test after any change to the Drive sync or storage code: it's easy to
  silently break persistence without an error surfacing.

## Working conventions
- Keep everything in the one file unless the user asks to split it up.
- New persisted fields should degrade gracefully for existing stored data
  (default to empty/blank rather than assuming a field exists) — the user has
  real data in Drive that shouldn't be lost across feature additions.
- Match the existing blueprint/dark-navy aesthetic (IBM Plex Mono + Source
  Serif 4, `#0F1B2D` background, cyan-ish gridlines) — see `:root` CSS
  variables at the top of the file for the palette already in use.
