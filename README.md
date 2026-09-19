# Garment Production Risk-Triage: An Honest Proof-of-Concept

*A small decision-support model for flagging which garment-factory production lines are at elevated risk of missing their daily target — built on public data, with as much attention paid to catching my own mistakes as to the final numbers.*

**The short version:**
- A logistic regression model, honestly validated, catches 60% of real misses in its top 10 daily flags — 6x better than random guessing, using only historical data.
- A more sophisticated model appeared to hit 90% — I didn't take that at face value, traced exactly where the improvement came from, and found something narrower and more specific than "a better model."
- The most important finding didn't come from a notebook — it came from standing on an actual factory floor.

---

## Why this project

Industrial engineering teams in garment factories work under a hard constraint: floor attention is scarce, and underperformance isn't distributed evenly across lines. This is a direct application of *management by exception* — surface the exceptions worth attention rather than monitoring every line equally.

This project asks a narrow, testable question: given yesterday's production data, which team+department lines deserve extra attention today? It deliberately uses public data, not any employer's internal records, and treats "does this actually help" as an open question rather than an assumption.

## Dataset

UCI Machine Learning Repository #597 — *Productivity Prediction of Garment Employees* (Rahim, Imran & Ahmed, 2021). 1,197 shift records, 12 teams across sewing and finishing departments, one Bangladeshi factory, January–March 2015.

## Methodology

**Target:** `actual_productivity < targeted_productivity` — did this line miss its target that day.

**Data audit, before any modeling:**
- `department` — cleaned a raw text inconsistency that split one category into three inconsistent string values.
- `wip` — **excluded.** Missing in 100% of finishing-department rows, 0% of sewing rows — a structural artifact of how the two departments are tracked, not random. Including it would just re-encode department.
- `idle_time` / `idle_men` — **excluded.** Nonzero in only 1.5% of all rows.
- `over_time`, `incentive` — retained, flagged as weak-to-null in their correlation with the outcome, with their exact timing relative to each day's result not fully resolvable from the data alone.

**Validation: chronological, never random.** Train on the earliest ~80% of dates, test on the most recent ~20%. Team+department lines are repeated, time-ordered observations — a random split lets a model see nearby days from the same line and produces an inflated, unrealistic result. (This isn't theoretical: an early random split in this project inflated one result from an honest 30% to a misleading 60%, which is why chronological validation became non-negotiable.)

**Metric: precision@10, not accuracy.** With misses at ~27% of shifts, a model that always predicts "safe" scores 75% accuracy while flagging nobody — useless for triage. Precision@10 asks the real operational question: of the 10 riskiest flagged shifts, how many are actual misses.

## Results

| Approach | Result | Notes |
|---|---|---|
| Always predict "safe" | 75.0% accuracy | Flags nobody — not a real baseline for this task. |
| "Missed target yesterday?" (zero-cost lookup) | 50.0% precision (33/66 flagged) | The single most informative baseline in the project. |
| Logistic regression (7 features, incl. lag) | 77.5% accuracy · **60% precision@10** | Headline interpretable model. |
| Random Forest (100 trees, depth 4) | 80.0% accuracy · 90% precision@10 | See below — verified, then decomposed. |

**What's actually driving Random Forest's number.** Instead of reporting 90% at face value, I checked what it was made of. Every one of Random Forest's correct catches shares the same "missed yesterday" flag as logistic regression's hits — it found zero cases where yesterday was safe but something else (SMV, overtime, staffing) signaled risk. Its real contribution is narrower: it learns team-specific base rates *within* the already-flagged population more precisely than logistic regression's single shared team coefficient allows — several of its top-ranked rows carry identical scores at the exact team+department+target-tier level, confirming this is a refinement in granularity, not a newly discovered pattern in the day's production data.

## Floor reality — the most important limitation

Direct observation on a real factory floor surfaced the constraint that matters most here: a line supervisor sees a stalled machine, piling work-in-progress, or an absent operator in real time, by eye. This model works from yesterday's end-of-day data — a full day behind. It is not a substitute for that kind of real-time judgment.

Given that lag, the realistic use case is narrower than "predictive tool": a **morning macro-audit triage list for managers overseeing many lines**, who can't personally watch every line and already work from end-of-day reports — not a replacement for floor supervision.

There's also a structural reason work like this rarely gets deployed in this industry: thin margins, constant style/SKU turnover, and a fragmented, mostly small-factory ownership structure that makes in-house data science capacity uncommon. That's an industry-economics explanation, not a comment on whether the method is sound.

## Related work

This dataset has real prior work, worth naming rather than implying this is untouched ground:
- Binary target-vs-actual classification on this exact dataset already exists publicly.
- SHAP-based explainability has already been applied to it separately.
- A 2025 peer-reviewed paper classifies productivity into three tiers across eleven algorithms and pairs it with a linear-programming layer for **worker reallocation** across departments — more sophisticated modeling than this project, answering a genuinely different operational question.

**What's different here:** a *floor-triage* framing for lines already staffed and running (not a reassignment question), strict chronological validation, precision@k as the evaluation metric, and a limitation — the real-time-vs-lagging-data gap — that only direct floor observation could have surfaced.

## What this doesn't claim

- Doesn't generalize beyond this one factory and this two-month window.
- Doesn't claim overtime or incentive causally affect productivity — overtime's model coefficient was effectively zero, and no causal claim is supportable from observational data regardless. (In practice, overtime in this industry is driven by order books and shipment deadlines, not model output — this caveat guards against a naive misreading of the numbers more than it reflects a real decision risk on the floor.)
- Isn't deployment-ready.
- Doesn't claim AI improves factory output — the model only modestly beats a free, zero-cost lookup rule, and no intervention was ever tested.

## The honest ceiling — and what survives it

No public, retrospective dataset — this one included — can prove that a model like this would actually help a real factory. Proving that requires an intervention: flag lines, have someone act on the flags, and measure whether misses actually drop compared to lines that weren't flagged. That is not a gap this project failed to close through better modeling. It is a gap no dataset assembled after the fact can close, in this or any field — the same ceiling applies to most observational research generally, and very likely to the 2025 paper cited above as well, which also reports offline metrics rather than a live trial.

A factory running 40–50 IE staff under a manager with 15 years of experience almost certainly already tracks daily line performance closely. What a project like this could plausibly add, given real data and a real trial, isn't replacing that expertise — it's testing whether formalizing informal, one-person pattern recognition into something explicit and consistent across a much larger operation than any one manager can hold in their head adds anything measurable. That is a real, testable question. It is also one that no public dataset, including this one, can answer.

## What I'd do next

- Get informal validation from IE practitioners on whether the flagged output matches real-world triage decisions.
- Add uncertainty estimates, given the small number of teams and short time window.
- Note explicitly what more historical data would and wouldn't fix: it would sharpen these same offline metrics, but it would still be observational — the only step that actually resolves the open question above is a live trial with real decisions and measured outcomes, not a larger CSV.

## Stack

Python, pandas, scikit-learn, Google Colab. Implementation was AI-assisted; every methodological decision, validation design choice, and reported result was independently checked before being trusted — including catching and correcting an inflated result from an initial random train/test split.

---

*Author: Majbha Uddin · [LinkedIn](https://www.linkedin.com/in/majbha-uddin-62a264219) · [Notebook](Garment_productivity_project.ipynb) · [Open in Colab](https://colab.research.google.com/drive/1d0u_fj3UGVpI_wk1iOGAqKF5WXjoNgEa)*
