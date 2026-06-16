# USDM v4 — A Data Manager's Walk-Through

### Digital SoA, mapped end to end. One assessment — the 6-Minute Walk Test — traced from a sentence in the protocol to the sixteen USDM v4 tables that make a protocol machine-readable.

---

## What this is

Most data managers have heard the acronym **USDM** (the CDISC Unified Study Definitions Model). Few have seen it land on a *real* assessment.

So this walk takes **one** assessment — a 6-Minute Walk Test (6MWT), performed at three visits — and follows it all the way down: from a single sentence in a protocol to the actual USDM v4 classes, columns, and IDs that hold it.

No abstraction. Just the tables, the column names, and where each one points.

**Who it's for:** clinical data managers, programmers, and standards folks who want to *see* USDM rather than read about it — and then try the same walk on an assessment from their own study.

> This is an **illustrative teaching example**, not normative guidance. The canonical source is CDISC USDM v4 and the [`usdm4`](https://pypi.org/project/usdm4/) Python package by Dave Iberson-Hurst. Names and structures here are drawn directly from that package.

---

## The story

**Study `1111_0000` · Amendment 0**

> *"At Visits 1, 2 and 3 (Day 1 of each), the subject performs a 6-Minute Walk Test. Distance walked is recorded in meters, with the assessment date and a flag for whether the test was performed."* — illustrative protocol text.

The 6MWT belongs to the **Questionnaires, Ratings & Scales (QRS)** category of assessments.

| Visit | Encounter | When | What is collected |
|-------|-----------|------|-------------------|
| Visit 1 | `ENC-V1` | Day 1 | 6MWT · distance (m) |
| Visit 2 | `ENC-V2` | Day 1 | 6MWT · distance (m) |
| Visit 3 | `ENC-V3` | Day 1 | 6MWT · distance (m) |

Each visit traces the same chain: `ACT-6MWT` → `BC-6MWT` → `BCC-QRS`.

---

## The sixteen sheets = sixteen USDM v4 classes

Every cell is a row in a USDM table. Every `*Id` is a foreign key into another sheet. Table names are `snake_case`; columns are kept `camelCase`, 1:1 with the USDM JSON payload format.

| Table (`snake_case`) | USDM v4 class | Our instance |
|----------------------|---------------|--------------|
| `study` | Study | `STD-1111-0000` |
| `study_version` | StudyVersion | `SV-1111-0000-1` |
| `study_amendment` | StudyAmendment | `AMD-0` |
| `study_design` | StudyDesign | `SD-1111-0000` |
| `study_epoch` | StudyEpoch | `EPO-TRT` |
| `encounter` | Encounter (×3) | `ENC-V1 / V2 / V3` |
| `biomedical_concept_category` | BiomedicalConceptCategory | `BCC-QRS` |
| `biomedical_concept` | BiomedicalConcept | `BC-6MWT` |
| `biomedical_concept_property` | BiomedicalConceptProperty (×4) | `BCP-DIST / UNIT / DATE / PERF` |
| `response_code` | ResponseCode | `BRC-YES/NO · m/ft` |
| `code` | Code | `LOINC · CDISC · UCUM · NCIt` |
| `alias_code` | AliasCode | `AC-6MWT-NCI` |
| `activity` | Activity | `ACT-6MWT` |
| `schedule_timeline` | ScheduleTimeline | `ST-MAIN` |
| `scheduled_activity_instance` | ScheduledActivityInstance (×3) | `SAI-V1 / V2 / V3` |
| `timing` | Timing (×3) | `TIM-V1 / V2 / V3` |

---

## The heart of the walk — the Biomedical Concept

A **Biomedical Concept** is a measured thing. A **Property** is what you record about it. A **ResponseCode** is what's allowed.

- **`biomedical_concept_category` · `BCC-QRS`** — `code` → CDISC C100129 · `memberIds` → `[BC-6MWT]`
- **`biomedical_concept` · `BC-6MWT`** — `reference` → LOINC 64098-7 · `properties` → 4 · `code` → CD-LP35-9

Four `biomedical_concept_property` rows — each carries its own `datatype` and, where applicable, its permitted `responseCodes`:

| Property | `datatype` | `responseCodes` |
|----------|------------|-----------------|
| `BCP-DIST` · Distance Walked | `decimal` | — (free) |
| `BCP-UNIT` · Unit of Measure | `code` | `BRC-METER`, `BRC-FOOT` |
| `BCP-DATE` · Assessment Date | `date` | — (free) |
| `BCP-PERF` · Performed? | `code` | `BRC-YES`, `BRC-NO` |

The specification doesn't live in a CRF. It lives in USDM.

---

## Where it gets collected

One Activity. One Timeline. Three instances (one per visit). Three timings (Day 1 of each).

- **`activity` · `ACT-6MWT`** — `biomedicalConceptIds` `[BC-6MWT]` · `bcCategoryIds` `[BCC-QRS]` · `timelineId` `ST-MAIN`
- **`schedule_timeline` · `ST-MAIN`** — `mainTimeline` true · `entryId` `SAI-V1` · `instances` `[SAI-V1, SAI-V2, SAI-V3]`
- **`scheduled_activity_instance` · `SAI-V1/V2/V3`** — `activityIds` `[ACT-6MWT]` · `encounterId` `ENC-Vn` · `epochId` `EPO-TRT` · `timelineId` `ST-MAIN`
- **`timing` · `TIM-V1/V2/V3`** — `type` Fixed Reference · `value` `P0D` · "Day 1 of each visit"

One specification — repeated three times — without ever copying-and-pasting the activity definition.

---

## An important distinction: USDM is the protocol, SDTM is the data

They live in different layers. The membrane between them is what we automate.

**USDM v4 — the protocol, as a dataset**
Study / Design / Epoch / Encounter · Activity / ScheduleTimeline / SAI / Timing · BiomedicalConcept + Property + ResponseCode · Code + AliasCode.
*Owned by Clinical, Statistics, Protocol authors. Lives once. Referenced everywhere downstream.*

**SDTM / CDASH / ADaM / CRF / edit checks — the collected data**
SDTM domains (QS, VS, FA, …) · CDASH eCRF conventions · ADaM analysis-ready · CRF specs, edit checks, mapping specs.
*Owned by Data Management, Programming, Statistics. Generated, not duplicated — from USDM where possible.*

---

## What this changes for a data manager

| | Today | With USDM |
|---|-------|-----------|
| **CRF generation** | Hand-built from the Word protocol + SDTM IG; review takes weeks. | Derived from `BC + Property + ResponseCode`. The 6MWT eCRF is generated, not transcribed. |
| **Edit checks** | Written separately per study — range, missing, conditional logic. | Emitted from `datatype + responseCodes`: distance decimal; unit ∈ {m, ft}; performed ∈ {Y, N}. |
| **Real-time review** | Waits for SDTM; endpoints live in the SAP, not the protocol. | SAI at V1/V2/V3 links to the endpoint. Data lands → dashboard updates. |

**The principle:** the further upstream we structure intent, the less we re-encode downstream.

---

## Try it yourself

1. **Pick one assessment** from your own study — a vital sign, a questionnaire, a lab — and walk it through the same sixteen sheets.
2. **Write the BC + properties + allowed values.** If you can name what's measured and the permitted values, the rest is mechanical.
3. **Find the membrane** — where does USDM end and SDTM begin in your trial? That's where automation lives.

---

## Appendix — the sixteen classes, attribute by attribute

Below, each class is expandable. For every class you'll see its key attributes (exact USDM v4 names), what each one holds, and the value it carries in our 6MWT example. `*Id` / `*Ids` attributes are foreign keys into another sheet. Each class has additional attributes in the full USDM v4 model — see the [interactive class browser](https://ankonyeni.github.io/usdm-v4-docs/diagram-viewer/) for the complete list.

<details>
<summary><b>study</b> &nbsp;·&nbsp; Study &nbsp;·&nbsp; <i>7 attributes in full</i></summary>


| Attribute | Holds | Our 6MWT value |
|-----------|-------|----------------|
| `id` | string identifier | STD-1111-0000 |
| `name` | short study name | Study 1111_0000 |
| `versions` | child StudyVersion ids | [SV-1111-0000-1] |
| `instanceType` | the class name | Study |

</details>

<details>
<summary><b>study_version</b> &nbsp;·&nbsp; StudyVersion &nbsp;·&nbsp; <i>28 attributes in full</i></summary>


| Attribute | Holds | Our 6MWT value |
|-----------|-------|----------------|
| `id` | string identifier | SV-1111-0000-1 |
| `versionIdentifier` | version label | 1 |
| `rationale` | why this version exists | Original protocol |
| `studyDesigns` | child StudyDesign ids | [SD-1111-0000] |
| `amendments` | child StudyAmendment ids | [AMD-0] |
| `biomedicalConcepts` | BC ids defined for the study | [BC-6MWT] |
| `bcCategories` | BC category ids | [BCC-QRS] |
| `instanceType` | the class name | StudyVersion |

</details>

<details>
<summary><b>study_amendment</b> &nbsp;·&nbsp; StudyAmendment &nbsp;·&nbsp; <i>17 attributes in full</i></summary>


| Attribute | Holds | Our 6MWT value |
|-----------|-------|----------------|
| `id` | string identifier | AMD-0 |
| `number` | amendment number | 0 |
| `summary` | plain-language summary | Original protocol — no changes |
| `instanceType` | the class name | StudyAmendment |

</details>

<details>
<summary><b>study_design</b> &nbsp;·&nbsp; StudyDesign &nbsp;·&nbsp; <i>28 attributes in full</i></summary>


| Attribute | Holds | Our 6MWT value |
|-----------|-------|----------------|
| `id` | string identifier | SD-1111-0000 |
| `name` | design name | Main Study Design |
| `epochs` | child StudyEpoch ids | [EPO-TRT] |
| `encounters` | child Encounter ids | [ENC-V1, ENC-V2, ENC-V3] |
| `activities` | child Activity ids | [ACT-6MWT] |
| `scheduleTimelines` | child ScheduleTimeline ids | [ST-MAIN] |
| `instanceType` | the class name | StudyDesign |

</details>

<details>
<summary><b>study_epoch</b> &nbsp;·&nbsp; StudyEpoch &nbsp;·&nbsp; <i>10 attributes in full</i></summary>


| Attribute | Holds | Our 6MWT value |
|-----------|-------|----------------|
| `id` | string identifier | EPO-TRT |
| `name` | epoch name | Treatment |
| `type` | epoch type (Code) | C-TREATMENT-EPOCH |
| `instanceType` | the class name | StudyEpoch |

</details>

<details>
<summary><b>encounter</b> &nbsp;·&nbsp; Encounter &nbsp;·&nbsp; <i>15 attributes in full</i></summary>


| Attribute | Holds | Our 6MWT value |
|-----------|-------|----------------|
| `id` | string identifier | ENC-V1  /  ENC-V2  /  ENC-V3 |
| `name` | visit name | Visit 1  /  Visit 2  /  Visit 3 |
| `type` | encounter type (Code) | C-SITE-VISIT |
| `previousId / nextId` | order in the visit chain | links V1 → V2 → V3 |
| `instanceType` | the class name | Encounter |

</details>

<details>
<summary><b>biomedical_concept_category</b> &nbsp;·&nbsp; BiomedicalConceptCategory &nbsp;·&nbsp; <i>10 attributes in full</i></summary>


| Attribute | Holds | Our 6MWT value |
|-----------|-------|----------------|
| `id` | string identifier | BCC-QRS |
| `name` | category name | Questionnaires, Ratings & Scales |
| `code` | the category's Code id | → CDISC C100129 |
| `memberIds` | BC ids that belong to it | [BC-6MWT] |
| `instanceType` | the class name | BiomedicalConceptCategory |

</details>

<details>
<summary><b>biomedical_concept</b> &nbsp;·&nbsp; BiomedicalConcept &nbsp;·&nbsp; <i>10 attributes in full</i></summary>


| Attribute | Holds | Our 6MWT value |
|-----------|-------|----------------|
| `id` | string identifier | BC-6MWT |
| `name` | concept name | 6-Minute Walk Test |
| `reference` | the anchoring code id | → LOINC 64098-7 |
| `properties` | child BiomedicalConceptProperty ids | [BCP-DIST, BCP-UNIT, BCP-DATE, BCP-PERF] |
| `code` | the concept's Code id | → CD-LP35-9 |
| `instanceType` | the class name | BiomedicalConcept |

</details>

<details>
<summary><b>biomedical_concept_property</b> &nbsp;·&nbsp; BiomedicalConceptProperty &nbsp;·&nbsp; <i>11 attributes in full</i></summary>


| Attribute | Holds | Our 6MWT value |
|-----------|-------|----------------|
| `id` | string identifier | BCP-DIST / BCP-UNIT / BCP-DATE / BCP-PERF |
| `name` | what is recorded | Distance / Unit / Assessment Date / Performed? |
| `datatype` | the value's type | decimal / code / date / code |
| `responseCodes` | permitted ResponseCode ids | DIST: — · UNIT: [BRC-METER, BRC-FOOT] · DATE: — · PERF: [BRC-YES, BRC-NO] |
| `isRequired` | must it be collected? | true (DIST, DATE, PERF) |
| `instanceType` | the class name | BiomedicalConceptProperty |

</details>

<details>
<summary><b>response_code</b> &nbsp;·&nbsp; ResponseCode &nbsp;·&nbsp; <i>7 attributes in full</i></summary>


| Attribute | Holds | Our 6MWT value |
|-----------|-------|----------------|
| `id` | string identifier | BRC-METER / BRC-FOOT / BRC-YES / BRC-NO |
| `name` | display label | meter / foot / Yes / No |
| `isEnabled` | is this option active? | true |
| `code` | the underlying Code id | → UCUM m / UCUM [ft_i] / NCIt C49488 / C49487 |
| `instanceType` | the class name | ResponseCode |

</details>

<details>
<summary><b>code</b> &nbsp;·&nbsp; Code &nbsp;·&nbsp; <i>7 attributes in full</i></summary>


| Attribute | Holds | Our 6MWT value |
|-----------|-------|----------------|
| `id` | string identifier | CD-LP35-9 (example) |
| `code` | the code value | 64098-7 |
| `codeSystem` | the source system | LOINC |
| `codeSystemVersion` | system version | 2.77 |
| `decode` | human-readable meaning | 6 minute walk test (distance) |
| `instanceType` | the class name | Code |

</details>

<details>
<summary><b>alias_code</b> &nbsp;·&nbsp; AliasCode &nbsp;·&nbsp; <i>5 attributes in full</i></summary>


| Attribute | Holds | Our 6MWT value |
|-----------|-------|----------------|
| `id` | string identifier | AC-6MWT-NCI |
| `standardCode` | the primary Code id | → LOINC 64098-7 |
| `standardCodeAliases` | equivalent Code ids | [→ NCIt C100129 alias] |
| `instanceType` | the class name | AliasCode |

</details>

<details>
<summary><b>activity</b> &nbsp;·&nbsp; Activity &nbsp;·&nbsp; <i>15 attributes in full</i></summary>


| Attribute | Holds | Our 6MWT value |
|-----------|-------|----------------|
| `id` | string identifier | ACT-6MWT |
| `name` | activity name | 6-Minute Walk Test |
| `biomedicalConceptIds` | BC ids measured here | [BC-6MWT] |
| `bcCategoryIds` | BC category ids | [BCC-QRS] |
| `timelineId` | the timeline it sits on | ST-MAIN |
| `instanceType` | the class name | Activity |

</details>

<details>
<summary><b>schedule_timeline</b> &nbsp;·&nbsp; ScheduleTimeline &nbsp;·&nbsp; <i>13 attributes in full</i></summary>


| Attribute | Holds | Our 6MWT value |
|-----------|-------|----------------|
| `id` | string identifier | ST-MAIN |
| `name` | timeline name | Main Timeline |
| `mainTimeline` | is this the primary timeline? | true |
| `entryId` | first instance id | SAI-V1 |
| `instances` | ordered SAI ids | [SAI-V1, SAI-V2, SAI-V3] |
| `instanceType` | the class name | ScheduleTimeline |

</details>

<details>
<summary><b>scheduled_activity_instance</b> &nbsp;·&nbsp; ScheduledActivityInstance &nbsp;·&nbsp; <i>12 attributes in full</i></summary>


| Attribute | Holds | Our 6MWT value |
|-----------|-------|----------------|
| `id` | string identifier | SAI-V1 / SAI-V2 / SAI-V3 |
| `activityIds` | activities performed here | [ACT-6MWT] |
| `encounterId` | the visit it happens at | ENC-V1 / ENC-V2 / ENC-V3 |
| `epochId` | the epoch it falls in | EPO-TRT |
| `timelineId` | its timeline | ST-MAIN |
| `instanceType` | the class name | ScheduledActivityInstance |

</details>

<details>
<summary><b>timing</b> &nbsp;·&nbsp; Timing &nbsp;·&nbsp; <i>15 attributes in full</i></summary>


| Attribute | Holds | Our 6MWT value |
|-----------|-------|----------------|
| `id` | string identifier | TIM-V1 / TIM-V2 / TIM-V3 |
| `type` | timing type (Code) | Fixed Reference |
| `value` | ISO-8601 offset | P0D |
| `valueLabel` | human-readable timing | Day 1 of the visit |
| `relativeToFrom` | anchor point | Start to start |
| `instanceType` | the class name | Timing |

</details>


---

## Resources

- **USDM v4 interactive class browser** (by Ankon Yeni): https://ankonyeni.github.io/usdm-v4-docs/diagram-viewer/ — best companion for "what does this class actually contain?"
- **`usdm4` package** (Dave Iberson-Hurst): https://pypi.org/project/usdm4/ — the authoritative class model used here.
- **CDISC DDF / USDM**: https://www.cdisc.org/ddf

---

## Open questions for the group

- Which assessment in your study would be the cleanest first walk-through?
- Where do you currently lose protocol intent on its way to SDTM?
- If you had a BC layer today, which downstream artifact would you generate first?

---

*Tamer Chowdhury*

*Walk one assessment. Map it once. Automate everything downstream.*
