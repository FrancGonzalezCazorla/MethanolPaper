# Methanol Paper

Code for the process optimization and life cycle assessment (LCA) of methanol production by CO₂ hydrogenation. The repo has two parts:

1. **Process optimization** – a notebook that optimizes a two-reactor methanol synthesis flowsheet in Aspen HYSYS.
2. **LCA** – a Brightway notebook that computes the GWP100 of the process, regionalized by country, and compares it with conventional methanol routes.

## Repository structure

```
MethanolPaper/
├── optimization/
│   └── two_reactor_hysys_optimization.ipynb  # HYSYS optimization (DE + Nelder-Mead)
├── lca/
│   ├── lca_methanol.ipynb                  # regionalized LCA and maps
│   ├── inventories.xlsx                    # foreground inventories (GLO)
│   ├── .env.example                        # template for lca/.env
│   ├── trade_data.db                       # regioinvent trade database (git-ignored, download it as described in Setup)
│   ├── brightway/                          # Brightway data (git-ignored, created on first run)
│   └── results/                            # generated tables, logs and maps
├── docs/
│   └── system_diagram.drawio               # LCA system diagram
├── environment.yml                         # conda environment (both parts)
└── README.md
```

## Setup

Requires Windows (HYSYS is accessed through COM) and [Conda](https://docs.conda.io/). Part 1 also needs Aspen HYSYS installed and licensed; part 2 needs an ecoinvent account and the regioinvent trade database (downloaded in step 3).

1. Create and activate the environment:

   ```
   conda env create -f environment.yml
   conda activate methanol-paper
   ```

   On Windows, the `regioinvent` pip package contains files with very long names, so installation fails with `No such file or directory ... regioinvent\data\ReCiPe\...` unless long paths are enabled. Enable them once, from an **administrator** terminal, then restart the terminal and re-run the commands above.

   Command Prompt:

   ```
   reg add "HKLM\SYSTEM\CurrentControlSet\Control\FileSystem" /v LongPathsEnabled /t REG_DWORD /d 1 /f
   ```

   PowerShell:

   ```
   New-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\FileSystem" -Name LongPathsEnabled -Value 1 -PropertyType DWORD -Force
   ```
2. Create the local configuration for the LCA from its template, and fill it in (the optimization needs no `.env`; see part 1):

   ```
   copy lca\.env.example lca\.env
   ```

   `lca/.env`:

   | Variable                                   | Description                                                                          |
   | ------------------------------------------ | ------------------------------------------------------------------------------------ |
   | `BRIGHTWAY2_DIR`                           | Brightway data directory (default `brightway`, so `lca/brightway`, git-ignored)       |
   | `ECOINVENT_USERNAME`, `ECOINVENT_PASSWORD` | ecoinvent credentials, used for the one-time download                                |
   | `TRADE_DATABASE_PATH`                      | Trade database used by regioinvent (default `trade_data.db` in `lca/`, git-ignored)  |

   Relative paths in `lca/.env` are relative to the `lca/` folder, where the notebook runs. The `.env` file is git-ignored, so credentials and local paths are not committed.

3. Download the trade database (only needed for the LCA):

   ```
   curl -L -o lca/trade_data.db "https://zenodo.org/records/20596474/files/trade_data.db?download=1"
   ```

   The regioinvent package needs a trade database (`trade_data.db`: BACI import/export data for 2020–2024 plus production estimates). It is published on Zenodo ([record 20596474](https://zenodo.org/records/20596474), CC-BY 4.0) and is about 2.4 GB. If you save it elsewhere, set `TRADE_DATABASE_PATH` in `lca/.env` accordingly. The other files in the Zenodo record are only raw sources and are not needed.

   If `curl` does not work, download it manually instead:
   - **Manual download:** open the [Zenodo record](https://zenodo.org/records/20596474) in a browser, download only the file `trade_data.db`, and put it in the `lca/` folder (or anywhere, and set `TRADE_DATABASE_PATH` in `lca/.env` to it).
   - **Check the file:** it must be exactly 2,440,024,064 bytes (MD5 `43eda01d6d1efe387353f5c8da399240`). A smaller file is an incomplete download, and the notebook fails with `database disk image is malformed`. On Windows you can check the checksum with `certutil -hashfile lca	rade_data.db MD5`.

## 1. Process optimization

The notebook minimizes the cost `Z` (€/t, read from cell `E14` of the HYSYS spreadsheet `OptimizerSpreadsheet`) over six variables:

| Variable | Description                               | Bounds         |
| -------- | ----------------------------------------- | -------------- |
| P1       | Pressure of stream `Feed_2`             | 60 – 90 bar   |
| T1       | Temperature of stream `Feed_3`          | 200 – 250 °C |
| T2       | Temperature of stream `Feed_8`          | 200 – 250 °C |
| L1       | Tube length of reactor `PFR-100`        | 0.1 – 15 m    |
| L2       | Tube length of reactor `PFR-100-2`      | 0.1 – 15 m    |
| Purge    | Purge fraction (spreadsheet cell `E10`) | 0.01 – 0.15   |

It runs in two phases: a global search with SciPy `differential_evolution` (tuned: `popsize=15`, Sobol initialization), then polishing with `Nelder-Mead` (`adaptive=True`) from the best point found. At the end it picks the best point across the whole history, prints it, and applies it to the open HYSYS case.

If the solver times out (10 s) the point is discarded. If an evaluation fails, the notebook tries to reset the recycle (`RCY-1`) and continues; after 10 consecutive failures it aborts.

Open `optimization/two_reactor_hysys_optimization.ipynb` with the `methanol-paper` environment as kernel and run it from top to bottom (HYSYS opens the case, and every evaluation prints one line). Before running it, put your HYSYS simulation (`.hsc`) in the `optimization/` folder and set `hyFilePath` (the first line of the first code cell) to its file name.

The stream, operation and spreadsheet names are hard-coded in the connection cell of the notebook; change them there if your flowsheet differs. The bounds are in the `limites` list in the configuration cell.

## 2. LCA

Open `lca/lca_methanol.ipynb` with the `methanol-paper` environment as kernel. Run it from top to bottom; the first run is slow because it downloads ecoinvent and builds the regionalized databases (later runs reuse them from the Brightway project `Methanol_LCA`). The notebook:

1. Imports ecoinvent 3.10 (cutoff) into the Brightway project.
2. Imports the foreground inventories from `inventories.xlsx` (CO₂ hydrogenation methanol, PEM electrolysis hydrogen, point-source CO₂ capture) and computes their global (GLO) GWP100.
3. Regionalizes the background with [regioinvent](https://github.com/CIRAIG/regioinvent) and localizes the foreground to each country, matching inputs by reference product, unit and location with fuzzy name matching as a tiebreaker.
4. Computes the GWP100 of CO₂-hydrogenation methanol for every country.
5. Does the same for the conventional routes: natural gas reforming, coal gasification and the methanol market.
6. Draws a world map of GWP100 per country for each route.

Results are written to `lca/results/`: two Excel tables (`lca_results_glo.xlsx` with the global, unlocalized GWP100 of every route, and `lca_results_regionalized.xlsx` with the GWP100 of every route by country), the mapping logs and the PNG maps. The folder is tracked in git. Brightway projects and databases are stored in `lca/brightway/` (also git-ignored). The maps use a Natural Earth GeoJSON downloaded from GitHub, so an internet connection is needed.

## Notes

- The notebook and `inventories.xlsx` must stay in the same folder, since the notebook reads the inventory with a relative path.
- The LCA notebook uses the legacy Brightway2 stack (`brightway2=2.4.6`, `bw2data=3.6.6`), which is pinned in `environment.yml`. The newer Brightway 2.5 packages behave differently (for example, LCIA method names are prefixed with the ecoinvent version, and copying exchanges with `as_dict()` breaks the localization step), so don't upgrade them without adapting the notebook. Other packages are not pinned.
