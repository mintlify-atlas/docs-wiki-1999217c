# Supplementary Modules — Research Notes

> **Purpose**: These notes document the `fineract-charge`, `fineract-tax`, `fineract-rates`,
> `fineract-investor`, `fineract-validation`, and `fineract-document` modules plus the
> `creditbureau` sub-system. Use this reference when enriching Loan Subsystem, Savings &
> Deposits, Accounting, and Build & Deployment pages during the review phase.

---

## 1. `fineract-charge` — Charges Module

### Purpose
Models **fees and penalties** that can be applied to Loans, Savings accounts, Clients, Shares, and Working Capital Loans. A single `Charge` definition is a reusable template that can later be attached to product definitions or individual account instances.

### Key Domain Classes

| Class | Table | Description |
|---|---|---|
| `Charge` | `m_charge` | Core entity. Holds name, amount, currency, calculation type, time type, payment mode, penalty flag, min/max caps, fee schedule, linked GL account, and optional TaxGroup. |
| `ChargeAppliesTo` | enum | `LOAN(1)`, `SAVINGS(2)`, `CLIENT(3)`, `SHARES(4)`, `WORKING_CAPITAL_LOAN(5)` |
| `ChargeCalculationType` | enum | `FLAT(1)`, `PERCENT_OF_AMOUNT(2)`, `PERCENT_OF_AMOUNT_AND_INTEREST(3)`, `PERCENT_OF_INTEREST(4)`, `PERCENT_OF_DISBURSEMENT_AMOUNT(5)` |
| `ChargePaymentMode` | enum | `REGULAR(0)`, `ACCOUNT_TRANSFER(1)` |
| `ChargeRepository` / `ChargeRepositoryWrapper` | — | Spring Data JPA access; wrapper throws typed `ChargeNotFoundException`. |

#### `Charge` Fields (Selected)
- **`name`** — unique across all charges (`m_charge.name` has UNIQUE constraint).
- **`chargeAppliesTo`** / **`chargeTimeType`** / **`chargeCalculation`** — integer-coded enums.
- **`penalty`** — boolean; disbursement charges cannot be penalties; overdue-installment charges must be penalties.
- **`minCap`** / **`maxCap`** — only honoured for `PERCENT_OF_AMOUNT` and `PERCENT_OF_DISBURSEMENT_AMOUNT`.
- **`feeOnMonth`** / **`feeOnDay`** — used for monthly and annual fees.
- **`feeInterval`** — recurrence interval (1–12 months for monthly fees).
- **`feeFrequency`** — period type for recurring charges (requires `feeInterval` to be non-null).
- **`enableFreeWithdrawal`** / **`freeWithdrawalFrequency`** / **`restartFrequency`** — savings withdrawal charge free-tier support.
- **`enablePaymentType`** / **`paymentType`** — link charge applicability to a specific payment type.
- **`account`** (`GLAccount`) — income or liability GL account for the charge.
- **`taxGroup`** (`TaxGroup`) — optional tax association; once set, changing the tax group is blocked.

#### Working Capital Loan Charges
`ChargeAppliesTo.WORKING_CAPITAL_LOAN (5)` is a distinct product type. Only `FLAT` calculation type is valid. Payment mode defaults to `REGULAR`. Allowed time types are constrained to `ChargeTimeType.validWorkingCapitalLoanValues()`.

#### Soft Delete
`Charge.delete()` sets `deleted = true` and prepends the record ID to the name to allow future re-use of the same name.

### Service Layer

| Service | Description |
|---|---|
| `ChargeReadPlatformService` | Read operations: list all, retrieve by ID, retrieve template. |
| `ChargeWritePlatformService` | Create, update, delete. |
| `ChargeDropdownReadPlatformService` | Returns enum options for UI dropdowns (time type, calculation type, etc.). |
| `ChargeEnumerations` | Static utility converting integer values to `EnumOptionData` for API responses. |

### REST API — `ChargesApiResource` (`/v1/charges`)

| Method | Path | Operation | Permission |
|---|---|---|---|
| `GET` | `/v1/charges` | List all charge definitions | READ_CHARGE |
| `GET` | `/v1/charges/{chargeId}` | Retrieve charge; append `?template=true` for dropdown metadata | READ_CHARGE |
| `GET` | `/v1/charges/template` | Retrieve new-charge template (dropdowns), supports `chargeAppliesTo` + `chargeTimeType` query params | READ_CHARGE |
| `POST` | `/v1/charges` | Create a charge definition | CREATE_CHARGE |
| `PUT` | `/v1/charges/{chargeId}` | Update charge details (note: `chargeAppliesTo` cannot be changed after creation) | UPDATE_CHARGE |
| `DELETE` | `/v1/charges/{chargeId}` | Soft-delete a charge | DELETE_CHARGE |

### Validation Rules (Enforced in Domain)
- Disbursement charges cannot be penalties (`ChargeDueAtDisbursementCannotBePenaltyException`).
- Overdue installment charges must be penalties (`ChargeMustBePenaltyException`).
- Savings charges: only `FLAT` and `PERCENT_OF_AMOUNT` calculation types; `PERCENT_OF_AMOUNT` allowed only for withdrawal or no-activity charges.
- Client charges: only `FLAT` calculation type.
- Working Capital Loan charges: only `FLAT` calculation type, charge time type restricted to allowed values.

---

## 2. `fineract-tax` — Tax Module

### Purpose
Manages **tax components** and **tax groups** that can be associated with charges. When a charge has a tax group, the platform automatically computes and posts tax journal entries when the charge is applied.

### Key Domain Classes

| Class | Table | Description |
|---|---|---|
| `TaxComponent` | `m_tax_component` | A single tax rate with a percentage, optional debit/credit GL accounts, start date, and history. |
| `TaxComponentHistory` | `m_tax_component_history` | Immutable snapshots of past percentages and their effective date ranges. Enables audit trail. |
| `TaxGroup` | `m_tax_group` | Named container for one or more `TaxGroupMappings`. |
| `TaxGroupMappings` | `m_tax_group_tax_component_mappings` | Join entity linking a `TaxGroup` to its `TaxComponent` list with optional start/end date windows. |

#### `TaxComponent` Detail
- **`percentage`** — the tax rate (e.g., `15.0` for 15%).
- **`debitAccountType`** / **`debitAccount`** — where tax is debited (maps to `GLAccountType`).
- **`creditAccountType`** / **`creditAccount`** — where tax is credited.
- **`startDate`** — effective date of the current percentage.
- **`taxComponentHistories`** — `Set<TaxComponentHistory>` storing all superseded rates. When the percentage is changed, the old rate is archived in this history with the old start date and a new end date = today.
- `getApplicablePercentage(LocalDate date)` — looks up the historically correct percentage for any past date, enabling correct backdated calculations.

#### `TaxGroup` Detail
- Groups multiple `TaxComponent` references, each potentially limited to a date window.
- Only end dates can be updated after creation; new components can be added. No deletions once in use.

### Service Layer

| Service | Description |
|---|---|
| `TaxReadPlatformService` | `retrieveAllTaxComponents()`, `retrieveTaxComponentData(id)`, `retrieveAllTaxGroups()`, `retrieveTaxGroupData(id)`, `retrieveTaxGroupWithTemplate(id)`, and template methods. |
| `TaxWritePlatformService` | Create/update for both tax components and groups. |
| `ChargeTaxApplicationService` / `ChargeTaxApplicationServiceImpl` | Computes `Map<TaxComponent, BigDecimal>` of tax amounts for a given `TaxGroup`, base amount, and effective date. Delegates to `TaxUtils.splitTax(...)`. Returns empty map if group or amount is null/zero. |
| `TaxUtils` | `splitTax(baseAmount, effectiveDate, taxGroupMappings, scale)` — iterates active mappings, calls `getApplicablePercentage`, returns per-component tax amounts. |
| `TaxValidator` | JSON deserializer / validator for create and update commands. |

### REST API

#### Tax Components — `TaxComponentApiResource` (`/v1/taxes/component`)

| Method | Path | Description |
|---|---|---|
| `GET` | `/v1/taxes/component` | List all tax components |
| `GET` | `/v1/taxes/component/{taxComponentId}` | Retrieve a specific tax component |
| `GET` | `/v1/taxes/component/template` | Retrieve template (GL account dropdowns) |
| `POST` | `/v1/taxes/component` | Create new tax component. Mandatory: `name`, `percentage`. Optional: `debitAccountType`, `debitAccountId`, `creditAccountType`, `creditAccountId`, `startDate`. |
| `PUT` | `/v1/taxes/component/{taxComponentId}` | Update percentage (future components replaced); debit/credit accounts cannot be modified. |

#### Tax Groups — `TaxGroupApiResource` (`/v1/taxes/group`)

| Method | Path | Description |
|---|---|---|
| `GET` | `/v1/taxes/group` | List all tax groups |
| `GET` | `/v1/taxes/group/{taxGroupId}` | Retrieve specific group; `?template=true` adds component dropdowns |
| `GET` | `/v1/taxes/group/template` | Retrieve template |
| `POST` | `/v1/taxes/group` | Create group. Mandatory: `name`, `taxComponents[]` with `taxComponentId`. Optional per component: `id`, `startDate`, `endDate`. |
| `PUT` | `/v1/taxes/group/{taxGroupId}` | Update: only `endDate` editable on existing mappings; new components can be appended. |

### Integration Points
- A `Charge` entity has a `@ManyToOne` link to `TaxGroup`. When set, the `ChargeTaxApplicationService` is invoked during charge application to compute and post tax journal entries.
- Once a `TaxGroup` is set on a charge, the reference cannot be changed (enforced in `Charge.update()`).

---

## 3. `fineract-rates` — Floating Interest Rates Module

### Purpose
Implements **floating / variable interest rates** for loan products. A `FloatingRate` is a named market benchmark (e.g., LIBOR, SOFR) with a time series of rate periods. Loan products can reference a floating rate so that interest calculations automatically pick up the current rate.

### Key Domain Classes

| Class | Table | Description |
|---|---|---|
| `FloatingRate` | `m_floating_rates` | Named rate curve. Has `isBaseLendingRate` flag and a list of `FloatingRatePeriod`. Unique name constraint. |
| `FloatingRatePeriod` | `m_floating_rates_periods` | A single rate effective from a `fromDate`. Has `interestRate` and `isDifferentialToBaseLendingRate` flag. |

#### `FloatingRate.fetchInterestRates(FloatingRateDTO dto)` Algorithm
1. Iterates all active `FloatingRatePeriod` records ordered by `fromDate`.
2. Identifies the applicable periods covering the loan's `startDate`.
3. If `isDifferentialToBaseLendingRate` is true, the rate is added on top of the base lending rate from `FloatingRateDTO.baseLendingRatePeriods`.
4. Returns a `Collection<FloatingRatePeriodData>` for the loan engine to iterate.

#### `FloatingRateDTO`
- Carries: `isFloatingInterestRate`, `startDate`, `interestRateDiff` (the spread configured on the loan product), `baseLendingRatePeriods`.
- `fetchBaseRate(date)` — binary searches the base lending rate for the effective date.
- `addInterestRateDiff(diff)` — accumulates differential spreads.

#### Update Logic
When a `FloatingRate` is updated, all `FloatingRatePeriod` records with a `fromDate` **in the future** (relative to business date) are deactivated (`isActive = false`). New rate periods are appended. Past periods are immutable.

### Service Layer

| Service | Description |
|---|---|
| `FloatingRatesReadPlatformService` / `FloatingRatesReadPlatformServiceImpl` | `retrieveAll()`, `retrieveOne(id)`, `retrieveLookupAll()`, and `retrieveBaseLendingRate()` (returns the single rate marked as base lending rate). |
| `FloatingRateWritePlatformService` / `FloatingRateWritePlatformServiceImpl` | Create and update. |
| `FloatingRateDataValidator` | Validates rate period data — from dates must not overlap; percentage must be non-null. |

### REST API — `FloatingRatesApiResource` (`/v1/floatingrates`)

| Method | Path | Description |
|---|---|---|
| `POST` | `/v1/floatingrates` | Create floating rate. Mandatory: `name`. Optional: `isBaseLendingRate`, `isActive`, `ratePeriods[]` (each with `fromDate`, `interestRate`, optional `isDifferentialToBaseLendingRate`). |
| `GET` | `/v1/floatingrates` | List all floating rates |
| `GET` | `/v1/floatingrates/{floatingRateId}` | Retrieve specific floating rate with periods |
| `PUT` | `/v1/floatingrates/{floatingRateId}` | Update. Past periods immutable; future periods replaced. |

### Integration with Loans
- Loan products reference `FloatingRate` when `isFloatingInterestRate = true`.
- During interest calculation, the loan engine constructs a `FloatingRateDTO` and calls `floatingRate.fetchInterestRates(dto)` to obtain the applicable rate timeline.
- `InterestRatePeriodData` is the read-side DTO used in loan account API responses to expose the effective rate timeline.

---

## 4. `fineract-investor` — External Asset Owners / Investor Module

### Purpose
Enables **loan securitisation and loan sale** workflows. Loans can be transferred to external investors (asset owners), tracked with journal entries, and bought back. The module integrates with the COB (Close of Business) pipeline and generates business events for downstream processing.

### Activation
The module is conditionally enabled via `@Conditional(InvestorModuleIsEnabledCondition.class)`. All REST resources and the COB step are absent unless the investor module is enabled.

### Key Domain Classes

| Class | Table | Description |
|---|---|---|
| `ExternalAssetOwner` | `m_external_asset_owner` | Represents an external investor, identified by an `ExternalId`. |
| `ExternalAssetOwnerTransfer` | `m_external_asset_owner_transfer` | A transfer record: links owner + loan, holds `status`, `subStatus`, `settlementDate`, `effectiveDateFrom/To`, `purchasePriceRatio`, `externalId`. |
| `ExternalAssetOwnerTransferDetails` | (mapped via OneToOne) | Financial snapshot at transfer time: `totalPrincipalOutstanding`, `totalInterestOutstanding`, `totalFeeChargesOutstanding`, `totalPenaltyChargesOutstanding`, `totalOverpaid`. |
| `ExternalAssetOwnerTransferLoanMapping` | `m_external_asset_owner_transfer_loan_mappings` | The "active owner" mapping for a loan. Existence means the loan is currently sold. |
| `ExternalAssetOwnerJournalEntryMapping` | — | Links journal entries to the owning investor. |
| `ExternalAssetOwnerTransferJournalEntryMapping` | — | Links journal entries to a specific transfer transaction. |
| `ExternalAssetOwnerLoanProductAttributes` | `m_external_asset_owner_loan_product_attributes` | Key-value attributes per loan product controlling investor transfer behaviour. |

#### Transfer Status Lifecycle (`ExternalTransferStatus`)
```
PENDING → ACTIVE                (sale executed at settlement date via COB)
PENDING → DECLINED              (loan not transferable at COB — balance zero, negative, or user-requested cancel)
PENDING_INTERMEDIATE → ACTIVE_INTERMEDIATE  (delayed settlement: intermediary leg)
ACTIVE_INTERMEDIATE → ACTIVE    (final settlement to investor)
ACTIVE → BUYBACK                (buyback initiated)
BUYBACK → (expire active)       (buyback executed at COB)
PENDING + BUYBACK (same day) → CANCELLED/SAMEDAY_TRANSFERS
```

#### Transfer Sub-Status (`ExternalTransferSubStatus`)
- `BALANCE_ZERO` — loan has zero outstanding balance; sale declined.
- `BALANCE_NEGATIVE` — loan has negative balance; sale declined.
- `SAMEDAY_TRANSFERS` — sale and buyback scheduled for the same settlement date; both cancelled.
- `USER_REQUESTED` — manually cancelled by user.
- `UNSOLD` — buyback requested but loan was never sold to this owner.

#### Loan Product Attributes (`ExternalAssetOwnerLoanProductAttribute`)
- **`OUTSTANDING_INTEREST_STRATEGY` = `TOTAL_OUTSTANDING`** — Transfer includes total (due + not yet due + projected) interest.
- **`OUTSTANDING_INTEREST_STRATEGY` = `PAYABLE_OUTSTANDING`** — Transfer includes only due + not yet due interest.

### COB Integration — `LoanAccountOwnerTransferBusinessStep`
- Step name: `EXTERNAL_ASSET_OWNER_TRANSFER`
- Human-readable: `"Execute external asset owner transfer"`
- Runs during Close-of-Business for each loan.
- Finds all pending/buyback transfers with `settlementDate = today` and `effectiveDateTo = 9999-12-31`.
- Handles three scenarios:
  1. **Single PENDING** — calls `handleSale()`: validates transferability, creates ACTIVE record, posts journal entries via `loanJournalEntryPoster`.
  2. **Single BUYBACK** — calls `handleBuyback()`: expires active record, posts reverse journal entries.
  3. **Two transfers (PENDING + BUYBACK same day)** — calls `handleSameDaySaleAndBuyback()`: cancels both with sub-status `SAMEDAY_TRANSFERS`.
- Delayed Settlement: when `DelayedSettlementAttributeService.isEnabled(loanProductId)` is true, the flow uses `PENDING_INTERMEDIATE` → `ACTIVE_INTERMEDIATE` → `ACTIVE` staging.

### Service Layer

| Service | Description |
|---|---|
| `ExternalAssetOwnersReadService` | Retrieve transfers, active transfer, journal entries per transfer or owner, all external owners. |
| `AccountingServiceImpl` | `createJournalEntriesForSaleAssetTransfer(...)` and `createJournalEntriesForBuybackAssetTransfer(...)`. Maps journal entries to both the transfer record and the new/previous owner. Uses `FinancialActivity.ASSET_TRANSFER` GL account for the contra entry. |
| `ExternalAssetOwnerJournalEntryService` | Creates `ExternalAssetOwnerJournalEntryMapping` records to link GL entries to owners. |
| `DelayedSettlementAttributeService` | Checks if the `DELAYED_SETTLEMENT` feature is enabled for a loan product. |
| `LoanTransferabilityService` | Validates whether a loan can be transferred (checks balance, etc.). |

### Data Enrichers
Implement `DataEnricher<T>` to inject investor information into external event Avro messages:
- **`LoanAccountDataV1Enricher`** — adds `externalOwnerId`, `settlementDate`, `purchasePriceRatio` and per-charge `externalOwnerId` to loan events.
- **`LoanChargeDataV1Enricher`** — enriches charge events with the current owner.
- **`LoanTransactionDataV1Enricher`** / **`LoanTransactionAdjustmentDataV1Enricher`** — enriches transaction events.

### REST API

#### External Asset Owners — `ExternalAssetOwnersApiResource` (`/v1/external-asset-owners`)

| Method | Path | Command | Description |
|---|---|---|---|
| `POST` | `/v1/external-asset-owners/transfers/loans/{loanId}` | `sale` | Initiate loan sale to external owner |
| `POST` | `/v1/external-asset-owners/transfers/loans/{loanId}` | `buyback` | Initiate buyback |
| `POST` | `/v1/external-asset-owners/transfers/loans/{loanId}` | `intermediarySale` | Intermediary/delayed settlement sale |
| `POST` | `/v1/external-asset-owners/transfers/loans/{loanId}` | `cancel` | Cancel a pending transfer |
| `POST` | `/v1/external-asset-owners/transfers/loans/external-id/{loanExternalId}` | (same commands by external loan ID) | — |
| `POST` | `/v1/external-asset-owners/transfers/{id}` | `cancel` | Cancel transfer by transfer ID |
| `POST` | `/v1/external-asset-owners/transfers/external-id/{externalId}` | `cancel` | Cancel by transfer external ID |
| `GET` | `/v1/external-asset-owners/transfers` | — | List transfers; filter by `transferExternalId`, `loanId`, or `loanExternalId`; supports `offset`/`limit` |
| `GET` | `/v1/external-asset-owners/transfers/active-transfer` | — | Get the currently active transfer for a loan |
| `GET` | `/v1/external-asset-owners/transfers/{transferId}/journal-entries` | — | Journal entries for a transfer |
| `GET` | `/v1/external-asset-owners/owners/external-id/{ownerExternalId}/journal-entries` | — | Journal entries for an owner |
| `POST` | `/v1/external-asset-owners/search` | — | Full-text/date-range search for transfers |
| `POST` | `/v1/external-asset-owners` | `create` | Create an external asset owner by external ID |
| `GET` | `/v1/external-asset-owners` | — | List all external asset owners with details |

#### Loan Product Attributes — `ExternalAssetOwnerLoanProductAttributesApiResource` (`/v1/external-asset-owners/loan-product`)

| Method | Path | Description |
|---|---|---|
| `POST` | `/v1/external-asset-owners/loan-product/{loanProductId}/attributes` | Create attribute |
| `GET` | `/v1/external-asset-owners/loan-product/{loanProductId}/attributes` | List attributes; filter by `attributeKey` |
| `PUT` | `/v1/external-asset-owners/loan-product/{loanProductId}/attributes/{id}` | Update attribute |

### Business Events
- `LoanOwnershipTransferBusinessEvent` — fired after each COB transfer step (sale, buyback, decline, cancel). Carries both the `ExternalAssetOwnerTransfer` and the `Loan`.
- Standard `LoanAccountSnapshotBusinessEvent` is also fired post-transfer so downstream systems receive the updated loan state.

---

## 5. `fineract-validation` — Validation Constraints Module

### Purpose
Provides **custom JSR-380 (Bean Validation) annotations** for request model validation across the platform.

### Constraint Annotations

| Annotation | Validator | Target | Description |
|---|---|---|---|
| `@DateFormat` | `DateFormatValidator` | FIELD, PARAMETER | Validates that a `String` is a valid date format pattern (e.g., `"dd MMMM yyyy"`). |
| `@EnumValue` | `EnumValueValidator` | FIELD | Validates that a `String` value is one of the declared constants of the specified `enumClass`. Usage: `@EnumValue(enumClass = SomeEnum.class)`. |
| `@LocalDate` | `LocalDateValidator` | FIELD, PARAMETER | Validates that a `String` parses correctly as a `LocalDate` given the locale and date format. |
| `@Locale` | `LocaleValidator` | FIELD, PARAMETER | Validates that a `String` is a valid Java `Locale` tag. |

### Usage Pattern
```java
public class LoanRequest {
    @DateFormat
    private String dateFormat;

    @Locale
    private String locale;

    @EnumValue(enumClass = ChargeAppliesTo.class)
    private String chargeAppliesTo;
}
```

These annotations are applied to request DTOs throughout the codebase and are evaluated by the Jakarta Validation framework during request deserialization in JAX-RS / Spring MVC controllers.

---

## 6. `fineract-document` — Content Store Module

### Purpose
Provides a **pluggable content storage abstraction** for documents (images, files, reports) attached to clients, loans, and other entities. Supports local file system and AWS S3 backends. Implements a pipeline of content policies (security enforcement) and processors (transformations).

### Storage Backends (`ContentStoreType`)
- `FILE_SYSTEM (1)` — `FileContentStoreService` writes to the local file system path configured in `FineractProperties`.
- `S3 (2)` — `S3ContentStoreService` writes to an AWS S3 bucket.

The active backend is selected via Spring configuration.

### `ContentStoreService` Interface
```java
InputStream download(String path);
String upload(String path, InputStream is, String mimeType);
void delete(String path);
ContentStoreType getType();
```

### Content Security — Policy Chain
Policies implement `ContentPolicy.check(ContentPolicyContext ctx)` and are executed before upload and download:

| Policy | Description |
|---|---|
| `WhitelistContentPolicy` | Enforces a filename regex whitelist (`fineract.content.regex-whitelist`) and/or a MIME type whitelist (`fineract.content.mime-whitelist`). Both independently toggleable. |
| `MimeContentPolicy` | Uses `ContentDetectorManager` (Apache Tika backed by `TikaContentDetector`) to detect actual MIME type from the byte stream and compares it against the declared MIME type in the request. Prevents content-type spoofing. |
| `TraversalContentPolicy` | Blocks path traversal attacks (e.g., `../`) in upload paths. |
| `DefaultPreUploadContentPolicy` | Aggregates and runs all pre-upload checks. |
| `DefaultPostUploadContentPolicy` | Post-upload policy hooks. |
| `DefaultDownloadContentPolicy` | Download access checks. |
| `DefaultDeleteContentPolicy` | Delete access checks. |

### Content Processors — Transformation Pipeline
`ContentProcessor` is a `@FunctionalInterface` with a `then(ContentProcessor next)` combinator to chain processors:

| Processor | Direction | Description |
|---|---|---|
| `Base64DecoderContentProcessor` | Decode | Decodes Base64-encoded input to binary. |
| `Base64EncoderContentProcessor` | Encode | Encodes binary to Base64 for API responses. |
| `DataUrlDecoderContentProcessor` | Decode | Strips `data:<mime>;base64,` prefix from data URLs. |
| `DataUrlEncoderContentProcessor` | Encode | Wraps binary in data URL format. |
| `GzipDecoderContentProcessor` | Decode | Gunzips compressed streams. |
| `GzipEncoderContentProcessor` | Encode | Gzips streams for storage. |
| `ImageResizeContentProcessor` | Transform | Resizes images to configured dimensions. |
| `SizeContentProcessor` | Validate | Enforces maximum upload size. |

### Utility Classes
- **`ContentPathSanitizer`** / **`DefaultContentPathSanitizer`** — normalises and validates storage paths to prevent path injection.
- **`ContentPathRandomizer`** / **`DefaultContentPathRandomizer`** — generates randomised storage paths so file names are not predictable.
- **`ContentPipe`** — utility for chaining `ContentProcessor` instances into a linear pipeline.

### Content Detection
`ContentDetectorManager` (default: `DefaultContentDetectorManager`) orchestrates `ContentDetector` implementations:
- **`TikaContentDetector`** — uses Apache Tika to detect MIME type from the byte stream.
- **`FileContentDetector`** — falls back to file-extension-based detection.

The detected MIME type is placed in `ContentDetectorContext.mimeType` and used by `MimeContentPolicy`.

### Configuration
`ContentStoreConfig` registers a cached-thread-pool `ExecutorService` bean named `contentProcessorExecutor` for asynchronous content processing chains.

---

## 7. Credit Bureau Integration (`fineract-provider` / `infrastructure.creditbureau`)

### Purpose
Enables integration with external **credit bureaus** for fetching credit reports, submitting loan data, and mapping loan products to credit bureau services.

### Domain Entities

| Entity | Table | Description |
|---|---|---|
| `CreditBureau` | `m_creditbureau` | Credit bureau registry (name, country, product). |
| `OrganisationCreditBureau` | `m_organisation_creditbureau` | Links the organisation to an active credit bureau with `startDate`/`endDate`. |
| `CreditBureauConfiguration` | `m_creditbureau_configuration` | Key-value config entries per bureau (credentials, endpoints, etc.). |
| `CreditBureauLoanProductMapping` | `m_creditbureau_loanproduct_mapping` | Maps a loan product to a credit bureau. |
| `CreditReport` | `m_creditreport` | Cached credit report for a given NRC (national registration number). |
| `CreditBureauToken` | — | Stores authentication tokens for bureau API calls. |

### Service Layer

| Service | Description |
|---|---|
| `CreditBureauReadPlatformService` | Retrieves credit bureau list. |
| `OrganisationCreditBureauReadPlatformService` | Retrieves the organisation's credit bureau bindings. |
| `CreditBureauReadConfigurationService` | Reads configuration entries for a specific bureau. |
| `CreditBureauLoanProductMappingReadPlatformService` | Reads loan product ↔ bureau mappings. |
| `CreditReportReadPlatformService` | Retrieves previously cached credit reports. |
| `CreditReportWritePlatformService` | Fetches and saves credit reports from the bureau. |
| `ExternalCreditBureauIntegrationWritePlatformService` | Calls external bureau API and handles authentication. |

### REST API

#### Credit Bureau Configuration — `CreditBureauConfigurationApiResource` (`/v1/CreditBureauConfiguration`)

| Method | Path | Description |
|---|---|---|
| `GET` | `/v1/CreditBureauConfiguration` | List credit bureaus |
| `GET` | `/v1/CreditBureauConfiguration/mappings` | List loan product ↔ bureau mappings |
| `GET` | `/v1/CreditBureauConfiguration/organisationCreditBureau` | List org-level bureau bindings |
| `POST` | `/v1/CreditBureauConfiguration/organisationCreditBureau` | Create org-level bureau binding |
| `PUT` | `/v1/CreditBureauConfiguration/organisationCreditBureau/{orgCBId}` | Update org bureau binding |
| `POST` | `/v1/CreditBureauConfiguration/{orgCBId}` | Add configuration entry |
| `PUT` | `/v1/CreditBureauConfiguration/{configId}/creditBureauLoanProductMapping` | Map a loan product to a bureau |

#### Credit Bureau Integration — `CreditBureauIntegrationApiResource` (`/v1/creditBureauIntegration`)

| Method | Path | Description |
|---|---|---|
| `POST` | `/v1/creditBureauIntegration/creditReport` | Fetch credit report from external bureau (params passed as JSON map) |
| `POST` | `/v1/creditBureauIntegration/addCreditReport` | Upload client credit data to bureau (multipart file upload) |
| `GET` | `/v1/creditBureauIntegration/creditReport/{nationalId}` | Retrieve cached credit report by national ID |
| `DELETE` | `/v1/creditBureauIntegration/creditReport/{nationalId}` | Delete cached report |

---

## Cross-Module Integration Summary

### How Charges, Taxes, and Floating Rates Interact with Loans

```
Loan Product Definition
  ├─ Charge[] ─────────────────────────────→ fineract-charge (Charge entity)
  │     └─ TaxGroup? ──────────────────────→ fineract-tax (TaxGroup + TaxComponent)
  └─ FloatingRate? ─────────────────────────→ fineract-rates (FloatingRate + periods)

Loan Account
  ├─ LoanCharge[] ─── references → Charge definition
  └─ InterestRate timeline ─── built from → FloatingRate.fetchInterestRates(FloatingRateDTO)

When Charge Is Applied:
  1. LoanCharge payment triggers ChargeTaxApplicationService
  2. Tax computed: Map<TaxComponent, BigDecimal> = splitTax(baseAmount, date, mappings, scale)
  3. Journal entries posted for both the charge amount and each tax component

Floating Rate Recalculation:
  1. LoanReschedule / interest recalculation triggers FloatingRateDTO construction
  2. FloatingRate.fetchInterestRates(dto) returns applicable rate periods
  3. Loan engine applies rate per period, adding differential on top of base if configured
```

### How the Investor Module Interacts with COB and Accounting

```
COB Pipeline (per loan, at settlement date):
  LoanAccountOwnerTransferBusinessStep
    ├─ PENDING → sellAssetOrDecline()
    │     ├─ loanTransferabilityService.isTransferable(loan)
    │     ├─ AccountingServiceImpl.createJournalEntriesForSaleAssetTransfer()
    │     │     └─ Posts: Principal, Interest, Fees, Penalties transfer entries
    │     │     └─ Maps JEs to: ExternalAssetOwnerTransfer + ExternalAssetOwner
    │     └─ Fires: LoanOwnershipTransferBusinessEvent + LoanAccountSnapshotBusinessEvent
    └─ BUYBACK → buybackAsset()
          └─ AccountingServiceImpl.createJournalEntriesForBuybackAssetTransfer()
          └─ Fires: LoanOwnershipTransferBusinessEvent + LoanAccountSnapshotBusinessEvent

External Events (Avro):
  DataEnricher chain adds investor context to:
    - LoanAccountDataV1 (externalOwnerId, settlementDate, purchasePriceRatio)
    - LoanChargeDataV1 (externalOwnerId per charge)
    - LoanTransactionDataV1 / LoanTransactionAdjustmentDataV1
```

---

## Notes for Documentation Writers

### Loan Subsystem Pages
- The **charges** section should explain the `ChargeAppliesTo` distinction between `LOAN`, `SAVINGS`, `CLIENT`, `SHARES`, and `WORKING_CAPITAL_LOAN`.
- The **Working Capital Loan** concept (value `5`) is a newer product type with restricted charge options (flat only).
- Loan charges can be paid via account transfer (`ChargePaymentMode.ACCOUNT_TRANSFER`) — relevant to the repayment page.
- Overdue-installment penalties and the `minCap`/`maxCap` constraints should be noted on the charges page.

### Savings & Deposits Pages
- Savings charges support free-withdrawal tiers: `enableFreeWithdrawal`, `freeWithdrawalFrequency`, `restartFrequency`, `restartFrequencyEnum`.
- Savings charges can be payment-type-specific via `enablePaymentType` + `paymentTypeId`.

### Accounting Pages
- The investor module adds two new accounting flows: **asset sale** and **asset buyback**. These require a `FinancialActivity.ASSET_TRANSFER` GL account mapping.
- Tax journal entries are generated automatically when a charge with a tax group is applied. The debit/credit accounts come from `TaxComponent`.

### Build & Deployment Pages
- The investor module is conditionally compiled/activated; document the `INVESTOR_MODULE_ENABLED` configuration property.
- The content store backend (`FILE_SYSTEM` vs `S3`) is controlled by `fineract.content.store-type` with related properties for S3 bucket, region, and file system root path.
- Content security policies are configured via:
  - `fineract.content.regex-whitelist-enabled` / `fineract.content.regex-whitelist[]`
  - `fineract.content.mime-whitelist-enabled` / `fineract.content.mime-whitelist[]`
