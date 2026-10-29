# At-Home Avalanche-Noise QRNG: An Honest Experiment

A hardware random number generator built from a reverse-biased transistor junction,
tested with NIST's own tools, and written up so that a curious beginner can follow it
and an expert would not wince.

**Status: Phase 1 (planning).** No hardware has been built and no data has been captured.
Every number in this repository that looks like a result is a placeholder until then.

## Start here

- The plan and hardware decision: [docs/PLAN.md](docs/PLAN.md)
- The pre-registered experiment (hypotheses, procedure, pass/fail rules, written before any data): [experiment/PREREGISTRATION.md](experiment/PREREGISTRATION.md)
- What to buy: [hardware/BUY_LIST.md](hardware/BUY_LIST.md)
- Learning path for beginners: `docs/learn/` (Phase 2)

## Repo map

```
hardware/        schematic, bill of materials, build guide, buy list
firmware/        capture-device code (Raspberry Pi Pico)
capture/         host-side capture and quick-look diagnostics
processing/      raw, von Neumann, Toeplitz and SHA-256 pipelines, with tests
controls/        control-stream generators (counter+SHA-256, source-off, bad PRNG, os.urandom)
tools/           scripts that fetch and build NIST STS, SP 800-90B EA, dieharder
analysis/        run the tests, parse outputs, make figures, write results/summary.json
experiment/      PREREGISTRATION.md, capture logs, data manifest
results/         summary.json and small figures only
paper/           write-up source and build script
slides/          HTML deck source and build script
docs/learn/      learning materials and the philosophy essay
data/            (git-ignored) raw captures and derived bit streams
```

## Results

_No data yet. This section is populated by `analysis/run_all.py` from `results/summary.json`._

## Reproducing

Quickstart arrives with Phase 3 (tooling). The short version: install WSL2 Ubuntu,
run `tools/setup.sh` to build the NIST tools, then `python analysis/run_all.py`.
