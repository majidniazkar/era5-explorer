# ERA5 Explorer

Interactive temperature and precipitation over Italy at selectable resolution in **time** and
**space**, served as a static site, plus the notebooks that build the data behind it.

**Live site:** `https://majidniazkar.github.io/era5-explorer/`

---

## Running it

Live at https://majidniazkar.github.io/era5-explorer/ — pushing to `main` redeploys
via `.github/workflows/pages.yml`.

Locally: `python -m http.server 8000`, then http://localhost:8000. Opening
`index.html` from the filesystem does not work — `file://` blocks the `fetch`
calls that load `data/`.

`.nojekyll` must stay in the repo, or Jekyll drops files from `data/`.

---

## What is in the repository

```
index.html                 the whole application (43 KB, no dependencies, no CDN)
data/manifest.json         grid, coastline, presets, and a pointer to each array
data/*.bin.gz              gzipped byte-shuffled int16 arrays (~4.1 MB total)
notebooks/01_build_icechunk_store.ipynb    build the virtual Icechunk store
notebooks/02_extract_and_bake_site.ipynb   regenerate data/ for any region or year
notebooks/03_explore_live.ipynb            widgets over the live store, any region
CODE_REVIEW.md             review of the original era5-sl-icechunk.ipynb
```

`index.html` is written to work either way: if `data/manifest.json` is present it fetches the
arrays at load time (this repository), and if a payload is inlined into the page it uses that
instead — which is how the single-file offline build `era5_explorer.html` works. Same viewer code
in both.

---

## Using the site

| control | what it does |
|---|---|
| Variable | total precipitation or 2 m temperature |
| Temporal resolution | hourly → 3 h → 6 h → daily → 5-daily → weekly → monthly → seasonal → annual |
| Temporal statistic | sum / mean / min / max (sum is the default for precipitation) |
| Spatial resolution | 0.25° → 0.5° → 1.0° → 2.0°, by cos(latitude)-weighted block mean **or** plain subsampling, so the aliasing difference is visible |
| Area of interest | drag a rectangle on the map, type W/E/S/N, or pick a catchment preset |

Tabs: animated map with hover readout, AOI time series (bars plus a cumulative curve for
precipitation), the aggregated table, and a **code** tab that emits the xarray/Icechunk snippet
reproducing whatever is currently displayed. Series export to CSV, map to PNG.

For precipitation with the `sum` statistic the statistics panel also gives the **water volume**
over the AOI in km³.

### Data

| | |
|---|---|
| source | ARCO-ERA5 `raw/date-variable-single_level`, public GCS, anonymous HTTPS |
| domain | 36.00–47.75 °N, 6.00–19.75 °E — 48 × 56 cells at 0.25° |
| hourly base | 1–31 Oct 2020, 744 steps (includes Storm Alex, 2–3 Oct) |
| daily base | 1 Jan – 31 Dec 2020, 366 steps |
| variables | `t2m` (K → °C), `tp` (m → mm, accumulation over the hour *ending* at the stamp) |
| extraction | HTTP range reads of the Italian latitude band only: 3.3 MB per daily file instead of 50 MB. 17,568 range reads, 0 failures, 0 missing values. |
| encoding | int16 at 0.05 °C / 0.005 mm h⁻¹ / 0.05 mm d⁻¹, byte-shuffled + gzip |
| coastline | 0.5 contour of the ERA5 land–sea mask on the same grid — the model's coast, hence blocky |

All times UTC. Spatial means are cos(latitude)-weighted; cell areas use the exact spherical band
formula with R = 6371.0088 km. The AOI time series is the spatial mean taken on the native 0.25°
grid and *then* aggregated in time, so it does not move when you change the spatial-resolution
control — deliberate, so the two controls stay separable. The map and statistics panel do follow it.

Browser requirement: `DecompressionStream` (Chrome 80+, Firefox 113+, Safari 16.4+).

---

## Changing the region or the period

Edit the four numbers at the top of `notebooks/02_extract_and_bake_site.ipynb`:

```python
LAT_N, LAT_S = 47.75, 36.00
LON_W, LON_E =  6.00, 19.75
YEAR         = 2020
HOURLY_MONTH = 10
```

Run it, commit, push. It rewrites `data/` only — `index.html` is untouched, because the grid,
coastline, presets and about-text all come from `manifest.json`.

Budget: a full year of two variables is ~730 files, about 2.4 GB transferred (against 36 GB if
the files were downloaded whole) and roughly 20 minutes. A single month is about a minute.

---

## Notebooks

`01_build_icechunk_store.ipynb` builds the virtual Icechunk store the original notebook was
aiming at, with the `CODE_REVIEW.md` fixes applied. Two of those change results: precipitation is
included, and each source file's int16 packing is written as per-timestep coordinates instead of
array attributes — without that, everything after the first day decoded with the first day's
`scale_factor` (up to ≈11 K of error on temperature, silently). **A store built by the original
notebook carries those wrong values and has to be rebuilt.**

`03_explore_live.ipynb` is the same exploration as the website but against the store, so it is not
limited to the baked region and year. `panel serve notebooks/03_explore_live.ipynb --show`, or run
the cells in Jupyter.

The website itself needs none of this — it is static data plus one HTML file.
