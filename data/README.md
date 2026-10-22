# Data guide: synthetic MFC dataset (v2)
This guide explains what is in this `data/` folder, what every file and column means, and how to use them
without mixing anything up. You don't need any other document to start working with the data.
---

## 1. The experiment in one paragraph

A **microbial fuel cell (MFC)** is a small reactor where bacteria growing on an electrode (the *biofilm*) eat a
substrate (acetate) and release electrons, which produces a voltage across an external resistor (**R_ext**).
We simulate **48 runs** (one reactor each) for **28 days** each:

- **16 runs at normal gravity (1 g):** the reactor stands still.
- **32 runs under simulated microgravity:** the reactor turns on a **Random Positioning Machine (RPM)**.
  There are 8 runs at each of **10, 15, 21 and 26 rpm**.

During a run:
- **Days 0–14:** the biofilm grows (acclimation) at a fixed R_ext of 1000 Ω.
- **After day 14:** R_ext is changed every 12–48 h, between 100 Ω and 10 kΩ.
- **Feeding:** fresh substrate is added every ~2 days (occasionally a feed is skipped, so the cell starves).
- **Polarization tests (LSV):** about 5 times per run, the cell is disconnected for 30 min, and then a
  voltage–current curve is measured.

A data logger records voltage every **15 min**. An accelerometer (IMU) records gravity every **5 min**.

---

## 2. Folder layout

```
data/
├── README.md                          ← this file
├── synthetic_v2/                      ← MAIN dataset (use this one)
│   ├── run_log.csv            ┐
│   ├── events.csv             │
│   ├── das_timeseries.csv     │  INPUTS: what a real experiment would give you
│   ├── lsv_curves.csv         │
│   ├── imu.csv                ┘
│   ├── splits.csv                     ← suggested train / val / test split by run
│   ├── ground_truth.csv       ┐  🔒 HIDDEN ANSWERS: only for scoring your models,
│   ├── run_params_hidden.csv  ┘     never use as model input
│   └── overview.png                   ← plot of 4 example runs
└── synthetic_v2_no_gravity_effect/    ← NEGATIVE CONTROL (same files, same names)
```

### Which folder to use

| Folder | What it is | When to use it |
|---|---|---|
| `synthetic_v2/` | Microgravity changes the cell's behaviour (as reported by Yi 2024, Li 2026, Wang 2025a) | All normal development and results |
| `synthetic_v2_no_gravity_effect/` | **Identical** runs, schedules, noise, gaps and spikes, but microgravity has **no** effect | Sanity check: a correct model must find *no* gravity effect here. If it does, something is leaking or overfitting |

### Which files a model may use

| File | Model input? | Why |
|---|---|---|
| `run_log.csv`, `events.csv`, `das_timeseries.csv`, `lsv_curves.csv`, `imu.csv` | ✅ Yes | Exists in a real experiment |
| `splits.csv` | ✅ Yes (for splitting) | Our suggested split |
| `ground_truth.csv`, `run_params_hidden.csv` | ❌ **Never** | Hidden true values that don't exist in a real experiment. Use them **only to measure** how well the EKF, physics fit or forecast did |

---

## 3. How the files connect

- **`run_id`** (`R01` … `R48`) is the key in every file. One run = one reactor = one 28-day experiment.
- **`timestamp`** is local lab time, format `YYYY-MM-DD HH:MM:SS`, with no time zone. Runs start on different
  dates (staggered by 3 days, from 2026-10-01 to 2027-02-19), so **don't compare runs by calendar date**. Use the
  time since inoculation: `timestamp − run_log.inoculation_time`.
- **Sorting:** `das_timeseries.csv` and `ground_truth.csv` are sorted by `run_id`, then `timestamp`.
- **Joining the DAS log and ground truth:** match on (`run_id`, `timestamp`); both use the same 15-min grid.
- **Finding R_ext at any time:** take the latest `R_ext_change` event at or before that time (pandas
  `merge_asof(..., direction="backward")`).

---

## 4. File-by-file column reference

### 4.1 `run_log.csv`: one row per run (48 rows)

| Column | Unit | Meaning | Example |
|---|---|---|---|
| `run_id` | – | Run identifier | `R02` |
| `reactor_id` | – | Reactor name (one reactor per run) | `SIM-R02` |
| `gravity` | – | `1g` or `microgravity` | `microgravity` |
| `simulator` | – | `static` (standing still) or `RPM` (rotating) | `RPM` |
| `rpm_mode` | – | RPM rotation mode: `IRM` (inner random mode), or `-` for static | `IRM` |
| `rpm` | rev/min | Rotation speed: 0, 10, 15, 21 or 26 | `21` |
| `inoculation_time` | datetime | Start of the run (day 0) | `2026-10-04 12:00:00` |
| `electrode_area_cm2` | cm² | Anode area (4.85 cm², all runs). Use it to convert to power density | `4.853` |
| `anode_volume_mL` | mL | Anode chamber volume | `5.0` |
| `inoculum_batch` | – | Bacteria batch `B01`–`B16`. **Each microgravity run has a 1 g "partner" run from the same batch.** Compare µg vs 1 g within the same batch | `B09` |
| `temperature_nominal_C` | °C | Target temperature | `25.0` |
| `das_interval_min` | min | Voltage logging interval | `15` |
| `data_type` | – | Always `SYNTHETIC` | |

### 4.2 `events.csv`: everything that was changed during a run

Columns: `run_id`, `timestamp`, `event`, `value`, `unit`.

| `event` | `value` / `unit` | Meaning | Count |
|---|---|---|---|
| `R_ext_change` | ohm | New external resistance from this time onward. The first one (1000 Ω) is at the start | 521 |
| `feeding` | g COD/L | Medium replaced; the substrate is reset to this concentration. Also happens at every `open_circuit_start` | 829 |
| `feeding_skipped` | – | A planned feed that didn't happen (the cell may starve afterwards) | 45 |
| `open_circuit_start` | min (30) | Cell disconnected for 30 min before a polarization test | 240 |
| `lsv_start` | mV/s (1.0) | Polarization sweep starts (lasts ≤ 15 min) | 240 |

Rule built into the design: a load change never happens within 2 h of a feeding.

### 4.3 `das_timeseries.csv`: the main sensor log (every 15 min, 127 452 rows)

This is what the DAS-5000 data logger would record.

| Column | Unit | Meaning |
|---|---|---|
| `run_id`, `timestamp` | | Run and time |
| `V_meas_V` | **V** (volts, not mV) | Measured cell voltage. Typical range 0–0.5 V |
| `I_meas_A` | **A** (amperes, not mA) | Current, **calculated** as `V_meas_V / R_ext`, as the real logger does. It is not an independent sensor |
| `T_meas_C` | °C | Measured temperature (daily cycle of about ±1.5 °C, plus sensor noise) |

**Deliberate problems your cleaning code must handle:**

| Problem | How it looks | How many |
|---|---|---|
| Disconnected (open circuit + LSV) | `V_meas_V` and `I_meas_A` are **empty (NaN)** for ~45 min around each LSV | 5 times per run |
| Random gaps | Rows simply **missing** (the timestamp jumps by more than 15 min) | ~1 % of rows |
| Outages | 1–6 h of missing rows | in ~40 % of runs |
| Spikes | Single-point jumps of ±50–150 mV (the current jumps with them) | ~3 per run |
| Negative voltages | Can appear when a spike hits a near-zero voltage (a starved cell) | rare |
| Sensor noise | Correlated noise of ~1–2 mV, plus 0.3 % of V | everywhere |

### 4.4 `lsv_curves.csv`: polarization curves (240 sweeps, ~60–78 points each)

Each sweep lowers the cell voltage from open circuit (~0.75 V) down to 0.02 V at 1 mV/s, recording the current
every 10 mV. This is the input for the **Butler–Volmer physics model**.

| Column | Unit | Meaning |
|---|---|---|
| `run_id` | – | Run |
| `lsv_timestamp` | datetime | Start of the sweep. **One sweep = one (`run_id`, `lsv_timestamp`) pair** |
| `phase` | – | Biofilm phase when measured: `early` (< 7 d), `growth` (7–14 d), `mature` (≥ 14 d) |
| `I_A` | A | Current at this point (tiny negative values near open circuit are noise) |
| `V_V` | V | Cell voltage at this point |
| `OCV_V` | V | Open-circuit voltage measured just before the sweep (the same on every row of a sweep) |

Sweeps are on days ~3, 10, 17, 24 and 27.4 of each run.

### 4.5 `imu.csv`: accelerometer (every 5 min, 387 072 rows)

Each row is the average of 10 Hz samples over 5 minutes.

| Column | Unit | Meaning |
|---|---|---|
| `acc_x_mean_g`, `acc_y_mean_g`, `acc_z_mean_g` | g | Mean acceleration per axis |
| `acc_inst_std_g` | g | Spread of the instantaneous samples (large on the RPM because the sample keeps rotating) |

**How to read it:** gravity magnitude = √(x² + y² + z²).
- 1 g runs: ≈ (0, 0, 1), magnitude ≈ 1.
- RPM runs: magnitude ≈ 0.02–0.04 g.

This is how a real experiment confirms that microgravity was achieved.

### 4.6 `splits.csv`: suggested split (by whole run)

| Column | Meaning |
|---|---|
| `run_id` | Run |
| `split` | `train` (31 runs), `val` (6), `test` (11) |
| `gravity`, `rpm` | Repeated here for convenience |

Every condition (1 g, 10, 15, 21, 26 rpm) appears in every split. **Always split by run, never by random rows.**
Neighbouring 15-min rows are almost identical, so a row-level split leaks the answer. Use the same split for every
model you compare (TFT, LSTM, GRU, ARIMA, …).

### 4.7 🔒 `ground_truth.csv`: hidden true state (every 15 min, 129 024 rows)

The model's internal "truth" at every DAS timestamp, including the rows the DAS log is missing.
**Only for evaluation.**

| Column | Unit | Meaning |
|---|---|---|
| `run_id`, `timestamp` | | Same grid as `das_timeseries.csv` |
| `mode` | – | `load` (connected to R_ext) or `open_circuit` (disconnected for an LSV) |
| `biofilm_X` | – | Amount of electroactive biofilm (0 = none, 1 = mature 1 g biofilm; up to ~1.3 in microgravity). **EKF target** |
| `substrate_S_gCOD_L` | g COD/L | Food left in the anode (0 = starving). **EKF target** |
| `mediator_M` | – (0–1) | Fraction of the bacteria's internal electron carrier (NADH-like) that is "charged". **EKF target** |
| `E_ocv` | V | True open-circuit voltage |
| `I0` | A | True exchange current (Butler–Volmer) |
| `alpha` | – | True charge-transfer coefficient |
| `R_ohmic` | Ω | Ohmic part of the internal resistance (membrane, electrolyte, cathode) |
| `R_anode` | Ω | Biofilm/anode part of the internal resistance (falls as the biofilm grows) |
| `R_int` | Ω | Total internal resistance = `R_ohmic + R_anode`. **Compare your physics fit and EKF to this** |
| `I_L` | A | True limiting current |
| `gravity_effect_level` | 0–1 | How strongly microgravity acts at this moment (0 at 1 g; ramps up over 0.5–4 days on the RPM) |
| `T_K` | K | True temperature |
| `R_ext_ohm` | Ω | External resistance at this moment |
| `V_true_V`, `I_true_A` | V, A | Noise-free voltage and current |

### 4.8 🔒 `run_params_hidden.csv`: parameters drawn for each run

One row per run with the settings the simulator used. Most are in **area-specific units** (Ω·m², A/m²).
They are mainly for checking. Main ones:

| Column | Meaning |
|---|---|
| `E_ref`, `j0_max`, `alpha`, `R_min`, `R_max`, `jL_max` | Electrochemistry of that run |
| `mu_max`, `K_S`, `k_d`, `X0`, `CE` | Biology: growth rate, substrate half-rate, decay, starting biofilm, coulombic efficiency |
| `tau_mediator_min` | Time constant of the slow electrical response after a load change (minutes) |
| `m_jL`, `m_X`, `m_E`, `m_act`, `m_Ra`, `tau_g_d` | Microgravity multipliers at full effect and onset time. **They only act when `rpm_strength` > 0**, so for 1 g runs they are present but unused. In the negative-control folder they are all 1 |
| `rpm_strength`, `shear_factor` | How much of the microgravity effect each rpm gets, and the extra resistance from rotation |

---

## 5. Units cheat-sheet (avoid the common mistakes)

| Quantity | Unit in files | Common mistake |
|---|---|---|
| Voltage | V | Treating it as mV (0.25 means 250 mV) |
| Current | A | Treating it as mA (2.4e-04 A = 0.24 mA) |
| Resistance | Ω | – |
| Power | compute P = V × I (W) | To get mW/m², divide by the area (4.85 cm² = 4.853e-4 m²) and × 1000 |
| Substrate | g COD/L | – |
| Gravity | g (1 g = 9.81 m/s²) | – |
| Time | datetime | Compare runs by **day since inoculation**, not by date |

---

## 6. Glossary

| Term | Meaning |
|---|---|
| **1 g / µg** | Normal gravity / (simulated) microgravity |
| **RPM** | Random Positioning Machine: rotates the reactor randomly so gravity averages out to ~0 |
| **IRM** | Inner Random Mode: the RPM setting used (Wang 2025a found it gives the cleanest microgravity) |
| **R_ext** | External resistor (the "load"); we control it |
| **R_int** | Internal resistance of the cell; it falls as the biofilm grows |
| **OCV** | Open-circuit voltage (no current flowing) |
| **LSV / polarization curve** | Voltage vs current curve measured by sweeping the voltage; used to fit the physics model |
| **Butler–Volmer** | Physics equation for the voltage losses; gives V_phys |
| **Max power point** | The R_ext that gives the most power (~1 kΩ here); the controller should find it |
| **Biofilm** | The layer of bacteria on the anode that makes the electricity |
| **COD** | Chemical oxygen demand: a standard way to measure how much "food" is in the liquid |
| **Inoculum batch** | The bacteria source a run was started from. Runs from the same batch are similar, so compare within a batch |
| **Negative control** | The copy of the data where gravity has no effect, to catch false discoveries |

---

## 7. Quick start (Python)

```python
import pandas as pd

D = "data/synthetic_v2/"
runs   = pd.read_csv(D + "run_log.csv", parse_dates=["inoculation_time"])
events = pd.read_csv(D + "events.csv", parse_dates=["timestamp"])
das    = pd.read_csv(D + "das_timeseries.csv", parse_dates=["timestamp"])
lsv    = pd.read_csv(D + "lsv_curves.csv", parse_dates=["lsv_timestamp"])
split  = pd.read_csv(D + "splits.csv")

# day since inoculation
das = das.merge(runs[["run_id", "gravity", "rpm", "inoculum_batch", "inoculation_time"]], on="run_id")
das["day"] = (das.timestamp - das.inoculation_time).dt.total_seconds() / 86400

# attach R_ext (latest change at or before each row)
rext = (events[events.event == "R_ext_change"][["run_id", "timestamp", "value"]]
        .rename(columns={"value": "R_ext"}).sort_values("timestamp"))
das = pd.merge_asof(das.sort_values("timestamp"), rext, on="timestamp", by="run_id",
                    direction="backward").sort_values(["run_id", "timestamp"])

# power density (mW/m2)
das["P_mW_m2"] = das.V_meas_V * das.I_meas_A / 4.853e-4 * 1000

# split by run
train = das[das.run_id.isin(split.query("split == 'train'").run_id)]

# one LSV sweep
one = lsv[(lsv.run_id == "R01") & (lsv.lsv_timestamp == lsv[lsv.run_id == "R01"].lsv_timestamp.max())]
```

For a ready-made cleaning and EKF pipeline, `../physics_synthetic/mfc_ekf` (`preprocess`, `run_ekf`) reads this folder
directly: `preprocess("data/synthetic_v2", PreprocConfig(dt_min=15))`.

---

## 8. Things that are intentional (not bugs)

- **Very low voltage in the first ~10 days:** the biofilm is still growing.
- **Voltage dropping to ~0 for hours or days:** the substrate ran out (a skipped feed or a fast-eating cell). The cell recovers after the next feed.
- **Small overshoot after R_ext is lowered:** the bacteria's internal electron store discharges; it settles within ~1–2 h.
- **Microgravity runs aren't better in every single case:** batches differ a lot. Compare µg vs 1 g **within the same `inoculum_batch`**.
- **21 and 26 rpm show smaller gains than 10–15 rpm:** too much rotation adds shear (as Wang 2025a reports).

---

## 9. More detail

| File | What it has |
|---|---|
| `../README.md` | Why the data was built this way, sources, limitations |
| `../PARAMETER_SOURCES.md` | Every simulator parameter and the paper it comes from |
| `../VALIDATION_REPORT.md` | All checks the data passed |
| `../DATA_DICTIONARY.md` | Shorter technical column list |
| `../generate_synthetic_v2.py` | Regenerates the data (`python generate_synthetic_v2.py`) |
