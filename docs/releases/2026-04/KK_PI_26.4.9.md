# Release: KK PI 26.4.9

**Release ID:** #318015
**Prepared by:** [VERIFY — dev lead name]
**Date:** 2026-04-09
**Branch:** release/318015_KK_PI_26.4.9

---

## 1. Summary

- **eForms:** Cleanup orphaned enrollment reports on Send Invitations screen (#316482); fix cancel-invitation not removing reporting records (#317375); fix FRP Basis sort order on Approve & Renew (#298280).
- **Tiering (CACFP):** Fix Child Income Eligibility (Tier 1) incorrectly auto-set for all children in Tier 1 homes — now only applies to provider's own children by income (#318945); includes data migration script and cross-repo changes (KK, HxCloud, Database).
- **Claims:** Fix List Claims default filter showing only unsubmitted claims instead of all (#319629); fix Adjust Claim Count validation blocking save after removing approved meals (#293930).
- **Attendance:** Fix Certificate pop-up unresponsive when both Certify Accuracy and Auto-Calc Qty settings enabled (#317820, #319627) — covers both classrooms-attendance and classrooms-daily-attendance screens.
- **Printing/Checks:** Fix check printing at ~70% scale, alignment issues, itemized detail showing wrong amount/label for consolidated checks (#318211, #320038); remove memo line from bottom section.
- **Reports:** Fix Child List missing in Review Worksheet from Observer Mode (#317938); fix MPR milk quantity not calculating for Open Enrolled (ARAS/SFSP) centers (#317867); fix print MPR incorrect from sponsor view.
- **Enrollment/Children:** Fix expired/expiring-soon filter 1-day gap on exact expiration date (#319633); fix ChildSendInvitations API NullReferenceException (#314517); fix food audit fields saved as NULL or -1 (#307083).
- **Centers Policy:** New Policy D.33 — Use Historic Claims Data for Rosters with 6 sub-policies (D.33a–D.33f) for enrollment date, IEF expiration, FRP designation/basis, and classroom (#318497).
- **Parachute Integration:** Address sync (City, State, Zip) between Parachute and KidKare for all account types (#317756); remove Accounting PIN from KidKare pages (#319102); enhance child classroom assignment to prevent duplicates (#318618).
- **Guardian Management:** Filter out inactive contacts (status 289) from Manage Guardians tab (#316483).
- **TX Compliance:** Update Texas IEF for Child Care Center from July 2022 to March 2026 version — English and Spanish (#319410).
- **Reverted:** #311567 (Eform signature date) — withdrawn, issue confirmed resolved in current state. #317708 (Provider Training validation) — reverted.

---

## 2. Infrastructure Impact

**Overall Risk Level:** High

### 2.1 Database Changes

- `MinuteMenu.Database: 318945_Fix_child_income_eligibility_tier1.sql` — Data migration: backup child tier data, clear tier for Census/School provider children, clear tier for non-own Income children, sync dates for own Income children. Audit trail: `mod_login_id = '-318945'`.
- `MinuteMenu.Database: 317375_cleanup_orphaned_reports_after_cancel_invitation.sql` — One-time cleanup of orphaned `KK_EnrollmentReport` records left behind by prior cancel-invitation operations. Scoped to CX and HX, Submitted by Parent only. TRY/CATCH with rollback.
- `MinuteMenu.Database: 311567_revert.sql` — Revert of signature date changes from #311567.
- `MinuteMenu.Database: select_child_by_provider_id_filter_for_UP332_sp.sql` — Fixed Tier display at 4 locations: Tier 1 provider → ALL children display Tier='1' regardless of child's tier_code. Fixed bug comparing tier vs itself.
- `MinuteMenu.Database: select_rpt_422_*.sql` (Child Roster SPs) — Extended ClaimChildKeyData JOIN for D.33 policy; added per-field CASE logic for D.33a–D.33f sub-policies; fixed residual @mode IN (2,3) block and incorrect D.33 override ordering.
- `MinuteMenu.Database: SelectRptClaimsRosterSponsor.sql` — Added ClaimChildKeyData JOIN for D.33; replaced flat D.22 overlay with per-field CASE expressions.
- `MinuteMenu.Database: insert_default_independent_policies.sql` — Added D.33 and D.33a–D.33f policy defaults (all N).
- `MinuteMenu.Database: get_MilkQuantity_NonInfant_ForNeeded` — Added Open Enrolled (ARAS/SFSP) center support for milk quantity calculation using `SFSP_ATTENDANCE_ITEM`.
- `MinuteMenu.Database: MRP.build.config / MRP.sql` — Build config and release script updated.

### 2.2 Auth / Session Changes

- None
- Expected re-authentication impact: None — no auth changes

### 2.3 API Changes

- CHANGED: `POST enrollment/sponsor/ChildSendInvitations` — Fixed NullReferenceException by replacing `_userSessionState` (constructor injection) with `UserSessionState` (property injection); added null guards for `GuardianId` parsing (#314517).
- CHANGED: `PUT UpdateCenterSite` — Now also persists `city_name`, `state_code`, `zip_code` when updating center site (#317756).
- CHANGED: `GET GetMySite` — Response now includes `City`, `StateCode`, `Zip` fields (#317756).
- CHANGED: `GET GetSponsorReviewWorksheetReport` — Now uses sponsor's `current_claim_month/year` instead of `DateTime.UtcNow`; passes `@IncludePending` and `@IncludeExpired` to SP (#317938).
- CHANGED: `GET GetRecurringInvoiceProviders` — Now accepts `sponsorId` parameter for optimized provider lookup.
- NEW: `POST process-enrollment-report-cleanup` — Batch cleanup endpoint for stale enrollment reports; uses keyset pagination, batched writes of 50, `ReadUncommitted` isolation for reads (#316482).
- DISABLED: `SyncClassroomToParachute` call in `AddCxChild` is commented out — classroom sync to Parachute disabled for CX child creation (#318618).

### 2.4 Performance & Capacity

- Enrollment report cleanup API scanned 1,916,246 completed enrollment reports and cleaned up 100 orphaned record sets on QA. **Monitor** — watch execution time and DB load when run on production (one-time operation).
- D.33 roster SPs add ClaimChildKeyData JOINs to Child Roster and Claims Roster stored procedures. **Monitor** — watch SP execution time after deploy, especially for large sponsors with many claims.
- Tier data migration script (#318945) updates CHILD table records across all sponsors. **Monitor** — watch for lock contention during execution on production.
- `select_child_by_provider_id_filter_for_UP332_sp` changed at 4 locations. **Monitor** — verify child list performance for Tier 1 providers.

### 2.5 Cross-Repo Dependencies

- Repos involved: KK, POC_HxCloudConnectApi, Centers-CX, MinuteMenu.Database
- Deploy order: MinuteMenu.Database → POC_HxCloudConnectApi → KK → Centers-CX [VERIFY — confirm deploy order with DevOps]
- Rollback order: Centers-CX → KK → POC_HxCloudConnectApi → MinuteMenu.Database [VERIFY]

---

## 3. Web Configuration Changes

| Service | Key | Value | Notes |
|---------|-----|-------|-------|
| KK BE | `Settings.ReviewToolPendingChildren` | existing setting | Now read by Observer Mode review worksheet endpoint — must be present in config [VERIFY] |
| KK BE | `Settings.ReviewToolActiveChildrenWithExpiredEnrollment` | existing setting | Same — now read by Observer Mode endpoint [VERIFY] |

---

## 4. Rollback Plan

1. Deploy previous Centers-CX build via Octopus [VERIFY — build number]
2. Deploy previous KK BE + FE builds via Octopus [VERIFY — build numbers]
3. Deploy previous POC_HxCloudConnectApi build via Octopus [VERIFY — build number]
4. Run rollback SQL scripts for data migrations: reverse 318945 tier fix using backup table, reverse 317375 cleanup [VERIFY — confirm rollback scripts exist]
5. D.33 policy data: no rollback needed — new policies default to N and have no effect unless explicitly enabled by sponsors

---

## 5. Risk Assessment

- **Worst case:** Tier data migration (#318945) incorrectly updates child tier_code values, causing children to be claimed at wrong tier rates. D.33 roster SPs return incorrect data for sponsors who enable the new policy immediately after deploy.
- **How we detect it:** Sponsor reports of incorrect tier display or claims data; Application Insights exceptions on ChildSendInvitations or enrollment cleanup APIs; QA spot-check of tier data for known test sponsors (996, 117, 166, 227, 245).
- **How we recover:** Rollback per Section 4; tier data can be restored from backup table created by migration script (`mod_login_id = '-318945'`). D.33 policies can be disabled (set to N) per-sponsor without code rollback.

---

## 6. Deployment Checklist

| # | Step | Owner | Status |
|---|------|-------|--------|
| 1 | Dev lead prepared release note | [VERIFY] | Pending |
| 2 | DevOps reviewed infrastructure impact | [VERIFY] | Pending |
| 3 | Architect reviewed changes | [VERIFY] | Pending |
| 4 | SQL scripts verified on staging | [VERIFY] | Pending |
| 5 | Web config changes documented and ready | N/A | Done |
| 6 | Rollback plan confirmed | [VERIFY] | Pending |
| 7 | Production deployment approved | [VERIFY] | Pending |

---

## 7. Tickets

| Ticket | Title | Type | State | Infra Impact |
|--------|-------|------|-------|-------------|
| #318497 | Centers: New Policy D.33 – Use Historic Claims Data for Rosters | Story | Released | High |
| #318945 | Fix: Child Income Eligibility (Tier 1) incorrectly auto-set for all children in Tier 1 homes | Story | Released | High |
| #316482 | KCE: Some Expired IEFs and EFs Are Missing From The Send Invitations Screen | Story | Released | Medium |
| #317375 | Submitted (parent) eforms are not showing up for approval | Story | Released | Medium |
| #318211 | Printing check issues - HX023 | Story | Released | Low |
| #320038 | The check report is not the same as the HX app | Bug | Released | Low |
| #319629 | List Claims default filter shows only unsubmitted | Story | Released | Low |
| #317820 | Unable to close Certificate pop-up with Auto Save Actual Qty | Story | Released | Low |
| #319627 | Cert modal buttons unresponsive on Daily Attendance screen | Bug | Released | Low |
| #314517 | Error: Exception when calling API enrollment/sponsor/ChildSendInvitations | Story | Released | Medium |
| #307083 | Should log correct mod_login_id and mod_date_time when creating/updating foods | Story | Released | Low |
| #298280 | Sort Issue in eForms - Approve & Renew | Story | Released | Low |
| #317867 | OK - CX8288 - Actual Quantity Required Not Calculating | Story | Released | Medium |
| #293930 | Adjust Claim Count validation for approved meals | Story | Released | Low |
| #317938 | Child List is missing in Review Worksheet from Observer Mode | Story | Released | Low |
| #319633 | Application Insight Review - Expired/Expiring Soon filters 1-day gap | Story | UAT | None |
| #316483 | Food Program needs to handle p_participant records with record_status_code = 289 | Story | Released | Low |
| #319410 | Texas - Update IEF for Child Care Center | Story | Released | None |
| #319477 | Tier 1 Start/End Date incorrect when enrolling new Own child | Bug | UAT | None |
| #319102 | Remove Accounting PIN from KidKare pages | Story | Released | None |
| #317756 | Address Sync Between Parachute & Kidkare | Story | Released | Low |
| #318618 | Enhance assign child classroom in Parachute | Story | Released | Low |
| #311567 | Eform approved and renewed but signature date not updated | Story | Withdrawn | None |
