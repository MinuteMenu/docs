# Release: KK PI 26.4.22

**Release ID:** #318103
**Prepared by:** Tuan Nguyen
**Date:** 2026-04-21
**Branch:** release/318103_KK_PI_26.4.22

---

## 1. Summary

- **Claims engine:** Error 76 infant-disallowance logic added for HX030 (J.017 policy), Child Reconciliation report clears DO/PU instead of In/Out times (#319145 regression fix), OER R/I/R,I character correction for infants, HX211 snack-capacity formula fix, Full Month Attendance ISNULL wrap for missing child names.
- **Meal Counts with FRP:** New policy L.11 (`INCLUDE_ALL_RECORDED_MEALS_FRP_FLAG`) lets a client opt in to include all recorded meals regardless of disallowance (#318686).
- **Sponsor 102 Print Check:** Cross-repo print pipeline overhaul — new `Sponsor102Data` (Active Child List, tier, training hours, meal times) flows through HX API → KK → PDF; new ACH re-download without void/reissue; new CHECK_REGISTER covering index (#316306).
- **J.017 policy (HX030 own-infant edit-check):** New `SPONSOR_POLICY.include_infants_in_own_alone_edit_check_flag` column + HX API entity + VB6 property + HX Desktop v3.4.0 build; seeded Y for sponsor 030 only (#320244).
- **State-specific tiering:** New `STATE_POLICY.use_school_tier_exact_date_flag` set Y for Iowa (state 20) — allows custom school tier start/end dates (#319205).
- **Enrollment / eForm:** New admin endpoints `common/detectOrphanedEforms` and `common/cleanupOrphanedEforms` (security-key auth) to purge orphaned CX eForm assignments; CX Activate Children null-enrollment fix (#308617); home-sponsor-observer can add parents (#318629).
- **Reports & Texas IEF:** TX Child-Care IEF instruction pages removed (#320684); TX Adult-center IEF now correctly routed via `program_type_code` (#321830); `select_rpt_401_child_enrollment_sp` updated.
- **Spanish localization:** Added translations for Home Provider flows (#298391) — partial coverage, some keys still English (see Risk).
- **Provider policy defaults:** `highest_serving_allowed_code` defaults to 215 (Serving 1) when saved as 0/NULL (#320881).
- **Data patch:** Restore foster-child historical IE dates wiped by PI #318945 (#321413, HX102 request).
- **Parachute (bundled):** Standardized search/clear-X across accounting pages, kiosk PIN/phone numpad responsive, number-input caret fix, State Agency split-amount zero-out fix.

---

## 2. Infrastructure Impact

**Overall Risk Level:** **High**

### 2.1 Database Changes

- `MMADMIN/Updates/319205_Add_use_school_tier_exact_date_flag.sql` — adds `use_school_tier_exact_date_flag` column to `STATE_POLICY`, sets 'Y' for state_code=20 (Iowa).
- `MMADMIN_XXX/Updates/320244_ADD_NEW_FIELDS_TO_TABLE_SPONSOR_POLICY.sql` — adds `include_infants_in_own_alone_edit_check_flag` to `SPONSOR_POLICY` (NOT NULL DEFAULT 'N').
- `MMADMIN_XXX/Updates/320244_SET_DEFAULT_FIELDS_TO_TABLE_SPONSOR_POLICY.sql` — backfills existing rows to 'N'.
- `MMADMIN_XXX/Updates/320244_SET_SPONSOR_030_INCLUDE_INFANTS_FLAG.sql` — sets 'Y' for sponsor scan_code='030'. **[VERIFY — currently NOT included in regenerated MRP.sql manifest; requires manual DBA run, see section 2.4]**
- `MMADMIN_XXX/Updates/321413_Restore_Foster_Child_Income_Eligibility.sql` — one-off data patch to restore foster children's historical IE dates from `CHILD_TIER_BACKUP_318945`.
- `MMADMIN_XXX/Programmability/Stored Procedures/save_sponsor_policy_sp.sql` — new trailing parameter `@include_infants_in_own_alone_edit_check_flag flag = 'N'`.
- `MMADMIN_XXX/Programmability/Stored Procedures/save_provider_policy_sp.sql` and `save_provider_policy_sp_port_kk.sql` — 215 default applied when input is 0 or NULL.
- `MMADMIN_XXX/Programmability/Functions/claim_error_meal_attendance_count_fn.sql` — full rewrite; Error 76 branch adds CLAIM_ERROR→PROVIDER→SPONSOR_POLICY join per invocation to read J.017.
- `MMADMIN_XXX/Programmability/Functions/claim_error_meal_attendance_source_form_fn.sql` — adds MEAL_ATTENDANCE scan for source_form_code 151/152 to return 'I', 'R', or 'R,I'.
- `MMADMIN_XXX/Programmability/Stored Procedures/select_361126_full_month_attendance_worksheet_sub1_sp.sql` — wraps child name columns with `ISNULL(..., '')`.
- `MMADMIN_XXX/Programmability/Stored Procedures/select_361127_Child_Reconciliation_sp.sql` and CXADMIN equivalent — adds second UPDATE on `#PGRID` to clear DO/PU on RECTYPE='A' rows (fix for #319145).
- `MMADMIN_XXX/Views/STATE_POLICY.sql` and `STATE_POLICY_TD.sql` — rebuilt to expose new state flag column.
- `CXADMIN/Updates/316306_create_check_register_client_center_covering_index.sql` — new non-clustered covering index `IX_CHECK_REGISTER_client_id_center_id` on `CHECK_REGISTER` with 7 INCLUDE columns. **Rollback script exists.**
- `CXADMIN/Updates/318686_add_new_policy_L11_include_all_recorded_meals_frp.sql` — inserts policy L.11 into `POLICY`. Default L.11 CLIENT_POLICY value is effectively 'N' via ISNULL fallback (no CLIENT_POLICY row inserted by this release).
- `CXADMIN/Programmability/Stored Procedures/select_payment_history_list_sp.sql` — emits 3 new columns: `mod_date_time`, `mod_login_id`, `mod_username`. Consumed by CX `LogicService` + KK AutoMapper.
- `CXADMIN/Programmability/Stored Procedures/select_rpt_meal_count_attendance_sp.sql` — flips `@respectDisallowance` default 1→0; policy-L11-driven. Fallback `ISNULL` preserves legacy behavior when CLIENT_POLICY row absent.
- `CXADMIN/Programmability/Stored Procedures/select_rpt_401_child_enrollment_sp.sql` — blank IEF branch now uses `ISNULL(CLC.program_type_code, 0)` instead of hardcoded `0` (supports TX adult/child IEF split).

### 2.2 Auth / Session Changes

- None
- Expected re-authentication impact: None — no auth changes.

### 2.3 API Changes

- NEW: `POST /common/detectOrphanedEforms` (security-key auth) — enumerates CX clients with orphaned eForm assignments. Payload: `{ SecurityKey, ClientIds?[] }`.
  => Internal tool not public for client
- NEW: `POST /common/cleanupOrphanedEforms` (security-key auth) — hard-deletes orphaned CX eForm assignments. Payload: `{ SecurityKey, ClientIds?[] }`. **When ClientIds is null/empty, the endpoint sweeps every CX client with any active assignment.** [VERIFY — see section 2.4.]
  => Internal tool not public for client
- NEW: `POST /portcx/payment/redownloadACHFile` — regenerate ACH file for a set of check_ids without voiding. Payload: `{ CheckIds: int[] }` (permission-gated to `CX.IssuePayments`).
  => redownloadACHFile only regenerates the ACH file for existing DD records (payment_type_code=175). It does not void, modify, or otherwise touch any check_register data. Purely a read + file-reconstruction operation. Protected by [RequiredPermission(CX.IssuePayments)]
- NEW: `GET /site/isAtRiskOrArasForSponsor?centerId=...` — returns live at-risk/ARAS flag for a center. **[VERIFY — endpoint added but call site not visible in diff; confirm it is consumed.]**
  => Won't fix — the endpoint, resource, and service method (centerSiteDetailsService.getIsAtRiskOrArasForSponsor) are kept intentionally for future features that may need the live At Risk check (e.g., enrollment workflows). The Child Roster report no longer calls it, but removing it would require re-adding it later.
- CHANGED: `POST /portcx/fileservice/updateIEFChildren` — return shape changed from `bool` to `{ success, skippedChildNames[] }`.
- CHANGED: `GET /cxservice/payments/getCheckRegisterList` — response now includes `modDateTime`, `modLoginId`, `modUsername` on each row.
- CHANGED: `POST /payments/printIssuePayment` (HX API) — response now includes `Sponsor102Data { ActiveChildren[], ChildTierCounts[], ProviderTiers[], ServingTimes[], TrainingHours }` for Sponsor 102 providers.
- CHANGED: HX API `SponsorPolicy` entity — new property `include_infants_in_own_alone_edit_check_flag`. HX API `StatePolicy` DTO (via `SelectStatePolicyDTO`) — new `use_school_tier_exact_date_flag`.

### 2.4 Performance & Capacity

- **CHECK_REGISTER covering index (#316306)** — new non-clustered index with 7 INCLUDE columns on a multi-million-row table. CREATE NONCLUSTERED does not specify `ONLINE = ON` → Sch-M lock for duration of build, blocks all reads/writes to CHECK_REGISTER.
  **Action required** We are using standard edition license for sql server that does not support ONLINE = ON indexing.
- **Orphan detect/cleanup endpoints (#319630)** — `common/cleanupOrphanedEforms` with auto-detect (null `ClientIds`) performs serial per-client fan-out: opens fresh eFormsContext + CxContext per iteration and pulls entire active CHILD roster per client. No `CommandTimeout`. No row cap.
  **Action required** — Internal tool. No action needed.
- **Sponsor 102 PrintIssuePayment fan-out (#320213)** — HX API controller uses unbounded `Task.WhenAll` calling `BuildSponsor102Data` per check; each call fires 5 stored procedures in parallel. A 500-provider batch print = 2,500 concurrent DB round-trips.
  **Monitor** — spike in MMADMIN connection count during first post-release 102 print. Will monitor post release.
  Rollback: Medium — requires HX API redeploy to revert.
  Watch: `sp_who2`/`sys.dm_exec_connections` during first 102 batch print; HX API p99 latency on `/payments/printIssuePayment`.
  Evidence: No SemaphoreSlim, no caching across providers sharing tier/fiscal data.
- **`claim_error_meal_attendance_count_fn` Error 76 branch (#320244/#321350)** — adds CLAIM_ERROR→PROVIDER→SPONSOR_POLICY join per invocation. Scalar function called per CLAIM_ERROR row in OER/summary reports.
  **Monitor** — execution time of `select_claim_error_summary_sp` and `select_claim_errors_for_rpt_sp` post-deploy for sponsors with high Error 76 volume.
  Rollback: Low — revert function via CREATE OR ALTER.
  Watch: Report query duration on OER for HX030 and HX102.
  Evidence: Pattern #1-style transformation from pre-resolved flag to per-row lookup.
- **ACH re-download CommandTimeout = 300s (#316306)** — `GetCenterACHFileFromCheckRegister` sets 5-minute command timeout for an indexed IN-lookup.
  **Monitor** — occurrences of 5-minute SQL waits on `check_register`.
  Rollback: Low — code redeploy.
  Watch: SQL query execution time on ACH redownload calls.
  Evidence: Pattern #4 anti-pattern (oversized timeouts mask real issues).
- **MRP_Rollback scope gap** — `CXADMIN/MRP_Rollback.build.config` only covers `select_payment_history_list_sp`. Rollbacks for 318686 policy INSERT, `select_rpt_meal_count_attendance_sp`, `select_361127_Child_Reconciliation_sp`, `select_rpt_401_child_enrollment_sp` are NOT in the bundled rollback.
  **Action required** — Already ask developer to commit the rollback script.
- **MRP missing sponsor-030 enablement (#320244)** — `320244_SET_SPONSOR_030_INCLUDE_INFANTS_FLAG.sql` exists in repo but is NOT in the regenerated MRP manifest. Feature will not activate for HX030 unless DBA runs manually.
  **Action required** — either regenerate MRP with the script included (sequenced after SET_DEFAULT), OR document manual DBA run post-deploy with product-owner sign-off.
  Rollback: Low — `UPDATE SPONSOR_POLICY SET include_infants_in_own_alone_edit_check_flag='N' WHERE sponsor_id = (SELECT sponsor_id FROM SPONSOR WHERE scan_code='030')`.
  Watch: HX030's Error 76 behavior post-deploy — if no change observed, the script was not run.
  Evidence: MRP manifest inspection.

### 2.5 Cross-Repo Dependencies

- Repos involved: **MinuteMenu.Database, POC_HxCloudConnectApi (HX API), HX (Desktop/VB6 — build 3.4.0), Centers-CX, Shared-Core, KK, Parachute**
- Deploy order: **MinuteMenu.Database → Shared-Core → POC_HxCloudConnectApi → HX Desktop (3.4.0) → Centers-CX → KK → Parachute**
  - Rationale: DB schema change must precede any EF-mapped reader (HX API `SponsorPolicy` entity); HX Desktop v3.4.0 reads the new column unconditionally and will fail sponsor-policy load on a pre-upgrade DB; KK consumes the updated CX `MinuteMenu.Centers.ServerLib.dll` plus SP signature changes.
- Rollback order: **Parachute → KK → Centers-CX → HX Desktop → POC_HxCloudConnectApi → Shared-Core → MinuteMenu.Database**

---

## 3. Web Configuration Changes

| Service | Key             | Value | Notes                                                                            |
| ------- | --------------- | ----- | -------------------------------------------------------------------------------- |
| HX API  | (none per diff) | —    | `settings.xml` changed but no new key surfaced in review — **[VERIFY]** |
| KK BE   | (none)          | —    | No new appsettings key added in this release                                     |

**[VERIFY]** — Confirm whether `HxCloudConnect.Api/settings.xml` changes require corresponding prod-environment config update.

---

## 4. Rollback Plan

1. **Halt further traffic to printIssuePayment endpoint** (Sponsor 102 print) if rollback is decided mid-day to avoid incomplete PDF batches.
2. Deploy previous Parachute build.
3. Deploy previous KK BE + FE build.
4. Deploy previous Centers-CX build (restores the pre-change `MinuteMenu.Centers.ServerLib.dll`).
5. Reinstall prior HX Desktop installer (pre-3.4.0) on affected sponsors — **HX Desktop rollback is per-machine; communicate with affected sponsor operators.**
6. Deploy previous HX API build.
7. Deploy previous Shared-Core build (reporting service file replacement).
8. Run DB rollback scripts:
   - `CXADMIN/ROLLBACK/316306_create_check_register_client_center_covering_index.sql` (drops covering index)
   - `CXADMIN/RELEASES/MRP_Rollback.sql` (restores `select_payment_history_list_sp`)
   - **Manual rollback required** for: `select_rpt_meal_count_attendance_sp`, `select_361127_Child_Reconciliation_sp`, `select_rpt_401_child_enrollment_sp`, `claim_error_meal_attendance_count_fn`, `claim_error_meal_attendance_source_form_fn`, `select_361126_full_month_attendance_worksheet_sub1_sp`, `save_sponsor_policy_sp`, `save_provider_policy_sp`, `save_provider_policy_sp_port_kk` — DBA to check out prior commits from master and re-deploy each SP.
   - `DELETE FROM POLICY WHERE policy_name='INCLUDE_ALL_RECORDED_MEALS_FRP_FLAG'` and any related `CLIENT_POLICY` rows.
   - `UPDATE SPONSOR_POLICY SET include_infants_in_own_alone_edit_check_flag='N'` (reset J.017 to legacy) — or, if full column removal is required, `ALTER TABLE SPONSOR_POLICY DROP COLUMN include_infants_in_own_alone_edit_check_flag`.
   - `ALTER TABLE STATE_POLICY DROP COLUMN use_school_tier_exact_date_flag` — remove state tier flag.
   - `321413_Restore_Foster_Child_Income_Eligibility.sql` cannot be rolled back directly — foster IE dates written back into `CHILD` are correct by intent. If reversal is needed, restore `CHILD` rows from `CHILD_TIER_BACKUP_318945` for the affected rows.
9. Post-rollback: run HX030 Error 76 regression test and an OER smoke for one large sponsor to confirm claim-processing pipeline is back to pre-release behavior.

---

## 5. Risk Assessment

- **Worst case:** (1) HX API outage on production if deploy order is reversed — the EF-mapped `SponsorPolicy` entity reads a column that doesn't exist yet; every route using SponsorPolicies breaks. (2) Mass eForm data loss if the cleanup endpoint is invoked in auto-sweep mode against the current orphan predicate (withdrawn-children = orphan). (3) Silent overwrite of `PROVIDER_POLICY.highest_serving_allowed_code` to 215 on every HX Desktop save → incorrect meal allowance on next claim processing for any provider configured to Serving 2/3.
- **How we detect it:** (a) APM error rate on HX API for "Invalid column name 'include_infants_in_own_alone_edit_check_flag'" within 5 min of HX API deploy; (b) APM on `/cxservice/payments/getCheckRegisterList` for `IndexOutOfRangeException` if CX/KK deploys before DB; (c) sudden drop in `PROVIDER_POLICY.highest_serving_allowed_code != 215` count over a 24h window on HX030/HX102; (d) spike in `EnrollmentAssignmentCx` hard-delete rate (DB audit or `sp_BlitzCache` "Deletes per second" counter).
- **How we recover:** Follow Rollback Plan section 4. For (c), restore `PROVIDER_POLICY` from nightly backup. For (d), restore `EnrollmentAssignmentCx`, `EnrollmentForm`, `IncomeEligibilityForm`, and related storage from nightly backup; notify affected clients; re-enter lost eForms if recovery is incomplete.

---

## 6. Deployment Checklist

| #  | Step                                                                                                 | Owner            | Status  |
| -- | ---------------------------------------------------------------------------------------------------- | ---------------- | ------- |
| 1  | Dev lead prepared release note                                                                       | Tuan Nguyen      | Done    |
| 2  | DevOps reviewed infrastructure impact (esp. deploy order + orphan cleanup endpoint risk)             | [VERIFY — name] | Pending |
| 3  | Architect reviewed changes (J.017 cross-repo contract + Sponsor 102 fan-out)                         | [VERIFY — name] | Pending |
| 4  | SQL scripts verified on staging (MRP + 320244 set + 321413 + 316306 index)                           | [VERIFY — name] | Pending |
| 5  | Web config changes documented and ready                                                              | [VERIFY — name] | Pending |
| 6  | Rollback plan confirmed (including manual SP rollback steps)                                         | [VERIFY — name] | Pending |
| 7  | Confirm `320244_SET_SPONSOR_030_INCLUDE_INFANTS_FLAG.sql` added to MRP OR scheduled for manual run | [VERIFY — name] | Pending |
| 8  | Verify `CHILD_TIER_BACKUP_318945` exists on each HX sponsor DB before running 321413               | [VERIFY — name] | Pending |
| 9  | 316306 index DDL scheduled for maintenance window OR confirmed Enterprise Edition for ONLINE=ON      | [VERIFY — name] | Pending |
| 10 | HX Desktop 3.4.0 rollout plan coordinated with sponsor support (per-machine installer)               | [VERIFY — name] | Pending |
| 11 | Sponsor 102 print-batch SQL connection monitor staged                                                | [VERIFY — name] | Pending |
| 12 | Cleanup-orphaned-eForm endpoint firewalled OR code-gated before production access                    | [VERIFY — name] | Pending |
| 13 | Production deployment approved                                                                       | [VERIFY — name] | Pending |

---

## 7. Tickets

| Ticket                                                  | Title                                                       | Type  | State                | Infra Impact                                   |
| ------------------------------------------------------- | ----------------------------------------------------------- | ----- | -------------------- | ---------------------------------------------- |
| [#296656](https://minutemenu.tpondemand.com/entity/296656) | [Sponsor] Run button disabled switching SFSP & ARAS         | Story | Withdrawn            | None                                           |
| [#297720](https://minutemenu.tpondemand.com/entity/297720) | [MPR ARAS/SFSP] Rqd Serving Size not displayed              | Story | Withdrawn            | None                                           |
| [#298391](https://minutemenu.tpondemand.com/entity/298391) | Add Spanish translation to KidKare for Home Providers       | Story | Ready for Release    | Low                                            |
| [#299748](https://minutemenu.tpondemand.com/entity/299748) | KCE: At Risk Portion Of Child Roster Missing                | Story | UAT                  | Low                                            |
| [#300571](https://minutemenu.tpondemand.com/entity/300571) | CMK: Diet Statement on File Diet Exp Date disabled          | Story | Withdrawn            | None                                           |
| [#300738](https://minutemenu.tpondemand.com/entity/300738) | IL State Provider File incorrect Begin Date                 | Story | Withdrawn            | None                                           |
| [#300918](https://minutemenu.tpondemand.com/entity/300918) | Copy options not remembered Infant/Non-Infant tabs          | Story | Ready for Release    | Low                                            |
| [#308617](https://minutemenu.tpondemand.com/entity/308617) | [CX] Error when Set IEF with NULL Current Enrollment        | Story | UAT                  | Low                                            |
| [#316306](https://minutemenu.tpondemand.com/entity/316306) | KK Centers Re-Generate Payments Without Void/Reissue        | Story | Ready for Release    | **High**                                 |
| [#318117](https://minutemenu.tpondemand.com/entity/318117) | Geo-Location Missing HX300 Reviews                          | Story | UAT                  | Low                                            |
| [#318178](https://minutemenu.tpondemand.com/entity/318178) | Child Attendance Reconciliation overlapping DO/PU           | Story | Ready for Release    | Medium                                         |
| [#318581](https://minutemenu.tpondemand.com/entity/318581) | Ghost eForm Enrollment Record (Not Started)                 | Story | Dev                  | **[VERIFY — no code change]**           |
| [#318584](https://minutemenu.tpondemand.com/entity/318584) | Receipt Verification does not list receipts                 | Story | Code review          | Low                                            |
| [#318629](https://minutemenu.tpondemand.com/entity/318629) | Home sponsors cannot add parents when observing             | Story | Ready for Release    | Low                                            |
| [#318686](https://minutemenu.tpondemand.com/entity/318686) | Meal Counts & Attendance With FRP missing meal data         | Story | UAT                  | Medium                                         |
| [#318784](https://minutemenu.tpondemand.com/entity/318784) | Child Roster licenseId override uses live At-Risk           | Bug   | UAT                  | Low                                            |
| [#319145](https://minutemenu.tpondemand.com/entity/319145) | #318178 Fix clears In/Out instead of DO/PU on RECTYPE='A'   | Bug   | UAT                  | Medium                                         |
| [#319149](https://minutemenu.tpondemand.com/entity/319149) | Center Search Not Functioning After Clear                   | Story | UAT                  | Low                                            |
| [#319156](https://minutemenu.tpondemand.com/entity/319156) | Blank Attendance + Served Meals shows withdrawn children    | Story | UAT                  | Low                                            |
| [#319205](https://minutemenu.tpondemand.com/entity/319205) | HX100 School Start/End date update                          | Story | Ready for Release    | Medium                                         |
| [#319407](https://minutemenu.tpondemand.com/entity/319407) | Some children names missing on Full Month Attendance        | Story | Ready for Release    | Low                                            |
| [#319413](https://minutemenu.tpondemand.com/entity/319413) | HX 102 Void payment affected all providers                  | Story | Analysis             | **[VERIFY]**                             |
| [#319630](https://minutemenu.tpondemand.com/entity/319630) | CX client 8141: Child Not Found approving eForms            | Story | Code review          | **High**                                 |
| [#320213](https://minutemenu.tpondemand.com/entity/320213) | [Sponsor 102 Print Check] Data not populated                | Story | Test                 | **High**                                 |
| [#320224](https://minutemenu.tpondemand.com/entity/320224) | Eliminate depreciatecx dependency — Move eForm/FRP to KK   | Story | Dev                  | **[VERIFY]**                             |
| [#320244](https://minutemenu.tpondemand.com/entity/320244) | HX030 Error 76 disallow own infants                         | Story | Ready for Release    | **Critical**                             |
| [#320295](https://minutemenu.tpondemand.com/entity/320295) | Need assistance tracking meals for Louisiana                | Story | Withdrawn            | None                                           |
| [#320305](https://minutemenu.tpondemand.com/entity/320305) | #320224 - Specifications                                    | Task  | Open                 | None                                           |
| [#320443](https://minutemenu.tpondemand.com/entity/320443) | A provider cannot enter meal                                | Story | Released             | Low                                            |
| [#320684](https://minutemenu.tpondemand.com/entity/320684) | Texas - Remove IEF instructions pages (Child Care)          | Story | Ready for Release    | Low                                            |
| [#320881](https://minutemenu.tpondemand.com/entity/320881) | PCI Highest Serving Allowed empty                           | Story | UAT                  | Medium                                         |
| [#320883](https://minutemenu.tpondemand.com/entity/320883) | Error 113 firing after PI318945 cleared child tier          | Story | PDT Input            | **[VERIFY]**                             |
| [#320884](https://minutemenu.tpondemand.com/entity/320884) | Actual Quantity Served of all foods in March removed        | Story | Code review          | Low                                            |
| [#320891](https://minutemenu.tpondemand.com/entity/320891) | Cash In Lieu not calculated for KKFP client 3923            | Story | Completed by Support | None                                           |
| [#321350](https://minutemenu.tpondemand.com/entity/321350) | OER shows R character for Infant (J.017=Y)                  | Bug   | UAT                  | Medium                                         |
| [#321398](https://minutemenu.tpondemand.com/entity/321398) | Missing data in the OER report                              | Bug   | UAT                  | **[VERIFY — empty ticket description]** |
| [#321408](https://minutemenu.tpondemand.com/entity/321408) | HX211 Provider Capacity snack meal counts                   | Story | Ready for Release    | Low                                            |
| [#321413](https://minutemenu.tpondemand.com/entity/321413) | Restore Historical IE Dates for Foster Children             | Story | Test                 | **Medium — data patch**                 |
| [#321481](https://minutemenu.tpondemand.com/entity/321481) | Determine how March 2026 claim was deleted                  | Story | Support              | None                                           |
| [#321485](https://minutemenu.tpondemand.com/entity/321485) | Completed eForms not appearing under Reporting              | Story | Withdrawn            | None                                           |
| [#321486](https://minutemenu.tpondemand.com/entity/321486) | Kidkare Slowness — pages not loading                       | Story | Released             | None                                           |
| [#321487](https://minutemenu.tpondemand.com/entity/321487) | Issue Payments Partial Posting Recurrence 4/3               | Story | Released             | None                                           |
| [#321662](https://minutemenu.tpondemand.com/entity/321662) | Claim: Unable to disallow Pending children after Withdrawal | Story | Ready For Test       | **[VERIFY — no code change]**           |
| [#321830](https://minutemenu.tpondemand.com/entity/321830) | Texas - IEF for Adult center showing as IEF for child       | Story | Ready for Release    | Medium                                         |
| [#322138](https://minutemenu.tpondemand.com/entity/322138) | Data different between KK and HX app (Print Check)          | Bug   | UAT                  | Medium                                         |

**[VERIFY] items for dev lead:**

- **#318581, #321662** — no code change found in any release diff. Confirm with engineering whether in scope.
- **#319413, #320224, #320305, #320883** — in Analysis/Dev/PDT Input/Open states; confirm inclusion or move out.
- **#321398** — empty ticket description; no code references. Confirm scope.
