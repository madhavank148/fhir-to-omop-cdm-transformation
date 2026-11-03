# Benchmark questions

Purpose: give the pitch evidence rather than opinion. Run the same questions through an AI agent (for example a Cortex Analyst / Cortex Agent) twice, once over FOUNDATION and once over the OMOP product, and compare answer accuracy, the SQL produced, and how often a human has to correct it.

## How to run

1. Write the 20 to 30 questions below (extend with real stakeholder questions).
2. A data engineer writes a **reference SQL answer** for each on the OMOP model (and validates it against a hand check of a sample).
3. Run the agent over each model, with only the semantic definitions each model provides.
4. Score: **correct / partially correct / wrong**, plus latency and number of joins.
5. Report the difference together with the OMOP mapping coverage for the concepts used (a wrong answer caused by an unmapped local code is a mapping finding, not a modelling failure).

## Starter questions

| # | Question | Domains | Why FHIR is hard |
|---|---|---|---|
| 1 | How many patients aged 20–30 had an HbA1c above 9% in the last 12 months? | Person, Measurement | Multiple codes and systems for HbA1c, units, `valueQuantity`, age join |
| 2 | Which patients started metformin after a type 2 diabetes diagnosis? | Condition, Drug | Medication as reference, code or text; ordering of two event types |
| 3 | Number of patients with any statin exposure in the last year | Drug | Brand and generic codes; needs ingredient hierarchy |
| 4 | Average outpatient visits per year for patients with hypertension, by age band | Person, Visit, Condition | Encounter class codes; cohort definition across tables |
| 5 | Top 10 procedures by distinct patient count | Procedure | CPT, SNOMED and local procedure codes mixed |
| 6 | Median days from first diabetes diagnosis to first inpatient admission | Condition, Visit | Event ordering; visit type normalisation |
| 7 | Readmission within 30 days after an inpatient visit with heart failure | Visit, Condition | Self-join on visits; diagnosis hierarchy |
| 8 | Share of patients with a documented smoking status | Observation | Value codes vary by submitter |
| 9 | Distribution of systolic blood pressure for patients over 65 | Measurement | Blood pressure is a panel with components |
| 10 | Patients with both an HbA1c above 8% and a statin prescription | Measurement, Drug | Two domains, unit and code normalisation |
| 11 | New diagnoses of asthma per quarter, by sex | Condition, Person | Condition hierarchy; onset versus recorded date |
| 12 | Vaccination coverage for influenza by age group | Drug (immunization) | CVX versus local vaccine codes |
| 13 | Patients observed for at least 2 years with no emergency visit | Observation period, Visit | Needs a defined observation period |
| 14 | Which care sites see the most patients with chronic kidney disease? | Condition, Visit, Care site | Organization reference resolution |
| 15 | Count of patients whose most recent lab result for a code is abnormal | Measurement | Latest-result logic; reference ranges |

## Scoring sheet (template)

| # | Foundation: result | Foundation: correct? | OMOP: result | OMOP: correct? | Notes (mapping gap, ambiguity) |
|---|---|---|---|---|---|
| 1 | | | | | |

## Metrics to report in the pitch

- Agent accuracy: FHIR foundation vs OMOP (percentage of questions fully correct).
- Median joins and median SQL length per question.
- Mapping coverage by domain: percent of rows with `concept_id <> 0`.
- Percent of questions that were blocked by unmapped codes (shows the value of local code mapping).
