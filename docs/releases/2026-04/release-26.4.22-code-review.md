# Release 26.4.22 (318103) — Code Review Report

**Date**: 2026-04-21
**Release**: KK PI 26.4.22 — [TP #318103](https://minutemenu.tpondemand.com/entity/318103)
**Repos**: MinuteMenu/KK, MinuteMenu/Centers-CX, MinuteMenu/POC_HxCloudConnectApi, MinuteMenu/Shared-Core, MinuteMenu/MinuteMenu.Database, MinuteMenu/HX, MinuteMenu/Parachute
**Scope**: 7 release-to-master PRs consolidating 45 TP tickets
**Focus**: Critical (90-100) and High (80-89) findings

---

## Executive Summary

| Severity | Count |
|----------|-------|
| **Critical (90-100)** | 4 |
| **High (80-89)** | 17 |
| **Total** | 21 |

**Missing tickets**: 2 release-scope tickets have NO code change in any diff (#318581 Ghost eForm, #321662 Pending disallow). Flagged for product/engineering confirmation.

### Critical Issues (Must Fix Before Release)

| # | Repo | Area | TP Ticket | Category | Summary |
|---|------|------|-----------|----------|---------|
| C1 | CX + KK | ACH re-download | [#316306](https://minutemenu.tpondemand.com/entity/316306) | Security/Bug | Missing authorization scope; SQL string-concat IN; multi-date batch stamped with single arbitrary date |
| C2 | CX + KK | ACH re-download | [#316306](https://minutemenu.tpondemand.com/entity/316306) | FinancialAccuracy | Batch spanning multiple `check_date` values produces ACH file with only last row's date → bank compliance risk |
| C3 | HX API + DB | J.017 policy | [#320244](https://minutemenu.tpondemand.com/entity/320244) | CrossRepoImpact | EF-mapped SponsorPolicy entity queries `include_infants_in_own_alone_edit_check_flag`; any deploy before DB migration = HX API outage |
| C4 | Parachute | Vehicle search | [#320290](https://minutemenu.tpondemand.com/entity/320290) | Bug | Unguarded cast `(double)e.PurchasePrice` on nullable decimal → 500 on search when any vehicle has null price |

---

## Critical Findings (Detail)

### C1. [Centers-CX#4161](https://github.com/MinuteMenu/Centers-CX/pull/4161) + [KK#22699](https://github.com/MinuteMenu/KK/pull/22699) — ACH re-download missing authorization + raw SQL IN

**Ticket**: [TP #316306](https://minutemenu.tpondemand.com/entity/316306) — Enable KidKare For Centers The Ability To Re-Generate Payments Without Needing To Void And Reissue
**Score**: 92 | **Category**: Security/Bug
**File**: `products/Centers/Projects/MinuteMenu.Centers.ServerLib/Logic/LogicServiceToKK.cs:65-113` (`GetCenterACHFileFromCheckRegister`) + `Projects/KidKare.Service/Controllers/PortCX/PortCxPaymentController.cs:522-535` (`RedownloadACHFile`)

The ACH re-download SQL uses `WHERE check_id IN (" + checkIdList + ")` by string-concatenating an `int[]` into raw SQL. While the int array prevents injection today, it breaks the parameterization pattern — any later change to the type erases that safety. More important, the controller performs no authorization check that the supplied `CheckIds` belong to the current user's scope beyond `AND client_id = @client_id`. A sponsor user can force-download ACH batch files for arbitrary `check_id`s within the same client, and there is no null guard on `request.CheckIds` (null → 404 instead of 400).

**Impact**: Information disclosure (ACH bank account data for unrelated batches within same client); fragile non-parameterized IN list.

**Recommendation**: Parameterize the IN list (Dapper `@ids` or a TVP). Validate that all `CheckIds` belong to the expected scope before building the ACH file. Cap `checkIds.Length` (e.g., ≤ 2000). Null-check `request` and `request.CheckIds`.

---

### C2. [Centers-CX#4161](https://github.com/MinuteMenu/Centers-CX/pull/4161) — ACH multi-date batch produces wrong effective date

**Ticket**: [TP #316306](https://minutemenu.tpondemand.com/entity/316306) — CX re-download ACH
**Score**: 90 | **Category**: FinancialAccuracy
**File**: `products/Centers/Projects/MinuteMenu.Centers.ServerLib/Logic/LogicServiceToKK.cs:71-95` (`GetCenterACHFileFromCheckRegister`)

Inside the reader loop, `checkDate = Convert.ToDateTime(rdr["check_date"]).Date` is overwritten each row — the **last row's `check_date` wins** and is used for the whole ACH file. The KK batch-picker groups by `mod_date_time + mod_login_id` (not by `check_date`), so a single UI "batch" may legitimately contain rows with different check_dates. HX Homes' re-generate behavior preserves per-check posting dates; this fix does not. Additionally `cmd.CommandTimeout = 300` (5 min) is excessive for an indexed IN lookup.

**Impact**: Silent production of ACH files stamped with the wrong effective/settlement date. Since ACH files are submitted to banks, wrong-date files cause compliance/timing problems.

**Recommendation**: Take `checkDate` from the KK caller (`filter.checkDate`) via `RedownloadACHRequest` DTO; OR group rows by `check_date` inside the SP loop and emit one ACH file per distinct date; OR assert consistency across rows. Reduce `CommandTimeout` to 30s.

---

### C3. [POC_HxCloudConnectApi#1595](https://github.com/MinuteMenu/POC_HxCloudConnectApi/pull/1595) + [MinuteMenu.Database#686](https://github.com/MinuteMenu/MinuteMenu.Database/pull/686) — EF entity read of DB column requires strict deploy order

**Ticket**: [TP #320244](https://minutemenu.tpondemand.com/entity/320244) — HX030 Error 76 disallow own infants
**Score**: 90 | **Category**: CrossRepoImpact
**File**: `HxCloudConnect.Data/Entities/SponsorPolicy.cs` (property added) + ≥14 call sites in ProviderService, AdministrationService, ChildService, ExportFileService, SponsorService

HX API `SponsorPolicy` EF entity gains `include_infants_in_own_alone_edit_check_flag`. EF generates `SELECT [all mapped columns]` on every `db.SponsorPolicies.Where(...)` query. 14+ call sites hit this during routine traffic. If the HX API artifact deploys before DB migration `320244_ADD_NEW_FIELDS_TO_TABLE_SPONSOR_POLICY.sql`, every one of those queries throws `Invalid column name 'include_infants_in_own_alone_edit_check_flag'` → **full HX API outage**.

Companion risk: `clsSPONSORPOLICY.cls` in HX Desktop does `rsPopulate("include_infants_in_own_alone_edit_check_flag")` unconditionally. HX Desktop v3.4.0 installed on any sponsor whose DB has not yet been upgraded throws ADO "Item cannot be found" at sponsor-policy load — HX Desktop fails to open.

**Impact**: Any deploy-order violation = HX API outage (server side) and HX Desktop inoperable for affected sponsors (desktop side).

**Recommendation**: Enforce strict deployment order in the DevOps runbook: **DB Updates 320244_ADD + 320244_SET_DEFAULT must run BEFORE** any HX API artifact is promoted AND before HX Desktop v3.4.0 is pushed to any sponsor desktop. Add app-startup smoke test: `SELECT TOP 1 include_infants_in_own_alone_edit_check_flag FROM SPONSOR_POLICY`. Consider wrapping the VB6 `rsPopulate` read in `On Error Resume Next` for tolerance.

---

### C4. [Parachute#4330](https://github.com/MinuteMenu/Parachute/pull/4330) — Vehicle search removes null-check on PurchasePrice

**Ticket**: [TP #320290](https://minutemenu.tpondemand.com/entity/320290) — Search Parity for Accounting Pages
**Score**: 90 | **Category**: Bug
**File**: `Projects/Parachute.Service/Controllers/Expenses/MileageController.cs:113`

The Vehicle search refactor removed the `e.PurchasePrice != null` guard. New line: `isNumeric && SqlFunctions.StringConvert((double)e.PurchasePrice, 18, 2).Trim().Contains(stringNumber)`. Unconditional cast of nullable decimal → double throws `InvalidOperationException: Nullable object must have a value` during EF translation, or 500s the endpoint depending on provider behavior.

**Impact**: Any user searching the Large Inventory / Vehicles tab will see the entire endpoint 500 out if their account has **any** vehicle with a NULL purchase price. This is a very common data shape in production (legacy vehicles often lack purchase price).

**Recommendation**:

```csharp
(e.PurchasePrice != null && isNumeric
  && SqlFunctions.StringConvert((double)e.PurchasePrice.Value, 18, 2)
     .Trim().Contains(stringNumber))
```

---

## High Severity Findings (80-89)

### Cross-Repo J.017 Policy

#### H1. [MinuteMenu.Database#686](https://github.com/MinuteMenu/MinuteMenu.Database/pull/686) — `save_provider_policy_sp` silently overwrites `highest_serving_allowed_code` to 215

**Ticket**: [TP #320881](https://minutemenu.tpondemand.com/entity/320881) — PCI Highest Serving Allowed empty
**Score**: 88 | **Category**: DataIntegrity
**File**: `MMADMIN_XXX/Programmability/Stored Procedures/save_provider_policy_sp.sql` and `save_provider_policy_sp_port_kk.sql`

Change from "`0 → NULL`" to "`0 OR NULL → 215`" applies on **every update path**, not just INSERT. VB6 Integer parameters default to 0 when a form field was never loaded. Any existing HX Desktop update path that does not explicitly round-trip `highest_serving_allowed_code` now forcibly writes `215` over the stored value. HX API `ProviderService.cs` code-level fix is careful (loads existing value first); the SP change does NOT do that lookup.

**Impact**: Providers configured with Serving 2 or Serving 3 silently dropped to Serving 1 on next bulk edit → wrong meal allowance on next claim processing, cascades to reimbursement errors.

**Recommendation**: Only default to 215 in the INSERT branch of the SP:

```sql
IF @highest_serving_allowed_code = 0 OR @highest_serving_allowed_code IS NULL
  SELECT @highest_serving_allowed_code = ISNULL(highest_serving_allowed_code, 215)
  FROM PROVIDER_POLICY WHERE provider_id = @id
```

Move the default to the app layer (as `ProviderService.cs EnrollProvider` already does).

---

#### H2. [MinuteMenu.Database#686](https://github.com/MinuteMenu/MinuteMenu.Database/pull/686) — MRP manifest omits sponsor-030 enablement script

**Ticket**: [TP #320244](https://minutemenu.tpondemand.com/entity/320244)
**Score**: 80 | **Category**: CrossRepoImpact / DeploymentGap
**File**: `MMADMIN_XXX/Updates/320244_SET_SPONSOR_030_INCLUDE_INFANTS_FLAG.sql` + `MRP.sql`

The `SET_SPONSOR_030_INCLUDE_INFANTS_FLAG.sql` script (sets J.017='Y' for the ticket's requesting sponsor HX030) is committed to the repo but is **not listed in the regenerated MRP manifest**. MRP-driven deploy will not apply it → the feature ships disabled for the very client that asked for it.

**Impact**: Ticket appears complete but has no effect in production for HX030. Follow-up "not working" ticket expected.

**Recommendation**: Regenerate MRP.sql with the SET_SPONSOR_030 script included (sequenced after SET_DEFAULT so column is populated), OR document a manual DBA runbook step with product-owner sign-off.

---

### Sponsor 102 Print Check + ACH (Cross-Repo)

#### H3. [POC_HxCloudConnectApi#1595](https://github.com/MinuteMenu/POC_HxCloudConnectApi/pull/1595) + [KK#22699](https://github.com/MinuteMenu/KK/pull/22699) + [Centers-CX#4161](https://github.com/MinuteMenu/Centers-CX/pull/4161) + [MinuteMenu.Database#686](https://github.com/MinuteMenu/MinuteMenu.Database/pull/686) — Cross-repo deploy-order contract

**Ticket**: [TP #320213](https://minutemenu.tpondemand.com/entity/320213) — Sponsor 102 Print Check
**Score**: 85 | **Category**: CrossRepoImpact
**File**: CX `LogicService.cs` + `CheckPaymentHistoryList.cs`, KK `PaymentMapping.cs` + `PaymentCheckRegisterResponse.cs`, DB `select_payment_history_list_sp.sql`

`select_payment_history_list_sp` emits 3 new columns (`mod_date_time, mod_login_id, mod_username`). CX `LogicService` reads `dr["mod_date_time"]`. KK AutoMapper binds the new fields. KK consumes via the checked-in `MinuteMenu.Centers.ServerLib.dll`. Deploy ordering must be DB → CX service → KK → HX API; wrong order crashes the Check Register page for every sponsor with `IndexOutOfRangeException`.

**Recommendation**: Enforce deploy order in release runbook. Post-deploy smoke-test: `/cxservice/payments/getCheckRegisterList` returns `modDateTime/modLoginId/modUsername`.

---

#### H4. [POC_HxCloudConnectApi#1595](https://github.com/MinuteMenu/POC_HxCloudConnectApi/pull/1595) — Unbounded Task.WhenAll fan-out on Sponsor 102 print

**Ticket**: [TP #320213](https://minutemenu.tpondemand.com/entity/320213)
**Score**: 80 | **Category**: InfrastructureStress
**File**: `HxCloudConnect.Api/Controllers/IsssuePaymentController.cs` + `HxCloudConnect.Services/PaymentService.cs`

Controller spins an unbounded `Task.WhenAll` calling `BuildSponsor102Data` **per check**. Each `BuildSponsor102Data` fires 5 stored procedures in parallel (active child, serving times, provider tier, child tier counts, training hours). A 500-provider "print all" = **2,500 concurrent DB round-trips** with no semaphore, no batching, no caching across providers that share fiscal/tier data.

**Impact**: Spike DB connections to hundreds; each `QueryListByStoredProcedureAsync` opens a new connection. Risk of SQL thread pool / connection pool exhaustion during monthly batch prints.

**Recommendation**: Add `SemaphoreSlim(10)` around `sponsor102Tasks`. Cache `SELECT_PROVIDER_SERVING_TIMES_SP` / tier data per providerId within the request. Monitor `sp_who2` during first 102 print.

---

#### H5. [POC_HxCloudConnectApi#1595](https://github.com/MinuteMenu/POC_HxCloudConnectApi/pull/1595) — EF-tracked entity mutation in `AggregateLinkedClaimsForSponsor102`

**Ticket**: [TP #320213](https://minutemenu.tpondemand.com/entity/320213)
**Score**: 80 | **Category**: DataIntegrity
**File**: `HxCloudConnect.Services/PaymentService.cs:542-596`

Fetches HxClaim rows for a check and mutates the chosen `primary` claim **in-place** by summing counts and amounts from siblings. The entity is EF-tracked — any subsequent `SaveChanges()` in the same request (or caching by IRepository) persists the summed values to DB, **corrupting the original claim row**.

**Recommendation**: `AsNoTracking()` on the initial query, or detach the entity, or clone to a new `HxClaim` for transient view logic.

---

#### H6. [POC_HxCloudConnectApi#1595](https://github.com/MinuteMenu/POC_HxCloudConnectApi/pull/1595) — `Sponsor102LinkedClaimDTO.ClaimYear` typed `decimal?`

**Ticket**: [TP #320213](https://minutemenu.tpondemand.com/entity/320213)
**Score**: 82 | **Category**: Bug
**File**: `HxCloudConnect.Data/DTO/Payment/PrintReportDTO.cs:359-367`

`ClaimYear` and `StatusCode` declared as `decimal?`. On the FE, `moment({ year: lc.ClaimYear, month: lc.ClaimMonth - 1, day: 1 })` and `getClaimLabel(lc.StatusCode, ...)` expect integers. Works today via JS coercion but semantically wrong — any future JSON serializer change (scientific form for large decimals) breaks the check stub.

**Recommendation**: Change `ClaimYear` to `int?` and `StatusCode` to `int?` to match `HxClaim.claim_year/claim_status_code`.

---

### Enrollment & eForm Cleanup

#### H7. [KK#22699](https://github.com/MinuteMenu/KK/pull/22699) — Cleanup endpoint fan-out with no CommandTimeout

**Ticket**: [TP #319630](https://minutemenu.tpondemand.com/entity/319630) — CX Child Not Found on eForm approval
**Score**: 88 | **Category**: InfrastructureStress
**File**: `Projects/KidKare.Bll/Enrollment/CxEnrollmentBll.cs:1444-1628`

`DetectOrphanedCxEforms` / `CleanupOrphanedCxEforms` execute a **serial per-client fan-out**. Each iteration opens a fresh eFormsContext + CxContext, then runs `GetChildQuery(cx, clientId, null, null).Select(x => x.child_id).ToList()` — pulls the **entire active CHILD roster** for the client into memory. No `CommandTimeout`. No row cap. Auto-detect branch (null `ClientIds`) sweeps every CX client with any active eForm assignment.

**Impact**: N-query sweep where N can be hundreds of clients. Connection pool pressure, long-running thread held for the duration, OOM risk on large tenants. Matches known incident patterns 3/4.

**Recommendation**: Require non-empty `ClientIds` in prod (or cap per invocation). Add explicit `Database.CommandTimeout` of 30-60s. Move to a Hangfire/queue job rather than sync HTTP. Replace the HashSet-based orphan check with a server-side `NOT EXISTS` anti-join.

---

#### H8. [KK#22699](https://github.com/MinuteMenu/KK/pull/22699) — "Orphan" definition includes withdrawn children → mass hard-delete risk

**Ticket**: [TP #319630](https://minutemenu.tpondemand.com/entity/319630)
**Score**: 85 | **Category**: DataIntegrity
**File**: `Projects/KidKare.Bll/Enrollment/CxEnrollmentBll.cs` (orphan predicate) + `EnrollmentBll.DeleteEformAssignments`

Orphan predicate: `a.ChildId == 0 || !activeChildIds.Contains(a.ChildId)`. `activeChildIds` is populated by `GetChildQuery` which filters `record_status_code == Active`. **Any child whose status is Withdrawn or Inactive is treated as an orphan** and its eForm assignment is HARD-deleted via `HardDelete()` on `EnrollmentForm`, history, signatures, IEF, reports, and storage files.

**Impact**: Production data loss. A child withdrawn yesterday (a common lifecycle event) will have all historical enrollment forms/IEFs permanently deleted on first cleanup run.

**Recommendation**: Tighten predicate to "no matching CHILD row at all" (drop the Active filter for this check). Explicitly exclude withdrawn/inactive children from cleanup scope. Require `confirm: true` token before delete. Consider soft-delete instead of hard-delete.

---

#### H9. [KK#22699](https://github.com/MinuteMenu/KK/pull/22699) — Admin cleanup endpoint authz is shared secret only

**Ticket**: [TP #319630](https://minutemenu.tpondemand.com/entity/319630)
**Score**: 82 | **Category**: Authorization
**File**: `Projects/KidKare.Service/Controllers/CommonController.cs:7436-7472`

`common/cleanupOrphanedEforms` and `common/detectOrphanedEforms` are guarded **only** by a shared-secret `SecurityKey`. No `[RequiredRole]`, `[RequiredPermission]`, `[Authorize]`, or IP restriction. If the key leaks (logs, Postman collections, client-side bundles), an attacker can hard-delete eForm data org-wide by omitting `ClientIds` (auto-sweep mode).

**Recommendation**: Add `[RequiredRole(Role.MM.System.Admin)]` in addition to the security key. Move behind VPN / IP allow-list. Log every invocation with caller identity, `ClientIds`, and row counts.

---

### Localization & Data Patches

#### H10. [KK#22699](https://github.com/MinuteMenu/KK/pull/22699) — Spanish warning `ACTIVE_CHILDREN_WARNING_SETIEF_SKIPPED` left in English

**Ticket**: [TP #298391](https://minutemenu.tpondemand.com/entity/298391) — Spanish translations
**Score**: 85 | **Category**: Localization
**File**: `Projects/KidKare.Web/app/i18n/es/states/cx-active-children.json:28`

The key `ACTIVE_CHILDREN_WARNING_SETIEF_SKIPPED` was added to the Spanish file with the English text verbatim. Shown every time a sponsor performs Set IEF and at least one child lacks a current enrollment date.

**Recommendation**: Translate to Spanish.

---

#### H11. [KK#22699](https://github.com/MinuteMenu/KK/pull/22699) — Spanish `{{participant_kids_plural}}` placeholder dropped

**Ticket**: [TP #298391](https://minutemenu.tpondemand.com/entity/298391)
**Score**: 80 | **Category**: Localization
**File**: `Projects/KidKare.Web/app/i18n/es/states/my-kids-enroll-child.json:165`

Spanish translation hard-codes "varios miembros de una familia" instead of interpolating `{{participant_kids_plural}}`. Singular `{{participant_kids}}` kept — inconsistent. For sponsors configured with a non-default participant term ("Estudiantes", "Participantes", "Adultos"), the Spanish sentence loses terminology customization and also conveys a different meaning ("family members").

**Recommendation**: Restore the plural placeholder.

---

#### H12. [MinuteMenu.Database#686](https://github.com/MinuteMenu/MinuteMenu.Database/pull/686) — Foster Income-Eligibility restore too narrow

**Ticket**: [TP #321413](https://minutemenu.tpondemand.com/entity/321413) — Restore Historical IE Dates for Foster Children
**Score**: 80 | **Category**: RequirementWrong / DataIntegrity
**File**: `MMADMIN_XXX/Updates/321413_Restore_Foster_Child_Income_Eligibility.sql:72-101`

WHERE clause narrows to `bk.tier_code = 126 AND bk.tier_start_date IS NOT NULL AND bk.tier_end_date IS NOT NULL`. Three gaps versus ticket scope:
1. Tier I only — foster children who legitimately had Tier II (127) or other tier_code data cleared by #318945 are **not** restored.
2. RCA reference uses `OR` between start/end dates; foster children with only one date populated are skipped.
3. `is_child_income` checkbox is NOT restored — ticket explicitly asks: "Restore checkbox state if applicable".

**Recommendation**: Broaden to `bk.tier_code IS NOT NULL AND (bk.tier_start_date IS NOT NULL OR bk.tier_end_date IS NOT NULL)`. Also restore `is_child_income` from the backup table. Confirm scope with HX102 before release.

---

### Parachute

#### H13. [Parachute#4330](https://github.com/MinuteMenu/Parachute/pull/4330) — `MileageModel.Miles` API contract break (decimal → double)

**Ticket**: [TP #320290](https://minutemenu.tpondemand.com/entity/320290)
**Score**: 85 | **Category**: API Contract
**File**: `Projects/Parachute.Service/Models/MileageModel.cs:13`

`Miles` property type changed from `decimal` to `double`. This alters the JSON serialization contract for `/mileage`. Strongly-typed mobile clients (iOS `Decimal`, Android `BigDecimal`) may fail or lose precision. Values now serialize with IEEE-754 artifacts like `12.340000000000001`.

**Recommendation**: Revert to `decimal` or add `MilesDouble` as a new field while keeping the existing contract. Perform internal conversion only where needed.

---

#### H14. [Parachute#4330](https://github.com/MinuteMenu/Parachute/pull/4330) — `ExpensesController` NullReferenceException on search

**Ticket**: [TP #320290](https://minutemenu.tpondemand.com/entity/320290)
**Score**: 80 | **Category**: Bug
**File**: `Projects/Parachute.Service/Controllers/Expenses/ExpensesController.cs:509`

New predicate `e.Category.Name.ToLower().Contains(req.InputSearch)` without a null guard. `ExpenseModel.Category` is a reference type and the same file elsewhere defensively accesses `category?.Name`.

**Impact**: 500 on search when any expense has a null Category (legacy data, deleted categories).

**Recommendation**: `(e.Category != null && !string.IsNullOrEmpty(e.Category.Name) && e.Category.Name.ToLower().Contains(req.InputSearch))`

---

#### H15. [Parachute#4330](https://github.com/MinuteMenu/Parachute/pull/4330) — `number-input` directive empty→null semantic change

**Ticket**: [TP #320293](https://minutemenu.tpondemand.com/entity/320293)
**Score**: 80 | **Category**: Bug
**File**: `Projects/Parachute.Web/app/common/directives/number-input/number-input.js:86-92`

`$parsers` now returns `null` for empty input when `allowNull=false` (previously fell through to NaN via `parseFloat('')`). Consumers calling `.toFixed()`, arithmetic, or `ng-pattern` on non-nullable models may throw. `$formatters` also changed — fields previously showing "0.00" now render blank.

**Impact**: Cross-cutting — affects kiosk PIN/hours entry, payment amount, payroll hours, expense amount, mileage, recurring invoice amounts. Forms that guard against NaN but not null will silently submit null; `ng-required` now treats empty-valid.

**Recommendation**: Return `undefined` (so required validation still marks invalid), OR keep NaN fallthrough when `!allowNull`. Regression test across kiosk PIN/hours, payroll, payment, expense amount, mileage.

---

#### H16. [Parachute#4330](https://github.com/MinuteMenu/Parachute/pull/4330) — `.search-input` CSS class collides with existing `<input>` usage

**Ticket**: [TP #320291](https://minutemenu.tpondemand.com/entity/320291)
**Score**: 75 (just below High; flagging for visibility)
**File**: `Projects/Parachute.Web/app/common/directives/search-input/search-input.less`

The rewritten stylesheet redefines `.search-input` as a flex wrapper (`display: inline-flex; margin-right: 16px`). This is imported globally. Many existing pages apply `class="search-input"` directly to `<input>` elements (Messages sent view, Payer Invoice List, drag-and-drop lists, UI sandbox, expenses-income main-view). These inputs now receive flex/margin/position that was never intended.

**Recommendation**: Namespace: `app-search-input .search-input { ... }`, OR rename the wrapper class. (Listed here despite 75 score because the blast radius spans unchanged modules.)

---

### Claims Engine

#### H17. [MinuteMenu.Database#686](https://github.com/MinuteMenu/MinuteMenu.Database/pull/686) + [HX#1676](https://github.com/MinuteMenu/HX/pull/1676) — Default flip in `select_rpt_meal_count_attendance_sp`

**Ticket**: [TP #318686](https://minutemenu.tpondemand.com/entity/318686) — Meal Counts & Attendance with FRP missing Breakfast/Lunch
**Score**: 78 (just below High — included for visibility)
**File**: `CXADMIN/Programmability/Stored Procedures/select_rpt_meal_count_attendance_sp.sql:19`

The in-SP default `@respectDisallowance` was flipped from `1` to `0`. Behavior is preserved only because `ISNULL(..., 'N')` fallback re-creates prior behavior when the CLIENT_POLICY row is absent. The pattern is fragile: any future refactor removing the ISNULL silently enables disallowed meals in the FRP report for every CX client.

**Recommendation**: Keep the variable default = `1` and only flip to `0` when the new L11 policy is explicitly `'Y'`. Pattern: `IF @includeAllRecordedMeals = 'Y' SET @respectDisallowance = 0;`.

---

## Infrastructure Impact Assessments

### Batch 1 — Claims Engine & Meal Attendance
**Risk Level**: Medium
**Summary**: Changes are mostly additive but `claim_error_meal_attendance_count_fn` was fully rewritten (ticket asked only for Error 76 infant handling), and `select_rpt_meal_count_attendance_sp` flipped a default — both create latent regression/default-safety risks visible only at scale.
**Details**:
- `claim_error_meal_attendance_count_fn` Error 76 branch adds PROVIDER+SPONSOR_POLICY joins per call → scales with #(claim errors) × #(reprocess runs)
- `claim_error_meal_attendance_source_form_fn` adds MEAL_ATTENDANCE scan — single-scan, fine
- `select_361127_Child_Reconciliation_sp` adds a second full-scan UPDATE over `#PGRID` for RECTYPE='A' (duplicated across MMADMIN_XXX + CXADMIN)
- HX `clsClaimProcessor.cls` refactored for #321408; restored `iMiscGroupOverCount` in disallow formula
- Policy L.11 + policy seed default 'N' (legacy behavior preserved)
- `save_sponsor_policy_sp` parameter-add — safe default; deploy-order caveat applies
**Tables affected**: CLAIM_ERROR, MEAL_ATTENDANCE, CHILD, PROVIDER, SPONSOR_POLICY, CLIENT_POLICY, POLICY, #PGRID
**Rollback difficulty**: Medium (SPs rollback clean; HX client rollback is installer-level)
**Monitoring**: Watch execution time of `select_claim_error_summary_sp`, `select_claim_errors_for_rpt_sp`, `select_361127_Child_Reconciliation_sp`, `select_rpt_meal_count_attendance_sp`; HX crash logs for `IncludeInfantsInOwnAloneEditCheckFlag` binding errors
**DevOps Action Required**: **Yes** — deploy MMADMIN_XXX migrations + `save_sponsor_policy_sp` BEFORE HX installer; monitor post-deploy execution-time regression on Error 76 reports.

---

### Batch 2 — J.017 "Include-Infants" Policy (Cross-Repo)
**Risk Level**: **Critical**
**Summary**: 4-repo schema change (DB, HX API, HX Desktop, KK). Deployment order is NOT self-protecting. Out-of-order deploy = outage or silent data corruption. `save_provider_policy_sp` can silently overwrite `highest_serving_allowed_code` to 215 across all existing providers on next VB6 save. MRP manifest missing the sponsor-030 enablement script — feature will not activate for HX030.
**Tables affected**: SPONSOR_POLICY, PROVIDER_POLICY, STATE_POLICY, STATE_POLICY_TD view; any EF query over SponsorPolicies in HX API.
**Rollback difficulty**: High — HX Desktop installer rollback is per-machine; column-add requires DROP COLUMN for full reversal.
**DevOps Action Required**: **Yes — critical**
1. **Mandatory deploy order**: MMADMIN DB → HX API → HX Desktop → KK. Block reverse order in release pipeline.
2. **Add `320244_SET_SPONSOR_030_INCLUDE_INFANTS_FLAG.sql` to MRP** before ship, OR document manual run with product-owner sign-off.
3. **#320881 regression test**: snapshot `PROVIDER_POLICY.highest_serving_allowed_code` distribution before release; simulate HX Desktop save-with-unchanged-form; diff should show zero changes. If Desktop overwrites to 215, block release.
4. **Post-deploy smoke**: HX Desktop login for HX030, HX102; KK sponsor policy page; Save Sponsor Policy round-trip; OER render on claim with >50 errors.
5. **Monitoring**: HX API for `Invalid column name 'include_infants_in_own_alone_edit_check_flag'` and VB6 ADO "Item cannot be found" in the first 30 min window.

---

### Batch 3 — Sponsor 102 Print Check & Payments
**Risk Level**: **High**
**Summary**: Financial-path change across 4 repos with mandatory deploy order DB → CX → KK → HX API. Wide covering index on multi-million-row CHECK_REGISTER without `ONLINE=ON`. 100k-row browser fetch for dropdown. Unbounded fan-out of 5 SPs per provider on Sponsor 102 print. Cross-repo contract via compiled `MinuteMenu.Centers.ServerLib.dll` + SP signature.
**Tables affected**: CHECK_REGISTER (hot — index DDL window needed), SELECT_ACTIVE_CHILD_*, SELECT_PROVIDER_SERVING_TIMES_*, SELECT_PROVIDER_TIER_*, SELECT_CHILD_TIER_COUNTS_*, SELECT_TRAINING_HOURS_*.
**Rollback difficulty**: Medium-High — `MRP_Rollback.sql` only covers `select_payment_history_list_sp`; other SPs + L11 policy INSERT lack rollback.
**DevOps Action Required**: **Yes**
1. `sp_BlitzIndex 'dbo.CHECK_REGISTER'` in QA before deploy.
2. Add `WITH (ONLINE = ON, MAXDOP = 4)` to 316306 index creation if Enterprise Edition, else maintenance window.
3. Enforce DB → CX → KK → HX API deployment order in runbook.
4. Extend `MRP_Rollback.build.config` before GA OR document that rollback requires DBA intervention.
5. Stage a SQL connection / thread-pool monitor before first Sponsor 102 print post-release.
6. `sys.dm_db_index_usage_stats` check 7 days post-release — drop index if unused.
7. Smoke-test `/cxservice/payments/getCheckRegisterList` contains `modDateTime/modLoginId/modUsername`.

---

### Batch 4 — Enrollment & eForm Cleanup
**Risk Level**: **High**
**Summary**: Two admin endpoints perform serial per-client sweep of eForms + CXADMIN with no timeout guard, unbounded row fetches, hard-deletes based on "orphan = non-Active child" semantics (a common lifecycle event), and shared-secret-only authz. Cross-cutting API shape change (`UpdateIEFChildrenByChildIds` bool → object) and JS notification signature change.
**Rollback difficulty**: Low for individual features; impossible for data hard-deleted via cleanup.
**DevOps Action Required**: **Yes — critical gate**
1. **BLOCK deployment of the cleanup endpoint** until the orphan predicate is tightened (or replaced with explicit orphan ID list).
2. Add explicit `Database.CommandTimeout` (30-60s) on both contexts in the per-client loop.
3. Add `[RequiredRole]` on top of `CheckSecurityKey`; firewall the endpoint to DevOps app pool / IP allow-list.
4. Rotate API security key before/after first prod use. Audit-log every invocation.
5. Run initial sweep off-peak under SQL monitor.
6. Audit `advancedErrorNotification` callers for the title→conf signature change.

---

### Batch 5 — Reports, UI, Localization
**Risk Level**: Medium
**Summary**: Mostly low-risk report swaps, Spanish i18n, UI patches. Introduces the two admin cleanup endpoints (same risk as Batch 4). New covering index on CHECK_REGISTER (same risk as Batch 3). Shared-Core `.rpt` changed without nuspec version bump (process gap, not runtime risk).
**DevOps Action Required**: **Yes**
1. Run `316306_create_check_register_client_center_covering_index.sql` DDL in maintenance window.
2. Firewall/IP-restrict the two new cleanup endpoints.
3. Verify `CHILD_TIER_BACKUP_318945` exists on each HX sponsor DB before running `321413_Restore_Foster_Child_Income_Eligibility.sql`; capture before/after counts; run in explicit transaction.
4. Confirm Shared-Core reporting service website is redeployed (file replacement, not NuGet).
5. Monitor CX API thread count + CXADMIN/EForm connection count for 48h post-deploy.

---

### Batch 6 — Parachute
**Risk Level**: Medium-High
**Summary**: UI refresh (standardized search + cell highlighting) + targeted accounting fixes. Highest risks: silent API contract change on `MileageModel.Miles` (decimal→double), NRE in expenses search, semantic change in shared `number-input` directive reaching beyond the ticket scope (kiosk/payments/payroll), `.search-input` CSS class collision with existing inline usage.
**DevOps Action Required**: **Yes**
1. **Pre-deploy**: verify mobile clients (iOS/Android) handle `double` Miles without precision/crash; gate behind contract version if needed.
2. **Pre-deploy**: apply null guard on `e.Category.Name` in `ExpensesController` — prevents guaranteed 500 on null-category legacy data.
3. **Pre-deploy smoke**: `number-input` clear/empty behavior in kiosk PIN/hours, payment amount, mileage, expense amount.
4. **Pre-deploy visual regression**: pages using `<input class="search-input">` directly (Messages, Payer Invoice List, drag-and-drop, UI sandbox, expenses).
5. **Post-deploy APM**: NullReferenceException on `ExpensesController.GetExpense`; JSON deserialization errors on `/mileage`.
6. **Perf watch**: RUM/LCP on large expense/income tables.
7. **Follow-up**: remove legacy `common/directives/search-input/*` + index.html script tag.

---

## Missing Tickets — No Code Change Found

Two tickets are in the release scope but no matching diff content was found in any of the 7 repos reviewed:

| Ticket | Title | Status | Concern |
|--------|-------|--------|---------|
| [TP #318581](https://minutemenu.tpondemand.com/entity/318581) | Ghost eForm Enrollment Record (Not Started) | Dev | No enrollment-status counting / View Status change in any diff. #319630 orphan framework operates on `EnrollmentAssignmentsCx` only — HX assignments (this ticket is for HX provider 019603522) are not scanned. |
| [TP #321662](https://minutemenu.tpondemand.com/entity/321662) | Unable to disallow Pending children after Withdrawal | Ready For Test | Searched all diffs for "pending/withdraw/disallow/D.19/F.5b" — no matching change. HX claim processor changes are for #321408 and #320244 only. |

**Recommendation**: Confirm with engineering whether these are in scope. If yes, add the missing code. If not, remove from the release entity so the release scope is accurate.

---

## Priority Action Items

### Must Fix (Critical — blocks release)

1. **C1** — [Centers-CX#4161](https://github.com/MinuteMenu/Centers-CX/pull/4161) ([TP #316306](https://minutemenu.tpondemand.com/entity/316306)): Parameterize the ACH `check_id IN (...)`, add scope authorization on `CheckIds`, null-check the request.
2. **C2** — [Centers-CX#4161](https://github.com/MinuteMenu/Centers-CX/pull/4161) ([TP #316306](https://minutemenu.tpondemand.com/entity/316306)): Validate all batch rows share the same `check_date` (or group by date server-side); reduce `CommandTimeout` to 30s.
3. **C3** — [POC_HxCloudConnectApi#1595](https://github.com/MinuteMenu/POC_HxCloudConnectApi/pull/1595) + [MinuteMenu.Database#686](https://github.com/MinuteMenu/MinuteMenu.Database/pull/686) ([TP #320244](https://minutemenu.tpondemand.com/entity/320244)): Enforce deploy order (DB → HX API → HX Desktop) in release runbook. Add app-startup smoke test for the new column.
4. **C4** — [Parachute#4330](https://github.com/MinuteMenu/Parachute/pull/4330) ([TP #320290](https://minutemenu.tpondemand.com/entity/320290)): Restore null guard on `e.PurchasePrice` in `MileageController.cs:113`.

### Should Fix (High — significant risk)

1. **H1** — [MinuteMenu.Database#686](https://github.com/MinuteMenu/MinuteMenu.Database/pull/686) ([TP #320881](https://minutemenu.tpondemand.com/entity/320881)): Scope `save_provider_policy_sp` 215 default to INSERT-only; preserve existing DB value on UPDATE.
2. **H2** — [MinuteMenu.Database#686](https://github.com/MinuteMenu/MinuteMenu.Database/pull/686) ([TP #320244](https://minutemenu.tpondemand.com/entity/320244)): Add `320244_SET_SPONSOR_030_INCLUDE_INFANTS_FLAG.sql` to MRP manifest.
3. **H3** — Cross-repo ([TP #320213](https://minutemenu.tpondemand.com/entity/320213)): Document DB → CX → KK → HX API deploy order. Post-deploy smoke `/cxservice/payments/getCheckRegisterList`.
4. **H4** — [POC_HxCloudConnectApi#1595](https://github.com/MinuteMenu/POC_HxCloudConnectApi/pull/1595) ([TP #320213](https://minutemenu.tpondemand.com/entity/320213)): Add `SemaphoreSlim(10)` and caching to `BuildSponsor102Data` fan-out.
5. **H5** — [POC_HxCloudConnectApi#1595](https://github.com/MinuteMenu/POC_HxCloudConnectApi/pull/1595) ([TP #320213](https://minutemenu.tpondemand.com/entity/320213)): `AsNoTracking()` or detach in `AggregateLinkedClaimsForSponsor102`.
6. **H6** — [POC_HxCloudConnectApi#1595](https://github.com/MinuteMenu/POC_HxCloudConnectApi/pull/1595) ([TP #320213](https://minutemenu.tpondemand.com/entity/320213)): Change `ClaimYear` and `StatusCode` to `int?`.
7. **H7** — [KK#22699](https://github.com/MinuteMenu/KK/pull/22699) ([TP #319630](https://minutemenu.tpondemand.com/entity/319630)): Require `ClientIds`, add `CommandTimeout`, move cleanup to a Hangfire job.
8. **H8** — [KK#22699](https://github.com/MinuteMenu/KK/pull/22699) ([TP #319630](https://minutemenu.tpondemand.com/entity/319630)): Tighten orphan predicate to "no matching CHILD row at all". Require explicit confirm token.
9. **H9** — [KK#22699](https://github.com/MinuteMenu/KK/pull/22699) ([TP #319630](https://minutemenu.tpondemand.com/entity/319630)): Add `[RequiredRole]` on top of `CheckSecurityKey`; firewall behind VPN.
10. **H10** — [KK#22699](https://github.com/MinuteMenu/KK/pull/22699) ([TP #298391](https://minutemenu.tpondemand.com/entity/298391)): Translate `ACTIVE_CHILDREN_WARNING_SETIEF_SKIPPED` to Spanish.
11. **H11** — [KK#22699](https://github.com/MinuteMenu/KK/pull/22699) ([TP #298391](https://minutemenu.tpondemand.com/entity/298391)): Restore `{{participant_kids_plural}}` placeholder in Spanish.
12. **H12** — [MinuteMenu.Database#686](https://github.com/MinuteMenu/MinuteMenu.Database/pull/686) ([TP #321413](https://minutemenu.tpondemand.com/entity/321413)): Broaden `321413_Restore_Foster...sql` WHERE (drop Tier-I filter; OR instead of AND). Restore `is_child_income`.
13. **H13** — [Parachute#4330](https://github.com/MinuteMenu/Parachute/pull/4330) ([TP #320290](https://minutemenu.tpondemand.com/entity/320290)): Revert `MileageModel.Miles` to `decimal` OR add `MilesDouble` as additive.
14. **H14** — [Parachute#4330](https://github.com/MinuteMenu/Parachute/pull/4330) ([TP #320290](https://minutemenu.tpondemand.com/entity/320290)): Null-guard `e.Category.Name` in `ExpensesController.cs:509`.
15. **H15** — [Parachute#4330](https://github.com/MinuteMenu/Parachute/pull/4330) ([TP #320293](https://minutemenu.tpondemand.com/entity/320293)): Review `number-input` empty→null change; either return `undefined` or keep NaN; regression-test all callers.

### Verify With Product (Behavioral changes needing confirmation)

1. **#318686** — Policy L.11 defaults to `'N'` (legacy broken behavior). Sponsor must opt-in per client. Is this intentional or should the fix be auto-applied to the affected client?
2. **#318581 & #321662** — No code change found in any diff. Confirm whether these tickets are meant for 26.4.22 or should be moved out of scope.
3. **#318784** — Fix is a workaround (frontend skips licenseId auto-override on roster reports) rather than reading historical At-Risk data. Confirm product intent.
4. **#319630** — Delivered as general-purpose orphan framework scanning all CX clients. Original ticket was targeted to 2 children on client 8141. Confirm broader scope was intended.
5. **#320290 income date format** — Changed from MM/DD/YY to MM/DD/YYYY on the Income grid. Intentional?
6. **#320293 `deleteFileReceipt` call-site change** — Unrelated to any of the 5 Parachute tickets. Confirm QA tested delete-expense with attached receipt.
7. **#321408 `iSchoolAgedDisallowCount` storage under non-disallow waivers** — New gating may zero out a count that was previously written under `bAttendanceFromTimes=True` + Warn/Ignore waiver. Verify with HX211 scenarios.
8. **#321350 full-function rewrite of `claim_error_meal_attendance_count_fn`** — Non-76 error branches restructured; run side-by-side regression on OER output pre/post for production-scale claim-errors.

---

*Report generated by 18 parallel code-review agents (6 batches × 3 specialized agents: Bug Detection, Architecture & Performance, Requirement Alignment) reviewing 7 release-to-master PRs. Only Critical (90-100) and High (80-89) findings are shown above; approximately 25 additional findings scored 70-79 (Notable) were filtered out — ask to include them if desired.*
