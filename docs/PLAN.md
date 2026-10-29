# Phase 1 Plan: hardware, environment, layout, phases

_Written 2026-10-29, before any hardware exists or any data has been captured._

This document records the decisions that shape everything else. Where I recommend
something rather than confirm it, the recommendation is marked **(recommend, needs your OK)**.

---

## 1. Capture device: Raspberry Pi Pico (RP2040) **(recommend, needs your OK)**

The brief left the capture device open. I recommend the **Raspberry Pi Pico** (original, RP2040,
with pre-soldered headers, sold as "Pico H").

| Candidate | ADC | Realistic sample rate to host | Verdict |
|---|---|---|---|
| **Raspberry Pi Pico (RP2040)** | 12-bit SAR, up to 500 kS/s, free-running with DMA | ~250 kS/s over USB CDC | **Pick this.** ~$6, native USB, DMA, excellent docs |
| Raspberry Pi Pico 2 (RP2350) | 12-bit, 500 kS/s | same | Also fine; same SDK. I have not confirmed its ADC linearity is better, so no reason to pay more |
| Arduino Uno | 10-bit, ~10 kS/s usable | limited by a 115200-baud UART bridge | Too slow and too coarse; a 100 M-sample capture would take hours |
| ESP32 | 12-bit but notoriously non-linear, Wi-Fi radio on-die | fine | ADC quality and RF noise make it a poor measurement front end |
| Raspberry Pi 4 / 5 | none | n/a | Needs an external ADC; more parts, more to explain |
| USB sound card (the classic hack) | 16-bit, 48 to 96 kS/s | fine | Works, but AC-coupled with unknown filtering and AGC in some devices; harder to reason about honestly |

Why the Pico wins for an *honest* experiment:

- **The whole path is inspectable.** ADC, DMA, USB: all in roughly 200 lines of C we write ourselves, with no hidden filtering, AGC or driver magic.
- **Fast enough.** 250 kS/s at 12 bits means a 500 M-sample confirmatory capture (see the preregistration) takes about 35 minutes.
- **Cheap enough to buy two**, so one can be a spare or a second, independent unit.
- **Known warts, documented in advance** (which is the point of this project):
  - The RP2040 ADC has documented differential-non-linearity anomalies at four specific codes (see the ADC errata in the RP2040 datasheet). This is one reason the preregistration assesses entropy on the top 8 bits of each sample, not on the raw LSB alone, and why the raw-LSB stream is a *measured* stream, not a trusted one.
  - The ADC reference is the Pico's own 3.3 V rail, fed by an on-board switching regulator. Its ripple shows up in the ADC. Mitigation: force the regulator into low-noise PWM mode (drive GPIO23 high), decouple well, and *measure* the effect with the source-off control (Control B) rather than assume it away.
  - USB CDC throughput tops out around 0.5 to 1 MB/s in practice. 250 kS/s times 2 bytes is 500 kB/s. The firmware will stream in framed chunks with a sequence counter so any dropped chunk is detected and logged, never silently glossed over.

### Firmware approach
C with the official `pico-sdk`: free-running ADC at a fixed, documented clock divider, DMA into a double buffer, 12-bit samples stored as little-endian `uint16` (upper 4 bits zero), framed with a sync word, chunk sequence number and CRC. Built with `arm-none-eabi-gcc` inside WSL (same environment as the analysis tools). A pre-built `.uf2` with its SHA-256 will be committed so you can flash without building. No hardware-specific code is written until you confirm the Pico.

---

## 2. Analysis computer: Windows 11 with **WSL2 Ubuntu 24.04** **(recommend, needs your OK)**

What I found on this machine:

| Item | Found |
|---|---|
| OS | Windows 11 Home 10.0.26200 |
| Python | 3.13.1 (also 3.11 and 3.9 via `py`) |
| C compiler / make | none on PATH |
| WSL | installed, **no distribution** |

NIST STS 2.1.2 and the SP 800-90B Entropy Assessment are C/C++ programs written for Linux
(90B needs libbz2, libdivsufsort, jsoncpp, OpenSSL, GMP and MPFR). dieharder is an `apt install`
on Ubuntu. Making all of that build natively on Windows is possible but a distraction from the
experiment, and would make the write-up less reproducible for others.

Plan:

- **Capture runs on Windows** (native Python 3.13 plus `pyserial`; the Pico shows up as a COM port). This avoids the USB-passthrough dance that WSL2 needs for serial devices.
- **Processing, controls and all NIST/dieharder tests run in WSL2** (`wsl --install -d Ubuntu-24.04`, one command, needs a reboot). WSL reads the repo at `/mnt/c/Users/14808/Devel/avalanche-noise-qrng`. The `/mnt/c` bridge is slow for heavy I/O, so `run_all.py` will copy large inputs into a WSL-side work directory (git-ignored) before running the C tools.
- **One `requirements.txt`** (numpy, scipy, matplotlib, pyserial, pytest) used on both sides; `tools/setup.sh` installs the Ubuntu packages and builds STS and 90B EA from pinned upstream versions with recorded checksums.
- Alternative if you would rather not install WSL: Docker Desktop with a small Ubuntu image. Same scripts, slightly more friction. Say so and I will target that instead.

---

## 3. Repo layout: as suggested, with three small changes

The suggested layout is adopted (see the top-level README's repo map). Changes:

1. **`run_all.py` instead of a Makefile** as the single entry point. It is cross-platform and can describe what it is doing. A thin `Makefile` wrapper may be added later for Linux users.
2. **Third-party sources and builds are git-ignored** under `tools/src/`, `tools/build/` and `tools/bin/`; `tools/setup.sh` fetches them with pinned versions and verifies checksums. The repo records *how* to get the tools, never the tools themselves.
3. **`experiment/` gains two files beyond the preregistration**: `DATA_MANIFEST.md` (every data file: name, size, SHA-256, capture metadata) and `CAPTURE_LOG.md` (a dated, human-written log of each capture session, including the ones that went wrong).

Result files policy: `results/summary.json` and figures under about 500 kB each are committed. Raw tool
output (STS writes thousands of small files) goes under `results/raw_tool_output/`, git-ignored, but a
compressed tarball with its SHA-256 can be attached to a release.

---

## 4. Phases and what "done" means

| Phase | Deliverables | Done when |
|---|---|---|
| **1. Plan** (this) | `docs/PLAN.md`, draft `experiment/PREREGISTRATION.md`, `hardware/BUY_LIST.md`, repo skeleton | You have confirmed the Pico and WSL choices and signed off (or edited) the preregistration |
| **2. Learning materials** | `docs/learn/01` to `15`, `docs/learn/philosophy.md` | Each page reviewed for accuracy against its references; no hardware needed |
| **3. Tooling** | `tools/setup.sh`, STS and 90B runners and parsers, `controls/`, `processing/` with unit tests, `analysis/run_all.py` | The full pipeline runs end to end on the four controls alone and produces a `summary.json` whose control verdicts match the preregistered predictions |
| **4. Hardware and capture** | `hardware/README.md` (schematic, BOM, build guide, safety), `firmware/`, `capture/capture.py`, `capture/quicklook.py` | You have built the circuit, a shakedown capture looks sane in the quick-look plots, and the preregistration is **finalised and committed** before the confirmatory capture |
| **5. Analysis** | `results/summary.json`, `results/figures/`, `experiment/DATA_MANIFEST.md` | Every stream and control has been through every applicable test; the verdicts follow the preregistered rules mechanically |
| **6. Write-up and slides** | `paper/`, `slides/`, final README results section | Built from `summary.json` with no hand-typed numbers; reviewed for accuracy and approachability |

Ground rules from the brief apply throughout, especially: **no fabricated or simulated results presented as real.** Until Phase 5, every result slot is a `{{RESULT: ...}}` placeholder.

---

## 5. Open questions for you (Phase 1 check-in)

1. **Pico OK?** If you would prefer a different device, say which, before any firmware is written.
2. **WSL2 Ubuntu OK?** Installing it needs admin rights and a reboot, so I will leave running the install command to you (or confirm and I will run it). Docker is the fallback.
3. **What do you already own?** Multimeter, breadboard, jumper wires, resistor and capacitor kits, 9 V batteries, oscilloscope. The buy list marks those as "skip if owned".
4. **Oscilloscope budget?** It is optional but makes the hardware phase much less blind. The buy list gives a $0, roughly $40 and roughly $70 option.
5. **Data volume**: the preregistration proposes two confirmatory captures of 500 M samples (1 GB each) plus a smaller source-off control. Fine on disk? If not, the preregistration includes a smaller fallback plan.
6. **Preregistration thresholds**: please read sections 6 to 9 of `experiment/PREREGISTRATION.md` critically. Those are the rules we will be bound by.
