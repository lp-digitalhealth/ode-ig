# Referral submission and response

> **Draft, proposed text.** The wording below is proposed for review by the IG authors. It describes what a referral recipient returns when a referral is submitted, and how the sender follows the referral afterwards. Items that are not decided are listed at the end.

## Summary

A sender submits a referral as a FHIR **transaction Bundle**. The recipient answers **at once** with the standard FHIR response to a transaction: a **`transaction-response` Bundle** if the referral was stored, or an **`OperationOutcome`** if it was not. Anything slower than storing the referral, such as retrieving supporting images from the sender, happens **after** that response and does not delay it.

## Submitting a referral

The sender POSTs a transaction Bundle to the recipient's endpoint. The endpoint, the events of interest and the content are agreed between the two parties beforehand; the [Clinical Order Workflows](https://hl7.org/fhir/uv/cow/) guide (COW) requires that agreement for any push-based exchange.

The Bundle carries the referral order and its coordination Task, which share the referral identifier (`urn:ohia:referral-id`), together with the supporting clinical resources the profile in use calls for:

| Resource | Role |
|---|---|
| `ServiceRequest` (one of the three directional ODE referral profiles) | The referral order: what is requested |
| `Task` (`ODEReferralTask`, based on the COW Coordination Task) | The coordination state of the referral |
| Supporting resources (for example `Condition`, `DocumentReference`) | The clinical context the receiving clinician needs |

Supporting content that is large, such as images, is not placed in the Bundle. The Bundle carries a pointer, and the recipient retrieves the content afterwards (see below).

## The response

| Outcome | HTTP status | Body |
|---|---|---|
| The referral was stored | `200` | A Bundle of type `transaction-response` with one entry for each entry of the request, in the same order |
| The body is not a usable transaction Bundle | `422` | An `OperationOutcome`; nothing is stored |
| The payload is larger than the recipient accepts | `413` | An `OperationOutcome`; nothing is stored |
| The recipient could not store the referral | `4xx` or `5xx`, as the failure warrants | An `OperationOutcome`; nothing is stored |

This is the response FHIR defines for a transaction: for a successful transaction the server **SHALL** return a Bundle of type `transaction-response` with one entry per request entry, in order, and for a failed transaction it returns a single `OperationOutcome` ([FHIR R4, HTTP: Batch/Transaction Response](https://hl7.org/fhir/R4/http.html#transaction-response)). The ODE recipient capability statement already declares the `transaction` interaction; this page states what comes back.

A recipient:

- **SHALL** answer a successful submission with the `transaction-response` and a failed one with an `OperationOutcome`, as above.
- **SHOULD** send the response as soon as the referral is stored, without waiting for any retrieval of supporting content.
- **SHOULD** tell the sender where to read the later outcome (see the next section), for example in a response header or a resource link, so a sender does not have to guess.

## After the response

**The referral's state.** The sender follows the referral through the coordination `Task`, found by the referral identifier. This is the COW pattern of a Task held at the fulfiller. The `Task.status` and `Task.businessStatus` change as the recipient works; results are collected in `Task.output`.

**Supporting content.** When the Bundle carries pointers to supporting content, the recipient retrieves it after the response, by the mechanism the parties agreed (for example an ARex pull). A slow or unreachable sender server therefore cannot make the original request time out. The outcome of the retrieval is recorded by the recipient and is separate from the success of storing the referral: a referral can be stored while its images were not yet retrieved or could not be retrieved.

## Why the response is not held back

During November 2026 testing, a recipient that finished the whole retrieval before replying left a sender waiting. The retrieval makes several calls to the sender's server, each of which can take up to a minute, so a typical client timeout expired and the sender saw no answer, although the referral had been stored. Answering first, with the standard response, removes that failure and lets any FHIR client library read the reply.

## Open items

These are not decided by this page and need the IG authors:

1. **Strength of the wording.** Whether "SHOULD answer without waiting" should be a stronger requirement, and whether the status codes for failures should be fixed or only the `OperationOutcome` body.
2. **A Bundle the recipient stores only in part.** One test-server case stores every entry except a `Subscription` that the underlying server refuses. FHIR's transaction rule is all-or-nothing, so the response then has fewer entries than the request. The guide should say whether a recipient may do this and how it reports it.
3. **Reporting the retrieval outcome to the sender.** This page says the outcome is recorded by the recipient. It does not yet define how the sender learns it in a standard way, for example through `Task.output`, a `Task.businessStatus` value or a linked resource.
4. **Where this belongs in the artifacts.** The statement should also appear in the `transaction` interaction documentation of the recipient capability statement (`ODEReferralRecipientServer`), which currently lives in the HL7 `ode` repository, not in this one.

## Sources

- FHIR R4, HTTP, [Batch/Transaction Response](https://hl7.org/fhir/R4/http.html#transaction-response).
- Clinical Order Workflows, [FHIR exchange architectures](https://hl7.org/fhir/uv/cow/fhir-exchange-architectures.html) (pre-coordination for push exchanges) and the Task guidance (the Coordination Task, `Task.output`).
