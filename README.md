# tetrapy

A containerized, config-driven wrapper around the USGS [Tetracorder](https://www.usgs.gov/labs/spectroscopy-lab/tetracorder) expert system for imaging-spectroscopy mineral identification. `tetrapy` drives the full pipeline — from convolving spectral libraries onto a scene's grid, through running Tetracorder, to aggregating its outputs into L2B mineral and uncertainty products — off a single YAML config.

## What it does

Tetracorder identifies surface materials by matching continuum-removed absorption features in a reflectance spectrum against a reference library. Running it against a new sensor or scene normally means hand-editing a tree of command files, convolving spectral libraries to the instrument's wavelength grid, and post-processing a directory of per-material rasters. `tetrapy` automates that end to end:

- **Pure-Python spectral convolution** ([`tetrapy/conv`](tetrapy/conv)) reimplements USGS specpr function 17 (Gaussian high-to-low resolution convolution). Given an unconvolved master library and a scene's ENVI header, it builds a complete convolved specpr library — no convolution recipe required, since each master spectrum carries its own wavelength/FWHM grid.
- **Expert-system decoding** ([`tetrapy/tetracorder.py`](tetrapy/tetracorder.py)) parses the Tetracorder `cmd.lib.setup.*` files into structured per-material records (library, record, features, constraints) and exports a material matrix CSV.
- **Sensor integration** ([`tetrapy/sensor.py`](tetrapy/sensor.py)) writes the per-sensor files the command tree looks up by name (restart, deleted-channels, enable/disable, color-channels) so a run uses the freshly convolved library.
- **Run orchestration** ([`tetrapy/tetra.py`](tetrapy/tetra.py)) drives the vendored `cmd-setup-tetrun` / `cmd.runtet` scripts.
- **L2B aggregation** ([`tetrapy/aggregate.py`](tetrapy/aggregate.py)) composites the group 1 / group 2 material rasters into band-depth and mineral-id stacks, with per-pixel band-depth uncertainty (Clark, 2003), written as NetCDF and/or GeoTIFF.

## Installation

Requires Python ≥ 3.10 (project targets 3.12). Using [uv](https://docs.astral.sh/uv/):

```bash
uv sync
```

This installs the `tetrapy` CLI entry point. A working Tetracorder installation (command tree + spectral libraries) is expected on disk — the pipeline shells out to its scripts and reads its libraries. Paths are configured in the YAML (see below); the defaults assume the container layout under `/root/tetracorder`, `/data`, and `/output`.

## Usage

The whole pipeline runs from a YAML config:

```bash
tetrapy run config.yml \
  --data.rfl /data/emit20230728t214153_rfl \
  --data.rfluncert /data/emit20230728t214153_uncert
```

Individual stages are also exposed as subcommands:

```bash
tetrapy export_matrix config.yml   # decode expert system -> material matrix CSV
tetrapy convolve      config.yml   # convolve reference/research libraries to scene grid
tetrapy sensor        config.yml   # wire convolved library into the command tree
tetrapy setup         config.yml   # configure a Tetracorder run (cmd-setup-tetrun)
tetrapy tetrun        config.yml   # execute the run (cmd.runtet)
tetrapy aggregate     config.yml   # aggregate outputs into L2B products
tetrapy preview       config.yml   # print the resolved config without running anything
```

`tetrapy run` executes the enabled stages in order:

```
export_matrix → convolve → sensor → setup → tetrun → aggregate
```

Each stage is gated by its own `{stage}.enabled` flag in the config, so any stage can be skipped.

### Configuration

Config is loaded with [python-box](https://github.com/cdgriffith/Box) and supports a few conveniences (see [`config.yml`](config.yml) for a fully-commented example):

- **Interpolation** — `${...}` references other config values. A leading dot (`${.key}`) is relative to the current section; otherwise it resolves from the top of the config. Fully-substituted strings are parsed as Python literals where possible.

  ```yaml
  output:
    base: /output/test
    tetrapy: ${output.base}/aggregate
  ```

- **CLI overrides** — any dotted key can be overridden on the command line. Values are parsed as Python literals, so lists and numbers work:

  ```bash
  tetrapy run config.yml --tetrun.args '["band", 10, "gif"]' --output.base /output/run2
  ```

- **Section inheritance** — a section can pull in defaults from others with the `^^` key (`^^: [base, shared]`); its own keys win on conflict.

- **`--section`** — load and operate on just one subsection of the file with `-s/--section`.

### Output products

`aggregate` writes two products (extension decides format — `.nc` for NetCDF, `.tif` for GeoTIFF) over `downtrack`/`crosstrack` dimensions:

- **Mineral product** (`out_min`): `group_1_band_depth`, `group_1_mineral_id`, and the group 2 equivalents.
- **Uncertainty product** (`out_minuncert`): `group_1_band_depth_unc`, `group_1_fit`, and the group 2 equivalents.

Mineral IDs are keyed to a stable reference matrix ([`tetrapy/data/v6.00a6.csv`](tetrapy/data/v6.00a6.csv)) so material indices stay consistent across runs. A group that identified nothing (e.g. full cloud, snow, or water) is zero-filled rather than omitted, so both products always carry all four group variables.

## Project layout

```
tetrapy/
├── __main__.py       # Click CLI (tetrapy run, and per-stage subcommands)
├── __init__.py       # app init: config load + logging setup
├── config.py         # YAML load, ${...} interpolation, ^^ inheritance, CLI overrides
├── pipeline.py       # maps config keys to each stage's underlying call
├── tetracorder.py    # TetraDecoder: parse the expert-system command files
├── sensor.py         # write per-sensor tetracorder integration files
├── tetra.py          # setup/exec Tetracorder + library convolution drivers
├── aggregate.py      # aggregate group outputs into L2B products + uncertainty
├── utils.py          # logging/timing helpers
├── conv/             # pure-Python specpr Gaussian convolution
│   ├── library.py    #   build a convolved specpr library from a master
│   ├── convolve.py   #   the Convolver
│   └── specpr.py     #   specpr file reader/writer
├── configs/          # example configs (aviris, test)
└── data/             # reference material matrices (v6.00a6.csv)
```

## Notes

- The default config paths (`/root/tetracorder`, `/data`, `/output`) reflect the intended containerized deployment; adjust them for a local Tetracorder install.
- Logs are written both to the console (via [rich](https://github.com/Textualize/rich)) and, when `log.file` is set, to a file. The resolved config is exported to `log.config` for reproducibility.
