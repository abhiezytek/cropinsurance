# Enrollment, Payment & Policy Issuance Design (Indian Crop Insurance / PMFBY)

This document describes the backend design for the **Enrollment → Payment → Policy Issuance** flow of an Indian crop insurance system (PMFBY-style). It is meant for developers working on the .NET Core + React + SQL implementation.

---

## 1. Tech Stack & Scope

- Backend: ASP.NET Core Web API, EF Core, C#
- Frontend: React
- Database: SQL Server (or other relational)
- This document covers:
  - Pre-season configuration
  - Enrollment (direct API + bank upload)
  - Payment & bank integration
  - Policy issuance batch

Later phases (claims, reporting) will build on these entities.

---

## 2. Core Domain Entities

### 2.1 Master / Configuration

**NotifiedCropConfig**

Per State/District/Unit/Season/Year/Crop, holds PMFBY configuration: [PMFBY guidelines and calculators require this kind of data][web:19][web:77][web:86].

- StateId, DistrictId, BlockId, VillageId
- InsuranceUnitCode
- SeasonCode, SeasonYear
- CropId
- SumInsuredPerHa
- ActuarialPremiumRatePercent
- FarmerMaxRatePercent (e.g., 2 / 1.5 / 5)
- CentreSharePercentOfBalance
- StateSharePercentOfBalance

This table is read by the rating module.

### 2.2 Enrollment Entity Graph

- **Farmer**
  - FarmerId (PK)
  - AadhaarNumber, FullName, MobileNumber, CategoryCode, etc.
  - Geo/address fields
  - Bank/KCC details

- **LandParcel**
  - LandParcelId (PK)
  - FarmerId (FK)
  - Geo fields + SurveyNumber, TotalAreaHa, OwnershipType
  - InsuranceUnitCode

- **EnrollmentApplication**
  - EnrollmentApplicationId (PK)
  - FarmerId (FK)
  - SeasonCode, SeasonYear, SchemeCode, InsurerId
  - Geo (State/District/Block/Village)
  - ChannelCode
  - IsLoanee, LoanAccountNumber
  - ApplicationStatus
  - PaymentStatus
  - ApplicationDate, PremiumDebitDate, etc.
  - TotalSumInsured, TotalGrossPremium
  - FarmerPremiumShare, StateSubsidyShare, CentreSubsidyShare

- **EnrollmentCropLine**
  - EnrollmentCropLineId (PK)
  - EnrollmentApplicationId (FK)
  - LandParcelId (FK)
  - CropId, CropVarietyId
  - InsuranceUnitCode
  - SowingDate
  - InsuredAreaHa
  - SumInsuredPerHa, SumInsured
  - PremiumRatePercent
  - GrossPremium
  - FarmerPremiumShare, StateSubsidyShare, CentreSubsidyShare

Relations:

- 1 Farmer ↔ * LandParcels
- 1 Farmer ↔ * EnrollmentApplications
- 1 EnrollmentApplication ↔ * EnrollmentCropLines
- 1 LandParcel ↔ * EnrollmentCropLines

### 2.3 Bank Upload & Payment Entities

**BankUploadBatch**

- BankUploadBatchId (PK)
- BankId, BranchId
- SeasonCode, SeasonYear, SchemeCode, InsurerId
- FileName, UploadedBy, UploadedOn
- Status: Uploaded | Validating | Validated | PartiallyValid | Failed | Processed

**BankUploadRow**

- BankUploadRowId (PK)
- BankUploadBatchId (FK)
- Raw columns from bank file (farmer, land, crop, area, etc.)
- MappedJson (optional serialized DTO)
- RowStatus: Pending | Valid | Invalid | Processed
- ErrorMessages (string / JSON)

**BankPaymentBatch**

Represents wallet/challan from bank to insurer as per PMFBY remittance user manuals. [web:36][web:87][web:38]

- BankPaymentBatchId (PK)
- BankId, BranchId
- SeasonCode, SeasonYear, SchemeCode, InsurerId
- BatchNumber / ChallanNumber
- TotalAttachedPremium
- RemittedAmount
- UtrNumber, UtrDate
- Status: Created | Remitted | Confirmed | Failed

**BankPaymentAttachment**

- BankPaymentAttachmentId (PK)
- BankPaymentBatchId (FK)
- EnrollmentApplicationId (FK)
- AttachedPremiumAmount

### 2.4 Policy & Issuance

**Policy**

- PolicyId (PK)
- PolicyNumber
- EnrollmentApplicationId (FK)
- FarmerId (FK)
- SchemeCode, SeasonCode, SeasonYear, InsurerId
- TotalSumInsured, TotalPremium
- FarmerPremiumShare, GovtSubsidyTotal
- PolicyStatus

**PolicyIssuanceBatch**

- PolicyIssuanceBatchId (PK)
- BatchNumber
- SchemeCode, SeasonCode, SeasonYear, InsurerId
- CreatedOn, CreatedBy

**PolicyIssuanceBatchItem**

- PolicyIssuanceBatchItemId (PK)
- PolicyIssuanceBatchId (FK)
- EnrollmentApplicationId (FK)
- PolicyId (FK)

---

## 3. Key Services & Interfaces

### 3.1 Validation Pipeline

`IEnrollmentValidator` orchestrates a set of validators over `CreateEnrollmentApplicationRequest`.

Sub-interfaces:

- `IFarmerValidator`
  - Aadhaar/name/mobile, minimum KYC, duplicate farmer/season rules.
- `ILandAndCropValidator`
  - Land presence (LandParcelId or SurveyNumber + area + ownership)
  - InsuredAreaHa > 0
  - InsuredAreaHa per parcel ≤ TotalParcelAreaHa (including mixed crops)
  - Crop notified for State/District/Unit/Season/Year (via NotifiedCropConfig or equivalent). [web:19][web:69][web:71]
- `ISchemeAndPremiumValidator`
  - PMFBY scheme rules (loanee vs non-loanee, document flags, etc.). [web:19][web:29]
- `IPaymentAndCutoffValidator`
  - ApplicationDate and PremiumDebitDate within notified cut-off windows. [web:19][web:36]

Validation output: `ValidationResultModel` with `FieldErrors` and `GlobalErrors`.

### 3.2 Rating Module

`IEnrollmentRatingService` computes sum insured and premium splits per crop line using PMFBY formulas. [web:19][web:22][web:77][web:79]

For each validated `EnrollmentCropLineDto`:

1. Load `NotifiedCropConfig`.
2. SumInsured = SumInsuredPerHa × InsuredAreaHa
3. ActuarialRate = ActuarialPremiumRatePercent
4. FarmerRate = min(ActuarialRate, FarmerMaxRatePercent)
5. GrossPremium = SumInsured × ActuarialRate / 100
6. FarmerPremium = SumInsured × FarmerRate / 100
7. Balance = GrossPremium – FarmerPremium
8. CentreSubsidy = Balance × CentreSharePercentOfBalance / 100
9. StateSubsidy = Balance × StateSharePercentOfBalance / 100

Returns `RatedEnrollmentResult` containing line-level and total values.

### 3.3 Enrollment Service

`IEnrollmentService.CreateEnrollmentAsync(CreateEnrollmentApplicationRequest)`:

1. Calls `IEnrollmentValidator`.
2. Calls `IEnrollmentRatingService`.
3. Creates/loads `Farmer`.
4. Creates/reuses `LandParcel` records.
5. Creates `EnrollmentApplication` with premium totals and initial statuses:
   - `ApplicationStatus = "Validated"`
   - `PaymentStatus = "Unpaid"` or `"PendingConfirmation"`
6. Creates `EnrollmentCropLine` rows with rated financial fields.

---

## 4. APIs & Workflows

### 4.1 Enrollment API (Direct)

`POST /api/enrollments`

- Request: `CreateEnrollmentApplicationRequest`
  - Header: season, scheme, insurer, geo, channel, loanee flag, dates.
  - FarmerDto: Aadhaar, name, mobile, bank details.
  - CropLines: land info, crop, area, insurance unit, sowing date.
- Processing:
  - Controller → `IEnrollmentValidator` → `IEnrollmentRatingService` → `IEnrollmentService.CreateEnrollmentAsync`.
- Response: `CreateEnrollmentApplicationResponse`
  - EnrollmentApplicationId
  - ApplicationStatus
  - Premium totals (sum insured, gross premium, farmer share, subsidies).

### 4.2 Bank Upload Workflow

1. `POST /api/bank-uploads`
   - Input: file + bank/branch/season/scheme/insurer params.
   - Creates `BankUploadBatch`, rows in `BankUploadRow`.
   - For each row, maps to `CreateEnrollmentApplicationRequest` and validates.
   - `BankUploadRow.RowStatus` = Valid/Invalid, `ErrorMessages` populated.

2. `GET /api/bank-uploads/{batchId}`
   - View rows and errors for branch to fix.

3. `POST /api/bank-uploads/{batchId}/process-valid-rows`
   - For each `RowStatus = Valid`:
     - Rebuild request and call `IEnrollmentService.CreateEnrollmentAsync`.
     - Store created `EnrollmentApplicationId` onto `BankUploadRow`.
     - Set `RowStatus = Processed`.
   - Set `BankUploadBatch.Status = Processed`.

This mirrors PMFBY/NCIP bank utility behavior but within your system. [web:36][web:46][web:47]

### 4.3 Payment & Bank Integration

**Create wallet/challan**

`POST /api/payments/batches`

- Input: bankId, branchId, season, year, scheme, insurer, expectedTotalPremium.
- Output: BankPaymentBatch with `Status = "Created"`.

**Attach applications**

`POST /api/payments/batches/{batchId}/attachments`

- Input: list of EnrollmentApplicationIds.
- Adds `BankPaymentAttachment` rows, updates `TotalAttachedPremium`.
- Applications remain `PaymentStatus = "PendingRemittance"`.

**Remittance / UTR entry**

`POST /api/payments/batches/{batchId}/remittance`

- Input: utrNumber, utrDate, remittedAmount.
- Verifies remittedAmount ≈ TotalAttachedPremium and date within cut-off. [web:36][web:75]
- Sets `BankPaymentBatch.Status = "Remitted"`.
- Updates attached `EnrollmentApplication.PaymentStatus = "Paid"` and stores UTR info.

**Optional reconciliation API**

`POST /api/payments/reconciliation`

- Accepts file/JSON with (ExternalApplicationRef, amount, dates, UTR).
- Matches to `EnrollmentApplication`, checks amounts, sets `PaymentStatus = "Paid"` or reports mismatches.

### 4.4 Policy Issuance Batch

`POST /api/policy-issuance/run`

Calls `IPolicyIssuanceService.IssuePoliciesAsync(schemeCode, seasonCode, seasonYear, insurerId)` which:

1. Queries eligible applications:
   - ApplicationStatus = "Validated"
   - PaymentStatus = "Paid"
   - No existing Policy
2. Creates `PolicyIssuanceBatch`.
3. For each eligible app:
   - Generates policy number via `IPolicyNumberGenerator`.
   - Creates `Policy`.
   - Marks application as `PolicyIssued`.
   - Records mapping in `PolicyIssuanceBatchItem`.

Returns counts of eligible, issued, failed.

---

## 5. High-level Component Interaction Diagram (Text)

```text
[Bank / Channel] 
    |
    v
 (A) Enrollment Entry
    |
    v
+----------------------------+
| EnrollmentsController      |
|   -> IEnrollmentValidator  |
|   -> IEnrollmentRatingSvc  |
|   -> IEnrollmentService    |
+----------------------------+
            |
            v
 ApplicationDbContext
 (Farmer, LandParcel,
  EnrollmentApplication,
  EnrollmentCropLine)

[Bank User] 
    |
    v
 (B) Bank Upload & Payment
    |
    v
 BankUploadsController
 PaymentController
    |
    v
 BankUploadBatch/Row,
 BankPaymentBatch/Attachment,
 EnrollmentApplication.PaymentStatus

[Insurer Ops / Scheduler]
    |
    v
 (C) Policy Issuance
    |
    v
 PolicyIssuanceController
   -> PolicyIssuanceService
   -> IPolicyNumberGenerator
    |
    v
 Policy, PolicyIssuanceBatch, PolicyIssuanceBatchItem
```

This shows how the Enrollment + Payment + Policy Issuance pieces connect around the shared domain model.

---

## 6. Next Steps for Implementation

- Scaffold EF Core entities and DbContext for the tables listed.
- Implement the validation interfaces and pipeline.
- Implement `IEnrollmentRatingService` using `NotifiedCropConfig`.
- Implement `IEnrollmentService.CreateEnrollmentAsync`.
- Build APIs:
  - `/api/enrollments`
  - `/api/bank-uploads` (+ process route)
  - `/api/payments/*`
  - `/api/policy-issuance/run`

Once these are stable, the next domain to design is the **claims engine** that uses `Policy` + yield/weather data at the insurance-unit level.
