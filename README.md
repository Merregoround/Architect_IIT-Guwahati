# 🧬 DiabetesTwin-AI: Personalized In Silico Glucose Monitoring Twin & Prospective Forecasting Engine

> **Important:** This repository is an academic/engineering research prototype. It is **not medical advice, a diagnostic tool, a CGM, or a regulated medical device**. It does not recommend medication or insulin dosages.

---

## 👥 Team & Institutional Affiliation

- **Team Members:**
  - Gaurang Srivastava
- **Institution:** Indian Institute of Technology Guwahati (IIT Guwahati), Assam, India

---

## 🎯 Problem Statement & Healthcare Use Case

### Clinical Problem
Metabolic conditions such as Prediabetes and Type 2 Diabetes require managing non-linear blood glucose variations driven by diet, exercise, stress, and circadian dynamics. Current Continuous Glucose Monitors (CGMs) operate **reactively**—alerting patients only after hyperglycemic (>180 mg/dL) or hypoglycemic (<70 mg/dL) thresholds are crossed. Moreover, standard ML models in diabetes literature often report misleading accuracy due to **person-level data leakage** (mixing identical patient data across train and test sets) and fail to outperform a basic **persistence baseline** ($y_{t+30} = y_t$).

### Healthcare Use Case & Solution
**DiabetesTwin-AI** bridges the gap between reactive sensing and anticipatory intervention via a dual-paradigm architecture:

1. **Lifestyle Counterfactual "What-If" Simulation:** A compartmental in-silico physiological simulator modeling diverse patient phenotypes (`balanced`, `insulin_resistant`, `active`). Clinicians and patients can simulate meal carbohydrate loads, exercise intensity, and sleep disruptions to observe prospective 24-hour CGM trajectories and clinical indices (Time in Range, Time Above Range, Time Below Range).
2. **Leakage-Safe Real-Data Forecasting (+30 Minutes):** Ingests and processes multi-modal data from the clinical PhysioNet CGMacros cohort (45 participants). It provides +30-minute anticipatory glucose predictions using strict grouped holdouts and chronological time-series splits, while enabling export into HL7 FHIR R5 standard format for electronic health record (EHR) interoperability.

---

## 🛠️ Technical Stack, Models & Framework Details

| Layer | Technologies | Details & Implementation |
| :--- | :--- | :--- |
| **Data Ingestion & ETL** | PhysioNet CGMacros (v1.0.0), Pandas, NumPy | Automated SHA-256 verified extraction; parses Dexcom G6 Pro and Abbott FreeStyle Libre Pro CGM data aligned with Fitbit physical activity, meals, and laboratory covariates. |
| **Feature Engineering** | Python, SciPy, Scikit-learn | Multi-modal lag construction ($t-15$, $t-30$ min), rate of change $\Delta G$, cyclical diurnal Fourier transforms, rolling 120-minute macronutrient intake (carbs, fat, protein, fiber), and rolling Fitbit exertion metrics (METs, heart rate). |
| **Predictive ML Core** | `HistGradientBoostingRegressor`, `RandomForestRegressor` | Tabular gradient boosting and decision tree ensembles optimized for non-linear feature interactions and high robustness to tabular missingness. |
| **Evaluation Suite** | GroupKFold, Chronological Splitters | Participant-level holdouts (36 train / 9 test) and patient-specific 70/30 chronological splits evaluated against a rigid persistence baseline. |
| **API & Microservices** | FastAPI, Uvicorn, Pydantic | Asynchronous REST endpoints (`/simulate`, `/predict`, `/export/fhir`) with strict schema validation. |
| **User Interface** | Streamlit, Plotly | Interactive clinical dashboard providing real-time trajectory visualization, counterfactual parameter controls, and model evaluation views. |
| **Interoperability** | HL7 FHIR R5 JSON Schemas | Interoperable formatting mapping glucose telemetry directly to FHIR `Observation` resources with LOINC standard codes. |
| **DevOps & QA** | Docker, Docker Compose, Pytest, Ruff, GitHub Actions | Multi-stage containerization, full CI testing suite, automated benchmark workflows, and static analysis. |

---

## 📊 Real-Data Benchmark & Validation

Evaluated across the complete official PhysioNet CGMacros dataset (**45 participants**, **621,069** usable samples):

- **Grouped Participant Holdout (36 Train / 9 Unseen Test Participants):**
  - Random Forest Grouped MAE: **13.11 mg/dL**
  - HistGradientBoosting Grouped MAE: **13.40 mg/dL**
  - Persistence Baseline MAE: **13.39 mg/dL**
- **Personalized Chronological Split (Historical 70% Train / Future 30% Test):**
  - Personalized HGB Median MAE across all 45 participants: **12.05 mg/dL**
  - Beats or ties persistence in **21 / 45** individual participants.
- **Reporting Integrity:** Standard ML models often only match persistence for short horizons (+30 min); this project explicitly reports these limitations rather than obscuring baseline comparisons.

---

## 🏗️ System Architecture

```text
+---------------------------------------------------------------------------------------------------------+
|                                    DATA ACQUISITION & ETL SUBSYSTEM                                     |
|  +--------------------------------+   +---------------------------------+   +-------------------------+  |
|  |   PhysioNet CGMacros v1.0.0    |   | Wearable & Biomarker Streams    |   | Checksum Verification   |  |
|  |   45 Real Clinical Subjects    |-->| CGM (Dexcom/Libre) + Fitbit     |-->| SHA-256 Ingestion Engine  |  |
|  |   (Healthy, Pre-DM, T2D)       |   | Meals (Macros) + Demographics   |   | Automated Pipeline      |  |
|  +--------------------------------+   +---------------------------------+   +-------------------------+  |
+----------------------------------------------------+----------------------------------------------------+
                                                     |
                                                     v
+----------------------------------------------------+----------------------------------------------------+
|                                    FEATURE EXTRACTION & ALIGNMENT                                       |
|  • 1-Minute Timestamp Grid Inference & Alignment                                                        |
|  • Temporal Lags: G(t), G(t-15), G(t-30), Delta-G                                                        |
|  • 120-Minute Integrated Nutrient Accumulation (Carbs, Fats, Protein, Fiber)                            |
|  • Exertion Indices: Rolling METs, Heart Rate, Activity Calories                                        |
|  • Clinical Covariates: Baseline HbA1c, Fasting Glucose, Fasting Insulin, BMI                           |
+--------------------------+---------------------------------------------------+--------------------------+
                           |                                                   |
                           v                                                   v
+--------------------------+------------------------+ +------------------------+--------------------------+
|            PARTICIPANT HOLDOUT ENGINE             | |          CHRONOLOGICAL PERSONALIZED ENGINE        |
|  • Grouped Split: 36 Train / 9 Unseen Test        | |  • 70% Historic Chronological Train Split         |
|  • Zero Person-Level Identity Leakage             | |  • 30% Future Chronological Evaluation            |
+--------------------------+------------------------+ +------------------------+--------------------------+
                           \                                                   /
                            \-----------------------+-------------------------/
                                                    v
+---------------------------------------------------+-----------------------------------------------------+
|                                          PREDICTIVE ML CORE                                             |
|  +-------------------------------------+                      +--------------------------------------+  |
|  | HistGradientBoosting Regressor      |                      | Random Forest Regressor              |  |
|  | (Grouped MAE: 13.40 mg/dL)          |                      | (Grouped MAE: 13.11 mg/dL)           |  |
|  +-------------------------------------+                      +--------------------------------------+  |
|                                                    v                                                    |
|                                +--------------------------------------+                                 |
|                                | Benchmark vs. Persistence Baseline   |                                 |
|                                | (Reference: 13.39 mg/dL Grouped)     |                                 |
|                                +--------------------------------------+                                 |
+----------------------------------------------------+----------------------------------------------------+
                                                     |
                                                     v
+----------------------------------------------------+----------------------------------------------------+
|                                    DEPLOYMENT & CLINICAL INTEGRATION                                    |
|  +----------------------------+   +-------------------------------+   +------------------------------+  |
|  | Streamlit UI               |   | FastAPI Microservices         |   | Interoperability Engine      |  |
|  | Interactive Counterfactual |   | Asynchronous In-Silico        |   | FHIR R5 Observation Export   |  |
|  | Visualizer & Trajectory    |   | Simulation & Model Inference  |   | (LOINC Glucose Standards)    |  |
|  +----------------------------+   +-------------------------------+   +------------------------------+  |
+---------------------------------------------------------------------------------------------------------+