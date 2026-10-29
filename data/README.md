# data/ (git-ignored)

Raw captures and processed bit streams live here and are **never committed**.

- `data/raw/`        raw ADC sample files (`.bin`) plus `.json` metadata sidecars written by the capture script
- `data/processed/`  bit streams derived from raw data by the processing pipelines
- `data/controls/`   generated control streams

Every file that contributes to a reported result is listed, with its SHA-256 and size,
in `experiment/DATA_MANIFEST.md`. To publish the data itself, use a GitHub release asset
or Git LFS (see the top-level README).
