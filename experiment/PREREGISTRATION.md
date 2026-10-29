# Pre-registration: At-Home Avalanche-Noise QRNG

| Field | Value |
|---|---|
| **Status** | **DRAFT v0.1, not yet binding.** Becomes binding when a commit titled `Finalise preregistration v1.0` lands, which must happen before the first confirmatory capture. |
| Drafted | 2026-10-29 |
| Author | Evan Hackett |
| Amendments | See section 12. Anything changed after v1.0 is logged there with a date and reason, never edited silently. |

## 0. Why this document exists

Randomness testing is unusually easy to fool yourself with: there are many streams to choose
from, many tests, and a strong temptation to keep the capture that "looks best". This document
fixes the hypotheses, the procedure, the sample sizes and the pass/fail rules **before** any
confirmatory data exists, so the git history can show that the rules came first and the
results second.

## 1. Question

Does a home-built avalanche-noise source (reverse-biased 2N3904 base-emitter junction, amplified
and sampled by a Raspberry Pi Pico ADC) produce raw samples with measurable min-entropy, and can
standard post-processing turn them into a bit stream that passes NIST SP 800-22, in a way that is
distinguishable from streams that pass for trivial reasons?

Secondary question, answered in the philosophy essay rather than by data: what, if anything, does
passing these tests say about the source being "quantum"?

## 2. Hardware and capture configuration (fixed design; measured values recorded at capture time)

- **Noise source:** 2N3904 NPN, base-emitter junction reverse-biased into avalanche from two 9 V
  batteries in series (nominal 18 V) through a 100 kΩ resistor. Collector left open.
- **Gain stage:** second 2N3904 common-emitter amplifier powered from the Pico's 3.3 V rail,
  AC-coupled from the source, output biased near mid-rail (about 1.65 V). Final values of the
  gain-setting resistors are chosen during shakedown (section 5) and then frozen.
- **Shielding:** circuit inside a closed metal tin, tin connected to circuit ground, only the USB
  cable leaving.
- **Capture device:** Raspberry Pi Pico (RP2040), 12-bit ADC, free-running, DMA, streamed over
  USB CDC as little-endian `uint16` samples (values 0 to 4095) in framed chunks with sequence
  numbers and CRC. Pico regulator forced to PWM mode (GPIO23 high).
- **Sample rate:** fixed at **250,000 samples/s** (ADC clock divider recorded in the metadata
  sidecar). If shakedown shows USB cannot sustain this without dropped chunks, the rate is lowered
  to the highest of 200 k, 150 k or 100 kS/s that sustains zero drops for 10 minutes, and that
  value is frozen in v1.0.
- **Host:** Windows 11 PC, native Python capture script writing `data/raw/<name>.bin` plus
  `<name>.json` with: UTC start/end time, sample rate and divider, firmware git hash and `.uf2`
  SHA-256, Pico serial number, measured bias voltage (multimeter, before and after capture),
  battery type, gain-stage resistor values, shield on/off, room temperature if a thermometer is
  available (otherwise "unknown"), USB host port, chunk count, dropped-chunk count, SHA-256 of
  the `.bin` file.

A capture with **any** dropped chunks is still kept but is flagged and may not be used as a
confirmatory capture.

## 3. Streams derived from each raw capture

All streams are produced by scripts in `processing/`, each with unit tests, from the raw `.bin`
file. Sample index `i` runs from 0; `s_i` is the 12-bit value.

| ID | Name | Definition | Bits per sample |
|---|---|---|---|
| S1 | RAW-LSB | bit 0 of `s_i` | 1 |
| S2 | RAW-MED | 1 if `s_i` > running median of the previous 4095 samples, 0 if less, **discarded** if equal. First 4095 samples produce no bits. | ≤1 |
| S3 | VN | Von Neumann debiasing applied to S1 using non-overlapping pairs: 01→0, 10→1, 00 and 11 discarded | ≈0.25 at best |
| S4 | TOEPLITZ | Toeplitz-matrix extractor applied to the **8-bit symbols** `y_i = s_i >> 4`, with output size set by the rule in section 7 | depends on measured entropy |
| S5 | SHA256 | SHA-256 applied to blocks of `y_i` symbols, block size set by the rule in section 7; each block hashed independently (no chaining) | depends on measured entropy |

The 8-bit symbol `y_i = s_i >> 4` (the top 8 bits) is used for S4, S5 and for the SP 800-90B
entropy assessment, so the entropy estimate and the extractor input are the **same** stream.
The four low bits are deliberately discarded for those streams: the RP2040 ADC has documented
non-linearity at specific codes, and the SP 800-90B tools assess symbols of at most 8 bits.
Whatever entropy the low bits carry is forfeited, which is conservative.

## 4. Controls

| ID | Name | Definition | Purpose |
|---|---|---|---|
| CA | COUNTER-SHA256 | 64-bit big-endian counter 0, 1, 2, … each hashed with SHA-256; outputs concatenated. Zero entropy by construction. | Shows that passing SP 800-22 after hashing proves nothing about the source |
| CB | SOURCE-OFF | A real capture with the bias battery disconnected (junction not in avalanche), everything else identical to the confirmatory setup, 50 M samples. Processed into S1 to S5 exactly like a real capture. | Measures what the amplifier, ADC, regulator, USB and room contribute on their own |
| CC1 | BAD-BIAS | Bernoulli bits with P(1) = 0.52 drawn from MT19937 seeded with 20261029 | Shows the tests detect a modest bias |
| CC2 | BAD-PERIODIC | `os.urandom` bytes with bit 0 of every 64th byte forced to 1 | Shows the tests detect injected structure |
| CP | URANDOM | `os.urandom` output from the analysis machine, 100 M bits, generated once and checksummed | Reference CSPRNG; expected to pass |

Controls CA, CC1, CC2 and CP are generated by `controls/` with the seeds above; CB is a capture.
Each is 100 M bits unless stated. Every generated control file's SHA-256 goes in the data manifest.

## 5. Capture plan and sample sizes

| Capture | Samples | Size (uint16) | Time at 250 kS/s | Use |
|---|---|---|---|---|
| **C0 shakedown** (as many as needed) | ≤ 10 M each | ≤ 20 MB | ≤ 40 s | Tuning gain, bias, shielding, sample rate. **Never used for confirmatory results.** All kept and logged, including bad ones. |
| **C1 confirmatory** | 500 M | 1.0 GB | ~33 min | The primary result for every hypothesis |
| **C2 replication** | 500 M | 1.0 GB | ~33 min | Taken in a separate session at least 12 hours after C1, batteries re-measured. Used for stability (section 8) and to adjudicate marginal results |
| **CB source-off** | 50 M | 100 MB | ~3.5 min | Negative control B |

Why 500 M samples: NIST STS at the standard 100 sequences × 1,000,000 bits needs 10^8 bits per
stream. S1 and S2 give about one bit per sample. S3 (von Neumann) yields at most 0.25 bits per
input bit, so 4 × 10^8 input bits are needed for a full-size S3; 500 M gives headroom for bias.
S4 and S5 depend on the measured entropy (section 7).

**Fallback plan if 1 GB captures are impractical** (to be decided at v1.0, not after): C1 and C2
are 125 M samples each. S1, S2, S4 and S5 are tested at full size where the data allow; any stream
with fewer than 10^8 bits is tested with as many whole 1,000,000-bit sequences as exist, provided
that is at least 20, using the STS proportion threshold recomputed for that number of sequences.
A stream with fewer than 20 sequences is reported as "insufficient data", not pass or fail.

Sequence selection is mechanical: bits are consumed from the start of the stream in order; nothing
is skipped or chosen.

## 6. Tests and parameters

### 6.1 NIST SP 800-22, STS 2.1.2
- 100 sequences × 1,000,000 bits per stream, all 15 tests, STS default block parameters:
  Block Frequency M = 128, Non-overlapping and Overlapping Template m = 9, Linear Complexity
  M = 500, Serial m = 16, Approximate Entropy m = 10. This yields 188 p-value columns.
- Per-subtest criteria, exactly as STS reports them:
  - **Proportion** of sequences with p ≥ 0.01 must be at least the STS threshold
    (0.960150 for 100 sequences, i.e. 96 of 100; recomputed by STS for the Random Excursions
    tests, which use only the sequences meeting their precondition).
  - **Uniformity** of the 100 p-values (chi-square over 10 bins) must give P-value ≥ 0.0001.
- `finalAnalysisReport.txt` is parsed to JSON by `analysis/parse_sts.py`; verdicts are computed
  by code from that JSON, never read off by eye.

### 6.2 NIST SP 800-90B Entropy Assessment (usnistgov/SP800-90B_EntropyAssessment, pinned commit recorded in `tools/`)
- `ea_non_iid` on 1,000,000 symbols of `y_i` (8 bits per symbol) from **two windows** of each
  capture: samples 0 to 999,999 and the last 1,000,000 samples. Also run on 1,000,000 bits of S1.
- `ea_iid` is run for information; if its permutation tests pass, the **non-IID** estimate is
  still the one used (conservative).
- The entropy figure used downstream is the tool's reported min-entropy per symbol
  (the minimum of its estimators, and of the literal and bit-string tracks as the tool reports it):
  **h8 = min over the two windows of C1**. h8 is in bits per 8-bit symbol, 0 ≤ h8 ≤ 8.
- Restart testing (SP 800-90B section 3.1.4) is **not performed**: it needs 1,000 power-cycled
  restarts of the source and is out of scope for this build. This is stated as a limitation.

### 6.3 dieharder (secondary, informational only)
- `dieharder -a -g 201 -f <file>` on S4, S5, CA and CP if dieharder installs cleanly.
- Results are reported but carry **no weight** in the verdicts: dieharder's file input rewinds
  on short files, and several of its tests need more than 10^8 bits.

## 7. Extractor sizing rule (fixed before h8 is known)

Let h8 be the measured min-entropy per 8-bit symbol from section 6.2. Both extractors use a
**safety factor of 0.5** on h8 and a **128-bit security margin** per block, corresponding to a
leftover-hash-lemma distance of 2^-64.

- **S4 TOEPLITZ:** input block n = 8192 bits (1024 symbols). Output per block
  `m = 8 × floor((0.5 × 1024 × h8 − 128) / 8)`.
  Examples: h8 = 4.0 → m = 1920; h8 = 2.0 → m = 896; h8 = 0.5 → m = 128.
  If m < 64 the source is declared **insufficient for extraction**, S4 is not produced, and that
  is reported as a result.
  The Toeplitz matrix is m × n with entries from the first m + n − 1 bits of a seed stream:
  ChaCha20 keyed by 32 bytes from `os.urandom`, generated once before C1, stored as
  `results/toeplitz_seed.bin` with its SHA-256 in the manifest. The seed is public, fixed and
  independent of the data, as a strong extractor requires.
- **S5 SHA256:** input symbols per 256-bit output `n_in = ceil(768 / h8)` symbols, so that
  `0.5 × n_in × h8 ≥ 384 = 256 + 128`.
  Examples: h8 = 4.0 → 192 symbols; h8 = 2.0 → 384; h8 = 0.5 → 1536.
  If h8 < 0.05 (n_in > 15,360) the source is declared insufficient and S5 is not produced.

Nothing about these rules may change after v1.0 except by a logged amendment (section 12).

## 8. Verdict rules

### 8.1 Per stream, SP 800-22 (applies to S1 to S5, CA, CC1, CC2, CP)
Let F_p = number of the 188 subtests failing the proportion criterion and F_u = number failing
uniformity.

| Verdict | Rule |
|---|---|
| **PASS** | F_p ≤ 2 and F_u ≤ 1, and no subtest has proportion < 0.90 or uniformity P-value < 10^-6 |
| **MARGINAL** | not PASS, but F_p ≤ 5 and F_u ≤ 2, and no subtest has proportion < 0.90 or uniformity P-value < 10^-6 |
| **FAIL** | anything else |

Rationale: for a truly random stream at these parameters the expected number of proportion
failures is about 0.7 (each subtest fails with probability ≈ 0.0037) and of uniformity failures
about 0.02, so a genuinely random stream gets PASS roughly 95 % of the time and FAIL essentially
never. "1 to 2 marginal failures are expected" is therefore built into PASS, not excused afterwards.

**Replication rule:** a MARGINAL on C1 is resolved by C2. If the *same* subtest(s) fail on C2
the stream is FAIL; if C2 is PASS, the stream is reported as "PASS (marginal on C1, replicated
clean on C2)". A FAIL on C1 is a FAIL; C2 is reported alongside but does not rescue it.

### 8.2 Source usability (SP 800-90B)
The source is **usable** if h8 ≥ 0.5 bits per symbol on both C1 windows and the two windows agree
within a factor of 2. It is **marginal** if 0.1 ≤ h8 < 0.5. It is **unusable** if h8 < 0.1 or the
windows disagree by more than a factor of 2 (non-stationary).

### 8.3 Stability
C2's h8 must be within a factor of 2 of C1's, and C2's S4/S5 verdicts (computed with C1's h8, not
re-sized) must not be FAIL. Otherwise the source is reported as unstable, whatever C1 said.

## 9. Hypotheses (falsifiable, stated before data)

| ID | Prediction | Supported if |
|---|---|---|
| H1 | The raw source has usable but far from full entropy | 0.5 ≤ h8 ≤ 6 on C1 |
| H2 | Raw bits are not random enough to pass as-is | S1 or S2 (or both) is FAIL on C1 |
| H3 | Von Neumann fixes bias but not correlation | S3 passes the Frequency test but S3 is not PASS overall (specifically, fails at least one of Runs, Serial, Approximate Entropy or Overlapping Template) |
| H4 | Properly sized extraction works | S4 and S5 are PASS (or PASS after replication) on C1 |
| H5 | Hashing hides everything | CA is PASS despite having zero entropy |
| H6 | The electronics alone are not a good source | CB's h8 is lower than C1's by at least a factor of 4, and CB's S1 is FAIL |
| H7 | The tests can detect bad streams | CC1 and CC2 are both FAIL |
| H8 | The reference passes | CP is PASS |
| H9 | The source is stable across sessions | Section 8.3 is satisfied |

H5 and H7 are the "honesty" hypotheses: if H5 fails, the hashing punchline is wrong and the
write-up changes accordingly; if H7 fails, the test harness itself is suspect and nothing else
can be reported until it is fixed.

Any hypothesis not supported is reported as not supported. There is no "try another capture
until it works" path: C1 is C1.

## 10. Provenance and reproducibility

- SHA-256 of every raw `.bin`, every derived stream and every control file is recorded in
  `experiment/DATA_MANIFEST.md` together with the metadata sidecar contents.
- Tool versions: STS 2.1.2 zip SHA-256, SP800-90B EA git commit, dieharder package version,
  Python and package versions, firmware git hash and `.uf2` SHA-256, all written into
  `results/summary.json` by `run_all.py`.
- `python analysis/run_all.py` regenerates `results/summary.json` and every figure from the raw
  data and the manifest. Any number in the paper, slides or README is pulled from that file.
- `experiment/CAPTURE_LOG.md` is a dated human log of every capture session, including failures.

## 11. Limitations acknowledged in advance

1. No SP 800-90B restart testing.
2. One hardware unit, one transistor, one room; no temperature control beyond logging it.
3. RP2040 ADC non-linearity and regulator ripple are measured (via CB) but not eliminated.
4. 100 × 10^6 bits is the standard STS size, not a large one; subtle long-range structure may be missed.
5. Passing SP 800-22 and having positive 90B min-entropy says nothing about the *physical origin*
   of the noise. Nothing in this experiment can show the source is "quantum"; see the philosophy essay.
6. The experimenter is also the analyst. The preregistration and mechanical verdict rules are the
   mitigation, not a cure.

## 12. Amendment log

| Date | Version | Change | Reason |
|---|---|---|---|
| 2026-10-29 | 0.1 | Initial draft | — |
