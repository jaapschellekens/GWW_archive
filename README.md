# Global Water Watch — reservoir surface water area timeseries archive

Satellite-derived surface water area timeseries for **71,208 lakes and
reservoirs** worldwide, 1985 to 2026, preserved from the Global Water Watch
platform before it was decommissioned.

[![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.22643338.svg)](https://doi.org/10.5281/zenodo.22643338)

> **Licence:** [CC BY 4.0](LICENSE) — except the reservoir outlines in
> `geometry_utf8`, which are OpenStreetMap-derived and remain under
> [ODbL 1.0](#reservoir-outlines-are-odbl). See [Licence](#licence).

## Why this archive exists

[Global Water Watch](https://www.globalwaterwatch.earth/) (Deltares, WWF and WRI)
published near-real-time surface water timeseries for small and medium-sized
reservoirs, derived from Landsat and Sentinel-2 imagery. The platform is being
shut down for resource reasons, and its API is going offline with it.

This archive is a complete snapshot of the reservoir timeseries served by that
API, taken in **September 2026** (TODO: confirm final date), so the record
remains usable after the service is gone. It is a preservation copy: the data is
reproduced as the API served it, with no reprocessing, filtering or gap-filling.

# Provenance

- **Source:** the Global Water Watch API, `https://api.globalwaterwatch.earth`
- **Retrieved:** September 2026 (TODO: confirm final date)
- **Method:** every reservoir listed by the API's `/reservoir` endpoint, with all
  timeseries from `/reservoir/{id}/ts`, reproduced without modification. Values
  and timestamps were verified against fresh API responses after writing.
- **Underlying imagery:** Landsat and Sentinel-2, processed on Google Earth Engine
  as described in the paper above.

# Acknowledgements

Global Water Watch was developed by **Deltares**, the **World Wide Fund for
Nature (WWF)** and the **World Resources Institute (WRI)**, with support from
Google.org, the Water, Peace and Security Partnership, and the European Space
Agency.

# Citation

If you use this data, please cite **both** the method paper and this archive.

**The method behind the timeseries:**

> Donchyts, G., Winsemius, H., Baart, F., Dahm, R., Schellekens, J., Gorelick, N.,
> Iceland, C., & Schmeier, S. (2022). High-resolution surface water dynamics in
> Earth's small and medium-sized reservoirs. *Scientific Reports*, 12, 13776.
> https://doi.org/10.1038/s41598-022-17074-6

**This archive:**

> [![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.22643338.svg)](https://doi.org/10.5281/zenodo.22643338)

BibTeX for the paper:

```bibtex
@article{donchyts2022reservoirs,
  title   = {High-resolution surface water dynamics in Earth's small and medium-sized reservoirs},
  author  = {Donchyts, Gennadii and Winsemius, Hessel and Baart, Fedor and Dahm, Ruben
             and Schellekens, Jaap and Gorelick, Noel and Iceland, Charles and Schmeier, Susanne},
  journal = {Scientific Reports},
  volume  = {12},
  pages   = {13776},
  year    = {2022},
  doi     = {10.1038/s41598-022-17074-6}
}
```

----
----

# The data

## What is in this archive

| File | Description |
|---|---|
| `gww_reservoir_timeseries.nc` | the data — NetCDF4, CF-1.10 (TODO: confirm size) |
| `README.md` | this file |
| `example_timeseries.ipynb` | worked example: open the file, extract and plot a reservoir |
| `LICENSE` | CC BY 4.0 licence text |

Nothing else is required to use the data. The NetCDF file is self-describing and
readable by any CF-aware tool.

## Quick start

```bash
pip install xarray netCDF4 pandas matplotlib
```

```python
import xarray as xr

ds = xr.open_dataset("gww_reservoir_timeseries.nc")

# monthly surface water area for one reservoir, in m2
series = ds["surface_water_area_monthly"].sel(reservoir=8).to_series().dropna()
print(series.head())
```

Then open `example_timeseries.ipynb` for the full walk-through, including the raw
observations.

---


## Dimensions

| Dimension | Size | Meaning |
|---|---|---|
| `reservoir` | 71,208 | one entry per reservoir; the index into every per-reservoir variable |
| `time` | ~500 | the shared monthly calendar, 1985-01 onwards |
| `obs_surface_water_area` | 44,808,972 | one entry per individual satellite observation, all reservoirs concatenated |
| `geometry_bytes` | 315,120,630 | the reservoir outlines, as concatenated UTF-8 bytes |

The third dimension is what makes this file look unusual; see
[Two layouts](#two-layouts-and-why-it-matters).

## Variables

### Per reservoir — dimension `(reservoir)`

| Variable | Type | Description |
|---|---|---|
| `reservoir` | int64 | Global Water Watch reservoir id. The coordinate. Ids are sparse — **not** `1..N`. |
| `longitude`, `latitude` | float64 | centroid of the outline, degrees east / north (WGS84) |
| `bbox_west`, `bbox_east`, `bbox_south`, `bbox_north` | float64 | bounding box of the outline, degrees |
| `name`, `name_en` | string | name from the source dataset; **empty string** where not recorded (most reservoirs) |
| `source_name` | string | dataset the outline came from — `osm_way`, `HydroLAKES`, … |
| `source_id` | string | identifier within that source dataset |
| `grand_id` | string | [GRanD](https://www.globaldamwatch.org/grand) dam id where matched; empty otherwise (a small minority) |
| `geometry_row_size` | int64 | length in bytes of this reservoir's outline; 0 if none recorded |
| `download_ok` | int8 | 1 = timeseries retrieved cleanly, 0 = the request was abandoned after retries |
| `surface_water_area_row_size` | int32 | number of raw observations belonging to this reservoir |

### Monthly aggregate — dimension `(reservoir, time)`

| Variable | Units | Description |
|---|---|---|
| `surface_water_area_monthly` | m² | monthly aggregate of the surface water area, **filtered** — spikes from cloud, shadow and partial scenes removed; `NaN` where no observations exist |

A dense grid. Every reservoir shares the same `time` axis, so each is `NaN`
outside its own period of record.

### Raw observations — dimension `(obs_surface_water_area)`

| Variable | Units | Description |
|---|---|---|
| `surface_water_area` | m² | surface water area from a single satellite overpass |
| `surface_water_area_time` | ms since 1970-01-01 | timestamp of that observation |

These are **not** indexed by reservoir — see below.

### Reservoir outlines — dimension `(geometry_bytes)`

| Variable | Description |
|---|---|
| `geometry_utf8` | every outline's GeoJSON text, UTF-8 encoded and concatenated |

Stored the same ragged way as the observations, with `geometry_row_size` giving
each reservoir's byte count. Outlines vary enormously — a farm pond is 271 bytes,
the largest lake is 935 kB — and one string per reservoir would make
`xr.open_dataset` try to widen the variable to a fixed 935,507 characters across
all 71,208 rows, a 248 GiB allocation that fails outright. Bytes on a flat axis
avoid that, and compress well.

**Partly ODbL-licensed — see [Licence](#licence).**

### Global attributes

`title`, `summary`, `source`, `references`, `Conventions` (CF-1.10),
`featureType` (`timeSeries`), `history`, `n_reservoirs`, plus
`monthly_variables` and `ragged_variables`, which name the variables of each kind
that this file actually contains.

## Two layouts, and why it matters

The archive holds two kinds of timeseries, stored differently.

**Monthly aggregates** share one calendar across all reservoirs, so they form a
plain 2-D grid, `surface_water_area_monthly(reservoir, time)`. Indexing is
ordinary.

**Raw observations** are irregular: each reservoir is observed at its own
satellite overpass times, shared with no other reservoir. On a common time axis
they would form a grid that is over 99.9% missing — hundreds of gigabytes of
nothing.

They are stored instead as a CF *contiguous ragged array*: each reservoir's
observations laid end to end along one flat dimension.

```
reservoir index :  [   0   ][      1      ][  2  ] ...
row_size        :     25          371        238
                      ↓            ↓           ↓
obs_...         :  |---25---|-----371-----|-238-|     one flat dimension
surface_water_area_time : timestamp of every element
surface_water_area      : value of every element
```

`surface_water_area_row_size(reservoir)` gives each reservoir's block length, in
reservoir order. Its cumulative sum gives the slice boundaries. That is the whole
trick, and the function below implements it.

---

# Extracting timeseries

## Monthly series for one reservoir

```python
import xarray as xr

ds = xr.open_dataset("gww_reservoir_timeseries.nc")

RESERVOIR_ID = 8

monthly = (
    ds["surface_water_area_monthly"]
    .sel(reservoir=RESERVOIR_ID)
    .to_series()
    .dropna()                 # drop months outside this reservoir's record
)

print(monthly.head())         # pandas Series, m2, indexed by month
```

## Raw observations for one reservoir

```python
import numpy as np
import pandas as pd


def ragged_slice(ds, variable, reservoir_id):
    """(times, values) for one reservoir from a contiguous ragged array."""
    ids = ds["reservoir"].values
    hits = np.flatnonzero(ids == int(reservoir_id))
    if hits.size == 0:
        raise KeyError(f"reservoir {reservoir_id} is not in this file")
    i = int(hits[0])

    row_size = ds[f"{variable}_row_size"].values
    start = int(row_size[:i].sum())
    sl = slice(start, start + int(row_size[i]))

    # Slice BEFORE loading -- see the note below.
    return ds[f"{variable}_time"][sl].values, ds[variable][sl].values


times, values = ragged_slice(ds, "surface_water_area", RESERVOIR_ID)
raw = pd.Series(values, index=pd.DatetimeIndex(times), name="surface_water_area")

print(raw.head())
```

> **Performance note.** Write `ds[var][sl].values`, not `ds[var].values[sl]`.
> The first reads only that reservoir's block from disk. The second loads *every
> observation in the file* — around 500 MB — to return one reservoir's data.

## Both series together, exported

```python
both = pd.DataFrame({"monthly": monthly}).join(pd.DataFrame({"raw": raw}), how="outer")
both.index.name = "time"
both.to_csv(f"reservoir_{RESERVOIR_ID}.csv")
```

## Many reservoirs at once

The dense monthly grid is designed for this:

```python
ids = ds["reservoir"].values[:100]
frame = ds["surface_water_area_monthly"].sel(reservoir=ids).to_pandas().T
# rows = time, columns = reservoir id
```

## Finding reservoirs

```python
# by bounding box
west, east, south, north = 4.0, 8.0, 50.0, 54.0
inside = (
    (ds["longitude"] >= west) & (ds["longitude"] <= east)
    & (ds["latitude"] >= south) & (ds["latitude"] <= north)
)
selected = ds["reservoir"].values[inside.values]

# by name (most reservoirs are unnamed, so this finds only a subset)
names = np.array([str(n) for n in ds["name"].values])
matches = ds["reservoir"].values[np.char.find(np.char.lower(names), "vacca") >= 0]
```

## The reservoir outline

```python
import json


def reservoir_geometry(ds, reservoir_id):
    """The GeoJSON outline of one reservoir, as a dict."""
    ids = ds["reservoir"].values
    hits = np.flatnonzero(ids == int(reservoir_id))
    if hits.size == 0:
        raise KeyError(f"reservoir {reservoir_id} is not in this file")
    i = int(hits[0])

    row_size = ds["geometry_row_size"].values
    start = int(row_size[:i].sum())
    n = int(row_size[i])
    if n == 0:
        return None
    return json.loads(ds["geometry_utf8"][start:start + n].values.tobytes().decode("utf-8"))


geom = reservoir_geometry(ds, RESERVOIR_ID)     # GeoJSON Polygon / MultiPolygon
print(geom["type"])
```

---

# Coverage, conventions and caveats

## Units and conventions

- Areas are **m²**. Divide by 1e6 for km², 1e4 for hectares. Reservoirs range
  from farm ponds of a few hundred m² to major lakes, so choose the unit from the
  data rather than assuming km².
- Times are stored as int64 milliseconds since 1970-01-01; xarray and most CF
  tools decode them to datetimes automatically.
- Missing numeric data is `NaN`. Missing strings are the empty string `""`.
- Coordinates are WGS84 (EPSG:4326).
- CF-1.10, `featureType = "timeSeries"`; the ragged layout is declared through
  the `sample_dimension` attribute, so CF-aware tools can follow it.

## What this dataset does and does not contain

- **Two variables:** `surface_water_area` (one value per satellite overpass,
  unfiltered) and `surface_water_area_monthly` (the monthly aggregate, filtered
  to remove spikes caused by cloud, shadow and partially imaged scenes).
  **For anything aggregated — regional means, trends, multi-reservoir work — use
  the monthly series.** The raw per-overpass values are there for inspecting an
  individual reservoir, and still contain the outliers the monthly series drops.
- **No gap filling.** Months with no usable observation are `NaN`, not
  interpolated.

## Caveats

- **Observation density varies over time.** Landsat-only coverage before ~2015
  is sparser than the Landsat + Sentinel-2 era after it. Comparing variability
  across those periods needs care — apparent changes may reflect sampling rather
  than the reservoir.
- **The end of each series is uneven.** Each reservoir's record ends whenever its
  last usable scene was, so end dates differ between neighbouring reservoirs.
  The final month of a series may aggregate fewer observations than usual.
- **Cloud, ice and shadow** affect water detection. Individual observations can
  be outliers; the monthly aggregate is more robust but not immune.
- **Outlines are static.** `geometry_utf8` is the reservoir extent as defined
  by its source dataset (OpenStreetMap, HydroLAKES), not a per-date water mask.
- **Most reservoirs are unnamed**, and `grand_id` is present only for the small
  minority matched to the GRanD database.

---



# Licence

Released under the **Creative Commons Attribution 4.0 International licence
(CC BY 4.0)** — full text in [`LICENSE`](LICENSE), summary at
<https://creativecommons.org/licenses/by/4.0/>.

You are free to share and adapt this data, including commercially, provided you
give appropriate credit, link to the licence, and indicate if changes were made.
Please attribute using the [citation](#citation) above.

## Reservoir outlines are ODbL

**The CC BY licence above does not cover the reservoir outlines.** The
`geometry_utf8` variable reproduces geometries from OpenStreetMap and
HydroLAKES. OpenStreetMap data is licensed under the
[Open Database Licence 1.0 (ODbL)](https://opendatacommons.org/licenses/odbl/1-0/),
and those terms travel with the geometries no matter what licence covers the rest
of this archive. We cannot relicense them, and this archive does not attempt to.

If you use or redistribute the outlines:

- **Attribute** — credit "© OpenStreetMap contributors".
- **Share alike** — a derived database built from them must itself be offered
  under ODbL.
- **Keep it open** — do not apply technological restrictions that prevent others
  exercising the same rights.

The `source_name` variable tells you which reservoirs came from which dataset, so
the ODbL-covered subset can be separated cleanly:

```python
import numpy as np

source = np.array([str(s) for s in ds["source_name"].values])
osm = source == "osm_way"            # ODbL applies to these outlines
print(f"{osm.sum():,} of {len(osm):,} outlines are OpenStreetMap-derived")
```

HydroLAKES is distributed by HydroSHEDS under its own terms; see
<https://www.hydrosheds.org/products/hydrolakes>.


