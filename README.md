[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/drive/1d0u_fj3UGVpI_wk1iOGAqKF5WXjoNgEa#scrollTo=hrvM2kgwaZab)

# Garment Production Risk Triage

An operational proof-of-concept prioritizing daily line-level interventions in apparel manufacturing. Rather than attempting to predict exact continuous productivity days in advance, this system acts as a **morning risk-triage sheet**—identifying the 10 highest-risk lines requiring immediate supervisor attention.

---

## 1. Problem Framing & Operational Reality

In high-volume garment manufacturing, production managers oversee dozens of sewing and finishing lines simultaneously. Two key operational realities define this environment:
1. **Floor Bottlenecks Outpace Batch Data:** On the sewing floor, physical symptoms (WIP accumulating between workstations, machine downtime, sewing thread breaks) are visible to line supervisors in real time. Batch models cannot serve as early-warning systems for intra-day line balance.
2. **Management by Exception:** Plant managers cannot physically audit 30+ lines during morning line balancing. What they need is a prioritized triage list: *Which lines have structural risk profiles that warrant intervention before the shift starts?*

---

## 2. Methodology & Leakage Prevention

* **Dataset:** 1,197 production shift records from a garment manufacturing plant.
* **Target Metric:** Binary target miss ($$Actual\ Productivity < Targeted\ Productivity$$).
* **Strict Chronological Evaluation:** Standard randomized `train_test_split` creates temporal data leakage. We implemented an 80/20 chronological split:
  * **Train Set (80% / 957 shifts):** January 1, 2015 – February 26, 2015
  * **Test Set (20% / 240 shifts):** February 26, 2015 – March 11, 2015

### Department-Level Missingness Audit
A structural analysis revealed that the `wip` (work in progress) feature was missing for **100% of finishing department records** (506/506) while present in 100% of sewing records. Imputing or using `wip` directly would cause the model to use missingness as an artificial proxy for department identity. It was excluded from training.

---

## 3. Benchmark & Precision@10 Results

In an operational triage setting, whole-dataset accuracy is misleading due to class imbalance (75% baseline success rate). The operational metric that matters is **Precision@10**—out of the 10 highest-risk lines flagged each morning, how many actually missed their target?

| Model | Test Accuracy | Precision@10 (Top 10 Flagged Lines) | Operational Interpretation |
| :--- | :--- | :--- | :--- |
| **Naive Baseline ("Always Safe")** | 75.00% | 0 / 10 (0%) | Assumes every line hits target; misses all failures. |
| **Logistic Regression** | 77.50% | 6 / 10 (60%) | Flags general high-risk segments (finishing lines). |
| **Random Forest (max_depth=4)** | **80.00%** | **9 / 10 (90%)** | Refines sorting within known high-risk categories. |

---

## 4. Model Audit: What is Random Forest Actually Learning?

To prevent overclaiming, we audited the specific rows caught by the Random Forest versus Logistic Regression:

1. **Zero Unseen Signal:** All 9 of the Random Forest's true-positive catches had `yesterday_miss = 1`. The tree model did not uncover hidden nonlinear drivers (e.g., SMV or staffing variations) predicting misses from previously safe lines.
2. **Team-Specific Base Rates:** With `max_depth=4`, the Random Forest isolated granular subgroup failure rates (e.g., *Finishing lines for Team 8 run structurally riskier than Team 6 when both missed target yesterday*).
3. **Decision Utility:** While not discovering new physical signals, the model successfully resolves tie-breaking among flagged lines, directing limited supervisory resources with 90% precision.

---

## 5. Sample Morning Triage Sheet Output

Generated dynamically for floor supervisors prior to morning shift kick-off:

| Line / Team | Department | Target Productivity | Yesterday Missed? | Risk Score | Triage Action |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Team 8** | Finishing | 0.70 | YES | **75.0%** | **HIGH RISK** (Supervisor Audit) |
| **Team 6** | Finishing | 0.70 | YES | **72.5%** | **HIGH RISK** (Supervisor Audit) |
| **Team 10** | Finishing | 0.75 | YES | **66.2%** | **HIGH RISK** (Supervisor Audit) |
| **Team 7** | Finishing | 0.65 | NO | **53.1%** | **HIGH RISK** (Monitor Line Balance) |
| **Team 7** | Sewing | 0.65 | NO | **52.5%** | **HIGH RISK** (Monitor Line Balance) |
| **Team 8** | Sewing | 0.70 | NO | **52.1%** | **HIGH RISK** (Monitor Line Balance) |
| Team 2 | Finishing | 0.75 | NO | 36.8% | NORMAL |
| Team 12 | Finishing | 0.80 | NO | 34.1% | NORMAL |

---

## 6. How to Run

1. Open the interactive Colab notebook: [![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/drive/1d0u_fj3UGVpI_wk1iOGAqKF5WXjoNgEa#scrollTo=hrvM2kgwaZab)
2. Run all cells sequentially to reproduce the temporal split, audit logs, and morning floor triage table.
