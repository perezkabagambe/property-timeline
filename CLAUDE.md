# Property Timeline — Project Context

## What this is
A single-file static site (`index.html`) tracking a remortgage + onward property
purchase, hosted on GitHub Pages at https://perezkabagambe.github.io/property-timeline/
No build step, no framework — plain HTML/CSS/JS in one file. One lazy-loaded
external dependency (Tesseract.js, for the screenshot OCR feature) — see below.

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
- `properties` — viewing tracker cards (schools, status, photos)
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
`address`). On add/edit, `ensurePropertySchools` resolves a location for the
property in priority order: (1) the entered `postcode`, geocoded via `postcodes.io`;
(2) failing that, a full postcode extracted from `address` via regex; (3) failing
that, the whole `address` string sent to Nominatim (OpenStreetMap's free geocoder)
to handle partial Rightmove/Zoopla-style addresses like "Mithras Gardens, Wavendon
Gate, Milton Keynes MK7" that only carry an outward code. Whatever postcode gets
resolved from (2) or (3) is written back into `p.postcode` with `p.postcodeGuessed
= true`, shown to the user as a small "auto-detected" note, and cleared back to
`false` the next time they save the edit form (whether or not they changed it) —
that's what "confirms" a guess. Nominatim calls are serialized through
`queueNominatim` with a ~1.1s gap between them to respect its usage policy; this
matters most during `backfillAllPropertySchools`, which does the same resolution
for every existing property on load and after a Drive sync merge. Changing
`p.postcode` (confirming or correcting a guess) always invalidates and recomputes
`p.nearestSchools` — the cache key is the resolved postcode string. Two sortable
tables (primary above secondary) render in a `.prop-schools-panel` beside the
card, inside a `.prop-row` flex container — replaced the old `.props-grid`
card-only layout.

This replaced an earlier, deliberately-manual Snobe.co.uk-based design (Snobe's ToS
prohibits automated scraping of their rankings). The DfE data above is a different,
openly-licensed source, so that restriction doesn't apply here — but the Snobe ToS
restriction itself still stands if anyone ever proposes pulling from Snobe directly.

**Property statuses:** Property of Interest (default for new entries) / Viewing
Scheduled / Viewing Complete / Offer Made / Offer Rejected / Offer Accepted —
a dropdown per property, shown as a colored badge.

**Automated property data extraction** (the "Automated Property Data Extraction"
panel at the top of the Add/Edit property form) lets the user paste, drop, or
upload a screenshot of a listing and pulls a photo crop + address/price
suggestions out of it — entirely client-side, no server, no request to
Rightmove/Zoopla ever made (the image is one the user already captured, so this
doesn't implicate the scraping-ToS restriction above). Two independent pieces:
- **Photo**: a hand-rolled canvas crop tool (`extractCanvas` + pointer events,
  no cropping library) — the user drags a box over the screenshot, "Use
  selection as photo" resizes/compresses it exactly like the existing file-
  upload path and sets `pendingImageData`, so it flows through the normal save
  logic unchanged.
- **Address/price**: OCR via **Tesseract.js**, lazy-loaded from a CDN
  (`loadTesseract`) only when an image is first provided — this is the one real
  external dependency in the app beyond Google's own sign-in script. Extracted
  text is pattern-matched (`PRICE_RE`, `guessAddressLine`, and a priority check
  against `UK_POSTCODE_RE`) into suggestion chips the user must click "Use" to
  apply — OCR is never auto-applied to the form fields, since accuracy on
  arbitrary screenshots is inherently best-effort, not reliable enough to trust
  silently.

`resetExtractPanel()` runs on both opening and closing the property modal so
stale image/suggestion state never leaks between add/edit sessions.

## Deployment
- Deployed file must be named `index.html` at the repo root (GitHub Pages
  serves it automatically).
- No CI/build — edits are committed straight to the file that's served.
- Test after any change to the Drive sync or storage code: it's easy to
  silently break persistence without an error surfacing.
- The `.app-footer` "Build:" timestamp at the bottom of the page is a plain
  hardcoded string (no real build step to generate it from) — bump it to the
  current date/time whenever committing a change to `index.html`.

## Working conventions
- Keep everything in the one file unless the user asks to split it up.
- New persisted fields should degrade gracefully for existing stored data
  (default to empty/blank rather than assuming a field exists) — the user has
  real data in Drive that shouldn't be lost across feature additions.
- Match the existing blueprint/dark-navy aesthetic (IBM Plex Mono + Source
  Serif 4, `#0F1B2D` background, cyan-ish gridlines) — see `:root` CSS
  variables at the top of the file for the palette already in use.
