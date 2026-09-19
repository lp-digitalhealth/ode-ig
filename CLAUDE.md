# CLAUDE.md — Oral Health Data Exchange (ODE) Implementation Guide

This is the canonical **FHIR R4** IG for the Oral Health Data Exchange (ODE), published by the
Oral Health Interoperability Alliance (OHIA). **FSH is the only source of truth** — SUSHI + the
HL7 IG Publisher generate everything else (the site, the StructureDefinitions/ValueSets, the NPM
package). There is no hand-maintained OpenAPI/Swagger/crosswalk to keep in sync.

The IG has two interfaces:
- **Referral** — `ServiceRequest` (three directional profiles) over an HL7 **Clinical Order
  Workflows (COW)** `Task` backbone.
- **Claims submission** — split `Claim` profiles + a `$submit` operation, for dental-for-medical
  billing (837D/837P/837I generated downstream by a separate translator, not here).

## Build & validate — run after every change

```bash
sushi .          # compile FSH -> fsh-generated/  (must be clean)
./_build.sh      # full IG Publisher build -> output/  (open output/qa.html)
```

- Toolchain: Node.js 20+ (SUSHI), Java 17 (IG Publisher), Ruby + Jekyll (site render).
- The **one expected SUSHI warning** is the terminology-canonical mismatch (see below). Everything
  else should be resolved. `output/qa.html` is the IG Publisher feedback loop — drive toward zero
  errors.

## Dependencies (`sushi-config.yaml`)

- `hl7.fhir.us.core: 6.1.0` — inherit US Core wherever a profile exists.
- `hl7.fhir.uv.cow: 1.0.0-ballot` — the referral workflow backbone; the Task conforms to COW.
- **No CARIN Blue Button** — the EOB profile was removed; do not reintroduce it.

## Authoring conventions (non-negotiable)

- **Inherit US Core** where a profile exists (`$ucServiceRequest`, `$ucObs`, `$ucProcedure`,
  `$ucDocRef`, …). Base FHIR only for `Task` and `List` (US Core has neither) — note the exception
  in the profile `Description`.
- **COW basing:** `ODEReferralTask` inherits the COW Task profile. The three directional
  ServiceRequests stay parented on **US Core**, with COW conformance carried by the Task (this is
  decision D1's default; do not reparent the ServiceRequests onto COW without an explicit decision).
- **Aliases are global** — add to `input/fsh/aliases.fsh`, never inline.
- **Terminology namespace:** CodeSystems/ValueSets set an explicit `^url` under
  `http://ohia-codes.org`, while the IG canonical is `https://oralhealthalliance.net/fhir`. SUSHI
  warns about the mismatch — **intentional, do not "fix" it.**
- **IP boundaries:** CDT is ADA-licensed — reference the system URI (`http://www.ada.org/cdt`),
  **never enumerate codes**. Same rule for X12 837 mappings and X12 code systems — reference, never
  reproduce.
- **"Should-support":** model as an optional (non-MS) slice with `^short`/`^comment` stating
  SHOULD-populate. FHIR has no should-support flag — do not fake it with Must Support.
- **CapabilityStatements** are `Usage: #definition` (requirements), not `#instance`.

## Directional coding — keep these three referral profiles' slices intact

- `ODEMedicalToDentalReferral` — `reasonCode[icd10]` MS, `code.coding[cpt|hcpcs]` MS, no CDT.
- `ODEDentalToDentalReferral` — `code.coding[cdt]` MS, `code.coding[snodent]` optional (SHOULD),
  `bodySite.extension[tooth]` **1..1 MS**.
- `ODEDentalToMedicalReferral` — `reasonCode[icd10]` MS, `code.coding[cpt|hcpcs]` MS, `snodent`
  optional, screening/finding result MS in `supportingInfo`.

## Claims submission — split profiles, not a union profile

Learn the CARIN lesson: enforce completeness **structurally**, per type — do NOT build one union
profile gated by conditional invariants.

- `ODEClaimBase` (abstract) — universal core + shared slice defs (careTeam.role, supportingInfo.category,
  diagnosis.type); carries `ODEBillingRail` as the routing determinant.
- `ODEOralClaim` (→ 837D, **priority**) — required: ICD-10 linkage diagnosis, KX modifier, tooth
  `bodySite` (ADA Universal Tooth Designation), dual CDT + CPT/HCPCS; carries `ODEInextricableLinkage`.
- `ODEInstitutionalClaim` (→ 837I) and `ODEProfessionalClaim` (→ 837P) — scaffold on the base.
  If institutional inpatient vs outpatient requireds diverge, **split into two profiles** rather
  than adding conditional invariants.
- `ODEClaimBundle` + `OperationDefinition` `$submit` (modeled on Da Vinci PAS `Claim/$submit`).
  Every instance SHALL populate `meta.profile`.

## Extensions

- `ODEBillingRail` — atomic (`value[x]` CodeableConcept from `ODEBillingRailVS`: dental-benefit |
  dental-for-medical | medical-professional | dme). **One definition**, contexts `ServiceRequest`,
  `Claim`, `Task` — referral carries the *anticipated* rail, the claim the *actual* rail.
- `ODEInextricableLinkage` — complex, a **sibling to Da Vinci CRD `coverage-information`** (same host
  ServiceRequest, same Coverage). Sub-extensions: `coveredServiceCategory` (from
  `ODEInextricableLinkageCategoryVS`), `linkedCoveredService`, `linkingDiagnosis`, `attestation`.
  Wired onto `ODEMedicalToDentalReferral` and `ODEOralClaim`.
- Invariant: `billingRail = dental-for-medical ⇒ ODEInextricableLinkage present`.

## Don't invent

Radiation dosimetry, the AI screening-result profile, and FDI ISO 3950 tooth numbering are
**deferred gaps** — leave them documented as gaps unless explicitly asked to model them. Where no
established code system exists, use `code.text`; never fabricate a coding.

## Terminology maintenance

`ODEInextricableLinkageCategory` tracks the CMS inextricable-linkage list at 42 CFR 411.15(i)(3).
It currently runs through dialysis/ESRD (added CY2025). **Update it when the annual PFS rule
expands the list.**

## Boundaries & related work

- **`ode-360x-adapter`** (separate repo) is the 360X⟷ODE bridge. It targets the *referral*
  contract (the CapabilityStatement / a generated OpenAPI). **Claims submission is NOT bridged by
  360X** — FHIR→837 is a separate translator. Coverage, billing-rail, and linkage **never travel in
  360X v2** — they are FHIR-only (lossless on FHIR, loss-noted outbound).
- Phased work plan: `docs/ODE-COW-CLAIMS-IMPLEMENTATION-PLAN.md`.

## First tasks

Read this file and `docs/ODE-COW-CLAIMS-IMPLEMENTATION-PLAN.md`, then work the plan phase by
phase. Small commits; run `sushi .` after each change; iterate against `output/qa.html`.
