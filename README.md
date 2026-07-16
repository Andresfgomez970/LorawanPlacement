# Greenhouse LoRaWAN Coverage: Signal Modeling & Gateway Placement

This project predicts LoRaWAN signal coverage inside a commercial tomato
greenhouse and finds gateway positions that keep autonomous harvesting
robots reliably connected. It is built on a real walking-survey measurement
campaign (Milesight UC300 end devices + UG56 gateway, EU868).

**Reliable coverage** at a point means **RSSI > -105 dBm AND SNR > -3 dB**.
These thresholds already include a roughly 3 sigma safety margin above the point where the radio actually stops being able to decode the signal, so by default the pipeline treats the model's mean prediction as the coverage criterion  (`Z_MARGIN = 0`).

## Repository contents

| File | Purpose |
|---|---|
| `clean_data.csv` | Cleaned packet-level measurement dataset: RSSI/SNR per packet, grid position (`x_m`, `y_m`)|
| `train_model.ipynb` | Fits the models and saves all artifacts. Run this first. |
| `predictions.ipynb` | Loads the saved artifacts (never refits) and maps predicted coverage and uncertainty for the current gateway at the origin |
| `gateway_placement_clean.ipynb` | Finds optimal gateway positions for full-greenhouse coverage |

## How to run

Run the notebooks top to bottom, in this order:

1. **`train_model.ipynb`** reads `clean_data.csv`, fits everything, and
   saves the `.pkl` artifacts (path-loss coefficients, coordinate scaler,
   GPR models, CV results) into the working directory.
2. **`predictions.ipynb`** loads those artifacts and produces coverage
   heatmaps, uncertainty maps, and model performance figures for the
   measured gateway position.
3. **`gateway_placement_clean.ipynb`** loads the same artifacts and runs
   the placement optimization. It outputs `final_gateway_placement.csv`
   with the selected gateway coordinates, plus coverage maps.

Requirements: Python 3.11+, `numpy`, `pandas`, `scipy`, `scikit-learn`,
`joblib`, `matplotlib`, `libpysal`, `esda`.

## Method overview

**Two-stage signal model** (fitted separately for RSSI and SNR):

1. **Log-distance path loss** with a row-crossing penalty:
   `signal = intercept + slope * log10(distance) + beta * row_crossings`
   (rows spaced 1.65 m; gateway at the origin, x along rows, y across rows).
2. **GPR residual kriging** on what path loss can't explain. A Gaussian
   Process (Matern kernel, heteroscedastic per-cell noise from measured
   variance) learns the spatial pattern of local deviations, meaning dips
   and bumps tied to specific greenhouse locations. Moran's I confirmed the
   residuals are spatially structured, which justifies this second stage.
   All model evaluation uses GroupKFold CV grouped by grid cell to prevent
   spatial leakage.

**Gateway placement** (`gateway_placement_clean.ipynb`):

- Signal fades differently depending on direction. Along a row it only
  loses to distance, while across rows it also pays the row-crossing
  penalty. Each gateway's coverage footprint is therefore an **ellipse**
  (wider along rows), not a circle.
- The ellipse is sized from the fitted path loss, tightened by a
  conservative margin from the GPR (`RESIDUAL_QUANTILE`: the ellipse must
  still hold in the worst X% of locations by predicted local residual), and
  matched to the shape of the exact combined RSSI and SNR footprint.
- Because the ellipse only depends on distance from the gateway, it can be
  placed anywhere. **Greedy set cover** over a dense candidate grid picks
  positions until the greenhouse is covered.
- The chosen positions are then **re-checked against the full two-stage
  model**: path loss recomputed per gateway, plus the GPR's location-based
  residual map. That validated number is the one to quote, never the
  ellipse estimate.
- A **fallback mode** (`RUN_FALLBACK = True`) instead selects the best
  subset from a fixed list of real, feasible mounting positions (for
  example under structural constraints), using multi-restart greedy
  selection against the full model.

## Key settings (top of `gateway_placement_clean.ipynb`)

- `RSSI_THRESHOLD_DBM`, `SNR_THRESHOLD_DB`: the coverage definition.
- `Z_MARGIN`: extra uncertainty margin (`mean - z * std`). Default 0
  because the thresholds above already carry a worst-case margin. It can be
  set to any convenient value, but take into account that the margins
  stack: the thresholds already sit about 3 sigma below the hardware
  limits, so any additional z comes on top of that.
- `RESIDUAL_QUANTILE`: how conservative the ellipse is about known local
  signal dips. Lower means a smaller ellipse, more gateways, and a safer
  placement.
- `CANDIDATE_SPACING_M`, `MAX_GATEWAYS`: placement search granularity.

## Known limitations

- The GPR residual map was learned from measurements taken with **one
  gateway position**. Applying it to other candidate gateways assumes local
  dips belong to the *location* (canopy, structures) rather than to the
  specific gateway-to-point path. Validating this with measurements from a
  second gateway position is the main recommended follow-up.
- Prediction uncertainty grows in under-surveyed areas (the far north and
  south of the greenhouse). Additional walking-survey coverage there would
  tighten the model without any code changes.
- A reliable packet loss measurement would have made the model's predictions more accurate. With a clearer metric like packet delivery rate directly available, coverage wouldn't need to be gated on a worst-case RSSI/SNR margin in the first place, it could be judged on actual delivery success instead. .

---

Developed as part of a summer internship project with **Batenburg Beenen**
on LoRaWAN-supervised autonomous greenhouse robotics.
