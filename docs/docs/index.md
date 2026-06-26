# EO_ENVIRONMENTAL_PREPROCESSING

Upstream preprocessing module that fetches Sentinel-2 L2A scenes from the Copernicus Dataspace STAC API and produces **monthly mean band rasters** for two spatial domains simultaneously:

- **Water** – Gulf of Gdańsk coastal strip (bands B02, B03, B04, B05)
- **Land** – 10 km radius area around the Port of Gdańsk (bands B04, B08)

These rasters are the mandatory inputs for `EO_DISTANCE_WATER_SEAPORT` and `EO_DISTANCE_LAND_SEAPORT`.

---

## Source files

| File | Role |
|---|---|
| `run_preprocessing.py` | Single entrypoint – fetches, filters, and composites all bands for a given year/month range |
| `context/preprocessing_context.json` | Auto-generated after each run; consumed by downstream modules |
| `assets/maska_wody/warstwa_wody_terra_checked.tif` | Static water fraction mask used to separate water/land pixels |

---

## How to run

```bash
cd EO_ENVIRONMENTAL_PREPROCESSING
python run_preprocessing.py --year 2024 --start-month 1 --end-month 12
```

| Argument | Default | Description |
|---|---|---|
| `--year` | required | Year to process |
| `--start-month` | `1` | First month (inclusive) |
| `--end-month` | `12` | Last month (inclusive) |
| `--force` | off | Reprocess months that already have a `done.flag` |
| `--output` | `context/preprocessing_context.json` | Path for the output context JSON |

Each already-processed month (flagged with `done.flag`) is skipped unless `--force` is set.

---

## Processing pipeline (per month)

1. **STAC query** — fetches all Sentinel-2 L2A items within the union bounding box of both AOIs, filtered by `cloud_cover ≤ 35 %`.
2. **Asset resolution** — resolves local `.jp2` file paths for bands B02, B03, B04, B05, B08, SCL; drops scenes with missing assets.
3. **Stacking** — builds lazy `xarray` stacks with `stackstac` (EPSG:2180, 20 m resolution, bilinear resampling for reflectance bands, nearest for SCL).
4. **Masking** — SCL-based clear-sky filter (removes clouds, cloud shadows, snow, saturated pixels); separate water-fraction thresholds for each domain (`≥ 0.8` for water pixels, `≤ 0.2` for land pixels).
5. **Scene filtering** — scenes with fewer than 500 valid pixels in a domain are excluded from that domain's composite.
6. **Compositing** — pixel-wise temporal mean over accepted scenes, independently per domain.
7. **Writing** — outputs rasters, masks, inventory, and `done.flag`.

---

## Output structure

```
preprocessed/
├── water/
│   └── <year>_<MM>/
│       ├── b02_mean_<year>_<MM>.tif         # B02 mean reflectance (water pixels)
│       ├── b03_mean_<year>_<MM>.tif
│       ├── b04_mean_<year>_<MM>.tif
│       ├── b05_mean_<year>_<MM>.tif
│       ├── water_fraction_20m.tif           # reprojected water fraction mask
│       ├── valid_mask_water_base.tif        # binary water AOI × water-fraction mask
│       ├── inventory.csv                    # per-scene stats and keep flags
│       ├── dropped_items.csv               # scenes dropped due to missing assets (if any)
│       └── done.flag                        # written on successful completion
│
└── land/
    └── <year>_<MM>/
        ├── b04_mean_<year>_<MM>.tif         # B04 mean reflectance (land pixels)
        ├── b08_mean_<year>_<MM>.tif
        ├── valid_mask_land_base.tif
        ├── inventory.csv
        └── dropped_items.csv               (if any)

context/
└── preprocessing_context.json              # paths + per-month results, passed to downstream modules
```

### `inventory.csv` columns

| Column | Description |
|---|---|
| `id` | Sentinel-2 scene ID |
| `datetime` | Scene acquisition time (ISO 8601) |
| `cloud_cover` | Scene-level cloud cover (%) |
| `valid_water_px` | Valid water pixels after masking |
| `valid_land_px` | Valid land pixels after masking |
| `keep_water` | Whether the scene was included in the water composite |
| `keep_land` | Whether the scene was included in the land composite |

---

## Key parameters (hardcoded in `run_preprocessing.py`)

| Parameter | Value | Description |
|---|---|---|
| `EPSG` | 2180 | Output CRS (Polish national grid) |
| `RESOLUTION` | 20 m | Pixel size for all outputs |
| `CLOUD_MAX` | 35 % | Max scene cloud cover for STAC query |
| `WATER_THRESHOLD` | 0.8 | Min water fraction for water pixels |
| `LAND_WATER_FRACTION_MAX` | 0.2 | Max water fraction for land pixels |
| `MIN_SCENE_VALID_PX` | 500 | Min valid pixels to include a scene in a composite |
| Water AOI | `(54.834, 18.330) – (54.331, 19.211)` | Diagonal bbox, Gulf of Gdańsk |
| Land AOI | `lat 54.4008, lon 18.6992, r = 10 km` | Circle around Port of Gdańsk |

---

## Dependencies

- Requires local access to Sentinel-2 `.SAFE` archives (mounted at `/eodata` or via Copernicus Dataspace).
- The static water mask (`assets/maska_wody/warstwa_wody_terra_checked.tif`) must exist before running.
- Downstream modules (`EO_DISTANCE_WATER_SEAPORT`, `EO_DISTANCE_LAND_SEAPORT`) read from `preprocessed/water/` and `preprocessed/land/` respectively.
