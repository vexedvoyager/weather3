# Predict Weather Bot v3.0

An automated weather-market trading bot for Kalshi, built to run for free
on GitHub Actions — no server, no Python installation, no coding
required to operate it day to day.

**New here? Start with [`QUICKSTART.md`](QUICKSTART.md)** — a click-by-click
setup guide that assumes no GitHub or Python experience.

**This is speculative software, not financial advice.** Read
[`KNOWN_LIMITATIONS.md`](KNOWN_LIMITATIONS.md) before ever switching out
of paper-trading mode. You can lose the full amount you deploy.

**Migrating from v2.0?** This is a fresh, separate repo — v2.0 had a
critical bug (a nonexistent NOAA hostname) that meant it never fetched a
single real forecast, despite passing all local tests. Nothing in a v2.0
repo's trade history is worth carrying forward.

---

## What's new in v3.0

- **Fixed a critical, previously-invisible bug**: v2.0's NBM fetch used
  `blend.nomads.ncep.noaa.gov` — a hostname that doesn't exist. Every
  single Forecast Refresh failed at the first network call from day one,
  meaning Price Check was always comparing prices against an empty
  cache. v3.0 uses the confirmed correct host (`nomads.ncep.noaa.gov`)
  and a corrected NBP publish schedule (01/07/13/19 UTC, not the
  6-value schedule v2.0 used).
- **Resilient, honest URL fetching**: the exact folder structure for
  NBM's text bulletins couldn't be fully verified from the environment
  this was built in. Rather than ship a second unverified guess, v3.0
  tries multiple plausible path patterns per candidate run and logs
  which one actually works.
- **New: a standalone connectivity check** (`Check NBM Connectivity`
  workflow) — run this manually, first, before trusting Forecast Refresh.
  It has no trading logic attached; it just reports plainly which URL
  pattern (if any) actually works, straight from GitHub Actions' network.
- **New: daily summary diagnostics** — surfaces total markets scanned,
  eligible candidates, and per-city forecast-cache coverage, so "why is
  nothing happening" is answerable from the summary alone. This exact
  gap was what made the v2.0 bug hard to diagnose from the outside.
- **New: self-audit Brier score tracking** — the direct, working
  replacement for the backtest dropped in v2.0. Computes the bot's own
  Brier score against the naive base rate directly from settled paper
  trades, with the same SKILL / NO SKILL verdict methodology the source
  material's own team uses.

---

## What this does, in one paragraph

Every 5 minutes, it checks current Kalshi weather-market prices against
a forecast-derived probability (refreshed 6x/day, matching NOAA's actual
publish schedule) — and, if the gap between model and market is large
enough and a few quality checks pass, opens a small position. It starts
in **paper mode** (simulated, no real money) and stays there until you
deliberately switch it. Each morning, it posts a plain-English summary to
a GitHub Issue you can read without touching any code.

---

## Where to find things

| I want to... | Go to... |
|---|---|
| Set this up for the first time | [`QUICKSTART.md`](QUICKSTART.md) |
| Understand what a phrase in my daily summary means | [`TROUBLESHOOTING.md`](TROUBLESHOOTING.md) |
| Know what this can't do yet, or what's unproven | [`KNOWN_LIMITATIONS.md`](KNOWN_LIMITATIONS.md) |
| See what's planned for the next version | [`IMPROVEMENTS.md`](IMPROVEMENTS.md) |
| Adjust cities, budget, or risk settings | `config.yaml` (every line has a comment) |

---

## What's included

- **Weather bot** for 5 cities (Chicago, New York, Miami, Austin, Los
  Angeles) — configurable in `config.yaml`
- **Paper and live trading modes**, one line to switch, hard budget caps
  in either mode
- **Daily summary** posted automatically as a GitHub Issue comment,
  including a quick-glance P&L sparkline and (v3) scan/cache diagnostics
- **Self-audit Brier score tracking**, surfaced in the daily summary once
  enough trades have settled
- **Same-day alerts** (a separate GitHub Issue) if a position mismatch or
  the daily loss limit occurs
- **A standalone NBM connectivity check**, run manually, before trusting
  the automated pipeline
- **70 automated tests**, including dedicated regression coverage for
  the fixed-width column parsing bug and the wrong-hostname bug found
  while building this project

---

## Running tests locally (optional — only if you want to verify changes)

```bash
pip install -r requirements.txt pytest
pytest tests/ -v
```

You don't need to do this to use the bot day-to-day; it's only relevant
if you (or I, on your behalf) change the underlying code.
