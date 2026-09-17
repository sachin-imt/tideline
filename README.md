# Tideline

Daily valuation corridors for a book of 22 AI, semiconductor and mega-cap stocks, published as a dashboard and sent as a plain-English email brief.

**Dashboard:** https://sachin-imt.github.io/tideline/

Each stock gets a score for where its price sits in the range it normally trades in: 0 is the cheap edge, 100 the expensive edge. The corridor behind that range **rises every day** as earnings accrue. A stock whose price doesn't move therefore gets cheaper day by day, like a tide coming in under a boat. That daily rise is what the name refers to.

> This is a valuation gauge, not investment advice. It only knows where a price sits against the range that stock usually trades in. It knows nothing about the news or the business.

## How it works

**Corridor.** For each stock, the corridor is next-twelve-months (NTM) earnings per share multiplied by the range of forward P/E multiples the stock has actually traded at. The five bands are the median and ±1σ and ±1.5σ around it. Tideline tracks two lookbacks, 90 days and 12 months.

**Daily accrual.** NTM EPS for date *t* is each quarter's EPS weighted by how much of that quarter falls inside the window *[t, t+365]*. As the window slides forward, it takes in a slice of a later quarter and drops a slice of an earlier one. That's how earnings accrue, and the corridor rises with them.

**Cockpit.** Each stock is also plotted on implied upside (x) against PEG (y) and sorted into four quadrants: Upside + Inexpensive, Upside + Expensive, Downside + Inexpensive and Downside + Expensive. A single ratio moves a stock on both axes:

```
r      = (price / refPrice) × (refMedian / medianToday)
upside = (1 + iu) / r − 1
PEG    = peg × r
```

It's calibrated so the reference date (8 Sep 2026) reproduces the published Cockpit exactly. After that date, a falling price and a rising median both push a stock toward Upside + Inexpensive.

**Estimates.** Each stock starts on a frozen analyst estimate and switches to market consensus once that company next reports, since the report is what makes the old estimate stale. The switch is made per stock, as each report lands. A sanity gate holds any switch that would move EPS by more than 25%, or imply a forward P/E outside 3–250, for a human to decide. Every other switch applies automatically.

## The daily run

GitHub Actions (`.github/workflows/daily-update.yml`) runs the whole pipeline in the cloud:

| When (UTC) | What |
|---|---|
| 22:00 daily | Capture prices and estimates, rebuild the dashboard |
| 07:07 Tue–Sat | Same, then email the brief (17:07 Sydney on AEST) |

```
fetch-prices.js       Yahoo Finance daily closes (rolling 45 sessions)
fetch-estimates.js    Finnhub consensus: forward quarters + reported history
apply-estimates.js    switch stocks that have reported; hold anything suspicious
build-eps-series.js   daily NTM accrual curve
update-data.js        bands, quadrant snapshots
build.js              inject data into tideline.html → docs/index.html
briefing.js           today's changes → pipeline/data/briefing-latest.json
send-brief.js         email it (never twice for the same session)
```

The brief caps itself at 15 changes and works through stocks in market-cap order, so the largest stocks always make the cut. Move alerts fire at 2% for the ten mega caps and 4% for the rest.

## Running it locally

Needs Node 20+. The pipeline uses Node's built-in modules only, so there is nothing to install.

```bash
cp .env.example .env         # then fill in your own values
npm run capture              # full pipeline
npm run brief:preview        # print the email without sending
npm run serve                # dashboard at http://localhost:8765
```

| Variable | Used for |
|---|---|
| `FINNHUB_KEY` | Consensus estimates (free tier is enough) |
| `SMTP_USER` / `SMTP_PASS` | Sending the brief; for Gmail, `SMTP_PASS` must be an App Password |
| `BRIEF_TO` | Recipient |

In CI, set the same four as repository secrets. If the SMTP variables are missing, the send step skips cleanly instead of failing.

## Known limitations

- **Three forward quarters, not four.** Finnhub's free tier returns three. The fourth is extrapolated from the quarter-to-quarter trend and flagged `derived` in `estimates.json`.
- **Consensus, not conviction.** Once a stock switches, its corridor rests on market consensus, and an analyst's edge comes from disagreeing with consensus. The method carries over; that edge doesn't.
- **TSM** reports in TWD against a USD-priced ADR, so it retires from coverage at its next report rather than switching to consensus.
- **XPEV** is loss-making. A corridor built on negative earnings inverts, so XPEV stays out of scope.
- GitHub can delay scheduled workflows under load, sometimes by hours.

## Credit

The Corridor Method and the Cockpit layout are modelled on AJ Investment Research's published methodology. AJ's last published Cockpit and Details views are archived at [`docs/archive/2026-09-08.html`](https://sachin-imt.github.io/tideline/archive/2026-09-08.html), and every calibration starts from them.
