# Changes from upstream

Upstream: https://github.com/ECOSTRESS-Collection-2/ECOv002-L3T-L4T-JET, branch `main` at commit a785b14b155fa628ea45f876ba0f37caa53978f9 (2026-07-27; upstream has no tags or releases), by Gregory H. Halverson, Kerry Cawse-Nicholson, Madeleine Pascolini-Campbell, Evan Davis and Claire Villanueva-Weeks (NASA Jet Propulsion Laboratory), Apache License 2.0.

This copy starts from upstream's file tree at a785b14 without its git history. It is unchanged except for the items below. The science code, the run-config template, the tests and the LICENSE are untouched, so the products do not change. `.github/` was left out: it holds the upstream CI workflow and the workflow that publishes releases to PyPI, which belong to the upstream repository.

Versions were not changed: `pyproject.toml` and `ECOv002_L3T_L4T_JET/version.txt` say `1.0.0` (static), and `ECOv002_L3T_L4T_JET/PGEVersion.txt` says `v1.13.1`, which the PGE writes to each product's `PGEVersion`. Products from this copy still say v1.13.1 and build 0700.

## 2026-10-29: the GDAL command-line tools are declared

The PGE runs GDAL's command-line programs: `gedi-canopy-height` calls `gdalinfo` and `gdalbuildvrt` on the canopy height mosaic, and `rasters` writes every output layer as a COG with `gdal_translate` (through `os.system`, without checking the return code). pip does not install these programs, and neither `make environment`, the README's conda environment nor the Dockerfile added them. On a fresh install the run stops when BESS (set up through PTJPLSM and PTJPL, `ECOv002_L3T_L4T_JET.py:1018`) opens the canopy height mosaic: `gdalinfo` raises `FileNotFoundError: [Errno 2] No such file or directory: 'gdalinfo'`, reported as `GEDINotAvailable` (`BESS/BESS.py:149`). They cannot be declared in `pyproject.toml`, so `make environment` now creates the conda environment with conda-forge `gdal`, and the Dockerfile installs `gdal-bin`. Results do not change.

The Dockerfile's `python:3.10.16-bookworm` meets `requires-python = ">=3.10"` and was left as it is. (The image was not built here.)

## 2026-10-29: README

The README's `pip install ECOv002-L3T-L4T-JET` finds nothing: the package is not on PyPI (404, checked 2026-10-29). The Installation section now says to use `pip install .` from a clone, and the conda environment line in the Environment section adds `gdal`, with a line on why.

## Notes for running it

- Inputs: one ECOv002 L2T LSTE granule and the matching ECOv002 L2T STARS granule from LP DAAC (Earthdata login), as unzipped folders of COGs. Write a run-config with `generate_L3T_L4T_JET_runconfig` (L2T LSTE folder, L2T STARS folder, working directory, static directory) and pass it to `ECOv002-L3T-L4T-JET <runconfig.xml>`.
- Static data: left alone, the dependencies download whole-globe files for the GEDI canopy height, SoilGrids field capacity and wilting point, and the MODIS clumping index (about 40 GB). Placing small windows of them under the run-config's static folder, with the file names the code looks for (`GEDI_download/Forest_height_2019.vrt`, the two `SoilGrids_download` files, `MODISCI_download/global_clumping_index.tif`), avoids this.
- GEOS-5 FP: the PGE downloads the 12 files it needs from NCCS itself (public, about 670 MB per scene).
- Earthdata login: every run reads it from `~/.netrc` or `LPDAAC_USERNAME`/`LPDAAC_PASSWORD`, even when nothing is downloaded. The download paths in `LPDAAC/LPDAACDataPool.py` (`:324`, `:411`) and in `MODISCI` pass the password on a `wget` command line; they are not reached when the inputs and static files are already in place.

## Check

On 2026-10-29, in a fresh conda environment made the way the README now says (conda-forge Python 3.11.18, pykdtree and GDAL 3.13.3; `jupyter` left out, it is only for the notebook), from a clone of this repository, with no other GDAL programs on the PATH:

```sh
pip install .
pip install pytest && pytest -q        # 36 passed (the tests import the package and its dependencies)
python -c "from ECOv002_L3T_L4T_JET import generate_L3T_L4T_JET_runconfig as g; print(g(
    L2T_LSTE_filename='work/ECOv002_L2T_LSTE_35698_014_11SPS_20241022T183627_0713_01',
    L2T_STARS_filename='work/ECOv002_L2T_STARS_11SPS_20241022_0713_01',
    working_directory='work', static_directory='static/L3T_L4T_static'))"
ECOv002-L3T-L4T-JET work/runconfig/ECOv002_L3T_JET_35698_014_11SPS_20241022T183627_0700_01.xml
```

The scene is orbit 35698, scene 014, tile 11SPS (2024-10-22, Imperial Valley and Salton Sea), the scene of `example_L3T_L4T_JET_run_with_download.py`: the V002 L2T LSTE and STARS granules (34 MB) from LP DAAC, and small windows of the static files. The PGE exited with code 0 in 4 min 38 s on 4 cores (most of it the GEOS-5 FP download from NCCS), with 5.1 GiB peak memory, and wrote the six products (L3T JET, MET, SEB, SM, L4T ESI, WUE; 133 MB of zips). The same command in an environment without `gdal` (the README's previous conda line) stopped after 6 s with `FileNotFoundError: [Errno 2] No such file or directory: 'gdalinfo'` and `GEDINotAvailable` at `BESS/BESS.py:149`. A first run with `gdal` stopped with exit code 14 (`ANCILLARY_SERVER_UNREACHABLE`) when NCCS answered 503 and timed out; the rerun ten minutes later finished.

Against the official LP DAAC V002 granules for the same scene (build 0713, made with PGE v1.12.4), all 31 layers have the same grid and the same no-data mask, and the water and cloud masks are bit-identical. ETdaily has r = 0.99898 and a mean absolute difference of 0.0056 mm/day; ESI agrees within 0.1% for 99.6% of pixels; WUE has r = 0.99995. BESSinst reads about 9% high (33.57 against 30.70 W/m2, r = 0.9996), as in the earlier runs of the unchanged code; the meteorology, radiation, PT-JPL-SM, MOD16 and soil moisture agree to 3 or 4 significant digits. These are the same statistics as from the unchanged upstream code with the same inputs.
