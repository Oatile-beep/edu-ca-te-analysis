# Edu-ca-te Learner Data & Resource Allocation — Beacon Innovation Hub Challenge

**Client:** Mr P.G. Marapira | **Pathways:** Data Analyst / Data Scientist
**Core question:** Which subject should receive additional tutoring resources next term, and why?

## Status
- [x] Data-quality audit
- [x] Cleaning script/notebook + cleaning log
- [x] Validation (row counts, subject consistency, join integrity)
- [ ] Analysis (engagement, mark change, enquiry demand/conversion, capacity)
- [ ] KPIs
- [ ] Business recommendation + visualization
- [ ] Limitations section

## Repository Structure

```
edu-ca-te-analysis/
├── README.md                          ← you are here
├── data/
│   ├── raw/                           ← original workbook, untouched
│   │   └── educate_linked_challenge_dirty_assessment.xlsx
│   └── cleaned/                       ← output of the cleaning notebook
│       ├── learners_clean.csv
│       ├── engagement_clean.csv
│       ├── assessments_clean.csv
│       ├── support_capacity_clean.csv
│       └── enquiries_clean.csv
├── notebooks/
│   └── 01_data_cleaning.ipynb         ← reproducible cleaning (run in Colab or locally)
├── reports/
│   ├── data_quality_audit.md          ← pre-cleaning audit, issues quantified per sheet
│   └── cleaning_log.csv               ← every cleaning action: problem, action, justification, rows affected
└── outputs/
    ├── excluded_flagged_records.csv   ← rows removed from analysis, kept with a reason (never silently deleted)
    └── charts/                        ← visualizations (added during analysis phase)
```

## How to Reproduce

1. Open `notebooks/01_data_cleaning.ipynb` in Google Colab or Jupyter.
2. Run all cells. When prompted (Colab only), upload `data/raw/educate_linked_challenge_dirty_assessment.xlsx`.
3. The notebook writes cleaned CSVs to a local `cleaned/` folder and prints validation checks
   (subject consistency, join integrity) at the end — these should show 4 canonical subjects
   in every subject-bearing sheet and 0 unmatched rows on every join.
4. Cleaned CSVs already in `data/cleaned/` reflect the last verified run.

## Key Cleaning Decisions (see `reports/cleaning_log.csv` for full detail)

- All subject-bearing sheets standardized to 4 canonical subjects: **Mathematics, Physical
  Sciences, Life Sciences, Accounting**.
- Rows with unmatched `learner_id` (no corresponding learner record) are **excluded from
  analysis but retained** in `outputs/excluded_flagged_records.csv`, not deleted.
- Implausible marks (e.g. repeated values of 1050, 180 in `latest_mark`) were set to missing
  rather than "corrected," since the true value can't be reconstructed with confidence.
- Full detail, counts, and the reasoning behind every rule are in the cleaning log.

## Data Dictionary (brief summary)

| Table | Grain | Key fields |
|---|---|---|
| `learners` | one row per learner | `learner_id`, `gender`, `grade`, `registration_date` |
| `engagement` | one row per learner–subject | `learner_id`, `subject`, `tutoring_attendance_pct`, `videos_watched`, `watch_time_minutes` |
| `assessments` | one row per learner–subject | `learner_id`, `subject`, `baseline_mark`, `latest_mark`, `mark_change` (derived) |
| `support_capacity` | one row per subject | `subject`, `active_tutors`, `weekly_tutor_hours`, `waiting_list`, `estimated_cost_per_extra_hour` |
| `enquiries` | one row per prospective-learner enquiry | `enquiry_id`, `enquiry_date`, `grade`, `subject`, `converted_to_registration` |
