# Quad Audio-Fingerprinting

Implementation of the Quad-based audio fingerprinting algorithm after Sonnleitner & Widmer (2016)
as a scale-invariant comparison system for a bachelor's thesis on audio fingerprinting.

> Sonnleitner, R. & Widmer, G. (2016). *Robust Quad-Based Audio Fingerprinting.*
> IEEE/ACM Transactions on Audio, Speech, and Language Processing, 24(3), 409–421.
> https://doi.org/10.1109/taslp.2015.2509248

---

## Project Description

This module implements the quad-based audio fingerprinting algorithm proposed by
Sonnleitner & Widmer (2016). Rather than combining peaks pairwise (as in Shazam),
the algorithm groups four spectral peaks into geometric quadruples (quads). Hashes
derived from the quad geometry are invariant to translation and scaling along both the
time axis and the frequency axis — making the system robust against tempo changes,
pitch shifts, and speed modifications that break landmark-based approaches.

Each detected query returns the best-matching track ID together with estimated scaling
factors ($s_\text{time}$, $s_\text{freq}$), which record how strongly the query was
shifted in time or frequency relative to the reference. Matching uses tolerance
parameters $\varepsilon_t = \varepsilon_p = 0.3$ (paper defaults, Section IV-A).

The project serves as the scale-invariant comparison system in a study that benchmarks
Shazam (Wang, 2003), Quad (Sonnleitner & Widmer, 2016), and NeuralFP
(Chang et al., 2021) across controlled distortion conditions.

**Research Question:** How do classical audio fingerprinting algorithms — in particular
Shazam's landmark-based approach — differ in robustness and efficiency from newer
methods using quad-based or ML-driven techniques?

---

## Project Structure

```
quad_fingerprint/
├── config.py          # Central configuration of all parameters
├── audio_loader.py    # Load and normalise audio files
├── spectrogram.py     # STFT spectrogram computation
├── peak_finder.py     # Constellation map: extract local maxima
├── quad_builder.py    # Quad formation (4-peak groups) and hash encoding
├── database.py        # Fingerprint database (in-memory hash table)
├── matcher.py         # Hash lookup, scale estimation, and scoring
├── evaluate.py        # Robustness and efficiency metrics
├── pipeline.py        # End-to-end pipeline (ingest, query, evaluation)
├── visualization.py   # Plots for the bachelor's thesis
└── tests/
    ├── test_spectrogram.py
    ├── test_peak_finder.py
    ├── test_quad_builder.py
    ├── test_matcher.py
    └── test_pipeline.py
```

---

## Installation

### Prerequisites

- Python **3.10** or newer (`audiofp_classical` conda environment)
- pip

### Steps

```bash
# Clone repository
git clone <repo-url>
cd quad

# Create virtual environment (recommended)
python -m venv .venv
source .venv/bin/activate   # macOS/Linux
.venv\Scripts\activate      # Windows

# Install dependencies
pip install -r requirements.txt

# Install package in development mode
pip install -e .
```

---

## Quickstart

### 1. Load reference songs into the database

```python
from quad_fingerprint.database import FingerprintDatabase
from quad_fingerprint.pipeline import ingest_directory

db = FingerprintDatabase()
stats = ingest_directory("data/reference/", db)
print(f"{stats['processed']} songs, {stats['total_hashes']} hashes")

db.save()
```

### 2. Identify unknown audio

```python
from quad_fingerprint.pipeline import query

result = query("data/queries/unknown_clip.wav", db)

if result.match_found:
    print(f"Matched: {result.best_match}  (score: {result.best_score})")
    print(f"Time scale: {result.detected_time_scale:.3f}  "
          f"Freq scale: {result.detected_freq_scale:.3f}")
else:
    print("No match found.")
```

### 3. Query a clip segment (e.g., 10 seconds from offset 30 s)

```python
result = query("data/queries/unknown_clip.wav", db,
               start_sec=30.0, duration_sec=10.0)
```

### 4. Load a saved database

```python
db = FingerprintDatabase()
db.load()   # loads from data/fingerprint_db.pkl (config.DB_PATH)
```

### 5. Evaluate robustness

```python
from quad_fingerprint.pipeline import evaluate_robustness

report = evaluate_robustness(
    query_dir="data/queries/",
    database=db,
    duration_sec=10.0,
)
print(f"Hit rate:          {report.recognition_rate:.1%}")
print(f"False-negative:    {report.false_negative_rate:.1%}")
print(f"Avg. query time:   {report.avg_query_time_ms:.1f} ms")
print(f"Scale MAE (time):  {report.scale_mae_time:.4f}")
print(f"Scale MAE (freq):  {report.scale_mae_freq:.4f}")
```

### 6. Run tests

```bash
pytest quad_fingerprint/tests/ -v
```

---

## Parameter Reference (`config.py`)

All algorithmic parameters are defined centrally in `config.py`.
No module uses hardcoded values.

### Audio Preprocessing

| Parameter | Default | Description |
|---|---|---|
| `SAMPLE_RATE` | `22050` | Target sample rate in Hz. Covers the musically relevant frequency range up to 11 kHz. |
| `MONO` | `True` | Always convert to mono. Fingerprinting requires one channel only. |
| `DURATION` | `None` | Load the full file. Set to a fixed duration for query clips. |
| `QUERY_DURATION_SEC` | `10.0` | Default query clip length in seconds. |

### Spectrogram (STFT)

| Parameter | Default | Description |
|---|---|---|
| `N_FFT` | `4096` | FFT window size. Yields ~5.4 Hz frequency resolution at 22050 Hz. |
| `HOP_LENGTH` | `2048` | 50 % overlap. Good trade-off between time resolution and compute. |
| `WINDOW` | `"hann"` | Hann window reduces spectral leakage (standard for audio STFT). |
| `MIN_FREQUENCY_HZ` | `300.0` | Lower frequency bound. DC and sub-bass components are less robust under distortions. |
| `MAX_FREQUENCY_HZ` | `5000.0` | Upper frequency bound. Most discriminative musical structure lies below 5 kHz. |

### Peak Extraction (Constellation Map)

| Parameter | Default | Description |
|---|---|---|
| `PEAK_NEIGHBORHOOD_SIZE_FREQ` | `20` | Frequency-bin neighbourhood for `maximum_filter()`. A peak must exceed all neighbours in a rectangle (Sonnleitner & Widmer, Section III-A). |
| `PEAK_NEIGHBORHOOD_SIZE_TIME` | `20` | Time-frame neighbourhood for `maximum_filter()`. |
| `MAX_PEAKS_PER_SECOND` | `30` | Density criterion: max peaks per 1-second segment. Ensures reasonably uniform coverage. |
| `AMPLITUDE_THRESHOLD_DB` | `-60.0` | Minimum peak amplitude in dB. Peaks below this threshold are treated as noise. |

### Quad Formation and Hash Encoding

| Parameter | Default | Description |
|---|---|---|
| `QUAD_MAX_TIME_DIFF` | `60` | Maximum time-frame span for a valid quad. Limits the temporal extent of each 4-peak group. |
| `QUAD_MAX_FREQ_DIFF` | `100` | Maximum frequency-bin span for a valid quad. |
| `QUADS_PER_ROOT` | `5` | Number of quads formed per root peak. Analogous to fan-out in Shazam. |
| `EPSILON_T` | `0.3` | Time-scale tolerance parameter $\varepsilon_t$. Paper default (Section IV-A): a candidate match is accepted if its detected time scale deviates by at most $\varepsilon_t$ from the reference quad. |
| `EPSILON_P` | `0.3` | Frequency-scale tolerance parameter $\varepsilon_p$. Paper default (Section IV-A). |

The hash encodes the normalised internal geometry of a quad — specifically the
relative positions of the four peaks projected onto the unit square. This geometric
normalisation makes the hash invariant to global translation and scaling in both axes
(Sonnleitner & Widmer, Section III-B):

```
Hash layout (Sonnleitner & Widmer, Section III-B):
[x1 : N bit] [y1 : N bit] [x2 : N bit] [y2 : N bit]
where (x1, y1), (x2, y2) are the two inner peaks projected onto the unit square
defined by the outer peak pair.
```

### Matching and Scale Estimation

| Parameter | Default | Description |
|---|---|---|
| `MIN_HASH_MATCHES` | `5` | Minimum hash hits before a candidate is scored. |
| `MATCH_THRESHOLD` | `10` | Minimum score for a positive match. Calibrate experimentally for the target database size. |
| `SCALE_BIN_WIDTH_T` | `0.01` | Bin width for the time-scale vote histogram. Controls scale-estimation resolution. |
| `SCALE_BIN_WIDTH_P` | `0.01` | Bin width for the frequency-scale vote histogram. |

After matching, the system reports `detected_time_scale` and `detected_freq_scale`
as the peak-vote scales. For Group A (scaling conditions) these are compared against
`true_time_scale` and `true_freq_scale` to compute the scale estimation MAE.

---

## Theoretical Background

All algorithmic decisions are based on:

> Sonnleitner, R. & Widmer, G. (2016). *Robust Quad-Based Audio Fingerprinting.*
> IEEE/ACM Transactions on Audio, Speech, and Language Processing, 24(3), 409–421.
> [PDF: `docs/sonnleitner2016.pdf`]

Relevant sections:
- **Section III-A** — Constellation map and peak extraction
- **Section III-B** — Quad formation, unit-square projection, hash encoding
- **Section III-C** — Database lookup and scale estimation via vote histograms
- **Section IV-A** — Tolerance parameters $\varepsilon_t$ and $\varepsilon_p$, calibration
- **Section IV-B** — Experimental evaluation, robustness against tempo/pitch/speed shifts

---

## Evaluation (Bachelor's Thesis)

The evaluation pipeline (`evaluate_robustness`) computes:

**Robustness**
- `recognition_rate` — fraction of correctly identified queries (Hit Rate)
- `false_negative_rate` — missed queries where the song is in the database
- `false_positive_rate` — false matches where the song is not in the database

**Scale Estimation (Group A only)**
- `scale_mae_time` — mean absolute error between `detected_time_scale` and `true_time_scale`
- `scale_mae_freq` — mean absolute error between `detected_freq_scale` and `true_freq_scale`

**Efficiency**
- `avg_fingerprint_time_ms` — average fingerprint generation time per query
- `avg_query_time_ms` — average matching time per query
- `db_memory_mb` — database memory footprint
- `hashes_per_second` — fingerprinting throughput

Tested distortions: tempo shifts (80–120 %), pitch shifts (±1–2 semitones),
speed shifts (80–120 %), white noise (SNR 20–−5 dB), room impulse response,
MP3 compression (128 kbps, 64 kbps), and combined real-world scenarios.

Audio datasets: [FMA](https://github.com/mdeff/fma) (in-domain),
[GTZAN](http://marsyas.info/downloads/datasets.html) (out-of-distribution),
[MUSAN](https://www.openslr.org/17/) (noise augmentation).
