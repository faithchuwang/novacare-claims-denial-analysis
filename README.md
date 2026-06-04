# NovaCare Claims Denial Analysis — The Denial Pattern

A healthcare data analytics project investigating a 39.58% claim denial rate across 6,150
insurance claims for NovaCare Health Insurance. Built an end-to-end pipeline using SQL,
Python, and Power BI to uncover denial patterns, quantify revenue leakage, and predict
claim denials before submission using logistic regression. Completed as part of the
Dataverse Africa Cohort 4.0 Healthcare Data Analytics Track.

---

## Table of Contents
- [Project Overview](#project-overview)
- [Data Sources](#data-sources)
- [Tools](#tools)
- [Project Structure](#project-structure)
- [Data Cleaning](#data-cleaning)
- [Data Analysis](#data-analysis)
- [Denial Prediction Model](#denial-prediction-model)
- [Dashboard Preview](#dashboard-preview)
- [Results/Findings](#resultsfindings)
- [Recommendations](#recommendations)
- [Limitations](#limitations)

---

## Project Overview

NovaCare Health Insurance flagged a critical operational problem: approximately 40% of all
hospital claims processed over a 12-month period were being denied. The CFO commissioned
an analytics team to investigate root causes, quantify financial impact, and deliver
actionable recommendations ahead of a board presentation.

Acting as a Healthcare Data Analyst, the objectives were to:

- Clean and standardize 6,150 raw claims records using PostgreSQL
- Analyse denial patterns by department, provider, diagnosis, and payer type
- Quantify revenue leakage and identify the biggest cost drivers
- Build a denial risk flag using evidence-based risk factors
- Develop a logistic regression model to predict claim denials before submission
- Deliver a 4-page interactive Power BI dashboard structured around the CFO's questions
- Write a 400-500 word executive memo addressed to the CFO

The analysis answers the CFO's three core questions:

- **What is the scale of the problem?** — Denial rate, total volume, and claim status breakdown
- **Where is it concentrated?** — By department, provider, diagnosis, and payer type
- **What is it costing us?** — Revenue leakage, breakdown by reason, and recoverable revenue

A machine learning extension then answers a fourth question:

- **Can we predict a denial before it happens?** — Logistic regression denial prediction model

---

## Data Sources

- **Dataset:** NovaCare Health Insurance internal claims database (simulated)
- **Records:** 6,150 claims — full 12-month period
- **Raw File:** novacare_claims_raw.csv — available in the /data folder
- **Fields covered:** Patient and provider identifiers, hospital department, admission and
  discharge dates, ICD-10 diagnosis codes, CPT procedure codes, billed/allowed/paid amounts,
  claim status, denial reason, payer type, and claim processing days

---

## Tools

- **PostgreSQL (via pgAdmin)** — Data cleaning, standardization, and flag engineering
- **Python (Jupyter Notebook)** — Exploratory analysis, pattern detection, risk flagging, and ML modelling
- **Libraries:** Pandas, NumPy, Scikit-learn, Matplotlib, SQLAlchemy, SciPy
- **Power BI Desktop** — 4-page interactive dashboard for executive reporting
- **Microsoft PowerPoint** — Team presentation deck
- **Microsoft Word** — Comprehensive project documentation and CFO memo

---

## Project Structure

```
novacare-claims-denial-analysis/
│
├── data/
│   ├── novacare_claims_raw.csv          # Original raw dataset
│   ├── claims_clean.csv                 # SQL stage output (25 columns)
│   └── claims_output.csv               # Python stage output (30 columns)
│
├── sql/
│   ├── NOVACARE_CLAIMS_SQL_FINAL.sql    # Primary team submission
│   ├── claims_clean_experiment.sql      # Extended version with median imputation
│   └── Clean_claims.sql                # Supplementary analytical queries
│
├── python/
│   ├── stage_2_analysis/
│   │   ├── novacare_claims_analysis_local.ipynb
│   │   ├── novacare_claims_analysis_colab.ipynb
│   │   └── novacare_claims_analysis_postgresql.ipynb
│   │
│   └── stage_3_denial_prediction/
│       ├── novacare_denial_prediction_final.ipynb
│       ├── novacare_denial_prediction_extended.ipynb
│       └── novacare_denial_prediction_postgresql.ipynb
│
├── dashboard/
│   └── screenshots/
│       ├── page1_problem_scale.png
│       ├── page2_where_concentrated.png
│       ├── page3_what_costing_us.png
│       └── page4_denial_prediction.png
│
└── docs/
    └── NovaCare_Project_Documentation.docx
```

---

## Data Cleaning

The raw dataset had not been cleaned since extraction. All cleaning was performed in
PostgreSQL and every decision was documented in a Data Audit Log.

| Issue | Records Affected | Decision |
|---|---|---|
| Extra whitespace in text fields | All 6,150 rows | Applied TRIM() to all VARCHAR columns on table creation |
| Inconsistent provider names | Multiple per provider | Standardized using provider_id as the reliable key |
| Department abbreviations (e.g. CARDIO, ER, PEDS) | All departments | Mapped to canonical names: Cardiology, Emergency, Pediatrics, etc. |
| ICD-10 codes with dots (e.g. A09.0) | 15 diagnosis codes | Removed dots using REPLACE() |
| Mixed casing in claim_status | All status values | Standardized to Initcap: Paid, Denied, Pending, Appealed |
| Duplicate claims (same encounter_id) | Multiple records | Added is_duplicate_claim flag (1 = duplicate) |
| NULL billed/allowed amount with valid paid amount | 74 records | Added paid_error flag; filled allowed_amount from paid_amount |
| NULL billed/allowed amount (no paid amount) | 114 records | Imputed using department-level median |
| Claim submitted before discharge date | 121 records | Added claim_before_discharge flag |
| Claim submitted before admission date | 56 records | Added claim_before_admission flag |
| Denied claims with NULL denial_reason | 183 records | Filled with 'Unknown' |
| NULL payer_type | 249 records | Filled with 'Unknown' |
| Negative claim_processing_days | 123 records | Converted to absolute values using ABS() |

Six engineered columns were added to the clean table:

| Column | Description |
|---|---|
| is_duplicate_claim | 1 if the claim is a known duplicate |
| paid_error | 1 if paid amount exists but billed/allowed are NULL |
| claim_before_discharge | 1 if submission date precedes discharge date |
| claim_before_admission | 1 if submission date precedes admission date |
| days_to_submit | Days between discharge and claim submission |
| is_pending | 1 if claim status is Pending |

---

## Data Analysis

**Stage 2 — Python: Pattern Detection & Risk Flagging**

| Analysis | Key Finding |
|---|---|
| Overall denial rate | 39.58% — 2,434 of 6,150 claims denied |
| Highest denial rate by department | Pediatrics and Emergency — both at 41% |
| Highest denial rate by reason | Duplicate Claim (92%), Patient Not Eligible (90%) |
| Top denial diagnosis | Gastroenteritis |
| Top denial provider | Dr. Yetunde Okafor |
| Highest denial rate by payer | Government — 40.90% |
| Total revenue leakage | $7.22M out of $18M total billed (40%) |
| Leakage from duplicates alone | $1.41M |
| Potential recoverable revenue | $2.46M (Pending + Appealed claims) |
| Top provider-diagnosis combination | Dr. Yetunde Okafor + Viral Infection — 70.59% denial rate |

**Denial Risk Flag**

A binary high_denial_risk flag was built using three evidence-based risk factors:

| Risk Factor | Definition | Rationale |
|---|---|---|
| risk_payer | 1 if payer type is Government, Insurance, or Self-Pay | These payer types showed consistently higher denial rates |
| risk_processing | 1 if claim_processing_days > median | Longer processing correlates with higher denial tendency |
| risk_amount | 1 if billed_amount > median | Higher-value claims face more scrutiny and denial |

Claims scoring 2 or more out of 3 risk factors were flagged as high_denial_risk = 1.

---

## Denial Prediction Model

A logistic regression model was built to predict claim denials before submission,
giving the billing team a chance to correct high-risk claims proactively.

**Model Setup**

| Parameter | Decision | Rationale |
|---|---|---|
| Algorithm | Logistic Regression | Interpretable, outputs probability scores, suitable for binary classification |
| Encoding | One-Hot Encoding | Nominal categories with no ordinal relationship |
| Scaling | StandardScaler on numeric features | Prevents high-magnitude features from dominating |
| Train/Test Split | 80/20 stratified, random_state=42 | Preserves class ratio; reproducible results |
| Class weight | Balanced | Compensates for 40/60 class imbalance |
| Threshold | 0.45 (adjusted from default 0.50) | Default threshold caught zero denials; 0.45 improved recall to 66% |

**Model Results**

| Metric | Value |
|---|---|
| Model Accuracy | 64% |
| Recall Rate | 28% (on full test set) |
| Recall at 0.45 threshold | 66.1% |
| High-Risk Claims Flagged | 1,240 claims |
| Total Revenue at Risk | $3.68M |
| Revenue Protected at 0.45 threshold | ~$957,296 |

**Confusion Matrix**

| | Predicted: Denied | Predicted: Not Denied |
|---|---|---|
| **Actual: Denied** | 137 (True Positive) | 350 (False Negative) |
| **Actual: Not Denied** | 93 (False Positive) | 650 (True Negative) |

**Top Feature Importances**

| Rank | Feature | Direction |
|---|---|---|
| 1 | is_duplicate_claim | Increases denial risk (+) |
| 2 | department_Internal Medicine | Decreases denial risk (-) |
| 3 | payer_type_Insurance | Decreases denial risk (-) |
| 4 | is_peak_month | Increases denial risk (+) |
| 5 | payer_type_Unknown | Increases denial risk (+) |

---

## Dashboard Preview

### Page 1 — Problem Scale
<img width="1236" height="679" alt="Problem Scale" src="https://github.com/user-attachments/assets/e9a09a72-8e7c-4edf-99e4-52eddb465f07" />


### Page 2 — Where Is It Concentrated?
<img width="1211" height="682" alt="Where Is It Concentrated?" src="https://github.com/user-attachments/assets/560cf116-d1db-46e3-b61d-2b97501fe991" />


### Page 3 — What Is It Costing Us?
<img width="1210" height="682" alt="What Is It Costing Us?" src="https://github.com/user-attachments/assets/42ec281d-9fc5-4a1b-a400-facace0ca552" />


### Page 4 — Denial Prediction Model
<img width="1217" height="678" alt="Denial Prediction Model" src="https://github.com/user-attachments/assets/5460ef0a-c81c-4871-986e-1d2c3ab3dbb3" />


---

## Results/Findings

1. **The denial problem is systemic, not clinical.** With a 39.58% denial rate, NovaCare
   is losing $7.22M out of $18M billed. The top two denial reasons — Duplicate Claims (92%)
   and Patient Not Eligible (90%) — are operational failures, not clinical complexity.
   These are preventable with better process controls.

2. **Denial rates are nearly identical across all departments.** The range is only 38–41%,
   suggesting the root causes are hospital-wide process issues rather than department-specific
   problems. No single department is driving the problem alone.

3. **Provider-diagnosis combinations show dangerous concentration.** Dr. Yetunde Okafor's
   Viral Infection cases carry a 70.59% denial rate — nearly double the overall average.
   Five provider-diagnosis combinations each exceed 57%, pointing to specific coding or
   documentation gaps at the provider level.

4. **$2.46M in revenue is still recoverable.** Pending and Appealed claims represent money
   that has not yet been written off. Prioritising this pipeline represents the fastest path
   to cash flow recovery without requiring any process change.

5. **Duplicate claims are the single biggest controllable loss.** At $1.41M in leakage and
   a 92% denial rate, they are also the strongest predictor in the ML model. A pre-submission
   duplicate check would directly address the largest single source of denial.

6. **The prediction model flags $3.68M in revenue at risk before submission.** At the
   adjusted 0.45 threshold, the model catches 66% of actual denials before they are
   submitted, giving the billing team a meaningful intervention window.

---

## Recommendations

1. **Implement a pre-submission duplicate claim checker.** Duplicate claims cause $1.41M
   in leakage and carry a 92% denial rate. An automated flag before submission would
   eliminate this loss category almost entirely.

2. **Target provider-diagnosis training for high-risk combinations.** Dr. Yetunde Okafor's
   Viral Infection cases (70.59% denial rate) and four other combinations exceeding 57%
   should receive priority coding education and documentation review.

3. **Prioritise the $2.46M in pending and appealed claims.** A dedicated appeals team
   working through these in order of claim value would accelerate cash flow recovery
   without requiring new processes.

4. **Deploy the denial prediction model at a 0.45 threshold.** The model flags 1,240
   high-risk claims with $3.68M at stake. At the adjusted threshold it catches 66% of
   actual denials before submission, giving the billing team time to correct them first.

5. **Investigate Emergency and Pediatrics for process improvement.** Both departments
   show the highest denial rates (41%) and the highest concentration of high-risk claims
   in the prediction model (186 and 273 respectively).

---

## Limitations

- Analysis covers a 12-month claims snapshot; longer longitudinal data would improve
  trend detection and model training quality
- The logistic regression model has a 28% overall recall rate, meaning the majority of
  actual denials are still missed — further feature engineering and model iteration is needed
- The denial risk flag was built on three risk factors; a more robust flag would incorporate
  additional variables such as provider history and specific ICD-10 code combinations
- The 350 false negatives (missed denials) represent a significant gap in the model's
  current predictive capability and should be the focus of the next iteration
- Power BI dashboard was built on exported CSVs rather than a live database connection;
  any updates to the underlying data would require a manual refresh

---

*Completed as part of Dataverse Africa — Cohort 4.0 | Healthcare Data Analytics Track*  
*Author: Faith Chuwang-Kwa*
