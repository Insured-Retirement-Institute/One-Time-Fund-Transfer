# Fund Transfer API

## Overview
This document defines the **Fund Transfer** transaction as part of the broader IRI Digital First transformation effort. The transaction supports fund-to-fund and segment-based value transfers on annuity and life insurance policies, enabling carriers, distributors, and solution providers to reallocate policy investments through a modern, standardized interface.

The initiative modernizes legacy XML/SOAP-based In-Force Transactions (IFT) into RESTful APIs for secure, scalable, and interoperable processing. It leverages industry standards and provides a unified approach for carriers, distributors, and solution providers.

---

### Business Case
#### Problem Statement
- Legacy fund transfer processing is commonly exposed through tightly coupled XML/SOAP services and carrier‑specific file formats.
- Inconsistent request/response structures across carriers increase onboarding time, integration cost, and operational risk.
- Distributors and solution providers require a predictable, standards‑aligned interface to submit transfers and track processing status.

#### Objectives
- Provide a consistent, standards‑based API for submitting fund transfers across carriers and product types.
- Enable pre‑defined request structures for different transfer styles (amount, percent, full rebalance) to reduce downstream failures.
- Support asynchronous processing with a lifecycle endpoint for status retrieval.
- Deliver predictable JSON payloads aligned with IRI Digital First architecture and OpenAPI 3.1.x.

### Key Features
- RESTful API with JSON input/output.
- Asynchronous processing model with status polling via a dedicated request lifecycle endpoint.
- Unified request schema with correlationId and firm/participant identifiers.
- Supports fund transfers using **specified funds** and **specified segments** (where applicable).
- Standardized response structures: request acknowledgment, processing status, and effective date confirmation.
- Alignment with IRI OpenAPI 3.1.x documentation.
- Conditional validation based on transfer type and amount method.
- Data Dictionary for field-level definitions and code lists.

---

## Endpoints

#### 1. Submit Fund Transfer
- **Method:** POST
- **Path:** `/v1/policies/{policyNumber}/fund-transfers`
- **Purpose:** Submits a fund transfer transaction for a policy, supporting source and destination fund or segment allocations.

#### 2. Retrieve Fund Transfer Request Status (Async Lifecycle)
- **Method:** GET
- **Path:** `/v1/policies/{policyNumber}/fund-transfers/requests/{requestId}`
- **Purpose:** Retrieves the current processing status of a fund transfer request submitted asynchronously.

---

## Schema Overview
The schema generally includes:

- **Root Attributes:** effectiveDate, allocationOption, cusip, nsccParticipantId
- **transactionAmounts:** amountType (FULL_REBALANCE, FUND_BALANCE_TRANSFER, PERCENT_OF_CONTRACT_VALUE, SPECIFIED_AMOUNT)
- **funds:** transferFromFunds and transferToFunds arrays, each containing fund-level and optional segment-level details (fundId, assetClass, investmentType, currentRate, maturityDate, fundSegments)
- **fundSegments:** segmentId, requestedAmount, requestedPercentage — with strict mutual-exclusivity rules based on transfer type
- **arrangement:** productCode, arrangementType, arrangementSubType, startDate, endDate, modalAmount, sourceTransferAmountType, destinationTransferAmountType, arrangementSource, arrangementDestination — used for systematic or recurring transfer programs
- **investProduct:** rateLockInfo, isLockedIn — captures rate-lock details for applicable investment products
- **auditTransSummation:** auditTotalType, auditTotal, correlationGuid, correlationIdState — supports audit and reconciliation workflows
- **Producer:** producerNumber, npn, crdNumber — identifies the advisor or producer associated with the transaction
- **Parties:** individual/entity identity, relationships (owner, jointOwner, annuitant, jointAnnuitant, primaryBeneficiary, contingentBeneficiary)

Each fund transfer request delivers structured acknowledgment and status information supporting the corresponding transfer workflow.

---

## Response Schema Overview

All API operations—across synchronous and asynchronous processing models—adhere to a unified **Error schema**. This ensures consistent error handling, predictable integration behavior, and standardized troubleshooting across all transaction types.

## Success Response Expectations

### Asynchronous APIs
- Return **HTTP 202** upon successful acceptance of the fund transfer request for processing.
- The response includes a `requestId` and `status` field. Consumers should poll the status endpoint using the `requestId` to determine final processing outcome.

### Status Polling
- Return **HTTP 200** with a `GetFundTransferRequestStatus` payload when retrieving the processing status of a submitted request.

---

## Standard Error Schema
Every error response—regardless of transaction type—includes:
- An HTTP status code in the **400–599** range
- A structured and validated **error code**
- A **timestamp** of when the error was generated
- A safe, user-friendly **Message**
- A **field-level** or **rule-level** error collections

---

## Key Fields

| Field | Description |
|-------|-------------|
| **correlationId** | Carries forward the inbound request's correlation ID header to enable end-to-end traceability. |
| **httpStatus** | Numeric HTTP status code (400–599) representing the type and severity of the failure. |
| **code** | Structured identifier in the enforced format: `domain.category.subcategory`. Enables machine-readable error handling. |
| **message** | User‑friendly explanation, safe to display in portals or consumer‑facing applications. |
| **validationErrors** | Array describing domain/business rule violations; each entry requires its own code and message. |
| **requestId** | Unique identifier assigned to an accepted fund transfer request and used for lifecycle tracking. |
| **associatedFirmId** | Firm identifier supplied by the caller and propagated throughout processing and event notifications. |
| **nsccParticipantId** | NSCC participant identifier associated with the transaction. |
---

## Supported HTTP Error Codes

| Status Code | Description |
|-------------|-------------|
| `400` | Bad Request |
| `401` | Unauthorized |
| `403` | Forbidden |
| `404` | Not Found |
| `405` | Method Not Allowed |
| `406` | Not Acceptable |
| `409` | Conflict — a transaction for this policy is already in progress or a duplicate request has been detected |
| `413` | Payload Too Large |
| `415` | Unsupported Media Type |
| `422` | Unprocessable Entity |
| `429` | Too Many Requests |
| `500` | Internal Server Error |
| `502` | Bad Gateway |
| `503` | Service Unavailable |
| `504` | Gateway Timeout |

---
# Day‑2 Asynchronous Processing

Fund Transfer transactions use an asynchronous processing model.
Upon successful validation, the API returns HTTP 202 Accepted and
provides a Location header and requestId for lifecycle tracking.

Acceptance of a request does not indicate final transaction completion.
Final transaction processing occurs asynchronously in downstream systems.

## Delivery Model
Day‑2 confirmation events are published through:
- SAP Advanced Event Mesh
- Solace Event Broker

Consumers receive notifications through topic-based subscriptions.
Day‑2 confirmations are event-driven only.

## Day‑2 Confirmation Event
Fund Transfer processing outcomes are communicated through the
Day2FundTransferConfirmationEvent.

Supported outcomes include:
- SUCCESS
- SUCCESS_WITH_INFO
- FAILURE

Each event includes:
- requestId
- correlationId
- associatedFirmId
- policyNumber
- eventId
- eventType
- eventTimestamp
- transactionType
- status
- transExeDate
- transExeTime

to support end-to-end traceability.

## Event Schema
The canonical event includes:

### Event Metadata
- eventType
- eventId
- eventTimestamp

### Transaction Information
- requestId
- policyNumber
- transactionType

### Processing Outcome
- status
- message
- effectiveDate

### Producer and Participant Information
- npn
- nsccParticipantId

### Execution Details
- transExeDate
- transExeTime

Supported transactionType value:
- FUND_TRANSFERS

Supported eventType value:
- DAY2_CONFIRM

## Status Visibility
The lifecycle endpoint:
GET /v1/policies/{policyNumber}/fund-transfers/requests/{requestId}
provides operational visibility into transaction progress.

The lifecycle endpoint does not replace Day‑2 event delivery as the
authoritative source for final transaction outcomes.

---

## Purpose & Benefits
This standardized error structure ensures:
- A **predictable experience** across all APIs (synchronous + asynchronous)
- Clear differentiation between **developer diagnostics** and **user-safe messages**
- Enhanced **traceability** for carriers, distributors, and integrators
- Support for granular **validation feedback** and complex business rule logic
- Easier **monitoring, logging, and cross-system troubleshooting**

---

## OpenAPI Specs
Unified Swagger documentation for this fund transfer endpoint is available in the `openapi-specs/` folder.

---

## Change Submissions and Reporting Issues
- Use the **Issues** tab of the repository to report bugs or enhancement requests.
- **Security issues** should be reported directly to Katherine Dease at **kdease@irionline.org**.
- Follow the standards governance workflow on the main page for contribution guidelines.

---

## Code of Conduct
Refer to the repository's **Code of Conduct** and **Style Guide** for contribution standards.

---

## How to Contribute
- Fork the repository and submit pull requests.
- Report issues using the **Issues** tab.
- Join working groups: **hpikus@irionline.org**.

---

## Business Owners
- **Carrier Business Owner:** digitalfirst@irionline.org
- **Distributor Business Owner:** [contact]
- **Solution Provider Business Owner:** [contact]

---

## Versioning
## v1.4.0 Highlights

- **Async Processing:** Added Day-2 confirmation event schema (`Day2FundTransferConfirmationEvent`).
- **Validation Enhancements:** Updated conditional business-rule validation for `transactionAmounts.amountType`, `funds.transferFromFunds[].fundSegments[]`, and `funds.transferToFunds[].fundSegments[]`.
Moved the validation logic from `TransactionAmounts.requestedAmount` and `TransactionAmounts.requestedPercentage` to `FundSegment.requestedAmount` and `FundSegment.requestedPercentage`. And removed `TransactionAmounts.requestedAmount` and `TransactionAmounts.requestedPercentage`.
- **Party Model Standardization:** Standardized `IndividualIdentity.type`, `EntityIdentity.type`, and `PartyRelationship.relationships[]` enum values to Screaming Snake case, naming conventions.
Removed paymentForm, allocationPercentage fields.
- **Required Field Changes:** Added mandatory fields `FundTransferRequest.nsccParticipantId`, `FundTransferRequest.producer`, `FundTransferRequest.allocationOption`, `Producer.npn`, `correlationId` and `associatedFirmId`.
- **Documentation Improvements:** Updated descriptions, examples, and business-rule documentation across `FundTransferRequest`, `FundSegment`, `Party`, `Producer`, and `Error` schemas.
---
