# FHIR R4 → OMOP CDM v5.4 mapping matrix

Basis: the HL7 FHIR-to-OMOP IG (STU1 ballot, CDM 5.4 target, developed against FHIR R5 / IPA / US Core; concepts are version-agnostic) and the OMOP CDM v5.4 specification. Rows marked **ours** are not covered by the IG and need our own rule. Verify field-level detail against the IG StructureMaps when implementing.

Legend: **Slice 1** = first delivery, **Later** = planned, **Out** = not planned.

## Clinical resources

| FHIR R4 resource | OMOP target | Status | Notes |
|---|---|---|---|
| Patient | `PERSON` (+ `LOCATION`, `DEATH`) | Slice 1 | Sex at birth to `gender_concept_id`; race and ethnicity from US Core extensions; `deceasedDateTime` to `DEATH`. Gender identity belongs in `OBSERVATION`. `person_id` from the identity crosswalk. |
| (derived) | `OBSERVATION_PERIOD` | Slice 1 | Not in FHIR. Derived per person (for example first to last event, overlaps merged). Required by OHDSI tools. |
| Encounter | `VISIT_OCCURRENCE` | Slice 1 | `class` to `visit_concept_id`; `visit_end_date` is mandatory, so define a rule for open encounters. Discharge and admit source to concepts where available. |
| Condition | `CONDITION_OCCURRENCE` | Slice 1 | `onsetDateTime` or `recordedDate` to start date; `abatement` to end date; status in `condition_status_concept_id`. |
| Observation (labs, vitals) | `MEASUREMENT` | Slice 1 | `value[x]` to `value_as_number` / `value_as_concept_id`; unit to `unit_concept_id` (UCUM). `component[]` (for example blood pressure) splits into one row per component. |
| Observation (social history, survey, other) | `OBSERVATION` | Slice 1 | The target table is decided by the concept's domain, not by the FHIR category alone. |
| MedicationStatement | `DRUG_EXPOSURE` | Slice 1 | IG-mapped. |
| MedicationRequest | `DRUG_EXPOSURE` | Slice 1 (**ours**) | Not in the IG. `drug_exposure_end_date` is mandatory; derive from dosage and dispense duration, fall back to start date plus days supply. Document the rule (accuracy loss). |
| Immunization | `DRUG_EXPOSURE` | Slice 1 | CVX codes. |
| MedicationAdministration | `DRUG_EXPOSURE` | Later (**ours**) | Exact administration time is lost (information loss). |
| Procedure | `PROCEDURE_OCCURRENCE` | Later | CPT / SNOMED / HCPCS via Athena. |
| AllergyIntolerance | `OBSERVATION` | Later | IG-mapped. |
| DiagnosticReport | `MEASUREMENT` / `OBSERVATION` / `NOTE` | Later (**ours**) | Results are Observations; report text is a note. |
| Device | `DEVICE_EXPOSURE` | Later | |
| Specimen | `SPECIMEN` | Later | |
| CarePlan, CareTeam, Goal | `OBSERVATION` or none | Later / Out | Low analytic value; decide per use. |
| DocumentReference | `NOTE` | Out (initially) | Free text and NLP are out of scope. |

## Health-system and context resources

| FHIR R4 resource | OMOP target | Status | Notes |
|---|---|---|---|
| Organization | `CARE_SITE` | Later | Cross-file identifier references must be resolved (see architecture). |
| Location | `LOCATION` / `CARE_SITE` | Later | Patient address also feeds `LOCATION`. |
| Practitioner / PractitionerRole | `PROVIDER` | Later | Specialty to `specialty_concept_id`. |

## Payer data

| FHIR R4 resource | OMOP target | Status | Notes |
|---|---|---|---|
| Coverage | `PAYER_PLAN_PERIOD` | Later (phase 2+) | Received in some file types. |
| Claim, ExplanationOfBenefit | `COST` | Out | Not received; the IG also excludes claims. |

## Derived tables

| OMOP table | Status | How |
|---|---|---|
| `CONDITION_ERA`, `DRUG_ERA` | Later | Collapsed from occurrences or exposures with a gap rule; rebuilt per affected person. |
| `DOSE_ERA` | Out | Needs reliable dose data. |
| `CDM_SOURCE` | Slice 1 | Version, source description, vocabulary version. |

## Field-level policy

1. **Always keep** `*_source_value` and `*_source_concept_id` (original code) and a lineage column to foundation `RESOURCE_ID`.
2. **Unmapped** codes get `*_concept_id = 0`; never drop the row.
3. **Type concept** (`*_type_concept_id`) is set to the EHR provenance concept for all clinical rows.
4. **Dates:** where FHIR gives only a datetime, derive the date; leave `*_datetime` NULL when the source has no time.
5. **Version resolution:** one OMOP row per FHIR `resource_id` (latest version wins).

## Known information loss (from the IG's four categories)

| Loss type | Example here |
|---|---|
| Fidelity | A specific local lab code maps to a broader standard concept |
| Information | Exact witness time of a medication administration |
| Relationship | Links such as Condition `evidence` or `basedOn` are not preserved |
| Accuracy | End dates for medication exposures derived, not observed |

Full-fidelity FHIR remains in foundation and is reachable from any OMOP row through the lineage column.
