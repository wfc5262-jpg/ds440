# How the Processed Dataset Was Made

A plain-language walkthrough of what happened between the raw CDC file and the CSV you actually load into a model. Full technical detail lives in `model1-pipeline-design.md`; this is the short version.

## Where it started

The raw source is CDC's official NHAMCS ED survey file — one row per emergency department visit, with roughly 900+ columns covering everything from vitals to billing codes. It comes as a CDC-published SAS file, not something you can just read like a normal spreadsheet — every column uses its own numeric codes, and several different "this value is missing" conventions.

Two processed files came out of it:
- **`nhamcs_2022_model1_processed.csv`** — 2022 only, 16,025 visits
- **`nhamcs_2018_2022_model1_processed_combined.csv`** — five years stacked together, 86,864 visits, with a `survey_year` column and a `covid_era` flag on 2020–2021

Both went through the exact same steps below.

## Step 1 — Fix the missing-value codes

CDC doesn't use a blank cell for "missing" — it uses numbers like `-9`, `-8`, or `-7` depending on the field. Left alone, a model would treat `-9` as a real value instead of "unknown." First step was converting every one of these codes to a true blank (NaN) across every field we kept.

## Step 2 — Decide what a model is actually allowed to see

This is the most important step. The whole point of Model 1 is predicting how urgent a visit is *before* anyone has run tests or made a diagnosis — so anything that would only exist *after* a doctor already made that call had to be thrown out as a predictor: diagnosis codes, admission status, medication lists, drug IDs. Those fields are still in the raw file, but they're walled off — they're only ever allowed to help define the outcome (see Step 4), never to be an input feature. Mixing the two would let the model "cheat" by seeing the answer.

We also dropped `PAINSCALE` (self-reported pain, 0–10) entirely. Not because it leaks anything, but because it was missing on 36–44% of visits depending on the year — too unreliable to keep as a feature.

What's left as actual inputs: age, sex, the patient's stated reasons for the visit, and the five vitals (temperature, pulse, respiration rate, blood pressure, oxygen level).

## Step 3 — Flag what's still missing

Even after Step 1, some of the kept fields still have gaps (vitals are missing on 6–18% of visits). Rather than just filling those in silently, we added a `vitals_missing` flag column — 1 if any vital is missing for that visit, 0 otherwise — so a future model can tell the difference between "this patient had a normal reading" and "this patient wasn't measured at all."

## Step 4 — Build the two outcomes (the things a model predicts)

Two separate targets came out of the raw file:

- **`urgency_label`** — how urgent the visit was, on a 1 (Immediate) to 5 (Nonurgent) scale, taken from the ED nurse's own triage rating. About a third of visits don't have a usable rating (no triage was done, or it wasn't recorded), so those rows are marked missing rather than guessed at.
- **`resource_label`** — how much care the visit used, bucketed into "0," "1," or "2+" services (tests + procedures), following the same three-tier approach used in prior published research on this exact question.

## Step 5 — Assemble the final table

Everything from Steps 2–4 gets joined into one row-per-visit table: the input features, the missing-data flags, both outcome columns, and a national survey weight (`patwt`, used only for reporting national estimates later — never as a model input).

## What you get at the end

A clean table where every column is either something a model can use as an input, a flag saying "this input was missing," or one of the two things being predicted — with nothing in it that would let the model see the answer ahead of time.
