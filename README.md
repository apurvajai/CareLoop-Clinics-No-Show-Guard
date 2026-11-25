# CareLoop Clinics: No-Show Guard

GitHub repo: <PASTE YOUR PUBLIC REPO LINK HERE>

This is my capstone for the CareLoop Clinics case. The clinics lose money every time a patient doesn't turn up, so the question I tried to answer was: which appointments are most likely to be missed, why, and what should CareLoop do first?

## Quick summary

1. **Python:** The raw file had 1,020 rows. After removing duplicates, unreadable dates and blank statuses I was left with 990 bookings, and 218 of them (22.0%) were no-shows.
2. **SQL:** Lead time matters a lot. Bookings made 1-7 days ahead are missed 16.2% of the time, but bookings made 31+ days ahead are missed 60.5% of the time.
3. **Model:** I picked logistic regression (test ROC-AUC 0.789). At a threshold of 0.35 it flags 49 of the 198 test bookings and catches 57.1% of the real no-shows.
4. **GenAI:** I could not run the Gemini part (my Colab and API key were not working), so there is no model output. Reading the 10 notes by hand instead, 2 of the 9 non-blank notes are about forgetting or no reminder, and both of those appointments had `reminder_sent` = No.

## What I would recommend

CareLoop should start with the bookings made far in advance. Those made 15-30 days ahead are missed 30.0% of the time and 31+ days ahead 60.5%, compared with 16.2% for the shortest band, so a second reminder closer to the date for anything booked more than two weeks ahead is the easiest first step. The front desk can also call the bookings the model flags each evening. On the test set that was 49 of 198 bookings and it caught 57.1% of the real no-shows, while the overall no-show rate is only 22.0%. In my model, appointments with a reminder had roughly 69% lower odds of a no-show, so every booking should get a reminder, but this is only an association and I can't say reminders cause it. I would pilot all this at Hadapsar Care Centre in Pune (27.3% on 143 appointments) and Malviya Nagar Clinic in Jaipur (26.8% on 142 appointments). The 10 call notes I read by hand do not point to one single reason (forgetting, distance and work or shifts have 2 notes each), but both "forgot" notes were for bookings with no reminder, which fits the model result.

## What I did

**Part 1: cleaning.** I read the file with the csv module first (988 rows accepted, 32 rejected because of blank age or distance). The 5 impossible ages like 250 still went through, since a type check does not check if a value makes sense. Then I cleaned with pandas: dropped 20 duplicates, fixed the 16 spellings of booking channel down to 3, set bad ages to missing, filled gaps with the median (age 48, distance 8.25) and capped outliers with the IQR rule (26 distances and 3 fees, the fees were obvious extra-zero typing mistakes). I also made `lead_days` and `no_show`. For the clinics I tried to add a temperature from the Open-Meteo API, but the calls failed when I ran it, so `temp_c` is blank (NULL in the SQL) rather than a made-up number. The final file has 990 rows, 16 columns and no missing values.

**Part 2: SQL.** I loaded everything into MySQL and the counts matched my Python numbers (8 clinics, 990 appointments, 218 no-shows). Inserting a second `A0001` gave a duplicate key error, which is saved as a comment in `02_queries.sql`. Gomti Nagar has the highest rate (33.3%) but only 90 appointments, so I trust Hadapsar's 27.3% more. By month the rate stayed between 20.4% and 24.8%. I also checked the clinic counts with pandas groupby and they matched the SQL exactly.

**Part 3: model.** If I just predicted "Attended" for everyone I would get 75.25% accuracy, so I looked at recall, precision and ROC-AUC too. The unlimited decision tree scored 1.00 on training but only 0.672 on test, which is worse than that baseline, so it was overfitting. The depth-3 tree and the random forest did better but not as well as logistic regression. I chose the final model and the threshold using only the training data (cross-validation), and used the test set once at the end.

| Model | CV ROC-AUC (training) |
|---|---|
| Logistic regression | 0.774 |
| Tuned random forest | 0.767 |
| Random forest | 0.747 |
| Decision tree (depth 3) | 0.678 |

For the threshold, 0.5 gave recall 0.249 and F1 0.347, while 0.35 gave recall 0.467 and F1 0.489, so I used 0.35. Recall matters more here because a missed no-show is an empty doctor slot, while an extra phone call only costs a minute. In a real clinic the threshold would depend on how many calls the front desk can make in one evening.

One thing to be careful about: the reminder coefficient is 0.56 as an odds ratio per standard deviation of the scaled feature, but the real No-to-Yes effect is 0.31 after dividing by the scale. That is why I say 69% lower odds and not 44%.

**Part 4: GenAI.** I used the Gemini API to turn the first 10 call-centre notes into a root cause, confidence, summary and recommended action, with JSON output and a Pydantic check on every reply. N004 is blank so it was skipped without using an API call. I also tested the checks with two bad replies (confidence 1.7 and a missing field) and compared a plain prompt with my CRAFT prompt on N007 (Hinglish) and N009 (unclear).
- Results: the notebook is written but I could not run it against the API, so I have no model results, no V1 vs V2 table and no call count to report. The `notes_analysed.csv` in this repo is only a placeholder: the blank note N004 is correctly marked `skipped_blank` (no API call), and the other 9 rows are marked `not_run_api_required` with empty fields.
- 5 of the 10 notes belong to appointments where `reminder_sent` was No. From my own reading of the notes (not model output), the 2 "forgot or no reminder" notes (N001, N007) both had No, and so did the distance note N002, the work note N010 and the blank note N004.

## Limitations

1. I calculated medians and outlier limits on all 990 rows before splitting, so the test rows had a tiny influence on the cleaning. A real system should use only training rows.
2. The split is random and I only did one split. The same patient can be in both train and test, and in real life the model would be trained on the past and used on the future, so my scores may look a bit better than reality.
3. Only 10 notes were analysed, so they are just an example and can't show the real reasons across all clinics. The data is also observational, so things like "reminders go with fewer no-shows" are associations, not proof.

## Files

- `data/`: the three original files, not edited
- `01_ingest_clean.ipynb`: Part 1, with `rejected_rows.json`, `appointments_clean.csv`, `clinics_enriched.csv` and `load_data.sql` as outputs
- `02_queries.sql` and `02_pandas_check.ipynb`: Part 2
- `03_model.ipynb` and `test_risk_scores.csv`: Part 3
- `04_genai.ipynb` and `notes_analysed.csv`: Part 4 (my API key is not in the file; `notes_analysed.csv` is a placeholder because the API part was not run)

I used an AI assistant to help explain errors while working, but every number above comes from my own code and outputs.
