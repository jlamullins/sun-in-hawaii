# Where's the Sun on Kaua'i?

A single-page weather site that answers one question: which side of Kaua'i has
the best shot at sun right now?

`index.html` is the whole site. No build step, no dependencies, no API key,
no server. Open it in a browser and it works.

## Data

Forecasts come from the U.S. National Weather Service (`api.weather.gov`),
office **HFO** (Honolulu). It is free, needs no key, and sends
`access-control-allow-origin: *`, so the browser calls it directly.

Per town the page makes two requests:

| Endpoint | Gives us | Gzipped |
|---|---|---|
| `/gridpoints/HFO/{x},{y}/forecast/hourly` | hourly temp, `shortForecast`, precip probability | ~4.5 KB |
| `/gridpoints/HFO/{x},{y}` | `skyCover` (6h/12h blocks on HFO) | ~5.7 KB |

Five towns ≈ **50 KB gzipped** for the whole island, which is what keeps this a
purely static page instead of needing a serverless proxy.

Grid coordinates are baked into `TOWNS` so a normal load skips the `/points`
lookup. If NWS re-grids and a baked URL 404s, `fetchTown()` falls back to live
`/points` resolution automatically.

### Deriving conditions

`shortForecast` text leads; `skyCover` refines it by at most one step.

Doing it the other way round misreads Hawai'i: trade-wind cumulus keeps HFO's
`skyCover` in the 70–85% range on days that read — and feel — "mostly sunny",
so a sky-cover-first rule would paint the whole island cloudy nearly every day.

Five categories map to the five icon variants already drawn in the mockup:
`sunny`, `partly`, `cloudy`, `rain-light`, `rain-heavy`.

### Refresh strategy

Fetch on load, then every 15 minutes, plus on tab re-focus if the data is more
than 10 minutes old.

NWS caches hourly forecasts for about an hour (`s-maxage=3600`), so most
15-minute refreshes are free CDN hits rather than new model runs. Client-side
polling was chosen over static regeneration because the page has no build step
to regenerate and the payload is small enough that a phone can just ask
directly — it also means the forecast is never staler than the visitor's own
last refresh.

### When data is missing

Failures are handled per town, never silently hidden:

1. **Fresh data** — normal render.
2. **Fetch fails, cached value exists** — shows the last known reading with
   `· last known` appended, and names the towns in the footer status line.
   The cache is `localStorage`, trimmed to a 48h horizon (~49 KB).
3. **Fetch fails, nothing cached** — the row reads `Forecast unavailable` with
   a `—` temperature. If every town fails, the hero says so outright.
4. **`skyCover` alone fails** — degrades to text-only categorisation; the town
   still renders.

During the first load rows read `Checking…`.

## Typography note

Fraunces' latin-ext subset mispositions the kahako (macron), so `index.html`
borrows only the ten macron vowels from a system serif via `@font-face` +
`unicode-range`. Every other glyph still resolves to Fraunces.

## Local preview

```bash
python3 -m http.server 4173
```

Then open <http://localhost:4173>.

## Deploy — GitHub Pages

Chosen because the site is one static file: no build, no env vars, no
serverless runtime, and the free tier is permanent.

```bash
git remote add origin https://github.com/<you>/wheres-the-sun-kauai.git
git branch -M main
git push -u origin main
```

Then in the repo: **Settings → Pages → Source: Deploy from a branch →
`main` / `root` → Save**. The URL appears there after a minute or so.

There are no environment variables or secrets to configure.

Vercel or Netlify work identically if you prefer them — import the repo and
accept the defaults; leave the build command empty and the output directory
set to the repo root.
