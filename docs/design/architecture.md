# Architecture

Landing and foundation are shown only as context. The new work is everything to the right of FOUNDATION.

## 1. Where the OMOP product sits

```mermaid
flowchart LR
    subgraph ctx["Existing (context only)"]
        K["Kafka topic per<br/>FHIR resource type"] --> LND["LANDING<br/>RECORD_METADATA<br/>RECORD_CONTENT"]
        LND -- "stream + task" --> FND["FOUNDATION<br/>content-surrogate dedup<br/>root fields flattened<br/>complex fields VARIANT<br/>append-only"]
    end
    subgraph new["New in this design"]
        VOC[("REF_VOCAB<br/>CONCEPT · CONCEPT_RELATIONSHIP<br/>CONCEPT_ANCESTOR · SOURCE_TO_CONCEPT_MAP")]
        STG["L3_OMOP_STG<br/>version resolution<br/>FHIR to OMOP mapping<br/>concept lookup"]
        CDM["L3_OMOP<br/>OMOP CDM v5.4 tables"]
        XW[("ID crosswalks<br/>FHIR id to OMOP id")]
        MON["Mapping coverage<br/>and DQ reports"]
    end
    FND -- "streams + tasks" --> STG
    VOC --> STG
    XW <--> STG
    STG --> CDM
    CDM --> MON
    CDM --> USE["Analysts · AI agents · OHDSI tools"]
    FND -- "streams + tasks" --> RQ["RQ (existing L3 product)"]
```

## 2. Refresh flow (streams and tasks)

Event tables map row by row. Derived tables are rebuilt for the persons that changed.

```mermaid
flowchart TD
    S["Streams on FOUNDATION tables<br/>PATIENT, ENCOUNTER, CONDITION, OBSERVATION, ..."] --> CL["1. Capture change set (one transaction)<br/>new resource versions + affected person keys"]
    CL --> VR["2. Latest version wins<br/>per resource_id, order by meta.lastUpdated<br/>fallback FOUNDATION_LOAD_TS"]
    VR --> ID["3. Resolve identity<br/>patient ref to person key (existing logic)<br/>FHIR id to OMOP integer id (crosswalk)"]
    ID --> MAP["4. Map to OMOP rows<br/>pick best coding, look up standard concept,<br/>route by concept domain, keep source value"]
    MAP --> UP["5. Upsert event tables<br/>replace older version of the same resource"]
    UP --> DER["6. Rebuild derived rows for affected persons<br/>OBSERVATION_PERIOD, CONDITION_ERA, DRUG_ERA"]
    DER --> REP["7. Refresh coverage and DQ metrics"]
```

Design notes:

- **Append-only foundation, versioned resources.** If an Observation moves from `preliminary` to `final`, foundation holds two rows (different content, different surrogate). Step 2 keeps only the latest per `resource_id`; step 5 replaces the earlier OMOP row so there is never a duplicate fact.
- **Stream mechanics.** A stream's offset advances when its change set is consumed in a committed transaction. Step 1 copies the changes from every foundation stream into a change-log table in a single transaction, and later steps read that table. This keeps the multi-stream product consistent and replayable.
- **Late-arriving data.** A Condition may arrive before its Encounter. `visit_occurrence_id` is nullable; the person-level rebuild back-fills the link when the Encounter arrives.
- **Identity.** Person identity uses the existing resolution logic. L3 consumes its output as a crosswalk of FHIR Patient id to person key, which becomes `PERSON.person_id`.

## 3. Person-centric model (first-slice tables highlighted in the matrix)

```mermaid
erDiagram
    PERSON ||--|{ OBSERVATION_PERIOD : "has"
    PERSON ||--o{ VISIT_OCCURRENCE : "has"
    PERSON ||--o{ CONDITION_OCCURRENCE : "has"
    PERSON ||--o{ DRUG_EXPOSURE : "has"
    PERSON ||--o{ PROCEDURE_OCCURRENCE : "has"
    PERSON ||--o{ MEASUREMENT : "has"
    PERSON ||--o{ OBSERVATION : "has"
    PERSON ||--o| DEATH : "may have"
    VISIT_OCCURRENCE ||--o{ CONDITION_OCCURRENCE : "during"
    VISIT_OCCURRENCE ||--o{ DRUG_EXPOSURE : "during"
    VISIT_OCCURRENCE ||--o{ PROCEDURE_OCCURRENCE : "during"
    VISIT_OCCURRENCE ||--o{ MEASUREMENT : "during"
    CONDITION_OCCURRENCE }o--|| CONCEPT : "condition_concept_id"
    DRUG_EXPOSURE }o--|| CONCEPT : "drug_concept_id"
    MEASUREMENT }o--|| CONCEPT : "measurement_concept_id"
    PERSON }o--o| LOCATION : "lives at"
    VISIT_OCCURRENCE }o--o| CARE_SITE : "at"
    VISIT_OCCURRENCE }o--o| PROVIDER : "by"
    CONCEPT ||--o{ CONCEPT_ANCESTOR : "ancestor of"
    CONCEPT ||--o{ CONCEPT_RELATIONSHIP : "maps to"
```

## 4. Proposed schemas

| Schema | Contents | Notes |
|---|---|---|
| `REF_VOCAB` | Athena tables plus `SOURCE_TO_CONCEPT_MAP` for local codes | Shared reference; versioned by vocabulary release. Other L3 products can use it. |
| `L3_OMOP_STG` | Change log, version resolution, ID crosswalks, mapping views | Internal, not for consumers |
| `L3_OMOP` | OMOP CDM v5.4 tables, `CDM_SOURCE` | Consumer-facing |
| `L3_OMOP_QA` | Mapping coverage, unmapped-code report, DQ results | Drives the mapping backlog and the pitch metrics |

Every OMOP row keeps `*_source_value`, `*_source_concept_id` and a lineage column pointing to the foundation `RESOURCE_ID`, so any fact can be traced back to full-fidelity FHIR.

## 5. Concept mapping logic (per coded field)

```mermaid
flowchart TD
    A["FHIR CodeableConcept<br/>coding[] from the submitter"] --> B{"Any coding in a<br/>standard-mapped system?<br/>SNOMED · LOINC · RxNorm · ICD-10 · CPT · CVX"}
    B -- yes --> C["Athena: 'Maps to' standard concept<br/>(ICD-10 to SNOMED, etc.)"]
    B -- no --> D{"Local code present in<br/>SOURCE_TO_CONCEPT_MAP?"}
    D -- yes --> E["Use curated mapping for<br/>that submitter + code"]
    D -- no --> F["concept_id = 0<br/>keep source value<br/>add to unmapped report"]
    C --> G["Route by concept domain:<br/>Condition · Drug · Procedure · Measurement · Observation"]
    E --> G
    F --> H["Default domain by FHIR resource type"]
```

If several codings map, a documented priority order decides (for example the most specific standard system first, then fall back through the list).
