# Review — `era5-sl-icechunk.ipynb`

The pipeline is sound in outline: virtualise public ARCO-ERA5 NetCDF-3 files with VirtualiZarr,
commit the chunk manifests to Icechunk on your own S3, read back with xarray. Nothing is copied,
and the anonymous-HTTPS virtual chunk container is the right call.

Nine issues below, ordered by consequence. All nine are fixed in `era5_icechunk.py` — but note
that the fix for #1 changes what gets *written*, so an already-built store carrying wrong values
has to be rebuilt; the library cannot repair it in place.

---

## 1. Per-file `scale_factor` / `add_offset` are silently discarded — **wrong values** (critical)

Every daily ARCO-ERA5 file computes its own int16 packing from *that file's* min/max. Sampled
headers, read straight from the source files:

| date | variable | `scale_factor` | `add_offset` |
|---|---|---|---|
| 2020-01-01 | 2m_temperature | 1.538826e-03 | 268.888314 |
| 2020-04-15 | 2m_temperature | 1.755136e-03 | 258.958961 |
| 2020-07-01 | 2m_temperature | 1.756159e-03 | 264.698234 |
| 2020-10-03 | 2m_temperature | 1.659118e-03 | 262.536928 |
| 2020-12-31 | 2m_temperature | 1.432157e-03 | 267.908952 |
| 2020-01-01 | total_precipitation | 6.084143e-07 | 0.019935 |
| 2020-04-15 | total_precipitation | 7.121160e-07 | 0.023333 |
| 2020-07-01 | total_precipitation | 7.641124e-07 | 0.025037 |
| 2020-10-03 | total_precipitation | 5.212880e-07 | 0.017081 |
| 2020-12-31 | total_precipitation | 5.009144e-07 | 0.016413 |

Five sampled days → five distinct pairs, for both variables.

A Zarr array can carry only **one** `scale_factor`/`add_offset`. The notebook concatenates the
daily virtual datasets with

```python
xr.concat(daily_vds, dim='time', coords='minimal', compat='override', combine_attrs='override')
```

`combine_attrs='override'` keeps the **first** file's attributes, so every subsequent day's int16
values are decoded with January's packing. Applying the 2020-01-01 pair to 2020-07-01 raw
integers gives an error of **+11.1 K at the low end, +4.2 K mid-range, −2.8 K at the high end**.
There is no exception and the maps still look plausible — this is the dangerous kind of bug.

**Check your existing store** before trusting anything built from it:

```python
import fsspec, xarray as xr, pandas as pd
from era5_icechunk import open_era5, source_url

ds = open_era5("era5-sl-icechunk-v1")
t  = "2020-01-07T12:00"                      # any day that is NOT the first in the store
with fsspec.open(source_url(pd.Timestamp(t), "2m_temperature"), "rb") as f:
    direct = xr.open_dataset(f)
print(float(ds.t2m.sel(time=t).mean()) + 273.15, float(direct.t2m.sel(time=t).mean()))
```

**Fixes, in order of preference**

1. **Virtualise the analysis-ready store instead of `raw/`.** `ar/…-0p25deg-chunk-1.zarr-v*`
   is already float32 and uniformly chunked, so the problem does not exist. Best option if you
   do not specifically need the raw files.
2. **Keep the packing as data.** Strip `scale_factor`/`add_offset` from the virtual array's
   attributes so xarray does not CF-decode, and write the per-file values as two loadable
   coordinate variables on `time`; decode explicitly with `raw * sf.sel(time=…) + ao.sel(time=…)`.
   Stays zero-copy and is exact.
3. **Materialise** the affected variables (decode once, write real chunks). Correct, but you lose
   the virtual-store benefit.

**What `era5_icechunk.py` does** — option 2, on by default (`build_store(..., fix_packing=True)`):

`externalise_packing()` pops `scale_factor`/`add_offset` from each virtual array's attributes —
so xarray does no CF decoding at all — and writes that file's pair as two coordinate variables
along `time`, `t2m_scale_factor` / `t2m_add_offset`. Those *are* concatenated per timestep, so
nothing is overridden. `open_era5()` calls `apply_packing()`, which decodes with the per-timestep
values and then drops the helper coordinates. The chunk manifests are untouched, so the store
stays virtual and zero-copy.

Verified on a synthetic two-file case with the two real packing pairs from the table above:
the notebook's concatenation pattern gives **1.007 K** of error on day 2, the fix gives
**0.00087 K** (the int16 quantisation step itself). `apply_packing()` is a no-op on a store built
the old way, so it is safe to call unconditionally.

`build_store()` also logs every file's packing and raises a `RuntimeWarning` when the pairs
diverge, naming the variable and the count — so even with `fix_packing=False` the problem is
no longer silent.

**This cannot repair an existing store.** The wrong values are a decoding artefact of what was
committed, so the affected date range has to be rebuilt.

---

## 2. No precipitation variable (blocking, for a hydrological tool)

`VARIABLES` contained `2m_temperature`, `mean_sea_level_pressure` and the two 10 m wind
components. `total_precipitation` is available at exactly the same path and is what any
water-balance, flood or drought application needs:

```
raw/date-variable-single_level/{YYYY}/{MM}/{DD}/total_precipitation/surface.nc
```

Also available and relevant: `large_scale_precipitation`, `convective_precipitation`,
`mean_total_precipitation_rate`, `runoff`, `surface_pressure`, `land_sea_mask`.

**Convention to respect:** ERA5 `tp` is an accumulation **over the hour ending at the stamped
time**, in metres. Daily total = sum of the 24 hourly values × 1000 (mm). Never average it as
if it were a rate, and never resample it with `mean` when you want a total.

---

## 3. Unconditional recursive delete of the repository (high)

```python
if fs_proto.exists(f'{storage_bucket}/{icechunk_prefix}'):
    fs_proto.rm(f'{storage_bucket}/{icechunk_prefix}', recursive=True)
```

This runs on every execution of the cell. Re-running the notebook to extend the date range
destroys the whole store and its commit history first — the one thing Icechunk exists to give
you. It is also irreversible on object storage.

**Fix:** `build_store(..., append=True)` to extend along `time`; deletion only behind an
explicit `overwrite=True`, and `Repository.open_or_create` instead of `Repository.create`.

---

## 4. `os.environ['HOME']` (high — this fails on your machine)

```python
load_dotenv(f'{os.environ["HOME"]}/dotenv/protocoast.env', override=True)
```

`HOME` is not set on Windows (`USERPROFILE` is), so this raises `KeyError`. Use
`pathlib.Path.home() / "dotenv" / "protocoast.env"`, and treat a missing env file as
non-fatal so the credentials can also come from the ambient environment.

---

## 5. Virtual-chunk credentials are not re-attached on re-open (medium)

The verification cell works only because it reuses the in-process `repo` object created with
`authorize_virtual_chunk_access=credentials`. In a fresh session,

```python
repo = icechunk.Repository.open(storage, config)      # no authorize_virtual_chunk_access
```

opens fine and then fails at read time, when a virtual chunk is actually fetched — a confusing
place to discover it. `open_repo()` always passes it.

---

## 6. Serial virtualisation, one monolithic commit (medium)

`open_virtual_dataset` is called in a double `for` loop: `n_days × n_variables` sequential HTTPS
metadata requests. For a year × 5 variables that is 1825 round trips with no concurrency, and a
single commit at the end — one network hiccup at hour three loses everything.

**Fix:** `ThreadPoolExecutor` over days (`max_workers=8` is plenty; the bucket is fine with it)
and one commit per calendar month, so a long build is restartable and each snapshot is small.
For reference, the byte-range extraction used to build the web tool ran 17,568 concurrent range
reads against this bucket with zero failures.

---

## 7. `compat='override'` hides real mismatches (medium)

```python
xr.merge(var_vds, compat='override', combine_attrs='override')
```

`override` means "take the first, do not compare". If one variable's file were ever on a shifted
grid, had a different longitude convention (0–360 vs −180–180), or a short time axis, you would
get a silently mis-registered dataset. Use `join='exact'` and `combine_attrs='drop_conflicts'`
so a genuine inconsistency raises instead of being absorbed.

---

## 8. No handling of missing or short source files (low)

ARCO-ERA5 has gaps for some variable/date combinations, especially in the early years. A single
missing `surface.nc` aborts the whole loop. Collect failures, report them, and let the build
either skip or stop deliberately.

---

## 9. Plotting cell (low)

`hvplot(..., geo=True, tiles='OSM')` needs `geoviews` + `cartopy`; without them it fails with an
import error that reads like an hvplot problem. The data are on a plain lat/lon grid, so
`geo=False` is sufficient for a quick check. Also set `units` before plotting rather than after
converting, so the colourbar label follows the conversion.

---

## Note on alternatives in the same bucket

The bucket also holds analysis-ready Zarr stores spanning 1959–2022 (`ar/…240x121…`,
`ar/…64x32…`, `co/…`). They are genuinely useful for global or coarse work, but for a
0.25° regional application they are not a shortcut: both 1.5° stores are chunked
`[8, 240, 121]` — the **full global grid sits in every chunk** — so extracting one small AOI
over the full period means pulling the entire global record (~64 GB for the hourly one), and
the 64×32 stores place all of Italy in one or two cells. Reading the Italian latitude band
directly out of the `raw/` NetCDF-3 files with HTTP range requests costs 3.3 MB per daily file
instead of 50 MB, which is what the accompanying web tool does.
