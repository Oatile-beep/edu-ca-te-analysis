# Data-Quality Audit — Edu-ca-te Learner Data & Resource Allocation

**Prepared for:** Beacon Innovation Hub — Foundational Competence Challenge
**Stage:** Pre-cleaning audit (raw workbook, no records altered)
**Source file:** `educate_linked_challenge_dirty_assessment.xlsx`

---

## 1. Dataset Overview

| Sheet | Rows | Columns | Likely unit of observation |
|---|---|---|---|
| `learners` | 155 | 4 | One row per learner (learner master record) |
| `engagement` | 199 | 5 | One row per learner–subject (participation in a specific subject) |
| `assessments` | 199 | 4 | One row per learner–subject (baseline vs. latest mark for a subject) |
| `support_capacity` | 8 | 5 | Intended: one row per subject (tutoring capacity). Actual: 8 rows for what should be 4 subjects, due to label duplication |
| `enquiries` | 122 | 5 | One row per prospective-learner enquiry |

**How the sheets relate:** `learner_id` is the intended key linking `learners` → `engagement` → `assessments`. `subject` is the key linking `engagement`, `assessments`, `support_capacity`, and `enquiries`. `enquiries` does not carry a `learner_id` — it represents prospective demand, not existing learners, so it can only be joined to the other sheets at the `subject` (and possibly `grade`) level, not at the individual level.

Both keys (`learner_id` and `subject`) have quality problems described below, which affects every downstream join.

---

## 2. Issues by Dataset

### 2.1 `learners` (155 rows)

| Issue | Count | Detail |
|---|---|---|
| Missing `learner_id` | 2 | Cannot be linked to any other table |
| Missing `gender` | 4 | |
| Missing `registration_date` | 2 | |
| Fully duplicate rows | 2 | Identical across all 4 columns |
| Duplicate `learner_id` after normalizing case/whitespace | 5 | e.g. `lnd1025` vs `LND1025`; one ID (`LND1088`) appears to duplicate an existing learner |
| `learner_id` needing case/whitespace normalization | 17 | Lowercase prefix (`lnd...`) or leading/trailing spaces — will fail exact-match joins if untreated |
| Inconsistent `gender` labels | 11 distinct values | `Female`, `F`, `female `, `Male`, `M`, `m`, `MALE`, `Non-binary`, `Unknown`, `Prefer not say`, blank |
| Inconsistent `grade` labels | 10 distinct values | Numeric (`10`,`11`,`12`), text (`'12'`), and free text (`Grade 11`, `grade12`, `G11`, `Gr 10`, `Grade Ten`) all representing the same three grades |
| Out-of-scope `grade` | 1 | One learner recorded as Grade 9, outside the stated Grade 10–12 scope |
| Non-ISO `registration_date` formats | 11 | Mix of `YYYY-MM-DD` and `DD/MM/YYYY`; ambiguous without a fixed parsing rule |

**Distortion risk:** Duplicate/ambiguous `learner_id`s can inflate learner counts per subject or double-count a learner's engagement and marks. Inconsistent `grade` labels will fragment any grade-level breakdown (e.g. "Grade 11" and "G11" treated as different groups) unless standardized before aggregation.

### 2.2 `engagement` (199 rows)

| Issue | Count | Detail |
|---|---|---|
| Missing `subject` | 2 | |
| Missing `tutoring_attendance_pct` | 2 | |
| Missing `videos_watched` | 1 | |
| Missing `watch_time_minutes` | 1 | |
| Fully duplicate rows | 1 | |
| Duplicate learner+subject pairs (after normalization) | 2 pairs (4 rows) | Same learner's participation recorded twice for the same subject — will double-count engagement if not deduplicated |
| Subject label variants | 12 distinct values for what should be 4 subjects | e.g. `Maths`/`Math`/`Mathematics`/`   Mathematics  `/`maths `; `Sci`/`Physical Sciences`; `LS`/`Life Sciences`/`life sciences ` |
| `tutoring_attendance_pct` outside 0–100 | 7 | Impossible attendance percentages |
| Negative `watch_time_minutes` | 8 | Physically impossible |
| `learner_id` not found in `learners` | 4 rows | IDs `LND1042`, `LND1050`, `LND1118`, and a non-conforming ID `TEMP-004` have no matching learner record — these rows cannot be attributed to a known learner without further information |

**Distortion risk:** Unmerged subject-label variants will split what should be one subject's engagement total into several smaller, apparently-weaker subjects — this directly threatens the core business question, since engagement is one of the signals used to judge need. Orphaned `learner_id`s inflate row counts in a join but represent no real, attributable learner.

### 2.3 `assessments` (199 rows)

| Issue | Count | Detail |
|---|---|---|
| Missing `learner_id` / `subject` / `baseline_mark` / `latest_mark` | 1 each | Same row appears to be missing several fields |
| Fully duplicate rows | 1 | |
| Non-numeric `baseline_mark` | 9 | Literal text values such as `"MISSING"` stored in a numeric field |
| `baseline_mark` outside 0–100 | 2 | Includes 115 and -5 |
| `latest_mark` outside 0–100 | 11 | Includes repeated values of exactly 1050 and 180 — the round, repeated nature of 1050 strongly suggests a data-entry or unit error (e.g. a stray digit or percentage-as-integer error) rather than a genuine mark |
| Subject label variants | 14 distinct values for what should be 4 subjects | Same fragmentation pattern as `engagement`, plus `ACC`, `MATHEMATICS`, `phys sci` |
| `learner_id` not found in `learners` | 3 rows | `LND1042`, `LND1050`, `LND1118` |

**Distortion risk:** This is the highest-risk dataset for the recommendation. The `latest_mark` outliers (1050, 180) would badly distort any average-mark-improvement KPI if included as-is — a single 1050 value could swing a subject's average latest mark by tens of points depending on subject size. The `"MISSING"` string in a numeric column will break any direct arithmetic (e.g. `latest_mark − baseline_mark`) unless coerced and handled explicitly.

### 2.4 `support_capacity` (8 rows, should represent 4 subjects)

| Issue | Count | Detail |
|---|---|---|
| Subject represented by duplicate/inconsistent rows | 8 rows for 4 real subjects | `Maths` vs `maths `; `Physical Sciences` vs `Physical Science`; `Life Sciences` vs `Life sciences`; `Accounting` vs `ACC` — each pair is a near-duplicate of the same subject's capacity record |
| Missing `waiting_list` | 1 | |
| Non-numeric / invalid `weekly_tutor_hours` | 2 | One value stored as text (`"26 hrs"`), one negative (`-2`), which is not physically meaningful |
| Mixed-format `estimated_cost_per_extra_hour` | 2 | One value stored with a currency prefix (`"R180"`), one as a zero-padded string (`"175.00"`) rather than numeric |

**Distortion risk:** This is the smallest sheet but arguably the most decision-critical, since it directly represents capacity constraints. If the duplicate subject rows are not merged, any join to `engagement`/`assessments` on `subject` will silently multiply learner-level records for whichever subject's label happens to match, corrupting per-learner metrics for that subject. The negative tutor-hours value and text-typed cost value will also break numeric aggregation if not cleaned first.

### 2.5 `enquiries` (122 rows)

| Issue | Count | Detail |
|---|---|---|
| Missing `converted_to_registration` | 1 | |
| Fully duplicate rows | 1 | |
| Duplicate `enquiry_id` | 1 | |
| Invalid `enquiry_date` | 2 | Unparseable as a date |
| Out-of-scope `grade` | 1 | Value of 13, outside Grade 10–12 |
| Inconsistent `grade` format | includes `"Grade 12"` alongside plain `12` | |
| Inconsistent `converted_to_registration` encoding | 7 distinct representations | `Y`, `Yes`, `1`, `N`, `No`, `0`, `Maybe`, and blank all used for what should be a binary/ternary flag |
| Subject label variants | 7 distinct values for 4 real subjects | Same fragmentation pattern (`Maths`/`maths `, `Physical Sciences`/`phys sci`) |

**Distortion risk:** The mixed `converted_to_registration` encoding is the most serious issue here — without standardizing to a consistent binary (and deciding how to treat `"Maybe"` and blanks), enquiry conversion rate — one of the requested KPIs — cannot be calculated reliably or consistently across subjects.

---

## 3. Cross-Cutting Issues

- **Subject naming is inconsistent across all four subject-bearing sheets** (`engagement`, `assessments`, `support_capacity`, `enquiries`), using different variants in different sheets. This is the single highest-impact problem in the workbook: subject is the key used to compare engagement, marks, capacity, and demand against each other, and any subject-level KPI computed before standardization will be wrong.
- **`learner_id` inconsistency (case, whitespace, unmatched IDs)** affects every join between `learners` and the learner-level sheets (`engagement`, `assessments`). Left untreated, this both fragments a single learner into multiple apparent identities and introduces unattributable rows.
- **Out-of-range numeric values** (marks beyond 0–100, negative watch time, negative tutor hours, attendance beyond 0–100) appear in every sheet with a numeric field, suggesting systematic (not one-off) entry or extraction problems that need a consistent, documented rule rather than case-by-case fixes.

---

## 4. Summary Counts

| Dataset | Rows | Rows with ≥1 flagged issue (approx.) |
|---|---|---|
| learners | 155 | ~24 |
| engagement | 199 | ~24 |
| assessments | 199 | ~26 |
| support_capacity | 8 | 8 (all rows implicated by subject duplication) |
| enquiries | 122 | ~13 |

*(Counts are approximate where a single row carries more than one issue; exact per-row detail will be tracked in the cleaning log during the next stage.)*

---

## 5. Next Step

These findings will drive the cleaning log: each issue above will be resolved with a documented rule (standardize, coerce, flag-and-exclude, or impute-with-justification), never a silent deletion. No values have been altered in this audit stage.
