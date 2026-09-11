# Model 1 (Informed) — Data Pipeline & Algorithm Design

Scope: this document covers **Model 1 only** (age, sex, RFV1–RFV5, PAINSCALE, plus the five vitals). Model 2 (restricted) is being redesigned separately and isn't addressed here. Source file: **2022 NHAMCS ED public use file**.

**Update: the raw file has been acquired and Stages 1–4 below have actually been run.** Two deliverables now exist alongside this doc: `nhamcs_2022_ed_raw_full.csv` (all 913 official variables, 16,025 visits) and `nhamcs_2022_model1_processed.csv` (the Model 1 feature/outcome table described in Stage 4), plus `model1_data_dictionary.md` describing every column in the processed file. The column positions and value labels below were confirmed directly against CDC's own SAS input/format statements and cross-checked against the real data (e.g. `PATWT` sums to exactly 155,397,747, matching the documented national estimate) — not re-derived from a summarized PDF excerpt. Two figures below were corrected from the earlier reference doc as a result: `IMMEDR` missingness and the resource-outcome construction (see Stage 3).

---

## 0. Files needed (all confirmed on CDC's servers)

| File | Purpose | Link |
|---|---|---|
| `ed2022.zip` (1.67 MB) | Raw fixed-width ASCII data, 16,025 records | https://ftp.cdc.gov/pub/health_statistics/nchs/Datasets/NHAMCS/ed2022.zip |
| `readme2022.txt` | Notes on the release | https://ftp.cdc.gov/pub/health_statistics/nchs/Datasets/NHAMCS/readme2022.txt |
| `ed22inp.txt` | **SAS INPUT statement** — exact column position of every variable | https://ftp.cdc.gov/pub/Health_Statistics/NCHS/Dataset_Documentation/NHAMCS/sas/ed22inp.txt |
| `ed22for.txt` | SAS FORMAT statement — value-code → label mapping | https://ftp.cdc.gov/pub/Health_Statistics/NCHS/Dataset_Documentation/NHAMCS/sas/ed22for.txt |
| `ed22lab.txt` | SAS LABEL statement — variable descriptions | https://ftp.cdc.gov/pub/Health_Statistics/NCHS/Dataset_Documentation/NHAMCS/sas/ed22lab.txt |
| `doc22-ed-508.pdf` | Full technical documentation / codebook | https://ftp.cdc.gov/pub/Health_Statistics/NCHS/Dataset_Documentation/NHAMCS/doc22-ed-508.pdf |

**Rule carried over from the project instructions: never hand-guess column ranges.** Stage 1 below parses the file using the positions straight out of `ed22inp.txt`, not eyeballed positions.

---

## Stage 1 — Parse the raw fixed-width file

`ed2022.zip` unzips to a single fixed-width text file, one line per ED visit (16,025 lines). Each variable lives at a fixed `@column` position. Positions below are pulled directly from the official 2022 SAS input statement:

| Variable | Column (`@N`) | Width/informat | Description |
|---|---|---|---|
| `AGE` | 16 | 3 | Patient age (years) |
| `SEX` | 25 | 1 | Patient sex |
| `TEMPF` | 48 | 4 | Temperature (°F) |
| `PULSE` | 52 | 3 | Heart rate |
| `RESPR` | 55 | 3 | Respiratory rate |
| `BPSYS` | 58 | 3 | Systolic BP |
| `BPDIAS` | 61 | 3 | Diastolic BP |
| `POPCT` | 64 | 3 | Pulse oximetry (%) |
| `IMMEDR` | 67 | 2 | Triage immediacy (label candidate) |
| `PAINSCALE` | 69 | 2 | Pain scale (0–10) |
| `RFV1` | 73 | 5 | Reason for visit #1 |
| `RFV2` | 78 | 5 | Reason for visit #2 |
| `RFV3` | 83 | 5 | Reason for visit #3 |
| `RFV4` | 88 | 5 | Reason for visit #4 |
| `RFV5` | 93 | 5 | Reason for visit #5 |
| `DIAG1`–`DIAG3` | 122, 126, 130 | $CHAR4 | Diagnoses — **exclusion list, see Stage 3c** |
| `NUMMED` | 457 | 2 | Number of medications |
| `MED1` | 243 | $CHAR5 | First medication entry — **exclusion list** |
| `DRUGID1` | 630 | $CHAR6 | Drug ID — **exclusion list** |

Resource/outcome-checklist columns (needed for Stage 3b, not as Model 1 predictors):

| Variable | Column | Item |
|---|---|---|
| `CBC` | 184 | CBC |
| `CARDENZ` | 183 | Cardiac enzymes |
| `BLOODCX` | 186 | Blood culture |
| `TRTCX` | 186* | Throat culture (*verify — conflicted with BLOODCX in extraction, confirm against printed codebook) |
| `URINECX` | 188 | Urine culture |
| `ELECTROL` | 192 | Electrolytes |
| `GLUCOSE` | 193 | Glucose, serum |
| `LACTATE` | 194 | Lactate |
| `URINE` | 204 | Urinalysis / dipstick |
| `PREGTEST` | 202 | Pregnancy test |
| `TOXSCREN` | 203 | Toxicology screen |
| `ABG` | 178 | Arterial blood gases |
| `EKG` | 199 | EKG/ECG |
| `CARDMON` | 198 | Cardiac monitor |
| `XRAY` | 207 | X-ray |
| `CATSCAN` | 208 | CT scan |
| `MRI` | 216 | MRI |
| `ULTRASND` | 219 | Ultrasound |

Disposition/admission columns (exclusion list, not predictors — see Stage 3c):

| Variable | Column | Item |
|---|---|---|
| `LWBS` | 491 | Left without being seen |
| `LEFTAMA` | 493 | Left against medical advice |
| `DIEDED` | 495 | Died in ED |
| `TRANNH` | 496 | Transfer to nursing home |
| `TRANOTH` | 498 | Transfer, non-psych hospital |
| `ADMITHOS` | 499 | Admit to this hospital |
| `OBSHOS` | 500 | Admit to observation, then hospitalized |
| `ADISP` | 531 | Disposition of live discharges (summary) |

**Two items flagged for verification against the printed codebook / SAS format file before coding, since automated extraction couldn't fully confirm them:** the `TRTCX`/`BLOODCX` column collision above, and the exact column for `PATWT` (patient visit weight — needed for the weighted-vs-unweighted sensitivity check in evaluation, not for training itself, so it doesn't block Model 1 but should be located before the reporting stage).

**Output of Stage 1:** one row per visit, one column per variable above, all still in raw CDC-coded form (no recoding yet).

---

## Stage 2 — Recode sentinel missing codes to true NaN

Apply before anything else touches the data:

| Raw code | Meaning | Applies to |
|---|---|---|
| `-9` | Blank field | General numeric/vitals/RFV fields |
| `ZZZ0`, `ZZZ2`, `ZZZ3`, `ZZZ4`, `ZZZ5` | Various non-diagnosis / uncodable entries | `DIAG1`–`DIAG5` |
| `99980` | Unknown drug entry | Drug fields |
| `99999` | Illegible drug entry | Drug fields |
| `0` / `9` on `IMMEDR` | "No triage" / "Unknown" — decide whether to treat as missing or as its own category | `IMMEDR` |

Write this as one shared recode function/lookup so every stage downstream sees real `NaN`, not a numeric-looking sentinel.

---

## Stage 3 — Define the two outcomes

### 3a. Urgency label — `IMMEDR`
Nurse-assigned immediacy rating (1=Immediate, 2=Emergent, 3=Urgent, 4=Semi-urgent, 5=Nonurgent). **Decision already made for this phase: `IMMEDR` is the label, never a Model 1 predictor.** Confirmed value labels from CDC's own format statement: -9=Blank, -8=Unknown, 0="no triage for this visit but ESA does conduct triage", 7="visit occurred in ESA that does not conduct nursing triage". All four of these are unusable as an acuity rating and are set to missing — **36.3% of visits**, corrected from the 27.8% cited in the earlier reference doc (that figure only counted blank+unknown; the two "not applicable" triage codes add another ~10 points). Visits with missing `IMMEDR` are dropped from the urgency task, not imputed, since imputing a clinical judgment call is a bigger assumption than imputing a vital sign.

### 3b. Resource-use label — 0 / 1 / 2+ count (Sterling et al. 2020 style)
CDC already provides pre-aggregated counts: `TOTDIAG` ("total number of diagnostic services ordered or provided") and `TOTPROC` ("total number of procedures provided"). The processed dataset buckets `TOTDIAG + TOTPROC` into {0, 1, 2+} directly — simpler than manually summing the full imaging/lab/procedure checklist as originally planned. **Two judgment calls are still open before this is the final definition, not resolved here:** (1) whether medications (`NUMMED`) should count toward "resources" at all under the ESI convention Sterling et al. used, and (2) whether `TOTDIAG`/`TOTPROC` bundle tests the same way ESI resource-counting does (e.g. a lab panel as one resource, not one per test). Sign off on both with Will before treating `resource_label` as final — see the data dictionary for the same caveat.

### 3c. Hard exclusions from Model 1's predictor set
`DIAG1`–`DIAG5`, `MED1`–`MED30`, `DRUGID1`–`DRUGID30`, `RX*CAT*`, and every disposition/admission variable (`LWBS`, `LEFTAMA`, `DIEDED`, `TRANNH`, `TRANOTH`, `ADMITHOS`, `OBSHOS`, `ADISP`) — these either define an outcome or leak it. `NUMMED` is the one exception: it's allowed as a *label component* (3b) but must never also appear as a *predictor* in the same model run.

---

## Stage 4 — Final Model 1 feature schema

This is the target shape of the processed dataset — one row per ED visit:

| Column | Type | Notes |
|---|---|---|
| `visit_id` | string/int | Row identifier we assign, not a CDC field |
| `age` | numeric | From `AGE` |
| `sex` | categorical (2) | From `SEX` |
| `rfv_group_1`…`rfv_group_k` | categorical / one-hot | `RFV1`–`RFV5` collapsed into a small number of clinically meaningful groups (module-level or complaint-level — pending the grouping-scheme decision) |
| `pain_scale` | numeric (0–10) or NaN | From `PAINSCALE` |
| `pain_scale_missing` | binary flag | 1 if `PAINSCALE` was NaN |
| `temp_f` | numeric or NaN | From `TEMPF` |
| `pulse` | numeric or NaN | From `PULSE` |
| `resp_rate` | numeric or NaN | From `RESPR` |
| `bp_sys` | numeric or NaN | From `BPSYS` |
| `bp_dias` | numeric or NaN | From `BPDIAS` |
| `spo2` | numeric or NaN | From `POPCT` |
| `vitals_missing` | binary flag | 1 if any vital above is NaN |
| `urgency_label` | ordinal (dropped rows where NaN) | From `IMMEDR`, recoded |
| `resource_label` | categorical {0,1,2+} | Constructed per Stage 3b |
| `patwt` | numeric | Carried through for the weighted sensitivity check only — never a predictor |

Two parallel modeling tasks share this same feature block (`age` through `vitals_missing`): one predicts `urgency_label`, the other predicts `resource_label`.

---

## Stage 5 — Train/test split

- 2022 only for now (16,025 visits): **stratified random split** on `urgency_label` (and separately, when training the resource task, on `resource_label`), since the high-urgency / high-resource classes are the minority and both train and test need enough of them.
- If 2022 alone proves too small on urgent cases once class counts are checked: fall back to combining with 2018–2019 (skip/flag 2020–2021 for COVID distortion, per the project's existing plan), train on earlier year(s), test on 2022.
- Log the exact class counts at this stage — needed for the TRIPOD+AI sample-size reporting either way.

---

## Stage 6 — Preprocessing (fit on train only, applied to test)

- **Vitals (5.9–10.9% missing):** check whether missingness co-occurs across vitals within the same visit before picking an imputation method; median imputation plus a shared `vitals_missing` flag is the simple default.
- **Pain scale (44% missing):** keep both the raw value (imputed or not, to be decided) and the `pain_scale_missing` flag — run an ablation with/without `pain_scale` given how sparse it is.
- **RFV groups:** one-hot or target-encode the collapsed groups from Stage 4 — encoder fit on train only.
- **Scaling:** standardize numeric features for logistic regression; not required for random forest.
- **Class imbalance:** class-weighting (or resampling) for both logistic regression and random forest, given urgent/high-resource visits are the minority class.

---

## Stage 7 — Algorithms

Per the topic paper: two algorithms × two outcome tasks, all on the Model 1 (informed) feature set for this phase.

1. **Logistic regression** — baseline, regularized (L2), class-weighted.
2. **Random forest** — tuned via cross-validation on the training split (tree depth, n_estimators, min samples per leaf) with the same class-weighting.

Both are fit twice: once for `urgency_label`, once for `resource_label`. (Model 2's restricted-feature versions of these same four fits come later, once Model 2 is redesigned.)

---

## Stage 8 — Evaluation plan

- **Sensitivity/specificity across thresholds** for the urgency task, since missing a true emergency is the costly error.
- **Threshold selection:** pick an operating threshold on a validation split to hit a target sensitivity for high-urgency cases, then evaluate that fixed threshold once on the untouched test set.
- **Calibration:** how well predicted probabilities match observed outcome rates.
- **Decision curve analysis** (Raita et al. 2019 style): net benefit vs. "flag everyone" / "flag no one" baselines, across a range of thresholds.
- **Cost-matrix scenarios:** adjustable penalty for missed-urgent vs. false-alarm, reported as hypothetical outcomes per 1,000 visits — not a dollar estimate.
- **Uncertainty:** bootstrap or similar confidence intervals on the test-set metrics.
- **Weighted sensitivity check:** compare unweighted test performance to a `PATWT`-weighted version, as agreed.

---

## Open items — remaining after Stages 1–4 were run

1. ~~Raw file acquisition~~ — done: `nhamcs_2022_ed_raw_full.csv` and `nhamcs_2022_model1_processed.csv` exist.
2. **RFV grouping scheme** — how granular `rfv1_module`/`rfv*_raw` should be collapsed into for modeling (research-design call; CDC's full code→text table is available in `ed22for.txt` if fine-grained grouping is wanted).
3. **Resource-outcome sign-off** — confirm `NUMMED`'s role and ESI bundling conventions with Will before `resource_label` is final (see Stage 3b).
4. ~~`PATWT` exact column~~ — confirmed and included in the processed file; sums to 155,397,747, exactly matching CDC's documented national estimate.
5. ~~IMMEDR 0/7/unknown/blank handling~~ — confirmed via CDC's format statement and applied (36.3% excluded as unusable for the label; see Stage 3a).

Next step once this doc is reviewed: turn Stages 1–8 into a visual pipeline diagram, and run Stages 5–8 (split/train/evaluate) against `nhamcs_2022_model1_processed.csv`.
