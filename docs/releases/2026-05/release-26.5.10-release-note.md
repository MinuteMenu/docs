# Release: KK PI 26.5.10

**Release ID:** #319522
**Prepared by:** Tuan Nguyen
**Date:** 2026-05-06
**Branch:** release/319522_KK_PI_26.5.10

---

## 1. Summary

- **Centers Meal & Attendance:** Two-bug follow-up to PI #313113 — fixes meal-data-disappears-after-save by merging `firstServing`/`secondServing` payloads and preserving non-conflicting meal types during concurrency conflict resolution (#322557, KK#22774 supersedes KK#22764). Adds backend defense-in-depth guard for L2=N centers against phantom `claim_attendance` rows, plus frontend "approved meals failed to load" fail-closed banner (#318492). Restores auto-calculation of `actual_quantity_served` for Non-Infant menus when `Inf611` row is absent (#317707, cross-repo KK DLL drop + Centers-CX#4142). UI-only Copy Menus confirmation re-worded so sponsors see the affected center name (#320884; data restore performed offline 2026-04-11).
- **Provider Capacity (KK ↔ HX parity):** Coordinated KK + DB change to make Licensing-tab save behavior in KidKare byte-equivalent to HomeX. Fixes load-time school-aged display bug, refreshes Maximum Capacity label after save, preserves DB-loaded values on license-type round-trip, and zeros out hidden capacity columns on save (relying on the new SP `0 → NULL` normalization). Ports `save_provider_capacity_sp_port_kk` from HX (#320035 / Bug #322928).
- **eForms — Approve & Renew:** Resolves "Submitted (Site)/(Parent)" forms missing from Approve & Renew + View Status by adding a `UNIQUE NONCLUSTERED` index `UX_KK_EnrollmentReport_LogicalKey` (filtered on `InvitationId IS NOT NULL`), one-off duplicate cleanup, status-290 backfill on 5 affected assignments, and a bounded retry-on-unique-violation in `ReportEnrollmentBll.GenerateInvitationFormReport` (#322254). Removed centers no longer appear in the Approve & Renew center drop-down (#322840) — fixed via the shared `GetOeCenters` helper (also covers other OE consumers).
- **Tier integrity / Center enrollment:** Removes the AutoMapper `.Ignore()` that was suppressing `never_activated_flag` for Center-enrolled Pending children, so Rule44 Branch 2 fires correctly on Withdrawn-never-activated transitions (#322375).
- **State Provider review formula:** `select_max_capacity_fn2` adjusted so Maximum Capacity returns MAX(infant, preschool) when school-aged capacity is not entered, plus a new per-provider compute-method override column on `PROVIDER_CAPACITY` (#305316).
- **One-off data fix:** Corrects misassigned review provider id `6B829C8D…` → `80B80B61…` for Monitor 16601 / Review 115FC181 / Sponsor 166, plus next-review-required-date update (#322555).
- **Untracked changes (not in TP release ticket list):**
  - KK#22761 — single-line label typo "Out Child Claim Payment" → "Own Child Claim Payment" on Sponsor Payments print preview (branch references #322813 but only the typo shipped).
  - MinuteMenu.Database#729 + #730 — kkeform Managed Instance index additions (`KK_EnrollmentDataReport.ReportId`, `KK_EnrollmentCompletedCx(ChildId,ClientId,CenterId)`, `KK_StorageFiles.PublicName`) intended to relieve Win32Exception bursts during claim-week peak (#322954). **[VERIFY]** root cause not confirmed (see section 2.4).

---

## 2. Infrastructure Impact

**Overall Risk Level:** **High**

### 2.1 Database Changes

- `MMADMIN_XXX/Programmability/Stored Procedures/save_provider_capacity_sp_port_kk.sql` — port from HX `save_provider_capacity_sp` adding `0 → NULL` normalization on capacity counts and sentinel-date → NULL on license/variance/waiver dates plus `mod_date_time`/`create_date_time`. Wired into MRP build + rollback via #726.
- `MMADMIN_XXX/Programmability/Stored Procedures/select_max_capacity_sp.sql`, `select_max_capacity_from_history_sp.sql`, `generate_state_provider_MA_sp.sql` — updated to support new compute-method override column on `PROVIDER_CAPACITY` (#305316).
- `MMADMIN/Programmability/Functions/select_max_capacity_fn2.sql` — semantic change: `>` flipped to `>=` in the `@HighestProviderAgedCnt` (1008) branch and `@PreschoolCnt` replaced by `@preschoolAgeCnt = preschooler + school_age`; LEFT JOIN to `LICENSE` added with `COALESCE` for the new override column (#305316).
- `MMADMIN_XXX/Programmability/Functions/select_max_capacity_fn.sql` — paired update with the MMADMIN function above.
- `MMADMIN/Updates/305316_ADD_COMPUTE_METHOD_TO_PROVIDER_CAPACITY.sql` and `MMADMIN_XXX/Updates/305316_ADD_COMPUTE_METHOD_TO_PROVIDER_CAPACITY.sql` — adds `displayed_max_capacity_compute_method_code` (NULL) on `PROVIDER_CAPACITY`.
- `MMADMIN/Updates/305316_ADD_HISTORIABLE_FIELD_FOR_OVERRIDE.sql` and `MMADMIN_XXX/Updates/305316_ADD_HISTORIABLE_FIELD_FOR_OVERRIDE.sql` — registers the new column in `HISTORIABLE_FIELDS` so trigger picks it up.
- `MMADMIN_XXX/Programmability/Scripts/305316_Update_License_Max_Capacity_Compute_Method.sql` — back-fills compute-method override values per requirements.
- `MMADMIN_EFORM/Updates/322254_unique_index_kk_enrollment_report.sql` — `CREATE UNIQUE NONCLUSTERED INDEX UX_KK_EnrollmentReport_LogicalKey ON KK_EnrollmentReport (InvitationId, EnrollmentFormType, FormType, SourceSystem) WHERE InvitationId IS NOT NULL`. **`ONLINE = ON` is NOT specified — see section 2.4.**
- `MMADMIN_EFORM/Updates/322254_script_delete_report_dup.sql` — deletes ~115 hard-coded duplicate IDs that would block the index build.
- `MMADMIN_EFORM/Updates/322254_Fix_EnrollmentAssignment_Status290_NullForms.sql` — back-fills status 290 on 5 specific `KK_EnrollmentAssignmentCx` rows.
- `MMADMIN/Updates/322555_Fix_Review_Provider_Id.sql` and `MMADMIN_XXX/Updates/322555_Fix_Review_Provider_Id.sql` — one-off correction: REVIEW (MMADMIN_166), ReviewSubmission, ReviewSubmissionAnswer (MMADMIN), plus `PROVIDER.next_review_required_date = '2026-05-29'` (added in #728). Not in MRP — manual DBA execution required.
- `MMADMIN_EFORM/Updates/322954_kkeform_MI_tuning_*.sql` (729) and `MMADMIN_EFORM/Updates/322954_kkeform_may5_burst_followup_indexes.sql` (730) — three new nonclustered indexes on `KK_EnrollmentDataReport`, `KK_EnrollmentCompletedCx`, `KK_StorageFiles` with `DATA_COMPRESSION = PAGE, ONLINE = ON` (Enterprise/MI only). **NOT registered in `MMADMIN_EFORM/MRP.build.config` — manual DBA deploy required. Untracked in TP. [VERIFY]**

### 2.2 Auth / Session Changes

- None
- Expected re-authentication impact: **None** — no auth changes.

### 2.3 API Changes

- None — no new, removed, or changed REST routes in this release. Existing endpoints retain the same signatures; only internal logic changed.
- **CHANGED contract (internal, AutoMapper):** `CxChildModel ↔ dsChild.CHILDRow` now round-trips `never_activated_flag` (previously `Ignore()` on the forward map). All callers of `Mapper.Map<dsChild.CHILDRow>` in CX child save flows now write this field — see section 2.4.

### 2.4 Performance & Capacity

- **`KK_EnrollmentReport` unique index without `ONLINE = ON` (#322254 / DB#727)** — `CREATE UNIQUE NONCLUSTERED INDEX` on a multi-million-row hot eForm output table. Without ONLINE, a Sch-M lock is held for the duration of the build, blocking all eForm save/submit paths on all 15 API VMs.
  **Action required** — Standard Edition does not support `ONLINE = ON`. Schedule index build during a maintenance window OR add `WITH (ONLINE = ON, MAXDOP = 4)` if running on Enterprise / Azure SQL MI. Confirm row count first. Enforce deploy order: cleanup duplicates → index DDL → KK code deploy. Add a smoke check that the index exists before promoting KK#22773.
  Rollback: Low — `DROP INDEX IF EXISTS UX_KK_EnrollmentReport_LogicalKey ON KK_EnrollmentReport`.
  Watch: eForm save/submit p99 latency, blocking-process count on `KK_EnrollmentReport`, count of catches in `IsUniqueIndexViolation` retry block (add log/metric).
  Evidence: Cleanup script targets ~115 IDs but PK values reach 2.2M+, indicating millions of rows; PI 26.5.10 release window overlaps with eForm season write peak.

- **Provider Capacity SP/UI coordinated change (#320035 / DB#725, #726 + KK#22769)** — Frontend now sends literal `0` for hidden capacity fields and relies on the NEW SP `0 → NULL` normalization to translate that into NULL in DB. KK and DB **must deploy together**; KK-only deploy or DB-only rollback writes literal `0` instead of NULL into `provider_capacity` columns for every Licensing save.
  **Action required** — Coordinate atomic deploy of MinuteMenu.Database#725 + #726 with KK 26.5.10. Document atomic rollback in runbook.
  Rollback: High — partial rollback silently corrupts data; recovery requires `UPDATE provider_capacity SET col = NULL WHERE col = 0` audit per affected column.
  Watch: `provider_capacity` column NULL-ratio trend (should jump for hidden-field columns); Licensing-tab save P95 latency (one extra GET per save now); error rate on `getProviderLicensingByStateCode`.
  Evidence: Implicit cross-repo contract; KK code has no direct SP reference (calls service layer), and the contract is not surfaced at the API boundary.

- **Extra `licensingTab.init` GET on every Licensing-tab save (#320035 / KK#22765)** — Save endpoint returns only `{code}`, so the UI re-fetches the entire licensing payload after every save to refresh the `maxCapacity` label.
  **Monitor** — doubles DB+API load on the Licensing save path.
  Rollback: Low — revert KK#22765 frontend.
  Watch: Licensing tab save round-trip count and `getProviderLicensingByStateCode` latency.
  Evidence: Save handler unconditionally calls `licensingTab.init` on `tabView === 'licensingTab' && !params.final`.

- **`select_max_capacity_fn2` semantic change (#305316 / DB#698)** — Function consumed by tier/capacity reports cluster-wide. `>` flipped to `>=` and preschool count now includes school-age in the @HighestProviderAgedCnt branch. **Provider max-capacity values displayed will change for any provider with tied counts or non-zero school-age.**
  **Action required** — Confirm with QA/business owner that `>=` and combined preschool+school-age semantics are intentional. Run a PROD-LIKE before/after diff: `SELECT provider_id, dbo.select_max_capacity_fn2_old(...) AS old, dbo.select_max_capacity_fn2(...) AS new FROM PROVIDER_CAPACITY WHERE old <> new` to enumerate affected providers.
  Rollback: Medium — function is `CREATE OR ALTER`; restore prior body from master commit. Note: `MMADMIN/ROLLBACK/305316_*.sql` files are byte-identical to forward Updates (no-op rollback) — see next item.
  Watch: Tier-eligibility reports and Maximum Capacity displays for HX sponsors.
  Evidence: Three logic changes shipped in one PR (LEFT JOIN + comparison flip + variable swap).

- **305316 ROLLBACK scripts are no-ops (#305316 / DB#698)** — Both `MMADMIN/ROLLBACK/305316_*.sql` files are byte-identical to the forward `Updates/` scripts. They re-add the column / re-insert the HISTORIABLE_FIELDS row. Function rollback is missing entirely.
  **Action required** — Author proper rollback before deploy: `ALTER TABLE dbo.PROVIDER_CAPACITY DROP COLUMN displayed_max_capacity_compute_method_code;`, `DELETE FROM HISTORIABLE_FIELDS WHERE HISTORIABLE_FIELD_NAME = 'displayed_max_capacity_compute_method_code';`, plus `select_max_capacity_fn2` prior body.
  Rollback: High — without proper scripts, DBA must hand-write rollback statements during incident.
  Watch: N/A (action prevents future incident response failure).
  Evidence: Diff of `MMADMIN/ROLLBACK/305316_*.sql` vs `MMADMIN/Updates/305316_*.sql`.

- **AutoMapper write-path broadening on `never_activated_flag` (#322375 / KK#22763)** — Removing `Ignore()` at `CxChildMapping.cs:301` means EVERY `Mapper.Map<dsChild.CHILDRow>(child)` and `Mapper.Map(child, updatedDatasetRow)` call now writes the flag from the model — not only the targeted AddChild path. Affects all CX child save/update flows (Add, Update, Reactivate, Withdraw, Import).
  **Monitor** — daily count of CHILD rows where `never_activated_flag` flips for the first week post-deploy.
  Rollback: Low — revert mapping/Bll commit; KK redeploy.
  Watch: New row count `WHERE never_activated_flag = 0 AND status_code = 239` (Pending) trend; alert on unexpected mass-flip.
  Evidence: `IgnoreInvalidFields` does not preserve this field from the DB row; request DTO defaults to false on edit-only flows. **[VERIFY]** — see Risk Assessment.

- **kkeform Managed Instance indexes — root cause not confirmed (#322954 / DB#729 + #730 — UNTRACKED)** — Three new nonclustered indexes on hot kkeform tables intended to relieve Win32Exception bursts during claim-week peak. Win32Exception is normally a connection-layer error (TLS handshake, TCP exhaustion, ADO.NET pool starvation), not a query-plan issue. Neither PR provides execution plans, missing-index DMV output, wait stats, or the actual exception stack. The 24-hour gap between #729 (2026-05-05) and #730 (2026-05-06, filename `may5_burst_followup`) suggests #729 did not relieve the bursts.
  **Action required** — Capture during the next burst: `sys.dm_os_wait_stats` delta, connection-pool counters, full Win32Exception stack. Confirm SQL-side vs network/pool-side root cause BEFORE adding more indexes. Treat 322954 as Open. Release notes should NOT advertise 322954 as resolved.
  Rollback: Low (per index) — `DROP INDEX IF EXISTS …` (rollback scripts not yet authored).
  Watch: Win32Exception count during next claim-week peak; write latency on `KK_EnrollmentDataReport`, `KK_EnrollmentCompletedCx`, `KK_StorageFiles`.
  Evidence: `KK_EnrollmentCompletedCx (ChildId, ClientId, CenterId)` leading column does not match observed query patterns in existing SPs (most filter by ClientId/CenterId first).

- **Cross-repo DLL drop without source audit trail (#317707 / KK#22599 ↔ Centers-CX#4142)** — KK PR is binary-only (7 DLLs in `ExternalReferences/MinuteMenu.CX.Services/`). Actual fix lives in CX#4142. No file in KK repo records which Centers-CX commit built these binaries.
  **Action required** — Verify the DLLs were built from CX#4142's merged SHA. Lock as a single deploy/rollback unit.
  Rollback: Medium — CX rollback requires re-building the prior DLL artifact and committing it back to KK.
  Watch: Errors in `SaveActualQuantitiesAsRequired` on attendance-certify path; spot-check `claim_menu_quantity` non-infant Actual rows after attendance certify for a sample of M01a1=Y/M16=Y centers.
  Evidence: Manual DLL drop pattern (commit `a55775cee2 317707 build dll from 26.5.10`).

- **L2=N phantom claim_attendance row reduction (#318492 / KK#22650)** — Backend guard broadened: skips inserting empty `claim_attendance` rows for L2=N centers when child has no meal flags AND no in/out times. Net load REDUCER for L2=N centers (majority of CX). PR ships frontend "approved meals failed to load" fail-closed banner so users cannot save into a stale/error state.
  **Monitor** — `claim_attendance` insert volume per center per day should DROP on L2=N centers.
  Rollback: Low — revert KK BLL commit.
  Watch: AppInsights `CHECK_DEVICE_DATE_SETTINGS` events; attendance save error rate (`SaveClaimAttendance`).
  Evidence: PR describes 35-199 unique users/week and peak 8064 errors triggering this path. Cleanup of pre-existing phantom rows requires a separate gated DBA query.


### 2.5 Cross-Repo Dependencies

- **Repos involved**: MinuteMenu.Database, Centers-CX, KK
- **Deploy order**: **MinuteMenu.Database (MMADMIN_XXX MRP — provider capacity SP, capacity functions/SPs, max-capacity column) → Centers-CX (LogicService updates) → KK (consumes Centers-CX DLL via `MinuteMenu.Centers.ServerLib.dll`; depends on `save_provider_capacity_sp_port_kk` `0 → NULL` normalization and on `UX_KK_EnrollmentReport_LogicalKey` index)**.
  - Cleanup of duplicate `KK_EnrollmentReport` rows must run BEFORE the unique-index DDL.
  - The `save_provider_capacity_sp_port_kk` SP must be deployed BEFORE KK#22769 because KK#22769 sends literal `0` for hidden fields expecting the SP to normalize to NULL.
  - `MMADMIN_EFORM` updates (322254, 322954) are NOT in the MMADMIN_EFORM MRP — manual DBA execution per kkeform MI.
  - Manual one-off scripts (322555, 322954) are not in MRP — must be tracked via DBA runbook.
- **Rollback order**: **KK → Centers-CX → MinuteMenu.Database**. Any rollback of DB#725/#726 without rolling back KK silently writes literal `0` instead of NULL on every Licensing save — see section 2.4.

---

## 3. Web Configuration Changes

| Service | Key | Value | Notes |
|---------|-----|-------|-------|
| (none)  | —   | —     | No `web.config`, `appsettings`, or Octopus variable changes detected in the diff. |

**[VERIFY]** — Confirm with DevOps that no per-environment configuration update is required for the `MinuteMenu.Centers.*.dll` swap (KK#22599) or for the new MMADMIN_XXX SP signature.

---

## 4. Rollback Plan

1. **Halt eForm Approve & Renew traffic briefly if rollback decision happens during eForm peak** — to minimize duplicate-row regeneration after `UX_KK_EnrollmentReport_LogicalKey` is dropped.
2. Deploy previous KK BE + FE build (restores prior `MinuteMenu.Centers.ServerLib.dll`, prior `ChildBll`, `ReportEnrollmentBll`, `CentersBll`, AngularJS controllers, AutoMapper profile).
3. Deploy previous Centers-CX build (`LogicService.cs` + `LogicServiceToKK.cs` reverted to pre-#4142 state).
4. Run DB rollback:
   - `MMADMIN_XXX/RELEASES/MRP_Rollback_319522.sql` — restores `save_provider_capacity_sp_port_kk` to its prior body and reverts the SP family used by max-capacity (305316 family).
   - `DROP INDEX IF EXISTS UX_KK_EnrollmentReport_LogicalKey ON KK_EnrollmentReport;` (no rollback file shipped — DBA executes manually).
   - **Manual rollback required (no scripts shipped):**
     - `ALTER TABLE dbo.PROVIDER_CAPACITY DROP COLUMN displayed_max_capacity_compute_method_code;`
     - `DELETE FROM HISTORIABLE_FIELDS WHERE HISTORIABLE_FIELD_NAME = 'displayed_max_capacity_compute_method_code';`
     - Restore prior `select_max_capacity_fn2` body from master commit.
   - 322254 cleanup DELETEs cannot be reversed without backup restore — accept loss of those duplicate rows or restore from nightly backup.
   - 322555 review-provider correction is irreversible — accept the corrected provider id, or restore `REVIEW`/`ReviewSubmission`/`PROVIDER` rows from nightly backup.
   - 322954 indexes — `DROP INDEX IF EXISTS IX_KK_EnrollmentDataReport_ReportId, IX_KK_EnrollmentCompletedCx_ChildClientCenter, IX_KK_StorageFiles_PublicName;` (manual on each kkeform MI).
5. Post-rollback smoke: Centers attendance save (mixed first/second serving), eForm Approve & Renew load, HX Licensing-tab edit-and-save, Sponsor Payments print preview.

---

## 5. Risk Assessment

- **Worst case:** (1) KK 26.5.10 deploys before MinuteMenu.Database#725+#726 (or DB rolls back without KK) → every HX Licensing save silently writes literal `0` into `provider_capacity` for hidden columns; downstream max-capacity/eligibility consumers begin reading 0 where NULL was expected, distorting Tier reports and review worksheets. (2) `KK_EnrollmentReport` unique index built without `ONLINE = ON` on Standard Edition holds Sch-M lock long enough to time out every concurrent eForm save across all 15 API VMs. (3) Edit on a Center-enrolled Pending child after this release silently flips `never_activated_flag` from 1 back to 0 because the request DTO does not carry the field — defeats Rule44 fix in #322375. (4) Centers attendance new-child concurrency continues to drop one user's update because of the `-` vs `_` separator mismatch in `AttendanceBll.cs:1447`.
- **How we detect it:** (a) APM/log on `provider_capacity` writes — alert if a row is inserted with non-NULL `0` in a normalized column (would indicate the SP didn't deploy); (b) blocking-process count on `KK_EnrollmentReport` and eForm save p99 latency; (c) daily count of `CHILD WHERE status_code = 239 AND never_activated_flag = 0` — alert on unexpected drop; (d) production reports of meal data still disappearing on Centers attendance save (same symptom as PI 313113); (e) `IsUniqueIndexViolation` retry catch invocation count (add metric).
- **How we recover:** Follow Rollback Plan section 4. Note (4) — the separator mismatch is a Critical finding from code review and **should be fixed before deploy** rather than relying on rollback. For (1), cleanup query `UPDATE provider_capacity SET <col> = NULL WHERE <col> = 0` per affected column; for (2), drop and rebuild the index during a maintenance window with `ONLINE = ON` if available; for (3), restore from nightly CHILD backup or run a one-off `UPDATE CHILD SET never_activated_flag = 1 WHERE status_code = 239 AND mod_date_time > <deploy_time>`.

---

## 6. Deployment Checklist

| #  | Step                                                                                                       | Owner       | Status  |
| -- | ---------------------------------------------------------------------------------------------------------- | ----------- | ------- |
| 1  | Dev lead prepared release note                                                                             | Tuan Nguyen | Done    |
| 2  | DevOps reviewed infrastructure impact (deploy order, ONLINE=ON decision for unique index, MI index runbook) | [VERIFY — name] | Pending |
| 3  | Architect reviewed changes (320035 cross-repo SP/UI contract, 322375 AutoMapper write-path broadening)     | [VERIFY — name] | Pending |
| 4  | SQL scripts verified on staging (MMADMIN_XXX MRP, 322254 cleanup + index, 305316 family, 322555 one-off)   | [VERIFY — name] | Pending |
| 5  | Critical fixes from code review applied or accepted (`AttendanceBll.cs:1447` separator, `never_activated_flag` preserve-on-edit, `LogicService.cs` cereal/bread parity, KK DLL provenance) | [VERIFY — name] | Pending |
| 6  | 305316 rollback scripts authored (currently no-ops)                                                        | [VERIFY — name] | Pending |
| 7  | 322954 kkeform MI runbook authored (per-MI execution + verification + rollback SQL)                        | [VERIFY — name] | Pending |
| 8  | Web config changes confirmed (none expected)                                                               | [VERIFY — name] | Pending |
| 9  | Rollback plan confirmed                                                                                    | [VERIFY — name] | Pending |
| 10 | Production deployment approved                                                                             | [VERIFY — name] | Pending |

---

## 7. Tickets

| Ticket | Title | Type | State | Infra Impact |
|--------|-------|------|-------|-------------|
| [#322557](https://minutemenu.tpondemand.com/entity/322557) | Centers: Meal Data Still Disappears After Save - Two Remaining Bugs from PI 313113 | Story | Test | High |
| [#322254](https://minutemenu.tpondemand.com/entity/322254) | Data Inconsistency – "Submitted (Site)"/"Submitted (Parent)" eForms Missing from Approve & Renew Queue | Story | Code review | Critical |
| [#322840](https://minutemenu.tpondemand.com/entity/322840) | Eform Approve & Renew shows removed centers | Story | UAT | Low |
| [#320035](https://minutemenu.tpondemand.com/entity/320035) | Provider Capacity - Discrepancy between KK and HX | Story | (parent of bug 322928) | Critical |
| [#322928](https://minutemenu.tpondemand.com/entity/322928) | The Maximum Capacity should be reloaded after the provider information is saved | Bug | UAT | High |
| [#322375](https://minutemenu.tpondemand.com/entity/322375) | never_activated_flag not saved for Center-enrolled Pending children causes Withdrawn-never-activated claims to bypass Rule44 | Story | UAT | Critical |
| [#317707](https://minutemenu.tpondemand.com/entity/317707) | CX 3792: Actual Quantity Served is no longer automatically calculated for Non-Infant menus | Story | Ready For Test | Critical |
| [#318492](https://minutemenu.tpondemand.com/entity/318492) | Application insight review - Children were automatically marked in meal attendance | Story | UAT | High |
| [#320884](https://minutemenu.tpondemand.com/entity/320884) | Actual Quantity Served of all foods in March was removed | Story | UAT | None (UI/i18n only) |
| [#305316](https://minutemenu.tpondemand.com/entity/305316) | The maximum capacity is not updated when the school-aged capacity is unavailable | Story | Test | High |
| [#322555](https://minutemenu.tpondemand.com/entity/322555) | Client 166 requires help correcting the provider number for a review | Story | UAT | Medium (one-off DML) |
| (untracked) | KK#22761 typo: "Out Child Claim Payment" → "Own Child Claim Payment" | — | merged | None |
| (untracked) | DB#729 + #730 — kkeform MI indexes (#322954 — not in release ticket list) | — | merged | High (root cause unconfirmed) |
| [#322851](https://minutemenu.tpondemand.com/entity/322851) | Total income missing and signature incorrect on IEF report | Story | Dev | Not merged |
| [#322839](https://minutemenu.tpondemand.com/entity/322839) | Updates to 322549 Incorrect Provider Tier Type in State Provider Upload File | Story | Dev | Not merged |
| [#322722](https://minutemenu.tpondemand.com/entity/322722) | Payment Method Discrepancy - provider Tiffany Tucker | Story | Withdrawn | Not in release |
| [#322718](https://minutemenu.tpondemand.com/entity/322718) | KCE - Missing IEF/FRP Data Fields on Renewal Screen | Story | Released | (no PR in this release branch) |
| [#321660](https://minutemenu.tpondemand.com/entity/321660) | Providers cannot enroll children manually | Story | Withdrawn | Not in release |
| [#321481](https://minutemenu.tpondemand.com/entity/321481) | March 2026 claim for Provider 4C28A95D-67DE-... | Story | Withdrawn | Not in release |
| [#318671](https://minutemenu.tpondemand.com/entity/318671) | HX088 - Unable to Reissue Payments - February 2026 | Story | Dev | Not merged |
| [#318631](https://minutemenu.tpondemand.com/entity/318631) | Application Insight review - Provider 284003620 was removed | Story | Withdrawn | Not in release |
| [#318581](https://minutemenu.tpondemand.com/entity/318581) | Ghost eForm Enrollment Record (Not Started) | Story | Dev | Not merged |
| [#318212](https://minutemenu.tpondemand.com/entity/318212) | KKFP center needs an option to advance their claim month | Story | Code review | (no PR in this release branch) |
| [#317726](https://minutemenu.tpondemand.com/entity/317726) | OK - Berry Items Triggering "Insufficient Quantity" (Error 78) | Story | Code review | (no PR in this release branch) |
| [#317708](https://minutemenu.tpondemand.com/entity/317708) | Provider Training not validated with review | Story | Code review | (no PR in this release branch) |

**Note:** TP release entity custom fields (`Time of Release`, `Affected Machines`, `Risks`, `Test Methodology`, `Rollback Plan`) are all empty as of 2026-05-06. Dev lead to populate before deploy.
