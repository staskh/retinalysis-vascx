# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

`retinalysis-vascx` extracts retinal vascular biomarkers from color fundus images (CFIs), in two independent stages:

1. **Segmentation** (`vascx/inference/`, CLI `vascx run-models`) — torch ensembles from the `Eyened/vascx` HuggingFace repo produce vessel / artery-vein / disc masks, fovea locations, quality scores.
2. **Biomarker extraction** (`vascx/fundus/`, CLI `vascx calc-biomarkers`) — pure CPU/numpy/networkx; consumes stage-1 output folders.

Most work happens in stage 2. Stage 1 needs `torch` + `retinalysis-inference`, which are **not** declared dependencies — install separately when touching `vascx/inference/`.

## Environment & commands

```bash
uv venv --python 3.12
uv pip install -e ".[test]"
```

```bash
MPLBACKEND=Agg .venv/bin/python -m pytest              # full suite (~2.5 min)
pytest -m reference                                    # biomarker regression only
pytest -m plotting                                     # per-feature plot smoke tests
pytest -m profile                                      # runtime guard (machine-speed sensitive)
pytest "tests/test_feature_regression.py::test_feature_set_regression[full_v3]"   # single case
pytest --accept-vascx-reference -m reference           # rewrite stored baselines
pytest --run-cli-e2e -m cli_e2e tests/test_cli_e2e.py  # opt-in; downloads model weights
tox -e py312                                           # or `tox` for the full matrix
```

End-to-end on the packaged samples:

```bash
vascx run-models ./samples/fundus/original/ ./samples/fundus/segmentations
vascx calc-biomarkers ./samples/fundus/segmentations ./samples/fundus/biomarkers.csv \
  --feature_set full_v3 --n-jobs 8 --naming resolved
```

## Architecture

**Upstream package split.** `rtnls_enface` (pinned `retinalysis-enface==1.2.0`) owns the generic enface-image layer: `Fundus`, `Layer`, `OpticDisc`, and all grid specifications (`ETDRSGrid`, `EllipseGrid`, `CircleGrid`, `HemifieldGrid`, `DiscCenteredGrid`) plus bounds/ROI helpers. VascX subclasses those. Grid or bounds behavior that looks wrong is often upstream, not here. `rtnls_fundusprep` supplies `make_roi_mask_from_bounds` (optional extra `[fundusprep]`).

**`Retina` (`vascx/fundus/retina.py`)** subclasses `Fundus` and is the single extraction entry point. `Retina.from_file(av_path=, vessels_path=, disc_path=, fundus_path=, fovea_location=, bounds=/roi_mask=, mm_per_pixel=)` builds it; `calc_features(feature_set, naming=)` returns a `{name: value}` dict. Layers are `arteries` / `veins` (`VesselTreeLayer`) and `vessels` (`FundusVesselsLayer`).

**Vessel representation stages** (`vascx/fundus/layer.py`, all `cached_property`, built lazily and expensively):
`binary` / `binary_nodisc` → `skeleton` (skimage) → `graph` (networkx `Graph`, `Segment` objects on edges, built via vendored `vascx/shared/sknw/`) → `digraph` + `trees` + `nodes` (flow directed outward from the disc; `Endpoint` / `Bifurcation`) → `resolved_segments` (merged vessels, `vascx/fundus/vessel_resolve.py`). Feature families consume different stages: density uses `binary`, bifurcation features use `digraph`/`nodes`, caliber/tortuosity use `segments` + splines + diameters (`vascx/shared/{splines,diameters,segment,vessels}.py`).

**Feature dispatch.** A `Feature` subclasses one of three bases in `vascx/fundus/features/base.py`, and the base determines what it is computed on inside `Retina.calc_features`:

| Base | Target |
|---|---|
| `RetinaFeature` | the `Retina` once |
| `LayerFeature` | every `VesselTreeLayer` (arteries, veins) |
| `VesselsLayerFeature` | every `FundusVesselsLayer` (vessels) |

Each feature carries an optional `GridFieldSpecification`. Region visibility is enforced centrally: if the grid field's in-bounds fraction falls below `min_area_within_bounds` (biomarker → field → grid → feature default), `compute` returns `None` rather than a biased value. Exceptions inside a single feature become a warning + `None`, not a failure, unless `raise_on_error=True`.

**Feature sets** (`vascx/fundus/feature_sets/`) are `FeatureSet` instances — a name, a flat list of parametrized feature objects, a description. `FeatureSet.__init__` self-registers in a **global name registry**, so names must be unique and the module must be imported for `FeatureSet.get_by_name` to find it; `feature_sets/__init__.py` re-exports every set and `vascx/utils/analysis.py` star-imports it. `full_v3` is the recommended set.

**Naming** (`vascx/shared/naming.py`). Every feature emits ordered `NamePart`s; `make_feature_names` assembles column names in one of two conventions. `canonical` keeps all parts (`[AGGREGATION]_[BIOMARKER]_[PARAMETERS]_[REGION]_[LAYER]`); `resolved` (CLI default) drops parts that do not disambiguate within the feature set, so **the same feature yields different column names in different feature sets**. Duplicate canonical names raise at name-build time. `vascx/utils/feature_docs.py` writes the accompanying `.names.json` display mapping.

**Parallelism** (`vascx/utils/analysis.py`). `extract_in_parallel` splits examples into per-worker batches over joblib; each worker builds its own `Retina`, so nothing is shared and warnings are aggregated back and counted.

**FAZ pipeline** (`vascx/faz/`) mirrors the fundus one for OCTA foveal-avascular-zone images — separate `FAZRetina`, `FAZLayer`, `FAZLayerFeature`, and feature sets — but shares `vascx/shared/` and the naming machinery.

## Adding biomarkers or feature sets

- New biomarker → a module in `vascx/fundus/features/`, subclassing the right base, implementing `compute`, `_plot`, `display_name`, and `name_tokens`/`name_parts`.
- New feature set → a module in `vascx/fundus/feature_sets/` exporting a `FeatureSet`, re-exported from that package's `__init__.py`.
- `tests/test_feature_regression.py` **auto-discovers** every `FeatureSet` in that package, so a new set fails until you generate its baseline with `pytest --accept-vascx-reference -m reference`. Baselines live in `tests/reference/<name>.{parquet,meta.json,overrides.yaml}`; the `overrides.yaml` carries per-set tolerances, rename maps, and ignore lists used when a change intentionally moves numbers.
- Changing any shared geometry code moves numbers across *all* feature sets — check the reference diff before accepting it.

## Gotchas

- `samples/fundus` and `tests/reference` are committed deliberately so tests run from a clean checkout; `.gitignore` allowlists them path by path.
- Notebooks are stripped through an `nbstripout` git filter (`.gitattributes`) — have `nbstripout` installed before committing notebook edits.
- Both `versioneer.py` and `[tool.setuptools_scm]` are present; the installed version comes from setuptools-scm.
- Distances default to pixels. Passing `mm_per_pixel` scales physical grids and converts caliber/CRE to mm only; unitless features are untouched. Without it, grid scaling falls back to the 4.75 mm disc–fovea assumption.
- Tests must run with a non-interactive matplotlib backend (`MPLBACKEND=Agg`; tox sets this).
