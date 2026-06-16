# USDM v4 — A Data Manager's Walk-Through

### Digital SoA, mapped end to end. One assessment — the 6-Minute Walk Test — traced from a sentence in the protocol to the sixteen USDM v4 tables that make a protocol machine-readable.



---

## What this is

Most data managers have heard the acronym **USDM** (the CDISC Unified Study Definitions Model). Few have seen it land on a *real* assessment.

So this walk takes **one** assessment — a 6-Minute Walk Test (6MWT), performed at three visits on Day 1 of each visit— and follows it all the way down: from a single sentence in a protocol to the actual USDM v4 classes, columns, and IDs that hold it.

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

*Tamer Chowdhury · Chair, ACDM Data Management Technology Expert Group (DMTEG)*

*Walk one assessment. Map it once. Automate everything downstream.*
