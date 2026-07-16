# Patient Journey Analytics — SQL Lakehouse
---

## Overview
This project takes raw synthetic electronic health records and builds a production-style analytics lakehouse, then answers five clinically meaningful questions using advanced SQL. The data spans 11,447 patients with roughly 670,000 encounters and 8.8 million clinical observations. Seventeen source tables are ingested, cleaned and modelled into a proper start schema then used to answer five clinically meaningful questions using SQL.
some key findings include an alarming rate of follow up appointmets (33.7%) by newly diagnosed hypertensive patients within the first 90 days and prevalance of certain comorbid conditions occuring together such as kidney disease and diabetes, through the care cycle of most patients living with a chronic condition.

---
## Architecture

```
Raw CSVs (Synthea, ~15.6M rows across 17 tables)
        │
        ▼
┌─────────────────────────────────────────────┐
│  BRONZE — raw ingestion                      │
│  COPY INTO, idempotent, lineage metadata     │
└─────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────┐
│  SILVER — cleaned & conformed                │
│  Dedup (ROW_NUMBER), typing, derived fields, │
│  clinical plausibility bounds, DQ anti-joins │
└─────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────┐
│  GOLD — star schema + serving marts          │
│  5 dimensions, 4 facts, 7 aggregated marts   │
└─────────────────────────────────────────────┘
```

Built entirely in **Spark SQL** on Databricks (Unity Catalog, Delta Lake), version-controlled via Databricks Git folders.

---

## Entity Relationship Diagram of tables
<img width="2214" height="2283" alt="data_model" src="https://github.com/user-attachments/assets/277cb520-f7aa-4f3d-b7ca-7423e7eb0983" />

---
## Analytics

Five analytical notebooks, each answering a business question through SQL.

### 1. Hospital readmissions
Of patients discharged from a hospital stay, how many end up back in the hospital within a month -and does that risk vary across the patient population?
**Approach :** the startting point was to reconstruct,for all patients with more than one stay, the gap between one discharge and the next admission. After the overall rate was esthablised, the same patients were split out into age groups to see the spread of the risk.
**Finding :** the overall 30-day readmission rate was 18.2%, which sits comfortably within the range of real hospital systems. Breaking the rate down by age revealed a clear pattern: readmission climbs steadily through mid-life, peaks among patients aged 50 to 79 and then noticeably drops among the oldest patients: this can be interpreted as the frailest patients in the oldest group are more likely to move into hospice care or pass away than to be formally readmitted.

### 2. Which conditions genuinely occur together
Which combinations of health conditions tend to occur in the same patient more often than would be expected by chance- as opposed to simply being common conditions that show up everywhere?
**Approach :** The first, most obvious approach was to count how many patients shared each pair of conditions. That produced a list dominated by extremely common, everyday conditions — the pairs at the top of the list were things like minor dental issues and seasonal infections, simply because so many patients have them, not because they are medically related. That result was a signal to change approach rather than to report it: the analysis needed to measure association, not just co-occurrence. So the comparison was reframed around how much more often two conditions appeared together than would be expected if they were statistically independent — correcting for how common each condition already is on its own.
**Finding :** the progression from advanced chronic kidney disease to kidney failure, the escalation from infection to sepsis to septic shock, and cancer conditions pairing tightly with their own staging codes. The corrected method surfaced genuine disease pathways that the naive count had completely obscured.

### 3. Hypertension care cascade
Of patients diagnosed with high blood pressure, how many actually complete the path to good control — and if patients are falling off that path, where does it happen?
**Approach :** The ideal patient journey was modelled as a sequence of checkpoints: diagnosis, a timely follow-up visit, being started on medication, and eventually reaching a healthy blood pressure reading. Each patient's records were checked against each checkpoint in turn, so that patients could only be counted as reaching a later stage if they had genuinely passed through the one before it. This turns a single success rate into a funnel, showing exactly where patients are lost rather than just how many succeed overall.
**Finding :** The funnel showed one dramatic point of attrition and almost no loss anywhere else. Only a third of diagnosed patients returned for a follow-up visit within the guideline-recommended window — but of those who did, the pathway worked extremely well: essentially all were prescribed medication, and 96% went on to achieve a controlled blood pressure. That raised an obvious follow-up question: were the other two-thirds of patients falling out of care entirely, or just returning later than the guideline recommends? Checking their full histories showed the latter — nearly all eventually came back, most of them roughly a year later rather than within 90 days. The conclusion is that the bottleneck in this pathway is the timing of follow-up, not the effectiveness of treatment once it happens.

### 4. How chronic kidney disease progresses over time
How does kidney disease actually progress in patients over the years — and can the recorded disease stages be trusted as a genuine reflection of declining kidney function?
**Approach :** Each patient's kidney-disease diagnoses were placed in chronological order to reconstruct an individual timeline, from earliest stage through to the most advanced. This made it possible to measure how long patients typically remain at a given stage before progressing to the next, and to trace the most common overall trajectories through the disease. Because a coded stage is only useful if it reflects real physiology, the analysis was cross-checked against patients' actual kidney-function lab results, to confirm the stages weren't just administrative labels.
**An anomaly worth investigating** A small number of patients appeared to move backward — recorded as recovering from kidney failure to an early stage, which should not be clinically possible. Rather than discard these as data errors, their full histories were reviewed individually. All of them had received kidney transplants, which explained the apparent reversal: a transplant genuinely can restore kidney function and justify a lower disease stage.
**Finding :** 149 patients could be traced through the complete progression from the earliest stage to kidney failure, spanning an average of 16 years. Progression accelerates as the disease advances — patients typically spend years at the earliest stages but move to kidney failure within a couple of years of reaching the most severe stage. The lab-value cross-check confirmed that kidney function declines in step with the recorded stages, which means the staging in this data reflects genuine disease severity rather than arbitrary coding.

### 5. Cost of kidney disease treatment
Do patients with chronic kidney disease genuinely cost more to treat, or does it only look that way because those patients tend to be older?
**Approach :** The first comparison was the simplest one available: average costs for CKD patients versus everyone else. That comparison suggested CKD patients cost roughly twice as much. But CKD patients in this population are, on average, 26 years older than patients without the condition — and older patients cost more regardless of diagnosis. Trusting the raw comparison would have overstated the effect of the disease itself, so the comparison was rebuilt to compare CKD and non-CKD patients only within the same age group, which isolates the cost associated with the disease from the cost associated simply with being older.

**Finding :** Once age was controlled for, most of the apparent cost gap disappeared. Within the most populous age groups the CKD cost premium fell from a headline of roughly double to somewhere in the range of 10 to 20%, and among the oldest patients the gap closed almost entirely. The original, unadjusted comparison would have significantly overstated how much the disease itself drives cost — a useful caution against comparing patient groups without accounting for how their underlying demographics differ.

---
## Recommendations
1. Target post-discharge follow-up at the 50–79 age group specifically, since this is where readmission risk is highest and where earlier intervention (discharge planning, follow-up calls, medication reconciliation) is likely to have the greatest impact.
2. Close the gap between guideline-recommended and actual hypertension follow-up timing. Since treatment is highly effective once patients return, the highest-value intervention is proactive outreach — reminders or scheduled check-ins — within the first 90 days after diagnosis, rather than waiting for patients to self-initiate a visit roughly a year later.
3. When assessing the cost impact of a chronic condition such as CKD, always stratify by age before comparing groups. An unadjusted comparison here overstated the disease's cost impact by roughly two-fold; the same distortion would apply to any resource-planning decision based on a similarly naive comparison.

--

## Data-Quality Diligence

Every analysis surfaced a data-quality issue that was investigated rather than ignored:

- **Creatinine** values up to 115 mg/dL (physiologically impossible) — bounded at 20, documented.
- **Backward CKD transitions** (ESRD → Stage 1) — investigated, found to be transplant recipients.
- **Statin prescriptions** spanning 64 years — a Synthea generation artifact; persistence metric corrected to use capped covered-time rather than calendar span.
- **Condition table** conflating diagnoses with social determinants — classified separately so comorbidity analysis measures real clinical links.
  
---

## Tech Stack

**Databricks** · Unity Catalog · Delta Lake · **Spark SQL** · Databricks Workflows · Databricks AI/BI Dashboards · Tableau Public · Git

---
*All data is synthetic and generated by Synthea. No real patient data is used at any point.*
