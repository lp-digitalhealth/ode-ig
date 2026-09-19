# ODE IG + 360X Adapter — Implementation Plan (COW basing, claims submission, EOB removal)

**Audience:** Claude Code, executing across two repos.
**Repos:**
- `ohia-fhirr4-scratchpad` — the ODE IG, under `/interfaces`, in four synchronized views.
- `ode-360x-adapter` — the 360X⟷ODE bridge: language-neutral `spec/` + Python (reference), .NET, Java.

**What this plan does:**
1. Delete the EOB profile (`ODEDentalClaim`) — it is incorrect.
2. Base the referral formally on HL7 **Clinical Order Workflows (COW)** (`hl7.fhir.uv.cow`, R4), replacing today's narrative-only COW claim.
3. Add a **claims-submission** interface (split `Claim` profiles + `$submit`) as a first-class part of the IG.
4. Add routing/linkage nuance to the referrals: `ODEBillingRail` + `ODEInextricableLinkage` extensions.
5. Keep all four IG views in parity, and propagate the referral-contract changes into the adapter `spec/` and its three implementations.

---

## 0. Governing rules (do not violate)

- **FSH is the source of truth.** `interfaces/fsh/` is the only view promoted to `staging-transition/`. Every change made there MUST be mirrored to `interfaces/openapi.yaml`, the regenerated `interfaces/ode-api-swagger.html`, `interfaces/fhir-interfaces.md`, and the crosswalk `interfaces/INTERFACE-VIEWS.md`, plus a `interfaces/CHANGELOG.md` entry. See `.cursor/rules/fsh-authoring.mdc`.
- **Cross-repo contract drift is a real risk.** The ODE Native OpenAPI exists in BOTH repos: `ohia-fhirr4-scratchpad/interfaces/openapi.yaml` (IG view) and `ode-360x-adapter/spec/api/ode-openapi.yaml` (the contract the adapter targets). Referral-contract changes must land in both.
- **Adapter parity:** `spec/` is language-neutral truth; Python is the reference; `.NET` and Java mirror it. The normative mapping artifact is `spec/mapping/360x-cow-crosswalk.md`.
- **IP boundaries:** CDT is ADA-licensed — reference the system URI, never enumerate codes. X12 837 mapping/code systems are X12 IP — do not reproduce; reference. (Claims submission here is FHIR profiling only; the FHIR→837 crosswalk itself is out of scope for this plan.)
- **Terminology namespace:** CodeSystems/ValueSets publish under `http://ohia-codes.org`; profiles/extensions under the IG canonical `https://oralhealthalliance.net/fhir`. The SUSHI mismatch warning is intentional — do not "fix" it.

---

## 1. Decisions to resolve first

- **D1 — ServiceRequest parent (COW vs US Core).** FHIR is single-inheritance. **Recommended default:** keep the three directional referral ServiceRequests parented on **US Core ServiceRequest** (preserve the data contract + directional coding slices exactly), and carry **COW conformance on the Task**, where COW's model actually lives. Alternative: reparent the ServiceRequests onto COW's request profile and re-declare US Core must-supports by hand (only if the ServiceRequests themselves must assert COW conformance). *This plan assumes the recommended default; flip Phase 1.2 if D1 changes.*
- **D2 — EOB removal is total.** The IG's claims interface becomes **submission only**; there is no claims-sharing/EOB profile going forward. (Confirmed.)
- **D3 — Claims submission is out of 360X adapter scope.** 360X carries referrals, not claims. The adapter is untouched by the claims work except to confirm the boundary in its docs. FHIR→837 generation is a separate translator, not this bridge.
- **D4 — COW ballot pin.** COW is at `1.0.0-ballot`. The adapter already pins `COW_VERSION = "1.0.0-ballot"`. Accept the ballot dependency for the IG too (matches the adapter), or hold — decide before Phase 1.2.

---

## 2. Phase 1 — IG FSH source of truth (`ohia-fhirr4-scratchpad/interfaces/fsh/`)

Do this phase first and get a clean `sushi .` before touching any other view.

### 1.1 Delete the EOB
- Remove `input/fsh/profiles/claims.fsh` (the `ODEDentalClaim` profile).
- In `sushi-config.yaml`, remove the `hl7.fhir.us.carin-bb` dependency (nothing else uses it — referrals reference PAS ClaimResponse, not C4BB; confirm with a grep before removing).
- In `input/fsh/terminology/terminology.fsh`, check `ODEClaimCareTeamRoleVS`: if it was only used by the EOB, either delete it or (preferred) carry it forward for reuse by the claims-submission care-team slicing in 1.3.
- Grep the whole `fsh/` tree for `ode-dental-claim`, `ODEDentalClaim`, `c4bbEOBProf`, `carin-bb` and clear stragglers (aliases, capability statements, examples).

### 1.2 Base the referral on COW
- **First:** pull the COW IG artifacts (`hl7.fhir.uv.cow`, R4, 1.0.0-ballot) and identify the exact **Task profile** canonical (COW's Task-at-Placer / Task-at-Fulfiller pattern) and its Task.status / lifecycle constraints. Record the canonical in `aliases.fsh` (e.g. `$cowTask`).
- Add `hl7.fhir.uv.cow: 1.0.0-ballot` to `sushi-config.yaml` dependencies.
- In `input/fsh/profiles/workflow.fsh`, reparent `ODEReferralTask` from base `Task` to the COW Task profile. Reconcile:
  - `status` — confirm `ODEReferralTaskStatusVS` is a subset of / compatible with COW's allowed Task.status values.
  - `businessStatus` — `ODEReferralSubStatusVS` (the 360X-driven sub-status axis) layered on top; confirm COW permits it.
  - Retain the referral-id identifier slice, `focus`→`ODEReferralServiceRequest`, `for`→US Core Patient, `input`/`output`/`note` COW scope.
  - Update the Description: COW is now a **structural parent**, not just a narrative claim.
- ServiceRequests: per **D1 (default)**, leave the three directional profiles + `ODEReferralServiceRequest` base parented on US Core ServiceRequest. Add a conformance note that the referral workflow conforms to COW via the Task. (If D1 flips, reparent here instead.)
- Sanity-check the `Request.status` (revoked) vs `Task.status` (cancelled) relationship against COW's lifecycle — this currently differs between the adapter's `transactions.md` and `360x-cow-crosswalk.md`; align the IG to COW and note it for Phase 3.

### 1.3 Add the claims-submission interface
Create `input/fsh/profiles/claims-submission.fsh` with the split design (learned from CARIN — structural requireds per type, not conditional invariants on one union profile):
- `ODEClaimBase` (abstract) — universal core: `patient`, `insurance`→Coverage, billing `provider`, `insurer`/payer, ≥1 `diagnosis`, ≥1 service `item`; and the **shared slice definitions**: `careTeam.role` (reuse the care-team-role VS), `supportingInfo.category`, `diagnosis.type`. Carry the `ODEBillingRail` extension (actual rail) here as the routing determinant.
- `ODEOralClaim` (→ 837D) — **priority profile.** Required: ICD-10 linkage `diagnosis`, `item.modifier` = KX, tooth `item.bodySite` (ADA Universal Tooth Designation, reuse `ODETooth`), dual CDT + CPT/HCPCS on `item.productOrService`. Carries `ODEInextricableLinkage`.
- `ODEInstitutionalClaim` (→ 837I) — required: type-of-bill, ≥1 revenue-coded line, admission/discharge data. (Scaffold; flesh later. If inpatient vs outpatient requireds diverge, split into two profiles rather than adding conditional invariants — the CARIN lesson.)
- `ODEProfessionalClaim` (→ 837P) — required: rendering provider in `careTeam`, place of service. (Scaffold.)
- `ODEClaimBundle` — the submission Bundle profile whose contained Claim SHALL conform to one of the three; SHALL populate `meta.profile`.
- `$submit` — `OperationDefinition/ode-claim-submit` (model on Da Vinci PAS `Claim/$submit`).
- Set `status`/`use` appropriately for a real submission (not the EOB's draft/queued freeze).

### 1.4 Extensions + terminology
Create from the already-drafted reference FSH (`ode-referral-linkage.fsh`), refactored to repo conventions (global aliases, `ohia-codes.org` terminology namespace, `$uc*` US Core aliases):
- `input/fsh/extensions/ode-billing-rail.fsh` — atomic extension, `value[x]` CodeableConcept from `ODEBillingRailVS`; contexts: `ServiceRequest`, `Claim`, `Task`. One definition, used on both referral (anticipated rail) and claim (actual rail).
- `input/fsh/extensions/ode-inextricable-linkage.fsh` — complex extension (sub-extensions: `coveredServiceCategory` 1..1 from `ODEInextricableLinkageCategoryVS`; `linkedCoveredService` 0..1 Reference(ServiceRequest|Procedure|CarePlan); `linkingDiagnosis` 1..* Reference(Condition); `attestation` 1..1 from `ODELinkageAttestationVS`); contexts: `ServiceRequest`, `Claim`. Designed as a **sibling to Da Vinci CRD `coverage-information`** — same host, same Coverage, adds the linkage attestation CRD does not express.
- In `input/fsh/terminology/terminology.fsh`, add (under `http://ohia-codes.org`): `ODEBillingRailCS/VS` {dental-benefit, dental-for-medical, medical-professional, dme}; `ODEInextricableLinkageCategoryCS/VS` (the CMS-codified 42 CFR 411.15(i)(3) scenarios — organ-transplant, cardiac-valve, head-neck-cancer, chemotherapy, car-t, antiresorptive-cancer, dialysis-esrd; **maintain as CMS expands the annual PFS list**); `ODELinkageAttestationCS/VS` {asserted, not-asserted}.

### 1.5 Wire the extensions onto the referral profiles
In `input/fsh/profiles/referrals.fsh` (keep the existing directional coding slices intact):
- All three directional profiles: add `ODEBillingRail` (1..1 MS) = anticipated rail.
- `ODEMedicalToDentalReferral`: add `ODEInextricableLinkage` (0..* MS); add invariant `rail = dental-for-medical ⇒ linkage present`. This is the burden-relief case (the referral is the evidence that unlocks dental-for-medical payment).
- `ODEDentalToMedicalReferral`: rail typically `medical-professional` or `dme` (e.g. UC05 sleep-apnea → oral appliance is a DME claim, not dental). Confirm the value set covers it.
- `ODEDentalToDentalReferral`: rail typically `dental-benefit`.
- Optionally reference CRD `coverage-information` as an allowed sibling extension slice (requires the CRD dependency; optional).

### 1.6 Capabilities
In `input/fsh/capabilities/capabilitystatements.fsh`: remove the EOB/claims-sharing interaction; add the `Claim` resource + `$submit` operation to the appropriate server/client statement.

### 1.7 Build
`cd interfaces/fsh && sushi .` — resolve errors; the only expected warning is the intentional terminology-canonical mismatch.

---

## 3. Phase 2 — Mirror to the other three IG views + crosswalk + changelog

Only after Phase 1 builds clean.

- `interfaces/openapi.yaml`:
  - Remove the **Claims Sharing** tag and the EOB schema.
  - Add a **Claims Submission** tag; add `Claim` schemas (base + oral/institutional/professional) with `x-fhir-profile`; add the `$submit` path.
  - On the Referral/Task schemas: reflect COW basing and the new `ODEBillingRail` / `ODEInextricableLinkage` extension fields.
- `interfaces/ode-api-swagger.html`: regenerate from the updated `openapi.yaml`.
- `interfaces/fhir-interfaces.md`: remove the EOB narrative; add the claims-submission section; update the referral section for COW; document the extensions.
- `interfaces/INTERFACE-VIEWS.md`: update crosswalk rows (drop EOB, add claim profiles + extensions + COW).
- `interfaces/CHANGELOG.md`: entries — "Removed EOB (ODEDentalClaim)", "Based referral Task on COW", "Added claims-submission profiles + $submit", "Added billing-rail + inextricable-linkage extensions".

---

## 4. Phase 3 — Adapter spec (`ode-360x-adapter/spec/`)

- `spec/api/ode-openapi.yaml`: sync the **referral** contract with the IG's updated `openapi.yaml` (COW Task, the two extensions). Do **not** add claims — record a one-line scope note that claims submission is out of 360X bridge scope (per D3).
- `spec/mapping/360x-cow-crosswalk.md` (**normative — re-validate carefully**):
  - Confirm the `Task.status` / `Task.businessStatus` rows still hold against the now-formal COW Task profile (the crosswalk was written to "COW scoped to what 360X supports" — verify nothing in the formal COW parent breaks that scoping).
  - Reconcile `Request.status = revoked` vs `Task.status = cancelled` for PCC-58 against COW's lifecycle (currently inconsistent with `transactions.md`).
  - Add a row/section: `ODEBillingRail` and `ODEInextricableLinkage` are **FHIR-only** — lossless on the FHIR (COW) side, **not carried in 360X v2** (insurance/coverage never travels in v2). Outbound: emit a loss note. Inbound (PCC-55): these cannot be reliably populated from a 360X CDA — mark as a gap; they are a FHIR-native referral advantage. State this explicitly.
- `spec/mapping/transactions.md`: align the state table with the reconciled crosswalk.
- `spec/mapping/attachments.md`: no change for claims (out of scope); confirm linkage supporting docs don't alter the existing attachment convention.

---

## 5. Phase 4 — Adapter implementations (Python reference → .NET → Java)

Drive everything from the reconciled `spec/`. Python first (reference), then port.

**Python (`ode-360x-adapter/python/ode_adapter/`):**
- `config.py`: confirm/annotate `COW_VERSION = "1.0.0-ballot"` now matches a **formal** IG dependency (previously the IG only referenced COW narratively). No functional change expected, but update the comment.
- `state_machine.py` / `engine.py`: re-verify `INBOUND_TX_TO_TASK_STATUS` and `task_to_360x()` against the reconciled COW Task lifecycle (esp. revoked/cancelled). Add explicit **pass-through preservation** of the two new extensions on FHIR-side resources, and a **loss note** when an outbound 360X transaction drops them.
- No claims handling (D3).
- Tests: extend for the extension pass-through + loss-note behavior; keep the 6 Connectathon scenarios green. (Note the broader `TODO.md` v0.1→v1.0 path is separate from this change.)

**.NET and Java:** mirror the Python reference behavior against the same `spec/`. Keep the three in parity.

---

## 6. Sequencing & acceptance

1. Resolve D1 and D4.
2. Phase 1 (IG FSH) → `sushi .` clean.
3. Phase 2 (mirror three views + crosswalk + changelog) → INTERFACE-VIEWS parity check passes.
4. Phase 3 (adapter spec) → crosswalk re-validated; cross-repo openapi consistent for the referral contract.
5. Phase 4 (Python → .NET → Java) → adapter tests green in CI.

**Definition of done:** EOB gone from all views; `ODEReferralTask` structurally inherits COW; claims-submission profiles + `$submit` present and building; both extensions defined once and wired to referrals + claims; all four IG views + both repos' OpenAPI consistent; adapter crosswalk re-validated and the FHIR-only extension behavior documented and implemented in all three languages.

---

## 7. Watch items / known gaps

- **COW Task profile specifics** — must be pulled from the COW IG before 1.2; the biggest single ripple.
- **D1 ServiceRequest parent** — default is US-Core-parent + COW-on-Task; flip only deliberately.
- **Coverage/linkage cannot ride 360X v2** — reinforces the FHIR-native referral value story; make sure the loss behavior is explicit, not silent.
- **`revoked` vs `cancelled`** — reconcile once, in the crosswalk, then propagate.
- **Cross-repo OpenAPI drift** — the referral contract lives in two repos; a change in one without the other is the most likely regression.
- **Institutional inpatient/outpatient** — if requireds diverge, split the profile; do not reach for conditional invariants.
- **Claims submission ≠ 360X** — keep the boundary loud so no one wires claims through the bridge.
