# `nhamcs_2022_model1_processed.csv` — Data Dictionary

Built from the official 2022 NHAMCS ED public use file (`ed2022_sas.sas7bdat`, 16,025 visits), via `build_datasets.py`. This is the Model 1 (informed) feature set: restricted fields + vitals + both outcome candidates, ready to split and model.

| Column | Type | Source | Notes |
|---|---|---|---|
| `visit_id` | int | assigned | 1..16025, not a CDC field |
| `age` | float | `AGE` | years; -9 recoded to NaN (none observed) |
| `sex` | string | `SEX` | "female"/"male" (CDC: 1=Female, 2=Male) |
| `rfv1_raw`…`rfv5_raw` | float | `RFV1`–`RFV5` | 5-digit Reason-for-Visit Classification code; -9/-8/-7 → NaN. **Not yet grouped** — see open item below |
| `rfv1_module` | string | derived from `RFV1` | first digit of the RVC code (1=symptom module, 2=disease, 3=diagnostic/screening, 4=treatment, 5=injury/adverse effect, 6=test results, 7=administrative, 8/9=uncodable/other) |
| `pain_scale` | float | `PAINSCALE` | 0–10; -9/-8 → NaN. **44.0% missing** (validated against the actual file) |
| `pain_scale_missing` | int (0/1) | derived | 1 if `pain_scale` is NaN |
| `temp_f` | float | `TEMPF` | °F; -9 → NaN (5.8% missing) |
| `pulse` | float | `PULSE` | bpm; -9 → NaN (6.0% missing) |
| `resp_rate` | float | `RESPR` | breaths/min; -9 → NaN (5.3% missing) |
| `bp_sys` | float | `BPSYS` | -9 → NaN (10.7% missing) |
| `bp_dias` | float | `BPDIAS` | -9 → NaN (10.9% missing) |
| `spo2` | float | `POPCT` | pulse ox %; -9 → NaN (6.0% missing) |
| `vitals_missing` | int (0/1) | derived | 1 if **any** of the six vitals above is NaN (17.9% of visits) |
| `urgency_label` | float (1–5) | `IMMEDR` | 1=Immediate, 2=Emergent, 3=Urgent, 4=Semi-urgent, 5=Nonurgent. Codes 0 ("no triage done, though this ED normally triages"), 7 ("this ED does not do nursing triage"), -8 (Unknown), -9 (Blank) are all set to NaN — none of them is a usable acuity rating |
| `urgency_missing` | int (0/1) | derived | 1 if `urgency_label` is NaN. **36.3%** of visits — higher than the 27.8% figure in the earlier summary doc, which only counted blank+unknown; this corrected number (validated straight from the real value-label table) also excludes the two "not applicable" triage codes |
| `resource_count_raw` | float | `TOTDIAG` + `TOTPROC` | sum of CDC's own pre-computed "total diagnostic services" and "total procedures" counts — see caveat below |
| `resource_label` | string {"0","1","2+"} | derived | bucketed version of `resource_count_raw`; NaN where `TOTDIAG` or `TOTPROC` was blank (-9), 3.1% of visits |
| `nummed` | float | `NUMMED` | number of medications; kept separately, **not folded into `resource_label`** — see caveat |
| `patwt` | float | `PATWT` | patient visit weight; sums to 155,397,747 across the file, matching the documented national estimate exactly. Reporting/sensitivity-check use only — never a predictor |

## Caveat on `resource_label` (provisional — flagged in the pipeline design doc as an open item)

CDC already provides `TOTDIAG` ("total number of diagnostic services ordered or provided") and `TOTPROC` ("total number of procedures provided") as pre-aggregated counts, which is simpler than the manual checklist-summing originally planned. This is a defensible first pass, but two judgment calls are still open for the team:

1. Whether medications (`NUMMED`) should count toward "resources used" at all — ESI resource-counting conventions are specific about what counts as a "resource," and this hasn't been confirmed against the ESI definition Sterling et al. (2020) used.
2. Whether `TOTDIAG`/`TOTPROC` already de-duplicate the way ESI resource counting expects (e.g., a panel of several blood tests sometimes counts as one resource under ESI, not one per test).

Treat `resource_label` as a working draft, not the final outcome definition, until Will signs off.

## Known open item: RFV grouping

`rfv1_raw`…`rfv5_raw` are left as raw 5-digit codes. `rfv1_module` gives a coarse 8-way split. Full code-to-text labels exist in CDC's `ed22for.txt` (the `VALUE RFVF` block) if finer, clinically meaningful grouping is wanted later — this is the research-design decision flagged in the pipeline doc, not resolved here.
