# ⚡ ElectriBid — MI & OH Electrical Construction Bid Aggregator

Dashboard that aggregates open electrical construction bids from Michigan and Ohio government procurement portals.

## How it works

```
scrape_bids.py  →  bids.json  →  electrical-bids.html
   (Python)         (data)          (dashboard)
```

The scraper pulls from public-facing pages of:
- **Michigan:** U-M AEC, MDOT Bid Letting, DTMB SIGMA VSS, MSU IPF, BidNet/MITN
- **Ohio:** OFCC, ODOT, OSU, Franklin County, BidNet/Ohio

The HTML dashboard checks for `bids.json` on load. If found, it uses live data. If not, it falls back to a static snapshot.

## Quick start (local)

```bash
pip install -r requirements.txt
python scrape_bids.py          # creates bids.json
open electrical-bids.html      # or use any local server
```

For the HTML to load `bids.json`, serve the directory (browsers block `file://` fetch):

```bash
python -m http.server 8000
# then open http://localhost:8000/electrical-bids.html
```

## Free automated scraping (GitHub Actions)

1. Push this repo to GitHub
2. Go to **Settings → Pages** → set source to `gh-pages` branch
3. Go to **Actions** tab → enable workflows
4. The scraper runs every 6 hours automatically (free on public repos)
5. Click **"Run workflow"** for an immediate scrape
6. Your live dashboard is at `https://yourusername.github.io/electribid/`

GitHub Actions free tier gives 2,000 minutes/month — this uses ~1 minute per run × 4 runs/day = ~120 min/month, well within limits.

## Files

| File | Purpose |
|---|---|
| `electrical-bids.html` | Dashboard (open in browser) |
| `scrape_bids.py` | Python scraper |
| `bids.json` | Scraped data (auto-generated) |
| `requirements.txt` | Python dependencies |
| `.github/workflows/scrape.yml` | GitHub Actions auto-scrape |

## Limitations

- Several portals (DTMB SIGMA VSS, BidNet, BuildingConnected) require vendor login — the scraper can't access individual bids there, so those appear as "standing" aggregator entries linking to the portal
- MDOT and ODOT publish letting details as PDFs — the scraper gets what it can from HTML pages and falls back to standing entries
- The scraper checks what's publicly visible; for full bid documents and drawings you'll still need to register on each portal

## Adding more sources

Edit `scrape_bids.py` and add a new `scrape_*()` function. Return a list of dicts matching this schema:

```python
{
    "title": "Project name",
    "sub": "Description",
    "location": "City",
    "state": "MI",  # or "OH"
    "status": "open",  # or "closing"
    "deadline": "2026-03-15",  # ISO date, or "Varies"
    "value": "$1M–$3M",
    "valueSortable": 3000000,
    "posted": "2026-02-01",
    "source": "Portal Name",
    "url": "https://...",
    "drawings": "yes",  # "yes", "reg", or "tbd"
    "drawingsNote": "Where to get drawings"
}
```
