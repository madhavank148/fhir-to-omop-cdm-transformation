# Open questions, prerequisites and phased plan

## Decisions already made

| Topic | Decision |
|---|---|
| Role of OMOP | An additional L3 data product; FHIR foundation remains the base |
| Source | FOUNDATION tables only |
| Refresh | Streams and tasks (not dynamic tables) |
| FHIR version | R4 |
| Claims / EOB | Not received, so out of scope |
| Foundation behaviour | Append-only; the same resource can appear twice with different content (for example preliminary then final), so L3 must resolve the latest version |
| Person identity | Existing logic identifies a person; L3 consumes it as a crosswalk |
| CDM version (proposed) | 5.4: it is what the FHIR IG targets, and the CDM site notes tool support for 5.5 features is still in progress |

## Open questions

| # | Question | Why it matters | Owner |
|---|---|---|---|
| 1 | Does foundation keep `meta.lastUpdated` / `meta.versionId`? | Version ordering for "latest wins"; the fallback is `FOUNDATION_LOAD_TS`, which reflects load order, not clinical order | Data engineering |
| 2 | How many submitters send local codes, and who owns mapping them? | Determines achievable coverage; the largest source of effort | Product / clinical data |
| 3 | Form and stability of the person key from the identity logic | Becomes `PERSON.person_id`; must not change on replay | Data engineering |
| 4 | Which resource types and which medication resource (`MedicationRequest`, `MedicationStatement`, `MedicationAdministration`) arrive in production? | The IG maps only `MedicationStatement`; drug rules depend on this | Data engineering |
| 5 | Data volume (patients, resources, daily change rate) | Sizes the person-scoped rebuild and warehouse needs | Data engineering |
| 6 | Athena access and vocabulary scope | Prerequisite for any concept mapping | Platform |
| 7 | CPT4 licensing | CPT is AMA-licensed; the Athena download uses a UMLS licence key | Legal / compliance |
| 8 | Who owns vocabulary refreshes and re-mapping when a release changes concepts? | Vocabulary is a living dataset | Platform |
| 9 | Who are the first named consumers (AI agent team, analysts)? | Drives priorities and the benchmark set | Product |
| 10 | Is `Coverage` data reliable enough for `PAYER_PLAN_PERIOD`? | Phase 2 candidate | Product |

## Prerequisites (before build)

1. Register for Athena, select vocabularies (SNOMED, ICD10CM, LOINC, RxNorm, CVX, UCUM, NDC, CPT4 if licensed, plus domain and class reference data) and download a release.
2. Confirm CPT licensing path or decide to proceed without CPT4 initially.
3. Agree the person-key crosswalk contract.
4. Pick the benchmark questions and a baseline run on foundation.

## Phased plan

| Phase | Scope | Exit criteria |
|---|---|---|
| **0. Foundations** | Load Athena into `REF_VOCAB`; create schemas; crosswalk tables; `CDM_SOURCE`; coverage report skeleton | Vocabulary queryable; a code can be looked up end to end |
| **1. First slice** | `PERSON`, `OBSERVATION_PERIOD`, `VISIT_OCCURRENCE`, `CONDITION_OCCURRENCE`, `MEASUREMENT`, `DRUG_EXPOSURE` (incl. MedicationRequest rule and immunizations); version resolution; streams and tasks | Benchmark questions 1 to 4, 6, 9, 10 answerable; coverage report published; row counts reconcile to foundation |
| **2. Broaden** | `PROCEDURE_OCCURRENCE`, `OBSERVATION`, `DEATH`, `LOCATION`, `CARE_SITE`, `PROVIDER`, `CONDITION_ERA`, `DRUG_ERA`; local code map workflow | Full benchmark run; DQD checks executed and tracked |
| **3. Consume** | Semantic view and agent over OMOP; `PAYER_PLAN_PERIOD` from Coverage; documentation for consumers | FHIR vs OMOP benchmark comparison presented |

## Risks and mitigations

| Risk | Mitigation |
|---|---|
| Low coverage from unmapped local codes | Coverage report from day one; prioritise by row volume; mapping workflow per submitter |
| Duplicate or stale facts from resource versions | Latest-version step before mapping; replace by `resource_id` |
| Derived tables become expensive | Person-scoped incremental rebuild; size with volume numbers (Q5) |
| Vocabulary drift on refresh | Version the vocabulary, record it in `CDM_SOURCE`, re-run mapping regression |
| Over-claiming | Present OMOP as complementary and lossy by design; keep lineage to FHIR |
| No named consumer | AI benchmark is the consumer and the evidence |
