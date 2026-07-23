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
Gate, Milton Keynes MK7" that only carry an outward code. `geocodeFreeText` retries
with progressively shorter queries (drops one trailing comma-separated segment at
a time via `geocodeFreeTextOnce`) if a query returns nothing — confirmed by hand
that Nominatim can fail an address outright when it carries extra trailing context
together (e.g. "62 Hawley Drive, Leybourne, West Malling" matches nothing) even
though a shorter version of the same address matches immediately ("62 Hawley
Drive, Leybourne" alone does) — so never assume a single Nominatim miss means the
address is truly ungeocodable. Whatever postcode gets
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

Postcode isn't the only thing that can make cached results stale, though — the
nearest-N counts, ranking logic, or an embedded dataset can also change without
any property's postcode changing (e.g. bumping primary/secondary schools from 5
to 10). `LOCATION_DATA_VERSION` exists for exactly that: `hasFreshLocationData`
requires `nearestSchools`/`nearestStations`/`nearestHospitals` to carry a
matching `version` stamp, not just a matching postcode, before treating them as
fresh. Bump this constant whenever changing what/how many get computed — every
property then recomputes once automatically on next load via the existing
backfill, no manual re-save needed. `p.imd` isn't versioned since nothing about
its computation is parameterised the same way.

This replaced an earlier, deliberately-manual Snobe.co.uk-based design (Snobe's ToS
prohibits automated scraping of their rankings). The DfE data above is a different,
openly-licensed source, so that restriction doesn't apply here — but the Snobe ToS
restriction itself still stands if anyone ever proposes pulling from Snobe directly.

**Nearest train stations** — same pattern, third table under the two schools
tables. `window.STATIONS_DB` (~115KB, its own `<script>` tag right after
`SCHOOLS_DB`) holds 2,639 Great Britain rail stations — name, postcode, lat/lon,
positional arrays again (`meta.cols`). Source: DfT's **NaPTAN** (National Public
Transport Access Nodes), filtered to `StopType=RLY` + `Status=active`, deduped by
normalized name (the raw dataset has a handful of legitimate multi-node stations
like Clapham Junction, plus ~9 Elizabeth Line stations whose RLY record had null
coordinates — patched from their paired RSE/entrance record; see chat history for
which). NaPTAN has no postcode field, so postcodes here are *reverse*-geocoded
from NaPTAN's own surveyed coordinates via `postcodes.io`'s bulk `geolocations`
endpoint — the inverse of the forward geocoding used elsewhere. `applyNearestSchools`
computes the nearest 3 into `p.nearestStations` (same `nearestFromDb` helper,
same cache-by-postcode invalidation as schools) whenever it resolves a property's
location, so this rides on the exact same postcode-resolution pipeline above —
no separate lookup path to maintain.

**Index of Multiple Deprivation (IMD)** — fourth block under stations, `p.imd =
{postcode, rank, decile, country}`. No embedded dataset needed here: `postcodes.io`
already returns `index_of_multiple_deprivation` (a rank, not a decile) on every
postcode lookup it's already doing for geocoding, so `geocodePostcode` just
captures it alongside lat/lon. `imdDecileFromRank` converts rank → decile via
`ceil(rank / (32844/10))` — verified directly against the official IoD2019 File 1
data (gov.uk) that this reproduces its published Decile column exactly, zero
mismatches across all 32,844 English LSOAs. **England only, deliberately**:
England/Wales/Scotland/Northern Ireland each run independent, non-comparable IMD
scales (confirmed in postcodes.io's own schema docs), and 32,844 is only the
correct denominator for England's — a postcode outside England shows a "not
available outside England" message rather than a wrong or misleading number.
The Nominatim free-text path resolves coordinates without ever calling
`api.postcodes.io/postcodes/{postcode}` (it uses Nominatim + a coordinate-based
reverse lookup instead), so `applyNearestSchools` makes one extra `geocodePostcode`
call to backfill the IMD rank in that case — cached by postcode, so it doesn't
repeat on subsequent loads.

**Nearest hospitals** — renders beside the IMD block (`.near-row`, a flex row
inside the panel — the one exception to every other block stacking full-width).
`window.HOSPITALS_DB` (~44KB, its own `<script>` tag after `STATIONS_DB`) holds
750 English hospitals — name, postcode, lat/lon, same positional-array convention.
Source: **CQC's locations directory** (`cqc.org.uk/about-us/transparency/using-cqc-data`,
Open Government Licence, freshly dated CSV/ZIP regenerated regularly — re-fetch
that page for a current link rather than reusing an old dated URL), filtered to
`Service types` containing `Hospital` and deduped by name. That CQC tag is a
regulatory category, not a strict definition of "hospital" — it's a good-enough
proxy (spot-checked, mostly NHS trust/community/private hospitals) but a handful
of edge cases (e.g. a specialist clinic) may be tagged Hospital too. Deliberately
excludes the separate `Hospitals - Mental health/capacity` tag (698 more) — out
of scope for "nearest hospital for something like an emergency," which is the
assumed use case; revisit if that assumption is wrong. `p.nearestHospitals` is
computed in `applyNearestSchools` exactly like stations (top 2 instead of top 3),
same cache-by-postcode invalidation, same `hasFreshLocationData` gate.

**Flexbox gotcha for `.near-row`**: its children need `min-width:0` (not just a
small `min-width`) and the table inside needs `table-layout:fixed` — without
both, the compact table's intrinsic content width fights the flex-basis and the
two blocks wrap onto separate lines instead of sitting side by side, even when
there's clearly enough space. Verified via computed `getBoundingClientRect()`
(same `top` on both blocks), not a screenshot — this session's screenshot tool
was unreliable (returned stale/blank frames repeatedly), so prefer geometry
checks via `javascript_exec` over trusting a screenshot when verifying layout.

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
  external dependency in the app beyond Google's own sign-in script.

  A **Rightmove/Zoopla source toggle** (`extractSourceToggle`, defaults to
  Rightmove) drives **positional** extraction as the primary strategy:
  `guessAddressPositional` picks a fixed line offset from the *end* of the OCR
  output, because the real listing info block is always the tail of the text —
  the photo is always at the top of the screenshot, so any noise Tesseract
  hallucinates out of its texture (a real, common failure — fragmented
  letter-like garbage from brick/tile/foliage patterns) always precedes it.
  Content-based heuristics alone (`isPlausibleTextLine`'s length/punctuation/
  vowel checks) turned out to *not* be reliable enough on their own — garbled
  OCR output can still accidentally look word-shaped enough to pass a generic
  quality filter, so position is now load-bearing, not just a tiebreaker:
    - Rightmove (from the end): `[-3]` address, `[-2]` price qualifier, `[-1]` £ amount.
    - Zoopla (from the end): `[-1]` address, `[-2]` bed/bath count, `[-3]` filler, `[-4]` £ amount.
  The Rightmove offsets are confirmed against real screenshots (including a
  real garbage OCR dump a user pasted back). **The Zoopla offsets are only an
  interpretation of a user's verbal description, not yet confirmed against a
  real Zoopla screenshot** — if a user reports it's picking the wrong line,
  get an actual raw-OCR-text dump (the "Show all extracted text" toggle in the
  panel) from them and recalibrate the offset, the same way the Rightmove one
  was fixed.
  `isPlausibleTextLine` still gates the positional pick (and remains the
  fallback path via `guessAddressLine` if position fails or a source wasn't
  matched) and `guessPriceLine` scans for the £ amount from the end backward
  for the same reason. Suggestions are never auto-applied to the form fields —
  the user must click "Use" — since accuracy on arbitrary screenshots is
  inherently best-effort, not reliable enough to trust silently.

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
