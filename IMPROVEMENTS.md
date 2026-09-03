# Improvements for Next Version

A running list of things to fold into the next rebuild — gathered from
real usage, not speculative "nice to haves."

---

## Open items

### 1. Verify the `KXHIGHLAX` series prefix for Los Angeles

**Why:** best-guess extrapolation from the naming pattern of the other
four cities, never confirmed against Kalshi's live API.

**Proposed fix:** once Forecast Refresh has run a few times, check the
logs for "found ZERO markets" warnings for Los Angeles specifically. If
persistent, find the real prefix via Kalshi's `/series` endpoint.

**Priority:** medium — fails safe (skips the city) if wrong.

---

### 2. Monitor whether the 5-minute Price Check schedule is actually reliable

**Why:** GitHub's cron scheduling is documented as best-effort even at
its minimum 5-minute interval.

**Proposed fix (only if this turns out to be a real problem):** an
external pinger (e.g. cron-job.org) calling GitHub's API to trigger the
workflow, bypassing GitHub's own scheduler reliability.

**Priority:** low until observed data says otherwise.

---

### 3. GitHub Pages dashboard

**Why:** a visual dashboard (real P&L chart, open positions table,
per-city breakdown) would be a nicer daily check-in than reading GitHub
Issue comments. User has confirmed interest in this as the next
feature once the underlying data pipeline is confirmed working.

**Priority:** next up, once v3.0's fixes are confirmed live.

---

### 4. Migrate to Herbie for NBM data fetching

**Why:** Herbie (https://herbie.readthedocs.io) is a real, actively
maintained open-source package built specifically to download weather
model data - including NBM - from NOAA's various sources. It has a
genuinely valuable built-in solution to the hardest remaining unverified
piece of this project: `pick_points`, a proper haversine/BallTree-based
nearest-neighbor accessor that finds the correct grid cell for any
station on NBM's curvilinear grid without manual coordinate-transform code.

This is a direct instance of the "stop building, use what exists" lesson
from the source material - handwritten bulletin-fetching code has now
caused two of this project's most serious bugs (wrong bulletin type in
v1→v2, wrong hostname in v2→v3). A maintained library used by real
practitioners is less likely to have these specific failure modes.

**What's confirmed:**
- A real, working example URL for NBM's GRIB2 (binary) files exists and
  was independently verified across multiple Herbie documentation versions
- `pick_points` genuinely solves the curvilinear-grid nearest-station problem

**What's NOT yet confirmed (why this wasn't built into v3.0):**
- The exact GRIB2 variable name / search string needed to extract
  percentile temperature fields from NBM's "qmd" (quantile-mapped)
  product files specifically - a different, unverified detail from the
  core "co"/"core" files that have a confirmed working example
- Whether `cfgrib`'s system dependency (`eccodes`, a C library, not a
  pure Python package) installs cleanly and quickly enough on a GitHub
  Actions runner to be worth the added workflow complexity and runtime

**Proposed approach when this is picked up:** scope a small, standalone
proof-of-concept first (similar to the connectivity-check pattern used
for the hostname fix) - install cfgrib/herbie in a throwaway workflow,
fetch one real qmd file, and confirm the percentile field names and
values look sane - before committing to a full migration.

**Priority:** medium - a meaningful quality improvement, but real
unverified surface area remains. Worth doing once v3.0's simpler fix is
confirmed stable in production.

---

## Decisions (things considered and deliberately not done)

**Dropped the backtest feature entirely (in v2.0).** After finding that
NOAA doesn't retain the needed forecast bulletin archive beyond about a
week for free, and that the best available substitute (a university-run
archive) only guarantees roughly the same window, the feature's original
value proposition wasn't achievable as designed. The v3.0 self-audit
Brier tracker is the direct replacement - it validates against the bot's
own accumulating trade history instead of external historical data.

**Did not add more cities alongside the v3.0 hostname fix.** Deliberately
isolated this change to one variable at a time - confirm the fix
actually produces real trades with the existing 5 cities before adding
more surface area. Revisit once the fix is confirmed working live.

**Did not migrate to Herbie in v3.0.** See item #4 above - real
unverified surface area (qmd percentile field names, eccodes install
feasibility) remains; the hostname fix alone already meaningfully
de-risks the pipeline without taking on that additional scope in the
same change.

---

## Log of resolved issues (for context, not action)

**From v2.0 setup (GitHub/workflow mechanics):**
- `.github`, `.gitignore`, `data/.gitkeep` are dotfiles invisible during
  drag-and-drop uploads unless hidden files are shown — documented in
  `QUICKSTART.md`
- `git add data/alerts/` failed on an empty/nonexistent folder — fixed
  with `mkdir -p` + placeholder file before the add
- `gh issue create --json` isn't a valid flag — fixed by parsing the
  plain URL output instead
- `KALSHI_ID_KEY` vs `KALSHI_KEY_ID` naming mismatch — user-side typo;
  reminder that GitHub always displays secret names in uppercase
  regardless of how they're typed, which can mask other typos

**From v2.0 pre-build verification (found before any code shipped):**
- v1 fetched the wrong NBM bulletin (NBH instead of NBP) — root-cause fixed
- NBM bulletins are fixed-width with intentionally blank columns; naive
  whitespace-splitting silently misaligned values — fixed with
  column-position-based parsing, covered by regression tests
- Kalshi's documented production base URL is `external-api.kalshi.com`,
  not the `api.elections.kalshi.com` alias v1 used — fixed
- NBP's publish schedule (as understood at the time) — later found to
  need further correction in v3.0 (see below)
- No handling for Kalshi's 429 rate-limit responses — added exponential
  backoff, tested

**From v3.0 (found via a real production incident, not pre-build review):**
- v2.0's NBM fetch used `blend.nomads.ncep.noaa.gov` — a hostname that
  does not exist (confirmed DNS resolution failure in a live Forecast
  Refresh log). Every single run failed at the first network call from
  launch. Root-cause fixed: correct host is `nomads.ncep.noaa.gov`.
- NBP's true publish schedule is 01, 07, 13, 19 UTC only (4x/day) — not
  the 6-value schedule v2.0 used, which had mixed in two extra hours
  misattributed from a different NBM product. Confirmed twice,
  independently, against NOAA's own product page.
- The daily summary had no way to distinguish "scans are running but
  finding nothing eligible" from "the pipeline is silently broken
  upstream" — this exact ambiguity is what made the v2.0 bug hard to
  diagnose from the outside. Fixed with scan-total and forecast-cache
  diagnostics surfaced directly in the summary.

