Input data (nhamcs_2022_model1_processed.csv) — one row per ED visit, 16,025 rows total. Columns fall into three groups:

What the patient themselves would know: age, sex, stated complaint (RFV1–5), self-reported pain score
What's only known once they're in the ED (the extra stuff Model 1 gets): temperature, pulse, respiration, blood pressure, oxygen level
The answers we're trying to predict: nurse-assigned urgency (1=most urgent to 5=least urgent), and how many medical resources got used (0/1/2+)

There's also a patwt column (survey weight, for national estimates only) — that one never goes into the model.

The pipeline:

Pull the raw file from CDC plus its official layout file, and turn the fixed-width raw text into a normal table
Convert the weird missing-value codes (-9, -8, etc.) into real blanks — otherwise the model would treat them as real numbers
Define the two outcomes (urgency, resource use), and exclude anything that would "leak" the answer (diagnosis codes, admission outcome, etc.) so they don't sneak into the inputs
Split into train/test, stratified by the outcome (since urgent cases are rare, both sides need enough of them)
Fill in missing values and encode categories — only "learned" from the training data, never peeking at test
Feed it into two algorithms — logistic regression (baseline) and random forest — each predicting both outcomes
Evaluate — mainly whether it's missing truly urgent patients (sensitivity), plus decision curves and cost scenarios to judge overall usefulness
