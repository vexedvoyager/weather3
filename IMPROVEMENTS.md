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

### 5. Document the "CLI" naming discovery and cross-check resource

**Why:** the user found https://www.clilax.com/ during research, which
confirms "CLI" stands for NWS's official "Climatological Report" product
- explaining exactly why Kalshi's settlement text uses identifiers like
"CLIMDW" rather than the plain ICAO code (the exact mismatch fixed in
`src/stations.py`). The site also independently confirms the "Local
Standard Time climate day" settlement methodology, which matches
`weather_day.py`'s existing DST-handling logic - good external
validation, not something requiring a code change itself.

The same operator also runs equivalent live-tracking pages for NYC and
Chicago (`clilax.com/nyc`, `clilax.com/chicago`) - two more of the 5
configured cities. Worth keeping as a manual cross-check resource once
real trades start settling, and worth reviewing again once (if) the
Herbie/GRIB2 migration (item #4) is scoped, since it describes a
"precision ladder" of temperature data sources that may be relevant.

**Proposed fix:** add a short note to `src/stations.py`'s docstring
referencing this as the confirmed source for the "CLI" naming
convention. Purely documentation - no logic change needed.

**Priority:** low, easy. The user is adding this site to the project's
context folder separately.

### 6. Two settlement/market-type edge cases from Kalshi's official rulebook (low priority)

**Why:** found while reading `Kalshi_Global_Temperature_Terms.pdf` (added
to the project's context folder), Kalshi's own official "GLOBALTEMPERATURE"
contract terms. Neither has caused a real problem - both are just gaps
the bot hasn't been tested against, worth a written record rather than
relying on memory of a PDF read once.

**Gap A — a third settlement outcome the database doesn't model.**
The rulebook states: *"If no data is available for [the time period] by
the Expiration Date, all strikes shall resolve to the last fair price as
determined in the sole discretion of the Exchange."* `db.py`'s
`settle_trade()` currently only accepts `outcome` values of `'yes'`,
`'no'`, or `'void'`. A "resolved to last fair price" settlement doesn't
cleanly map to any of the three - it's not a win, not a loss, and not a
stake-returned void either (it's an arbitrary exchange-determined price).
If this ever occurs on a real trade, `settle_trade()`'s assertion would
reject the unrecognized outcome value outright, which is a safe failure
(loud error, not silent misclassification), but the code has no
deliberate handling for it.

**Gap B — an unsupported market type ("exactly").** The rulebook defines
comparison operators as `<above/below/exactly/at least/between>`. Every
real market seen so far (across all 5 cities, in live Forecast Refresh
logs) has been `greater`/`greater_or_equal`/`less_or_equal`/`between` -
never "exactly" (equal to a value, rounded to one decimal). `extract_threshold()`
in `src/market_parsing.py` only recognizes the four strike types already
observed; an "exactly" market would return `None` and be silently (and
correctly, per the existing fail-safe design) skipped, not mispriced.

**Why this is unlikely to matter:** temperature markets are inherently
continuous-valued, and "exactly" only makes sense for something with a
small number of discrete possible outcomes - it doesn't fit the
above/below/between ladder structure Kalshi actually uses for weather.
Neither gap has appeared in any real data pulled so far.

**Priority:** low. Recorded for completeness, not because either is
expected to be hit. If Gap A ever occurs, it'll surface as a loud,
diagnosable error (per the settle_trade assertion) rather than silent
misbehavior - consistent with this project's overall design philosophy.

### 7. Fix the 401 Unauthorized on /portfolio/positions

**Why:** the first real Price Check run that actually placed trades also
revealed a real, separate bug: `get_positions()` (used for the daily
position-mismatch consistency check) failed with `401 Client Error:
Unauthorized`, while `/markets` calls in the very same run, using the
same signing code, succeeded normally. This points to a permissions/scope
issue on the Kalshi API key itself - most likely it's currently scoped
to market-data read access only, not portfolio access - rather than a
bug in the RSA-PSS signing logic (which is clearly working correctly for
other endpoints).

**Why it didn't block trading:** `price_check.py` is deliberately built
to fail safe here - a positions-fetch error is logged and treated as "no
mismatch detected" rather than aborting the run. That's the right
default behavior for an error, but it also means the consistency check
is currently NOT actually able to verify anything, silently. It's
running with zero real protection until this is fixed.

**Proposed fix:** check the Kalshi account's API key settings for a
permissions/scope option and grant portfolio/positions read access. If
Kalshi's key model doesn't support partial scoping, this may just need
a fresh key. Re-run Price Check afterward and confirm the error is gone
before ever considering live mode - the consistency check is one of the
two conditions that can trigger a same-day alert, and it needs to
actually work for that safety net to mean anything.

**Priority:** high — this is a real safety-net gap, not a cosmetic issue,
and now that trades are actually flowing, it matters.

---

### 8. Watch for overconfident high-probability trades at the deployment stage

**Why:** the first real batch of trades included several where the model
expressed 95%+ confidence while buying at 1 cent (implying the market
priced the same outcome near 0-1%) - for example Miami T87 (95.1% model
vs ~1% market), Austin T95 (98.3% vs ~1%), and LA T72 (96.7% vs ~1%).
This is either a genuinely rare, large mispricing, or it's the exact
overconfidence failure pattern described in the source material's own
post-mortem ("Confident and Wrong: Why Our Model's High-Confidence
Trades Lost Money") - where the model's 90%+ confidence bucket was
specifically where most of the real losses concentrated, despite the
model looking best-calibrated in aggregate.

**Not a bug, no code change proposed yet.** This is exactly what the
self-audit Brier tracker (added this round) exists to catch once these
specific trades settle - particularly whether the high-confidence bucket
specifically underperforms, not just the aggregate Brier score.

**Priority:** watch, don't act. Revisit once enough of these
specific high-confidence trades have settled to say anything meaningful.
If a pattern like the source material's does emerge, the likely fix
would mirror theirs: a more conservative sigma_multiplier or an explicit
confidence cap, but that's premature before real settlement data exists.

---

### 9. Nearly the full budget deployed in a single Price Check run

**Why:** the first successful run deployed $48.42 of the $50 total
budget in one pass - 10 trades, hitting the 2-per-city position cap on
all 5 cities simultaneously, all priced off the same single cached
forecast snapshot. Harmless in paper mode, but worth thinking through
before live mode: real capital would deploy almost entirely on the
first successful run rather than gradually over time, which may not be
the risk profile intended.

**Proposed fix (not yet decided on):** possible options include a
per-run deployment cap (e.g. no more than $X or N trades per single
Price Check run, even if more are eligible), or spreading eligible
candidates across multiple runs deliberately. Needs a real decision, not
just a default - flagging for discussion rather than proposing a
specific fix now.

**Priority:** medium - doesn't matter for paper trading, but should be
resolved before ever switching `mode` to `"live"`.
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

