# USDM v4 — A Data Manager's Walk-Through

### Digital SoA, mapped end to end. One assessment — the 6-Minute Walk Test — traced from a sentence in the protocol to the sixteen USDM v4 tables that make a protocol machine-readable.

*A walk-through and reference for clinical data managers.*

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
<summary><b>study</b> &nbsp;&middot;&nbsp; Study &nbsp;&middot;&nbsp; <i>7 attributes in full</i></summary>
<table>
<thead><tr><th>Attribute</th><th>Holds</th><th>Our 6MWT value</th></tr></thead>
<tbody>
<tr><td><code>id</code></td><td>string identifier</td><td>STD-1111-0000</td></tr>
<tr><td><code>name</code></td><td>short study name</td><td>Study 1111_0000</td></tr>
<tr><td><code>versions</code></td><td>child StudyVersion ids</td><td>[SV-1111-0000-1]</td></tr>
<tr><td><code>instanceType</code></td><td>the class name</td><td>Study</td></tr>
</tbody>
</table>
</details>

<details>
<summary><b>study_version</b> &nbsp;&middot;&nbsp; StudyVersion &nbsp;&middot;&nbsp; <i>28 attributes in full</i></summary>
<table>
<thead><tr><th>Attribute</th><th>Holds</th><th>Our 6MWT value</th></tr></thead>
<tbody>
<tr><td><code>id</code></td><td>string identifier</td><td>SV-1111-0000-1</td></tr>
<tr><td><code>versionIdentifier</code></td><td>version label</td><td>1</td></tr>
<tr><td><code>rationale</code></td><td>why this version exists</td><td>Original protocol</td></tr>
<tr><td><code>studyDesigns</code></td><td>child StudyDesign ids</td><td>[SD-1111-0000]</td></tr>
<tr><td><code>amendments</code></td><td>child StudyAmendment ids</td><td>[AMD-0]</td></tr>
<tr><td><code>biomedicalConcepts</code></td><td>BC ids defined for the study</td><td>[BC-6MWT]</td></tr>
<tr><td><code>bcCategories</code></td><td>BC category ids</td><td>[BCC-QRS]</td></tr>
<tr><td><code>instanceType</code></td><td>the class name</td><td>StudyVersion</td></tr>
</tbody>
</table>
</details>

<details>
<summary><b>study_amendment</b> &nbsp;&middot;&nbsp; StudyAmendment &nbsp;&middot;&nbsp; <i>17 attributes in full</i></summary>
<table>
<thead><tr><th>Attribute</th><th>Holds</th><th>Our 6MWT value</th></tr></thead>
<tbody>
<tr><td><code>id</code></td><td>string identifier</td><td>AMD-0</td></tr>
<tr><td><code>number</code></td><td>amendment number</td><td>0</td></tr>
<tr><td><code>summary</code></td><td>plain-language summary</td><td>Original protocol — no changes</td></tr>
<tr><td><code>instanceType</code></td><td>the class name</td><td>StudyAmendment</td></tr>
</tbody>
</table>
</details>

<details>
<summary><b>study_design</b> &nbsp;&middot;&nbsp; StudyDesign &nbsp;&middot;&nbsp; <i>28 attributes in full</i></summary>
<table>
<thead><tr><th>Attribute</th><th>Holds</th><th>Our 6MWT value</th></tr></thead>
<tbody>
<tr><td><code>id</code></td><td>string identifier</td><td>SD-1111-0000</td></tr>
<tr><td><code>name</code></td><td>design name</td><td>Main Study Design</td></tr>
<tr><td><code>epochs</code></td><td>child StudyEpoch ids</td><td>[EPO-TRT]</td></tr>
<tr><td><code>encounters</code></td><td>child Encounter ids</td><td>[ENC-V1, ENC-V2, ENC-V3]</td></tr>
<tr><td><code>activities</code></td><td>child Activity ids</td><td>[ACT-6MWT]</td></tr>
<tr><td><code>scheduleTimelines</code></td><td>child ScheduleTimeline ids</td><td>[ST-MAIN]</td></tr>
<tr><td><code>instanceType</code></td><td>the class name</td><td>StudyDesign</td></tr>
</tbody>
</table>
</details>

<details>
<summary><b>study_epoch</b> &nbsp;&middot;&nbsp; StudyEpoch &nbsp;&middot;&nbsp; <i>10 attributes in full</i></summary>
<table>
<thead><tr><th>Attribute</th><th>Holds</th><th>Our 6MWT value</th></tr></thead>
<tbody>
<tr><td><code>id</code></td><td>string identifier</td><td>EPO-TRT</td></tr>
<tr><td><code>name</code></td><td>epoch name</td><td>Treatment</td></tr>
<tr><td><code>type</code></td><td>epoch type (Code)</td><td>C-TREATMENT-EPOCH</td></tr>
<tr><td><code>instanceType</code></td><td>the class name</td><td>StudyEpoch</td></tr>
</tbody>
</table>
</details>

<details>
<summary><b>encounter</b> &nbsp;&middot;&nbsp; Encounter &nbsp;&middot;&nbsp; <i>15 attributes in full</i></summary>
<table>
<thead><tr><th>Attribute</th><th>Holds</th><th>Our 6MWT value</th></tr></thead>
<tbody>
<tr><td><code>id</code></td><td>string identifier</td><td>ENC-V1  /  ENC-V2  /  ENC-V3</td></tr>
<tr><td><code>name</code></td><td>visit name</td><td>Visit 1  /  Visit 2  /  Visit 3</td></tr>
<tr><td><code>type</code></td><td>encounter type (Code)</td><td>C-SITE-VISIT</td></tr>
<tr><td><code>previousId / nextId</code></td><td>order in the visit chain</td><td>links V1 → V2 → V3</td></tr>
<tr><td><code>instanceType</code></td><td>the class name</td><td>Encounter</td></tr>
</tbody>
</table>
</details>

<details>
<summary><b>biomedical_concept_category</b> &nbsp;&middot;&nbsp; BiomedicalConceptCategory &nbsp;&middot;&nbsp; <i>10 attributes in full</i></summary>
<table>
<thead><tr><th>Attribute</th><th>Holds</th><th>Our 6MWT value</th></tr></thead>
<tbody>
<tr><td><code>id</code></td><td>string identifier</td><td>BCC-QRS</td></tr>
<tr><td><code>name</code></td><td>category name</td><td>Questionnaires, Ratings &amp; Scales</td></tr>
<tr><td><code>code</code></td><td>the category's Code id</td><td>→ CDISC C100129</td></tr>
<tr><td><code>memberIds</code></td><td>BC ids that belong to it</td><td>[BC-6MWT]</td></tr>
<tr><td><code>instanceType</code></td><td>the class name</td><td>BiomedicalConceptCategory</td></tr>
</tbody>
</table>
</details>

<details>
<summary><b>biomedical_concept</b> &nbsp;&middot;&nbsp; BiomedicalConcept &nbsp;&middot;&nbsp; <i>10 attributes in full</i></summary>
<table>
<thead><tr><th>Attribute</th><th>Holds</th><th>Our 6MWT value</th></tr></thead>
<tbody>
<tr><td><code>id</code></td><td>string identifier</td><td>BC-6MWT</td></tr>
<tr><td><code>name</code></td><td>concept name</td><td>6-Minute Walk Test</td></tr>
<tr><td><code>reference</code></td><td>the anchoring code id</td><td>→ LOINC 64098-7</td></tr>
<tr><td><code>properties</code></td><td>child BiomedicalConceptProperty ids</td><td>[BCP-DIST, BCP-UNIT, BCP-DATE, BCP-PERF]</td></tr>
<tr><td><code>code</code></td><td>the concept's Code id</td><td>→ CD-LP35-9</td></tr>
<tr><td><code>instanceType</code></td><td>the class name</td><td>BiomedicalConcept</td></tr>
</tbody>
</table>
</details>

<details>
<summary><b>biomedical_concept_property</b> &nbsp;&middot;&nbsp; BiomedicalConceptProperty &nbsp;&middot;&nbsp; <i>11 attributes in full</i></summary>
<table>
<thead><tr><th>Attribute</th><th>Holds</th><th>Our 6MWT value</th></tr></thead>
<tbody>
<tr><td><code>id</code></td><td>string identifier</td><td>BCP-DIST / BCP-UNIT / BCP-DATE / BCP-PERF</td></tr>
<tr><td><code>name</code></td><td>what is recorded</td><td>Distance / Unit / Assessment Date / Performed?</td></tr>
<tr><td><code>datatype</code></td><td>the value's type</td><td>decimal / code / date / code</td></tr>
<tr><td><code>responseCodes</code></td><td>permitted ResponseCode ids</td><td>DIST: — · UNIT: [BRC-METER, BRC-FOOT] · DATE: — · PERF: [BRC-YES, BRC-NO]</td></tr>
<tr><td><code>isRequired</code></td><td>must it be collected?</td><td>true (DIST, DATE, PERF)</td></tr>
<tr><td><code>instanceType</code></td><td>the class name</td><td>BiomedicalConceptProperty</td></tr>
</tbody>
</table>
</details>

<details>
<summary><b>response_code</b> &nbsp;&middot;&nbsp; ResponseCode &nbsp;&middot;&nbsp; <i>7 attributes in full</i></summary>
<table>
<thead><tr><th>Attribute</th><th>Holds</th><th>Our 6MWT value</th></tr></thead>
<tbody>
<tr><td><code>id</code></td><td>string identifier</td><td>BRC-METER / BRC-FOOT / BRC-YES / BRC-NO</td></tr>
<tr><td><code>name</code></td><td>display label</td><td>meter / foot / Yes / No</td></tr>
<tr><td><code>isEnabled</code></td><td>is this option active?</td><td>true</td></tr>
<tr><td><code>code</code></td><td>the underlying Code id</td><td>→ UCUM m / UCUM [ft_i] / NCIt C49488 / C49487</td></tr>
<tr><td><code>instanceType</code></td><td>the class name</td><td>ResponseCode</td></tr>
</tbody>
</table>
</details>

<details>
<summary><b>code</b> &nbsp;&middot;&nbsp; Code &nbsp;&middot;&nbsp; <i>7 attributes in full</i></summary>
<table>
<thead><tr><th>Attribute</th><th>Holds</th><th>Our 6MWT value</th></tr></thead>
<tbody>
<tr><td><code>id</code></td><td>string identifier</td><td>CD-LP35-9 (example)</td></tr>
<tr><td><code>code</code></td><td>the code value</td><td>64098-7</td></tr>
<tr><td><code>codeSystem</code></td><td>the source system</td><td>LOINC</td></tr>
<tr><td><code>codeSystemVersion</code></td><td>system version</td><td>2.77</td></tr>
<tr><td><code>decode</code></td><td>human-readable meaning</td><td>6 minute walk test (distance)</td></tr>
<tr><td><code>instanceType</code></td><td>the class name</td><td>Code</td></tr>
</tbody>
</table>
</details>

<details>
<summary><b>alias_code</b> &nbsp;&middot;&nbsp; AliasCode &nbsp;&middot;&nbsp; <i>5 attributes in full</i></summary>
<table>
<thead><tr><th>Attribute</th><th>Holds</th><th>Our 6MWT value</th></tr></thead>
<tbody>
<tr><td><code>id</code></td><td>string identifier</td><td>AC-6MWT-NCI</td></tr>
<tr><td><code>standardCode</code></td><td>the primary Code id</td><td>→ LOINC 64098-7</td></tr>
<tr><td><code>standardCodeAliases</code></td><td>equivalent Code ids</td><td>[→ NCIt C100129 alias]</td></tr>
<tr><td><code>instanceType</code></td><td>the class name</td><td>AliasCode</td></tr>
</tbody>
</table>
</details>

<details>
<summary><b>activity</b> &nbsp;&middot;&nbsp; Activity &nbsp;&middot;&nbsp; <i>15 attributes in full</i></summary>
<table>
<thead><tr><th>Attribute</th><th>Holds</th><th>Our 6MWT value</th></tr></thead>
<tbody>
<tr><td><code>id</code></td><td>string identifier</td><td>ACT-6MWT</td></tr>
<tr><td><code>name</code></td><td>activity name</td><td>6-Minute Walk Test</td></tr>
<tr><td><code>biomedicalConceptIds</code></td><td>BC ids measured here</td><td>[BC-6MWT]</td></tr>
<tr><td><code>bcCategoryIds</code></td><td>BC category ids</td><td>[BCC-QRS]</td></tr>
<tr><td><code>timelineId</code></td><td>the timeline it sits on</td><td>ST-MAIN</td></tr>
<tr><td><code>instanceType</code></td><td>the class name</td><td>Activity</td></tr>
</tbody>
</table>
</details>

<details>
<summary><b>schedule_timeline</b> &nbsp;&middot;&nbsp; ScheduleTimeline &nbsp;&middot;&nbsp; <i>13 attributes in full</i></summary>
<table>
<thead><tr><th>Attribute</th><th>Holds</th><th>Our 6MWT value</th></tr></thead>
<tbody>
<tr><td><code>id</code></td><td>string identifier</td><td>ST-MAIN</td></tr>
<tr><td><code>name</code></td><td>timeline name</td><td>Main Timeline</td></tr>
<tr><td><code>mainTimeline</code></td><td>is this the primary timeline?</td><td>true</td></tr>
<tr><td><code>entryId</code></td><td>first instance id</td><td>SAI-V1</td></tr>
<tr><td><code>instances</code></td><td>ordered SAI ids</td><td>[SAI-V1, SAI-V2, SAI-V3]</td></tr>
<tr><td><code>instanceType</code></td><td>the class name</td><td>ScheduleTimeline</td></tr>
</tbody>
</table>
</details>

<details>
<summary><b>scheduled_activity_instance</b> &nbsp;&middot;&nbsp; ScheduledActivityInstance &nbsp;&middot;&nbsp; <i>12 attributes in full</i></summary>
<table>
<thead><tr><th>Attribute</th><th>Holds</th><th>Our 6MWT value</th></tr></thead>
<tbody>
<tr><td><code>id</code></td><td>string identifier</td><td>SAI-V1 / SAI-V2 / SAI-V3</td></tr>
<tr><td><code>activityIds</code></td><td>activities performed here</td><td>[ACT-6MWT]</td></tr>
<tr><td><code>encounterId</code></td><td>the visit it happens at</td><td>ENC-V1 / ENC-V2 / ENC-V3</td></tr>
<tr><td><code>epochId</code></td><td>the epoch it falls in</td><td>EPO-TRT</td></tr>
<tr><td><code>timelineId</code></td><td>its timeline</td><td>ST-MAIN</td></tr>
<tr><td><code>instanceType</code></td><td>the class name</td><td>ScheduledActivityInstance</td></tr>
</tbody>
</table>
</details>

<details>
<summary><b>timing</b> &nbsp;&middot;&nbsp; Timing &nbsp;&middot;&nbsp; <i>15 attributes in full</i></summary>
<table>
<thead><tr><th>Attribute</th><th>Holds</th><th>Our 6MWT value</th></tr></thead>
<tbody>
<tr><td><code>id</code></td><td>string identifier</td><td>TIM-V1 / TIM-V2 / TIM-V3</td></tr>
<tr><td><code>type</code></td><td>timing type (Code)</td><td>Fixed Reference</td></tr>
<tr><td><code>value</code></td><td>ISO-8601 offset</td><td>P0D</td></tr>
<tr><td><code>valueLabel</code></td><td>human-readable timing</td><td>Day 1 of the visit</td></tr>
<tr><td><code>relativeToFrom</code></td><td>anchor point</td><td>Start to start</td></tr>
<tr><td><code>instanceType</code></td><td>the class name</td><td>Timing</td></tr>
</tbody>
</table>
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
