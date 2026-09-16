# Model 1 Pipeline — Quick Version

The full technical doc is `model1-pipeline-design.md`. This is the short version — what happens, in order.

1. **Parse the raw file.** Read CDC's file using their official column map (2018–2022, 86,864 visits).
2. **Fix missing codes.** CDC uses numbers like `-9` for "missing." Convert those to real blanks.
3. **Wall off anything that would leak the answer.** Diagnoses, admission status, medications — these only exist *after* the visit is over, so they can never be model inputs. Also drop `PAINSCALE` (too much missing data to trust).
4. **Build the two outcomes.** `urgency_label` (how urgent, 1–5) and `resource_label` (how much care used, 0/1/2+).
5. **Assemble the final table.** Age, sex, reason for visit, vitals, both outcomes — one row per visit.
6. **Split into train/test.** Keep the split balanced so rare high-urgency cases show up in both.
7. **Train two models** (logistic regression + random forest) on the training set only.
8. **Evaluate on the untouched test set** — how often it catches true emergencies, how often it's wrong, and what those mistakes would cost.

That's it — raw file in, clean table out, two models trained and scored on data they never saw.
