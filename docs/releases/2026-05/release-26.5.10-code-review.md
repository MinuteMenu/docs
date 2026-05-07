# Release 26.5.10 (319522) - Code Review Report

**Date**: 2026-05-06
**Release**: KK PI 26.5.10 — [TP #319522](https://minutemenu.tpondemand.com/entity/319522)
**Repos**: MinuteMenu/KK, MinuteMenu/MinuteMenu.Database, MinuteMenu/Centers-CX
**Scope**: 22 merged PRs (19 tracked, 3 untracked)
**Focus**: Critical (90-100) and High (80-89) severity findings

---

## Executive Summary

| Severity | Count |
|----------|-------|
| **Critical (90-100)** | 6 |
| **High (80-89)** | 12 |
| Total | 18 |

**Untracked changes**: 3 PRs not mapped to any release ticket (KK#22761, MinuteMenu.Database#729, #730)

### Critical Issues (Must Fix Before Release)

| # | PR | Repo | Ticket | Category | Summary |
|---|-----|------|--------|----------|---------|
| C1 | [#22774](https://github.com/MinuteMenu/KK/pull/22774) | KK | [TP #322557](https://minutemenu.tpondemand.com/entity/322557) | DataIntegrity | Separator mismatch (`-` vs `_`) defeats new-child conflict-merge path |
| C2 | [#22763](https://github.com/MinuteMenu/KK/pull/22763) | KK | [TP #322375](https://minutemenu.tpondemand.com/entity/322375) | DataIntegrity | Edit on Pending child overwrites `never_activated_flag` back to false |
| C3 | [Centers-CX#4142](https://github.com/MinuteMenu/Centers-CX/pull/4142) | CX | [TP #317707](https://minutemenu.tpondemand.com/entity/317707) | DataIntegrity | `Inf611Cereal -> Inf611Bread` fix only in `LogicServiceToKK.cs`, not `LogicService.cs`; also untracked scope |
| C4 | [#22599](https://github.com/MinuteMenu/KK/pull/22599) | KK | [TP #317707](https://minutemenu.tpondemand.com/entity/317707) | RequirementWrong | DLL-only PR has no source audit trail / version pin |
| C5 | [MinuteMenu.Database#725](https://github.com/MinuteMenu/MinuteMenu.Database/pull/725) + [#726](https://github.com/MinuteMenu/MinuteMenu.Database/pull/726) | DB | [TP #320035](https://minutemenu.tpondemand.com/entity/320035) | CrossRepoImpact | Cross-repo deploy order: KK before DB silently writes `0` instead of `NULL` to `provider_capacity` |
| C6 | [MinuteMenu.Database#727](https://github.com/MinuteMenu/MinuteMenu.Database/pull/727) | DB | [TP #322254](https://minutemenu.tpondemand.com/entity/322254) | InfrastructureStress | `CREATE UNIQUE INDEX` on `KK_EnrollmentReport` lacks `ONLINE = ON` — risks write outage on multi-million-row table |

---

## Critical Findings (Detail)

### C1. [KK#22774](https://github.com/MinuteMenu/KK/pull/22774) — Separator mismatch defeats new-child conflict-merge path

**Ticket**: [TP #322557](https://minutemenu.tpondemand.com/entity/322557) — Centers: Meal Data Still Disappears After Save - Two Remaining Bugs from PI 313113
**Score**: 92 | **Category**: DataIntegrity / Bug
**File**: `Projects/KidKare.Web/app/states/classrooms/index/classrooms-controller.js` (lookup ~lines 648-704); server side `Projects/KidKare.Bll/.../AttendanceBll.cs:1447`

The PR fixes #22764's broken keying by re-keying `latestChildrenMap` to `centerChildId` (composite "centerId_childId" string). The existing-child path on the server (`AttendanceBll.cs:1458`) produces `CenterChildId = centerId + "_" + childId` (underscore). However the **new-child path at `AttendanceBll.cs:1447`** still uses `"-"` as the separator: `parameters.CenterId.ToString() + "-" + latest.Id.ToString()`. When another user adds a child in parallel that is not in the current user's snapshot, that record arrives with a dash-separated key and the underscore-keyed lookup misses; the conflict-merge branch silently bypasses, and the user's stale snapshot overwrites the new child's record on save.

**Impact**: The exact concurrency scenario this PR claims to fix continues to drop one user's update for newly-added children, reproducing the "meal data disappears" symptom. This is a Critical defect because the PR is the third iteration on this code path (PI 313113 → 322557 → #22764 → #22774).

**Recommendation**: Standardize the server separator — change `AttendanceBll.cs:1447` from `"-"` to `"_"` so both new-child and existing-child paths match the client's lookup key. Add a unit test that asserts `latestChildrenMap[child.childId]` resolves on the composite-string key for both paths.

=> This is a special case and seems unlikely to occur, so I don’t think we need to fix it.
---

### C2. [KK#22763](https://github.com/MinuteMenu/KK/pull/22763) — Edit on Pending child overwrites `never_activated_flag` back to false

**Ticket**: [TP #322375](https://minutemenu.tpondemand.com/entity/322375) — never_activated_flag not saved for Center-enrolled Pending children causes Withdrawn-never-activated claims to bypass Rule44
**Score**: 92 | **Category**: DataIntegrity / Bug
**File**: `Projects/KidKare.Bll/Centers/Child/ChildBll.cs:1583-1584`, `Projects/KidKare.Bll/Utils/Mappings/CxChildMapping.cs:301`

Removing `.Ignore()` at `CxChildMapping.cs:301` plus the explicit assignment at `ChildBll.cs:1584` causes every UPDATE to write `child.NeverActivatedFlag` from the request DTO. `model.Child` deserialized from JSON in `ChildController.UpdateChild` does NOT receive `NeverActivatedFlag` from the UI. For any non-create edit on a still-Pending child (guardian edits, classroom changes, IEF updates), `child.NeverActivatedFlag` defaults to `false` and OVERWRITES the previously-set DB value of `true`. `IgnoreInvalidFields` does not preserve this field from the DB row. This re-introduces the exact bug the fix targets: any subsequent edit before activation flips the flag back to `0`, allowing Withdrawn-never-activated claims to bypass Rule44.

**Impact**: The Rule44 bypass remains exploitable any time a Pending child is edited (e.g., contact change) before activation/withdrawal. The fix only works if no edit ever occurs between AddChild and Withdrawal — unrealistic in production.

**Recommendation**: Read existing `never_activated_flag` from `childRow` into `updatedDatasetRow` before mapping, e.g., `updatedDatasetRow.never_activated_flag = childRow.never_activated_flag;`, and only override on the auto-activate path or for new children. Alternatively, preserve the value in `IgnoreInvalidFields`.

=> It's OK
---

### C3. [Centers-CX#4142](https://github.com/MinuteMenu/Centers-CX/pull/4142) — `LogicService.cs` missing the `Inf611Cereal -> Inf611Bread` fix; also untracked scope change

**Ticket**: [TP #317707](https://minutemenu.tpondemand.com/entity/317707) — CX 3792: Actual Quantity Served is no longer automatically calculated for Non-Infant menus
**Score**: 95 | **Category**: DataIntegrity / RequirementWrong
**File**: `products/Centers/Projects/MinuteMenu.Centers.ServerLib/Logic/LogicServiceToKK.cs:7637` (fixed) and `LogicService.cs:~7622` (NOT fixed)

Two findings consolidated:

1. **Cross-file inconsistency**: `LogicServiceToKK.cs:7637` was changed from `Constants.MealPatternItemCode.Inf611Cereal` to `Constants.MealPatternItemCode.Inf611Bread` (correctly assigning `BreadAltOzEqRequiredQty` to the Bread slot, not the Cereal slot). The structurally-identical `LogicService.cs` was NOT changed. If `LogicService.cs` is reachable from any code path (it has the same public method signature), the bug remains: bread/grain ounce-equivalent quantity is written to the cereal slot, corrupting infant meal-pattern actuals.

2. **Scope creep / requirement-extra**: The Cereal → Bread item-code switch is unrelated to the ticket's stated goal ("auto-calculated for **Non-Infant** menus") — line 7637 is the **infant 6-11 bread** mapping. This is a separate piggy-back fix without test coverage and not mentioned in the PR body.

**Impact**: For sites whose CACFP claim flow invokes `LogicService.cs`, the infant 6-11 cereal column is overwritten with the bread quantity, wiping the cereal entry and inflating cereal counts on disallowance/audit reports — CACFP-claim financial/compliance-data corruption.

**Recommendation**:
- Apply the same `Inf611Cereal` → `Inf611Bread` rename in `LogicService.cs` within the OzEq fallback block, OR confirm `LogicService.cs` is dead code and remove it. Verify routing via DI/factory bindings before merge.
- File a follow-up TP ticket to track the typo correction as its own entry with regression tests.
- Refactor the duplicated `LogicService.cs` / `LogicServiceToKK.cs` (structural code-smell that has caused historical drift bugs).

=> Reply
- Finding 1: Apply rename in LogicService.cs — NOT FIX
  The typo does NOT exist in LogicService.cs. The two files diverged structurally for the bread oz-eq path:
  
  LogicServiceToKK.cs:7637 (had typo, fixed): setMptItems(mptItems, Inf611Cereal, 1129, BreadAltOzEqRequiredQty.Value)
  LogicService.cs: writes bread oz-eq via SaveClaimMenuOzEq(...bread_food_id, ...) direct DB write — never had the typo
  
  $ grep -n "Inf611Cereal,\s*\d" LogicService.cs
  (no matches)
  Both Inf611Cereal references in LogicService.cs (lines 7571, 7578) correctly write cereal data. The CACFP corruption scenario described in the impact cannot occur via this file.

- Finding 2: Scope creep / piggy-back fix — NOT FIX (acknowledged)
  Correct that the typo is unrelated to the non-infant ticket goal. Kept in-place because:
  
  Caught by automated 5-agent review on this PR (Agent 3 flagged it at 90/100 confidence)
  1-character identifier rename, no logic change
  Splitting to a separate ticket = another PR + DLL build + deploy for a trivial fix
  Documented in commit 70bd565 (with code comment) and the TP #317707 implementation summary.

---

### C4. [KK#22599](https://github.com/MinuteMenu/KK/pull/22599) — DLL-only PR has no source audit trail

**Ticket**: [TP #317707](https://minutemenu.tpondemand.com/entity/317707) — CX 3792: Actual Quantity Served is no longer automatically calculated for Non-Infant menus
**Score**: 92 | **Category**: RequirementWrong
**File**: `ExternalReferences/MinuteMenu.CX.Services/MinuteMenu.Centers.*.dll` (binary-only)

PR contains only updated binary DLLs in `ExternalReferences/MinuteMenu.CX.Services/` — no source diff, no commit message detail, no build metadata. The actual fix lives in the paired Centers-CX#4142. Without an accompanying source-side change in the KK repo (release-notes / version-pin / DLL-update commit message documenting which Centers-CX commit produced these binaries), the binary swap is not auditable. Recent commit `a55775cee2 317707 build dll from 26.5.10` confirms a manual DLL drop. If CX#4142 is correct but a stale DLL is committed, the KK repo will silently regress.

**Impact**: If the DLL was not built from the merged commit of CX#4142 (post-merge SHA), production receives a different binary than the reviewed source. Combined with C3, this is the highest-risk finding in the release.

**Recommendation**:
- Verify the DLL artifact was built from the merged CX#4142 SHA (post-merge, not from a feature branch).
- Add a small text artifact (e.g., `ExternalReferences/MinuteMenu.CX.Services/VERSION.txt` or PR description) recording the source commit SHA and build timestamp.
- Lock KK#22599 ↔ CX#4142 as a single deploy unit; rollback must roll both back.
=> The DLL has been built.

---

### C5. [MinuteMenu.Database#725](https://github.com/MinuteMenu/MinuteMenu.Database/pull/725) + [#726](https://github.com/MinuteMenu/MinuteMenu.Database/pull/726) — Cross-repo deploy-order silently corrupts `provider_capacity`

**Ticket**: [TP #320035](https://minutemenu.tpondemand.com/entity/320035) — Provider Capacity - Discrepancy between KK and HX
**Score**: 90 | **Category**: CrossRepoImpact
**File**: `MMADMIN_XXX/Programmability/Stored Procedures/save_provider_capacity_sp_port_kk.sql`; `MMADMIN_XXX/MRP.build.config`

[KK#22769](https://github.com/MinuteMenu/KK/pull/22769) explicitly depends on the new SP's `0 → NULL` normalization to translate UI-zeroed hidden fields into NULL. If KK ships first (or DB rollback runs while KK forward is live), the UI saves literal `0` into `provider_capacity` columns for every hidden field — corrupting data where consumers (reports, eligibility, max-capacity calc) distinguish NULL from 0. PR #726 wires the SP into the MRP — without #726, #725's normalization never reaches production even though it's merged.

The KK code has no direct SP reference (calls service layer), so the contract is implicit and undocumented at the API boundary.

**Impact**: A KK-only deploy (without DB) inverts the bug — KK now actively writes 0 instead of "leaving it alone". Rollback of DB without rolling back KK does the same. Recovery requires manual `UPDATE ... SET col = NULL WHERE col = 0` audit.

**Recommendation**:
- DevOps must deploy MinuteMenu.Database#725 + #726 BEFORE or atomically with KK 26.5.10. Document this as a coordinated release in the deploy ticket.
- Block KK-only hotfix deploy of #22769.
- Make rollback all-or-nothing (DB rollback must be paired with reverting KK#22769 deploy).
- Add a runbook smoke test: save a capacity with `0` in a visible field and verify DB column is NULL post-deploy.
- Consider a temporary monitoring query to alert if `provider_capacity` rows are inserted with non-NULL `0` in any of the normalized columns (would indicate the SP didn't deploy).

=> It's OK
---

### C6. [MinuteMenu.Database#727](https://github.com/MinuteMenu/MinuteMenu.Database/pull/727) — `CREATE UNIQUE INDEX` lacks `ONLINE = ON` on hot table

**Ticket**: [TP #322254](https://minutemenu.tpondemand.com/entity/322254) — Data Inconsistency – "Submitted (Site)"/"Submitted (Parent)" eForms Missing from Approve & Renew Queue
**Score**: 90 | **Category**: InfrastructureStress
**File**: `MMADMIN_EFORM/Updates/322254_unique_index_kk_enrollment_report.sql:6-10`

`CREATE UNIQUE NONCLUSTERED INDEX UX_KK_EnrollmentReport_LogicalKey ... WHERE InvitationId IS NOT NULL` is created **without `WITH (ONLINE = ON)`**. `KK_EnrollmentReport` is the high-volume eForm output table (cleanup script targets ~115 rows by Id but Ids run up through 2.2M+, indicating millions of rows). A non-online unique index build holds a Sch-M lock on the table for the duration of the build, blocking all reads and writes — every eForm save flow stalls across all 15 API VMs.

**Impact**: Production eForm outage window proportional to row count; eForm save/submit/Approve & Renew/View Status all write here. Potential timeouts cascading to RateLimiter delayed-report consumers.

**Recommendation**:
- Add `WITH (ONLINE = ON, MAXDOP = 4)` (or `RESUMABLE = ON` on Azure SQL/Enterprise).
- Confirm row count and run during a maintenance window if Standard Edition (no ONLINE).
- DevOps deploy order: cleanup DML → index DDL → KK code deploy. Add filename prefixes (`01_`, `02_`, `03_`) to enforce sort order.
- Add a `SELECT COUNT(*) ... HAVING COUNT(*)>1` precheck before `CREATE UNIQUE INDEX` to fail fast with a clear message if any duplicates remain.
- Add a smoke check post-DB-deploy that `UX_KK_EnrollmentReport_LogicalKey` exists before promoting KK#22773.
=> It's OK
---

## High Severity Findings (80-89)

### Provider Capacity (320035) — KK + DB

#### H1. [MinuteMenu.Database#726](https://github.com/MinuteMenu/MinuteMenu.Database/pull/726) — Rollback path triggers data corruption without atomic KK rollback

**Ticket**: [TP #320035](https://minutemenu.tpondemand.com/entity/320035) — Provider Capacity Discrepancy
**Score**: 85 | **Category**: CrossRepoImpact
**File**: `MMADMIN_XXX/MRP.build.config`; `MMADMIN_XXX/.../save_provider_capacity_sp_port_kk.sql:8`

If DB rollback executes (#725/#726) without rolling back KK#22769, capacity saves write `0` permanently into rows. Recovery requires a manual `UPDATE ... SET col = NULL WHERE col = 0` audit. The `CREATE OR ALTER PROCEDURE` change is correct (idempotent) but the rollback file reverts the SP body to the prior version.

**Impact**: Whole-release coordinated rollback is required; partial rollback silently corrupts data on every Licensing save.

**Recommendation**: Tie KK release 26.5.10 to this SP version explicitly in the runbook. Document atomic rollback requirement.
=> It's OK, not fix
---

#### H2. [KK#22765](https://github.com/MinuteMenu/KK/pull/22765) — Extra licensing init round-trip on every Licensing-tab save

**Ticket**: [TP #322928](https://minutemenu.tpondemand.com/entity/322928) — Maximum Capacity should be reloaded after save (sub-bug of 320035)
**Score**: 80 | **Category**: DataAccessChange
**File**: `Projects/KidKare.Web/app/states/sponsor/enroll-provider/hx-enroll-provider/hx-enroll-provider-controller.js:1182`

Every Licensing-tab save now triggers an extra round-trip through `licensingTab.init()` → `getProviderLicensingByStateCode` to refresh the "Maximum Capacity" label. The save endpoint returns only `{code}`, so the UI re-fetches the whole licensing payload after every save.

**Impact**: Doubles DB+API load on the Licensing-tab save path. Compounds with #725's added procedural branches under load.

**Recommendation**: Have the save endpoint return the new `maxCapacity` directly (1-line server change), or limit the re-init to fields that actually changed.

---

#### H3. [KK#22769](https://github.com/MinuteMenu/KK/pull/22769) — Visibility-driven mass-zeroing creates implicit semantic coupling to SP

**Ticket**: [TP #320035](https://minutemenu.tpondemand.com/entity/320035) — Provider Capacity Discrepancy
**Score**: 80 | **Category**: DataAccessChange / DataIntegrity
**File**: `Projects/KidKare.Web/app/states/sponsor/enroll-provider/hx-enroll-provider/hx-enroll-provider-controller.js:3124-3170` (`getProviderCapacityForSave`)

Client mass-zeros hidden capacity columns on every save, relying on the new SP's `0 -> NULL` normalization to translate that into NULL in DB. UI visibility flags (`licensingTab.*.visible`) are now load-bearing for data integrity. If a future UI bug mis-evaluates a `.visible` flag (e.g., undefined during slow network init), the client silently zeros/nulls real data on save. Also: opening a legacy provider in KK and clicking Save without changing anything will silently zero out hidden-but-populated fields.

**Impact**: Possible silent data loss on save for any provider whose DB row contains values in fields that the currently-selected license hides. User receives no warning.

**Recommendation**: Send an explicit "fields-to-clear" list rather than overloading `0` as a magic clear-token. Prompt the user when hidden fields contain non-zero values that will be cleared. Verify with HX product owner that destructive normalization is intended for the legacy-data case.

---

#### H4. [KK#22767](https://github.com/MinuteMenu/KK/pull/22767) — Snapshot stores REQUEST payload (zeros) instead of server response, drifting from DB

**Ticket**: [TP #320035](https://minutemenu.tpondemand.com/entity/320035) — Provider Capacity Discrepancy
**Score**: 80 | **Category**: DataIntegrity
**File**: `Projects/KidKare.Web/app/states/sponsor/enroll-provider/hx-enroll-provider/hx-enroll-provider-controller.js:1190-1196`

After save, code does `$scope.providerCapacity = angular.copy(params.providerCapacity)` — but `params.providerCapacity` is the **request payload**, not the server response. With H3 zeroing hidden fields and the SP normalizing `0 → NULL` server-side, the in-memory `$scope.providerCapacity` and the snapshot hold `0` for hidden fields while the DB row holds `NULL`. The combination diverges from the persisted truth.

**Impact**: Snapshot/originalCapacity drift from DB after save. License-type round-trip post-save can re-introduce `0` values into a license type that interprets them differently. Concurrency-check logic comparing originalProviderCapacity to DB will produce false negatives.

**Recommendation**: Either (a) have `saveEnrollProviderWizard` return the canonical post-save row and copy `response.providerCapacity` into the scope/snapshot, or (b) re-fetch the provider's capacity row inside the licensing-tab post-save handler before refreshing the snapshot.

---

### Centers Meal & Attendance (322557, 318492, 320884, 317707)

#### H5. [KK#22599](https://github.com/MinuteMenu/KK/pull/22599) — DLL drop creates hard cross-repo deployment-order coupling

**Ticket**: [TP #317707](https://minutemenu.tpondemand.com/entity/317707) — Auto-calc for Non-Infant menus
**Score**: 88 | **Category**: CrossRepoImpact
**File**: `ExternalReferences/MinuteMenu.CX.Services/MinuteMenu.Centers.ServerLib.dll` (binary)

PR is a binary-only DLL drop — 7 DLLs updated, no source code in KK repo. The actual fix lives in Centers-CX#4142 (`LogicServiceToKK.cs` + duplicated in `LogicService.cs`). The build artifact must be produced from the Centers-CX merge SHA before KK#22599 merges. Hidden risk: the empty-catch in `AttendanceBll.UpdateAutoCalculateActual:1204-1212` (Task.Run swallow) is **not** addressed — any future failure here remains invisible. This is the same swallow-and-hide pattern that hid PI 318492 for weeks.

**Impact**: If KK is rolled back independently, the stale DLL reintroduces the `continue`-skips-`Add` defect.

**Recommendation**:
- Lock KK#22599 ↔ CX#4142 as a single deploy unit; rollback must roll both back.
- File a follow-up ticket to remove the empty `catch` in `AttendanceBll.cs:1204-1212`.

---

#### H6. [Centers-CX#4142](https://github.com/MinuteMenu/Centers-CX/pull/4142) — Untracked typo fix + duplicate code drift

**Ticket**: [TP #317707](https://minutemenu.tpondemand.com/entity/317707) — Auto-calc for Non-Infant menus
**Score**: 80 | **Category**: CrossRepoImpact / DataIntegrity
**File**: `products/Centers/Projects/MinuteMenu.Centers.ServerLib/Logic/LogicServiceToKK.cs:7537-7672` AND `LogicService.cs:7525-7558`

Two structurally-identical files (`LogicService.cs` and `LogicServiceToKK.cs`) carry the same public method `SaveActualQuantitiesAsRequired`. The null-guard refactor (replacing `if (rowForInfantActual == null) continue` with a null-guarded block) is applied to BOTH. The unrelated `Inf611Cereal → Inf611Bread` typo correction at line 7637 is applied to **only one** (see C3). Verify call site of `actualQuantities.Add` is unconditional and that empty `mptItems` for infant menus does not generate spurious zero rows where the prior code would have skipped — flipping "no record" to "zero" can break downstream reconciliation logic.

**Impact**: Duplicate code paths drift out of sync — environment-specific bugs.

**Recommendation**: Add a unit test for `policyM16 && rowForInfantActual == null` with mixed infant/non-infant menu sets. Refactor duplicated files in a follow-up ticket.

---

#### H7. [KK#22650](https://github.com/MinuteMenu/KK/pull/22650) — Backend guard broadened (defense-in-depth, net-positive but data correctness depends on flags)

**Ticket**: [TP #318492](https://minutemenu.tpondemand.com/entity/318492) — Children automatically marked in meal attendance
**Score**: 82 | **Category**: DataAccessChange
**File**: `Projects/KidKare.Bll/Centers/Services/ClaimAttendanceService.cs:285-298`

Extends the L2=Y phantom-child guard to also cover L2=N. New `HasAnyMealFlag` helper iterates `child.Meals.Values` (max 6 meals/child) — purely in-memory, no extra query. Net effect REDUCES insert volume into `cxadmin.MEAL_RECORD` / `claim_attendance` because prior code created CLAIM_ATTENDANCE rows for every child in the payload regardless of `IsIn`. Genuine load reducer for L2=N centers (majority of CX Centers).

**Impact**: Net-positive load. Risk is the inverse — a center where the payload genuinely needs a row created for an `IsIn=false` child with no meal flags would now be skipped.

**Recommendation**:
- Verify QA TC4 (regression L2=N check-in 1) and TC11 (infants unaffected) executed.
- DBA-review and gated execution of the prepared cleanup query for existing phantom CLAIM_ATTENDANCE rows.
- Schedule monitoring on `claim_attendance` insert volume for L2=N centers post-deploy — should DROP.

---

### eForms (322254, 322840)

#### H8. [KK#22773](https://github.com/MinuteMenu/KK/pull/22773) — Retry handler tightly coupled to DB index name; fail-open if name renamed

**Ticket**: [TP #322254](https://minutemenu.tpondemand.com/entity/322254) — eForms Missing from Approve & Renew
**Score**: 80 | **Category**: CrossRepoImpact
**File**: `Projects/KidKare.Bll/Enrollment/ReportEnrollmentBll.cs:1408-1500`

Retry catches only `DbUpdateException` whose inner `SqlException.Number` is 2627/2601 **and** message contains the literal `"UX_KK_EnrollmentReport_LogicalKey"`. If KK#22773 deploys before DB#727, retry never engages — duplicates revert to prior broken behavior. If the index name is ever renamed in DB, the catch becomes a silent no-op.

**Impact**: Tight coupling between code constant and DB index name. Deploy-order or rename can silently break the retry contract.

**Recommendation**:
- Mandate DB#727 deploys first — gate KK#22773 deploy on a smoke check that the index exists.
- Extract the index name to a single shared constant referenced by both retry code and migration script.
- Add an APM/log warning when the catch fires so duplicate rate is observable.

---

### Tier/Eligibility & Provider/School-Age (322375, 305316, 322555)

#### H9. [KK#22763](https://github.com/MinuteMenu/KK/pull/22763) — AutoMapper Ignore-removal broadens write-path beyond the targeted scenario

**Ticket**: [TP #322375](https://minutemenu.tpondemand.com/entity/322375) — never_activated_flag not saved for Center-enrolled Pending children
**Score**: 80 | **Category**: DataAccessChange
**File**: `Projects/KidKare.Bll/Utils/Mappings/CxChildMapping.cs:301`

Removing `Ignore()` on `never_activated_flag` means EVERY `Mapper.Map<dsChild.CHILDRow>(child)` and `Mapper.Map(child, updatedDatasetRow)` call now writes the flag from the model — not only the AddChild path. Combined with C2, this affects all CX child save/update flows (Add, Update, Reactivate, Withdraw, Import). Two competing sources of truth on the update path (mapper auto-write vs explicit `childTable.Rows[0].never_activated_flag = false` at line 957).

**Impact**: All CX child save flows are at risk. See C2 for the worst case.

**Recommendation**: Either (a) keep Ignore on the forward map and instead set `childRow.never_activated_flag` explicitly in AddChild after the Map call, or (b) audit every caller to confirm `NeverActivatedFlag` is hydrated before write. Add a unit test verifying Update path does not regress the flag.

---

#### H10. [MinuteMenu.Database#698](https://github.com/MinuteMenu/MinuteMenu.Database/pull/698) — `select_max_capacity_fn2` semantic change is silent

**Ticket**: [TP #305316](https://minutemenu.tpondemand.com/entity/305316) — Maximum capacity not updated when school-aged unavailable
**Score**: 80 | **Category**: DataAccessChange
**File**: `MMADMIN/Programmability/Functions/select_max_capacity_fn2.sql`

Three logic changes in a single PR: (1) added LEFT JOIN on LICENSE with COALESCE for new override column, (2) flipped strict `>` to `>=` in the @HighestProviderAgedCnt branch (CHANGES results when two age-group counts tie), (3) replaced `@PreschoolCnt` (preschooler only) with new `@preschoolAgeCnt = preschooler + school_age` in the same branch. Function is consumed by tier/capacity displays/reports across all providers.

**Impact**: Silent change to displayed max-capacity numbers. Could cascade to Tier policy decisions if max capacity is consumed by Tier eligibility queries.

**Recommendation**: Confirm with QA/business owner that `>=` and `preschoolAge` semantics are intentional. Produce before/after counts of providers whose `select_max_capacity_fn2` output changes (run on PROD-LIKE). Document the semantic shift in release notes.

---

#### H11. [MinuteMenu.Database#698](https://github.com/MinuteMenu/MinuteMenu.Database/pull/698) — ROLLBACK scripts are no-ops (identical to forward)

**Ticket**: [TP #305316](https://minutemenu.tpondemand.com/entity/305316)
**Score**: 85 | **Category**: ScopeCreep / Rollback
**File**: `MMADMIN/ROLLBACK/305316_ADD_COMPUTE_METHOD_TO_PROVIDER_CAPACITY.sql`; `MMADMIN/ROLLBACK/305316_ADD_HISTORIABLE_FIELD_FOR_OVERRIDE.sql`

Both rollback scripts are byte-identical to the forward Update scripts — they re-add the column / re-insert the HISTORIABLE_FIELDS row. True rollback would `ALTER TABLE ... DROP COLUMN` and `DELETE FROM HISTORIABLE_FIELDS`. Function rollback is missing entirely (would need to restore prior `select_max_capacity_fn2` body).

**Impact**: If 26.5.10 must be reverted in production, executing MRP_Rollback will not undo this PR's schema change; DBAs will need a manual hand-written drop.

**Recommendation**: Author proper rollback scripts BEFORE deploy:
```sql
ALTER TABLE dbo.PROVIDER_CAPACITY DROP COLUMN displayed_max_capacity_compute_method_code;
DELETE FROM HISTORIABLE_FIELDS WHERE HISTORIABLE_FIELD_NAME = 'displayed_max_capacity_compute_method_code';
```
Plus the prior function definition for `select_max_capacity_fn2`.

---

#### H12. [MinuteMenu.Database#723](https://github.com/MinuteMenu/MinuteMenu.Database/pull/723) + [#728](https://github.com/MinuteMenu/MinuteMenu.Database/pull/728) — One-off data correction with no rollback artifact + scope creep

**Ticket**: [TP #322555](https://minutemenu.tpondemand.com/entity/322555) — Client 166 correct provider number for review
**Score**: 85 | **Category**: ScopeCreep
**File**: `MMADMIN/Updates/322555_Fix_Review_Provider_Id.sql`; `MMADMIN_XXX/Updates/322555_Fix_Review_Provider_Id.sql`

Two-database one-off data correction. Neither MRP.build.config nor MRP_Rollback.build.config is updated, so the script is NOT part of the MRP package — it relies on manual DBA execution. PR #728 amends the same script to ALSO update `dbo.PROVIDER.next_review_required_date = '2026-05-29'` for the corrected provider — the ticket asked only to correct the provider number on the review record, not modify scheduling data on PROVIDER. The script REPLACEs a substring inside `RawSubmission` (likely JSON/XML payload) and overwrites `Value` columns; original values not preserved.

**Impact**: Rollback difficulty HIGH. If the wrong record is hit, the change is unrecoverable without backup restore.

**Recommendation**:
- Capture pre-image: `SELECT * INTO bak_322555_ReviewSubmission FROM ReviewSubmission WHERE ReviewSubmissionId = 30648878` (and same for ReviewSubmissionAnswer and PROVIDER).
- Add a WHERE-clause guard to verify the substring exists pre-replace.
- Include execution checklist in PR description (which DB instance, who runs it, when).
- Confirm with the requester (client 166) whether the `next_review_required_date` change is intended; document rationale and date source in script header.

---

### Untracked Changes (322954)

#### H13. [MinuteMenu.Database#729](https://github.com/MinuteMenu/MinuteMenu.Database/pull/729) — Not registered in MRP; manual DBA deploy required

**Ticket**: No ticket — branch ref 322954 is not in release ticket list
**Score**: 80 | **Category**: InfrastructureStress
**File**: `MMADMIN_EFORM/Updates/322954_kkeform_MI_tuning_to_stop_Win32Exception_bursts_under_claim_week_peak.sql`; `MMADMIN_EFORM/MRP.build.config`

Script is dropped into `MMADMIN_EFORM/Updates/` but `MRP.build.config` only lists `MRP.seed.sql`. Generated `RELEASES/MRP.sql` (16 lines, regenerated 2026-04-08) does NOT include any `Updates/*` scripts. There is no automated path for this index to ship with the release — must be applied manually by a DBA, on a per-customer/per-database basis.

**Impact**: Release ships with a fix that quietly does not deploy. Win32Exception bursts continue on un-patched MIs during claim week. Inconsistent state across tenants makes debugging harder.

**Recommendation**: Either (a) add 322954 entries to `MMADMIN_EFORM/MRP.build.config` and rebuild `RELEASES/MRP.sql`, or (b) create a tracked TP ticket and DBA runbook explicitly listing the script + MI(s) it must run against, with verification SQL.

---

#### H14. [MinuteMenu.Database#729](https://github.com/MinuteMenu/MinuteMenu.Database/pull/729) — Index addition without execution-plan justification; possible misdiagnosis of Win32Exception

**Ticket**: No ticket
**Score**: 80 | **Category**: InfrastructureStress
**File**: `MMADMIN_EFORM/Updates/322954_kkeform_MI_tuning_*.sql:7-10`

PR title attributes Win32Exception bursts to indexing — but classic Win32Exception bursts on MI under load are typically connection-layer issues (TLS handshake failures, TCP exhaustion, ADO.NET pool starvation, `Min Pool Size`/`Max Pool Size`, ARR/load balancer reset), NOT missing indexes. PR body provides no execution plan, no missing-index DMV evidence, and no expected query. This matches Pattern 3/4 (connection leak under error path, oversized timeouts) which is NOT addressed.

**Impact**: If Win32Exception is a connection-pool / socket error, this PR misdiagnoses the symptom and adds permanent write cost on the heaviest write window of the year (claim week) without fixing the bursts.

**Recommendation**: Capture during the next burst: server `sys.dm_os_wait_stats` delta, `sys.dm_exec_connections`/`sys.dm_exec_sessions` count vs pool max, app-side connection-pool counters, full Win32Exception stack. Confirm whether bursts are SQL-side or network/pool-side BEFORE adding any further indexes. Treat 322954 as Open in TP until evidence-based RCA is posted.

---

#### H15. [MinuteMenu.Database#730](https://github.com/MinuteMenu/MinuteMenu.Database/pull/730) — Fast-follow indicates #729 was insufficient; root cause not identified

**Ticket**: No ticket
**Score**: 85 | **Category**: ScopeCreep
**File**: `MMADMIN_EFORM/Updates/322954_kkeform_may5_burst_followup_indexes.sql`

#729 merged 2026-05-05 08:09 UTC; #730 merged 2026-05-06 04:11 UTC — under 24 hours later, after the symptom recurred ("may5_burst_followup"). The pattern indicates #729 did not solve the root cause; the team is iterating in production by guessing-and-adding indexes. Branch name literally `…Win32-create-some-index`.

**Impact**: If #730 also misses the root cause, a #731 will land mid-PI. Each iteration adds permanent write cost. Confidence in the "fix" is low.

**Recommendation**: Pause iterative-index approach until evidence-based RCA. Treat 322954 as Open. Release notes should NOT advertise 322954 as resolved.

---

#### H16. [MinuteMenu.Database#730](https://github.com/MinuteMenu/MinuteMenu.Database/pull/730) — Composite index leading column does not match query patterns

**Ticket**: No ticket
**Score**: 80 | **Category**: InfrastructureStress
**File**: `MMADMIN_EFORM/Updates/322954_kkeform_may5_burst_followup_indexes.sql:8-12`

Index keyed `(ChildId, ClientId, CenterId)`. ChildId is per-child (high cardinality, narrow predicate), but most known queries against `KK_EnrollmentCompletedCx` filter by `(ClientId, CenterId)` first and then narrow by ChildId — see `321432_restore_reports_after_renew.sql` and `315703_Unable_to_send_new_eForm_invitation_for_a_chlid.sql` patterns. With ChildId leading, queries that only know ClientId/CenterId (the eForm dashboard, invitations, restore-after-renew) cannot seek and will scan or be ignored by the optimizer, while every write incurs maintenance.

**Impact**: New permanent write cost without query benefit; bursts not relieved — accelerates cycle into #731.

**Recommendation**: Validate the actual query that motivates the index (post execution plan in the PR). Re-key as `(ClientId, CenterId, ChildId)` if the dashboard/list path is the target. Consider a filtered index excluding `RemoveDate IS NOT NULL`.

---

## Infrastructure Impact Assessments

This section consolidates Agent B (Architecture & Performance) findings per batch — designed to be shared with DevOps before deployment.

### Provider Capacity (320035) — KK + DB

**Risk Level**: **High**
**Summary**: Coordinated KK+DB change with hard cross-repo deployment-order dependency. KK frontend now actively zeros hidden capacity fields and relies on a new DB SP `0 → NULL` normalization to translate that zero into NULL — deploying KK without the DB pair (or rolling back DB without rolling back KK) silently corrupts `provider_capacity` data.

**Details**:
- **Architecture changes**: Save semantics moved from "send DB-as-is" to "send 0 for hidden fields, let SP normalize". Per-save behavior added an extra licensing-init round-trip on the Licensing tab. SP gained a procedural normalization block (low-cost, single-row UPDATE).
- **Tables/services affected**: `provider_capacity` (write + read path), `save_provider_capacity_sp_port_kk`, `select_max_capacity_fn`, KK enroll-provider service, licensing init endpoint.
- **Blast radius**: All sponsors saving on the KK Licensing tab — high-volume during eligibility / onboarding cycles.
- **Load pattern changes**: Every Licensing-tab save now fires an additional GET (`getProviderLicensingByStateCode`). `angular.copy` of full `providerCapacity` runs on initial bind and after every save.
- **Dependency touchpoints**: KK#22769 hard-depends on DB#725's normalization. DB#726 deploys #725 to production via MRP. Required deploy order: DB (#725 + #726) before or atomically with KK 26.5.10.
- **Rollback difficulty**: **High**. Atomic rollback of both repos required.
- **Evidence**: PR #22769 summary states "The save SP normalizes 0 → NULL"; SP diff adds matching block; KK code has no direct SP reference, contract is implicit and undocumented at the API boundary.
- **Monitoring signals**: After deploy: (a) `provider_capacity` column NULL ratios trend (should jump for hidden-field columns); (b) Licensing-tab save P95 latency (extra GET roundtrip); (c) error rate on `getProviderLicensingByStateCode` (now called per save).

**DevOps Action Required**: **Yes**
1. Confirm MinuteMenu.Database#725 + #726 deploy as part of MRP for release 319522 BEFORE or with KK 26.5.10.
2. Document coordinated rollback (must revert both KK and DB together) in the release runbook.
3. Add a smoke test: save a capacity with `0` in a visible field and verify DB column is NULL post-deploy.
4. Temporary monitoring query alerting if `provider_capacity` rows are inserted with non-NULL `0` in any normalized column (would indicate the SP didn't deploy).

---

### Centers Meal & Attendance (322557, 318492, 320884, 317707)

**Risk Level**: **Medium**
**Summary**: Batch is net-load-reducing (KK#22650 cuts phantom CLAIM_ATTENDANCE inserts; KK#22675 is UI-only) but has tight cross-repo and intra-batch deployment-order coupling that requires coordinated rollback.

**Details**:
- **Architecture changes**: KK#22650 adds in-memory short-circuit in `SaveClaimAttendance` (no DB impact); KK#22764+22774 restructure Centers attendance conflict-resolution control flow (frontend); KK#22599+CX#4142 fix a `continue`-discards-data bug in CX→KK sync path (DLL drop + source).
- **Tables affected**: `claim_menu_quantity` (#22599 path, write reduction on infant edge case + non-infant write restoration), `claim_attendance` / `cxadmin.MEAL_RECORD` (#22650 phantom write reduction). No schema changes, no new indexes/SPs.
- **Blast radius**: All CX Centers users on Record Attendance and Daily Menu pages. Highest concentration in L2=N centers and M01a1=Y/M16=Y centers.
- **Load pattern**: NO temp-table operations on million-row tables. NO timeout/pool changes. NO removed indexes. NO reauth churn. NO new connection sites. One latent concern: empty-catch+Task.Run in `AttendanceBll.UpdateAutoCalculateActual` is unchanged.
- **Dependency touchpoints**:
  - **HARD COUPLING #1**: KK#22599 (DLL) ↔ CX#4142 (source). DLL must be built FROM CX merge SHA. Rollback together.
  - **HARD COUPLING #2**: KK#22764 ↔ KK#22774. #22764 alone leaves conflict-merge as dead code AND throws in inOutTimes branch. Must deploy and rollback as a unit.
- **Rollback difficulty**: **Medium**. KK frontend rollback is independent. Backend (#22650) rollback restores phantom-row creation behavior. CX DLL rollback requires re-building the prior DLL artifact.
- **Evidence**: Three of six PRs are direct fixes to defects from the LAST PI cycle. High-iteration zone — defect-density signal.
- **Monitoring (first 48h)**: Attendance save error rate (`SaveClaimAttendance`); `claim_attendance` insert volume per center per day (should DROP on L2=N centers); `claim_menu_quantity` non-infant Actual quantity NULL rate after attendance certify (should DROP for M01a1=Y/M16=Y centers); AppInsights query for `CHECK_DEVICE_DATE_SETTINGS`.
- **Post-deploy data cleanup**: KK#22650 PR body references a prepared cleanup query for existing phantom CLAIM_ATTENDANCE rows — requires DBA review and gated execution.

**DevOps Action Required**: **Yes**
1. Verify KK#22599 DLL artifact provenance against CX#4142 merge SHA before promoting.
2. Lock KK#22764 + KK#22774 as a single deploy/rollback unit.
3. Set up 48-hour monitoring on the four metrics above.
4. Coordinate phantom-row data-cleanup query review with DBA team after backend deploy.
5. File follow-up tickets: (a) remove empty catch in `AttendanceBll.UpdateAutoCalculateActual`; (b) track the untracked `Inf611Cereal→Inf611Bread` typo fix in CX#4142; (c) refactor duplicated `LogicService.cs` / `LogicServiceToKK.cs`.

---

### eForms (322254, 322840)

**Risk Level**: **High**
**Summary**: The cross-repo unique-index migration (DB#727) lacks `ONLINE = ON` on a multi-million-row hot table and has a hard deploy-order dependency with KK#22773 — both must be coordinated by DevOps to avoid a write outage on `KK_EnrollmentReport`.

**Details**:
- **Architecture**: KK#22773 introduces a bounded (2-attempt, no-backoff) retry on a specific unique-index name. DB#727 creates that filtered unique index. KK#22772 narrows the OE-enabled center query by status — performance-neutral.
- **Tables**: `dbo.KK_EnrollmentReport` (large, write-hot during eForm season), `dbo.KK_EnrollmentAssignmentCx` (one-shot DML on 5 rows).
- **Blast radius**: A non-online unique index build will block all eForm save paths across all 15 API VMs until the build completes.
- **Load patterns**: 15 VM concurrent writes are the root cause of the duplicate-row race; the retry is correctly bounded at 2 attempts, so worst-case load amplification is 2×.
- **Dependencies / deploy order**: STRICT — (1) DB#727 cleanup script must run first to remove the ~115 known duplicates; (2) the unique index must exist before KK#22773 is deployed.
- **Rollback**: KK#22773 safe to roll back independently. DB#727 unique index can be dropped; the cleanup DELETEs are not reversible without backup.
- **Monitoring**: Add metric/log on every entry into the `IsUniqueIndexViolation` catch block. Alert if retries-engaged count exceeds threshold.

**DevOps Action Required**: **Yes**
1. Confirm `KK_EnrollmentReport` row count and decide whether `ONLINE = ON` is required (recommended) or whether to schedule a maintenance window.
2. Enforce deploy order: cleanup DML → index DDL → KK code deploy.
3. Add a smoke check post-DB-deploy that `UX_KK_EnrollmentReport_LogicalKey` exists before promoting KK#22773.
4. Add monitoring on duplicate-catch invocation rate for the first 7 days post-deploy.

---

### Tier/Eligibility & Provider/School-Age (322375, 305316, 322555)

**Risk Level**: **High**
**Summary**: PR-698 quietly changes the semantics of `select_max_capacity_fn2` (a function consumed by tier/capacity reports) and ships non-functional rollback scripts; PR-22763 broadens an AutoMapper write that previously was explicitly suppressed; PRs 723/728 are one-off data corrections without proper rollback or MRP integration.

**Details**:
- **Architecture**: 22763 changes a CX child save/update boundary. 698 modifies a SQL scalar UDF used in capacity/tier displays plus adds a PROVIDER_CAPACITY column. 723/728 surgically modify single rows in REVIEW, ReviewSubmission, and PROVIDER.
- **Tables touched**: CHILD (via mapping), PROVIDER_CAPACITY (+col), HISTORIABLE_FIELDS (+row), LICENSE (read-side, NOLOCK join), REVIEW, ReviewSubmission, ReviewSubmissionAnswer, PROVIDER.
- **Blast radius**: 22763 affects all CX child save flows. 698 affects every caller of `select_max_capacity_fn2` cluster-wide. 723/728 one row each.
- **Load**: `select_max_capacity_fn2` is invoked across reports — extra LEFT JOIN to LICENSE adds tiny per-call cost. NOLOCK semantics retained.
- **Dependencies**: 22763 ↔ 698 are functionally independent. 723 ↔ 728 cumulative: 728 must subsume 723.
- **Rollback**: 698 ROLLBACK scripts are NO-OPs (do not undo column or HISTORIABLE_FIELDS row, no prior function body). 723/728 have NO rollback artifact and irreversibly mutate JSON-like `RawSubmission` text.
- **Monitoring after deploy**: (a) capacity-report tickets for tier shifts; (b) child enrollment reports for mass-flip in `never_activated_flag` (run a daily count by `never_activated_flag` for one week); (c) review queue / next-review-date alerts for the corrected provider.

**DevOps Action Required**: **Yes**
1. Author proper rollback scripts for 698 (DROP COLUMN, DELETE HISTORIABLE row, restore prior `select_max_capacity_fn2` body) before deploy.
2. Capture pre-images for 723/728 single-row updates and confirm execution order in production.
3. Verify `MMADMIN_XXX` execution path (manual vs MRP) and document which DBAs run it on which databases.
4. Post-deploy: monitor capacity-report deltas and run a count of CHILD rows where `never_activated_flag` flips unexpectedly during the first week.

---

### Untracked Changes (KK#22761, MinuteMenu.Database#729, #730)

**Risk Level**: **High**
**Summary**: Two untracked DB index PRs (#729, #730) responding to a live production Win32Exception incident were merged into the release branch without being registered in `MMADMIN_EFORM/MRP.build.config`, and the rapid #730 fast-follow strongly suggests the underlying root cause has not been identified.

**Details**:
- **Architecture impact**: Three new nonclustered indexes (`IX_KK_EnrollmentDataReport_ReportId`, `IX_KK_EnrollmentCompletedCx_ChildClientCenter`, `IX_KK_StorageFiles_PublicName`). All use `DATA_COMPRESSION = PAGE` and `ONLINE = ON` (Enterprise/MI-only). One composite index has a leading-column choice (`ChildId`) that does not match observed query patterns.
- **Affected tables**: `dbo.KK_EnrollmentDataReport`, `dbo.KK_EnrollmentCompletedCx`, `dbo.KK_StorageFiles` — all heavily written during claim week.
- **Blast radius**: All kkeform MI tenants. Permanent write-amplification on three tables during the heaviest write window of the year.
- **Load patterns**: Diagnosis attributes Win32Exception bursts to query plans, but Win32Exception is a Win32 socket/IO error normally surfaced from the connection layer (Pattern 3/4). The PRs do not change any connection settings, timeouts, or pool sizes.
- **Dependencies**: Scripts in `MMADMIN_EFORM/Updates/` but `MRP.build.config` does not include any `Updates/*` files. Deployment is implicit-manual.
- **Rollback**: NO rollback scripts created.
- **Evidence quality**: Neither PR includes execution plans, missing-index DMV output, wait stats, connection-pool metrics, or the actual Win32Exception stack. The 24-hour gap between #729 and #730 (filename `may5_burst_followup`) is empirical evidence that #729 did not relieve the burst.

**DevOps Action Required**: **Yes**
1. DBA review of all three indexes (key column choices, UNIQUE-ness on PublicName, edition guards for `ONLINE=ON`).
2. Decide deploy strategy: register in MRP, OR produce explicit DBA runbook listing per-MI execution + verification + rollback SQL.
3. Add rollback scripts in `MMADMIN_EFORM/ROLLBACK/`.
4. Before next claim week: capture connection-pool / Win32Exception telemetry to validate root cause.
5. Map untracked PR #22761 + #729 + #730 to TP tickets in release 319522 for traceability.
6. Schedule deployment timing window outside claim-week peak.

---

## Untracked Changes

| PR | Repo | Title | Author | Findings |
|----|------|-------|--------|----------|
| [#22761](https://github.com/MinuteMenu/KK/pull/22761) | KK | Fix typo: "Out Child Claim Payment" → "Own Child Claim Payment" | quan-vu-evizi | 0 Critical, 0 High (cosmetic) |
| [MinuteMenu.Database#729](https://github.com/MinuteMenu/MinuteMenu.Database/pull/729) | DB | 322954 kkeform MI tuning to stop Win32Exception bursts under claim-week peak | vantuanevizi | 0 Critical, 2 High (H13, H14) |
| [MinuteMenu.Database#730](https://github.com/MinuteMenu/MinuteMenu.Database/pull/730) | DB | 322954: kkeform MI tuning follow-up indexes | congnguyen-evizi | 0 Critical, 2 High (H15, H16) |

**Note**: KK#22761's branch references TP ticket 322813 ("HX102 Issue Payment Orphaned Records") which is NOT in the release ticket list — only a label edit shipped. Confirm with author whether 322813 is being deferred or handled elsewhere; otherwise the orphaned-records work may be lost. PRs #729 + #730 reference TP 322954 which is also NOT in the release ticket list.

---

## PRs With No Critical/High Issues

| PR | Repo | Ticket | Title |
|----|------|--------|-------|
| [#22772](https://github.com/MinuteMenu/KK/pull/22772) | KK | [TP #322840](https://minutemenu.tpondemand.com/entity/322840) | Eform Approve & Renew shows removed centers |
| [#22675](https://github.com/MinuteMenu/KK/pull/22675) | KK | [TP #320884](https://minutemenu.tpondemand.com/entity/320884) | Actual Quantity Served of all foods in March was removed (UI/i18n only) |
| [#22764](https://github.com/MinuteMenu/KK/pull/22764) | KK | [TP #322557](https://minutemenu.tpondemand.com/entity/322557) | Centers: Meal Data Disappears After Save (superseded by #22774) |
| [MinuteMenu.Database#728](https://github.com/MinuteMenu/MinuteMenu.Database/pull/728) | DB | [TP #322555](https://minutemenu.tpondemand.com/entity/322555) | Provider correction follow-up (rolled into H12) |

---

## Priority Action Items

### Must Fix (Critical — blocks release)

1. **C1** — [KK#22774](https://github.com/MinuteMenu/KK/pull/22774) ([TP #322557](https://minutemenu.tpondemand.com/entity/322557)): Fix `AttendanceBll.cs:1447` to use `"_"` instead of `"-"` so the new-child conflict path actually matches.
2. **C2** — [KK#22763](https://github.com/MinuteMenu/KK/pull/22763) ([TP #322375](https://minutemenu.tpondemand.com/entity/322375)): Preserve `never_activated_flag` from DB on Update path; do not let request DTO default to false silently overwrite it.
3. **C3** — [Centers-CX#4142](https://github.com/MinuteMenu/Centers-CX/pull/4142) ([TP #317707](https://minutemenu.tpondemand.com/entity/317707)): Apply the `Inf611Cereal → Inf611Bread` rename in `LogicService.cs` (or remove the file if dead). Verify routing via DI/factory bindings before merge.
4. **C4** — [KK#22599](https://github.com/MinuteMenu/KK/pull/22599) ([TP #317707](https://minutemenu.tpondemand.com/entity/317707)): Verify the DLL was built from CX#4142's merged SHA. Add a `VERSION.txt` recording source commit + build timestamp.
5. **C5** — [DB#725 + #726](https://github.com/MinuteMenu/MinuteMenu.Database/pull/725) ([TP #320035](https://minutemenu.tpondemand.com/entity/320035)): Lock KK 26.5.10 and DB#725+#726 as a single deploy unit; document atomic rollback in the release runbook.
6. **C6** — [DB#727](https://github.com/MinuteMenu/MinuteMenu.Database/pull/727) ([TP #322254](https://minutemenu.tpondemand.com/entity/322254)): Add `WITH (ONLINE = ON, MAXDOP = 4)` to the index DDL, OR schedule the index build during a maintenance window.

### Should Fix (High — significant risk)

1. **H1** — [DB#726](https://github.com/MinuteMenu/MinuteMenu.Database/pull/726) ([TP #320035](https://minutemenu.tpondemand.com/entity/320035)): Document atomic rollback requirement (DB rollback must pair with KK#22769 revert).
2. **H2** — [KK#22765](https://github.com/MinuteMenu/KK/pull/22765) ([TP #322928](https://minutemenu.tpondemand.com/entity/322928)): Have save endpoint return new `maxCapacity` directly to avoid extra GET per save.
3. **H3** — [KK#22769](https://github.com/MinuteMenu/KK/pull/22769) ([TP #320035](https://minutemenu.tpondemand.com/entity/320035)): Verify with HX product owner that destructive zeroing of legacy-populated hidden fields is intended.
4. **H4** — [KK#22767](https://github.com/MinuteMenu/KK/pull/22767) ([TP #320035](https://minutemenu.tpondemand.com/entity/320035)): Cache server response (canonical row), not the request payload, in the snapshot.
5. **H5** — [KK#22599](https://github.com/MinuteMenu/KK/pull/22599) ([TP #317707](https://minutemenu.tpondemand.com/entity/317707)): Lock KK#22599 ↔ CX#4142 as a single deploy unit; file follow-up to remove empty catch in `AttendanceBll.UpdateAutoCalculateActual`.
6. **H6** — [Centers-CX#4142](https://github.com/MinuteMenu/Centers-CX/pull/4142) ([TP #317707](https://minutemenu.tpondemand.com/entity/317707)): Add unit test for `policyM16 && rowForInfantActual == null` mixed-menu case.
7. **H7** — [KK#22650](https://github.com/MinuteMenu/KK/pull/22650) ([TP #318492](https://minutemenu.tpondemand.com/entity/318492)): DBA-review and gated execution of phantom CLAIM_ATTENDANCE cleanup query.
8. **H8** — [KK#22773](https://github.com/MinuteMenu/KK/pull/22773) ([TP #322254](https://minutemenu.tpondemand.com/entity/322254)): Extract the index name to a shared constant; add APM/log warning when retry catch fires.
9. **H9** — [KK#22763](https://github.com/MinuteMenu/KK/pull/22763) ([TP #322375](https://minutemenu.tpondemand.com/entity/322375)): Audit all callers of `CxChildModel → CHILDRow` mapping; add unit test verifying Update path does not regress the flag.
10. **H10** — [DB#698](https://github.com/MinuteMenu/MinuteMenu.Database/pull/698) ([TP #305316](https://minutemenu.tpondemand.com/entity/305316)): Confirm `>=` and `preschoolAge` semantics are intentional; produce before/after counts of providers whose function output changes.
11. **H11** — [DB#698](https://github.com/MinuteMenu/MinuteMenu.Database/pull/698) ([TP #305316](https://minutemenu.tpondemand.com/entity/305316)): Author proper rollback scripts (DROP COLUMN, DELETE HISTORIABLE row, restore prior function body).
12. **H12** — [DB#723 + #728](https://github.com/MinuteMenu/MinuteMenu.Database/pull/723) ([TP #322555](https://minutemenu.tpondemand.com/entity/322555)): Capture pre-image; confirm `next_review_required_date = '2026-05-29'` is intended; add execution checklist.
13. **H13** — [DB#729](https://github.com/MinuteMenu/MinuteMenu.Database/pull/729) (No ticket): Register in MRP OR produce DBA runbook with per-MI execution + verification + rollback SQL.
14. **H14** — [DB#729](https://github.com/MinuteMenu/MinuteMenu.Database/pull/729) (No ticket): Capture connection-pool / Win32Exception telemetry to validate root cause BEFORE adding more indexes.
15. **H15** — [DB#730](https://github.com/MinuteMenu/MinuteMenu.Database/pull/730) (No ticket): Pause iterative-index approach; release notes should NOT advertise 322954 as resolved.
16. **H16** — [DB#730](https://github.com/MinuteMenu/MinuteMenu.Database/pull/730) (No ticket): Re-key the `KK_EnrollmentCompletedCx` index as `(ClientId, CenterId, ChildId)` after validating the actual query pattern.

### Verify With Product (Behavioral changes needing confirmation)

1. **H3** — [KK#22769](https://github.com/MinuteMenu/KK/pull/22769) ([TP #320035](https://minutemenu.tpondemand.com/entity/320035)): Confirm that opening a legacy provider whose hidden fields contain non-zero values and clicking Save (without changing anything) should silently null those fields.
2. **H10** — [DB#698](https://github.com/MinuteMenu/MinuteMenu.Database/pull/698) ([TP #305316](https://minutemenu.tpondemand.com/entity/305316)): Confirm semantic shift in max-capacity calculation is intentional and document in release notes.
3. **H12** — [DB#728](https://github.com/MinuteMenu/MinuteMenu.Database/pull/728) ([TP #322555](https://minutemenu.tpondemand.com/entity/322555)): Confirm with client 166 whether `next_review_required_date = 2026-05-29` is the correct date.
4. Untracked PRs (#22761, #729, #730): Confirm with sponsors / DBAs that these are intentional release content; if 322813 work is being deferred, file a tracking ticket.

---

*Report generated by parallel code review agents reviewing 22 merged PRs across 3 repos. Each tracked PR was analyzed by 3 specialized agents (Bug Detection, Architecture & Performance, Requirement Alignment). Untracked PRs were analyzed by 2 agents (Bug Detection, Architecture & Performance). Only Critical (90-100) and High (80-89) findings are shown. Approximately 25 additional findings scored 70-79 were filtered out.*
