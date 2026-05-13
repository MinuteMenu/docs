# Pre-Estimation Checklist

**Companion to:** [323000 brain storming](./323000%20brain%20storming.md) · [aras-today](./aras-today.md)
**Date:** 2026-05-13
**Status:** Pre-meeting prep

> **Goal.** Walk into the estimation meeting with a shared picture, a locked schema, and no surprises. The list below splits the work so dev1 and dev2 can run in parallel where possible and meet only on what needs both heads.

---

## The checklist

| # | Item | Owner | Deliverable | Suggested time |
|---|---|---|---|---|
| 1 | Read [`323000 brain storming.md`](./323000%20brain%20storming.md) — the full plan, including §7 engineering decisions and §8 with Jay's confirmed answers. | **Both** | Shared mental model | 30 min |
| 2 | Read [`aras-today.md`](./aras-today.md) — focus on the **per-child claim path** (`Process`, `ChildChecker`, Rules 95/97). Understand how closed-enrolled ARAS works today, so we know what we are *not* using. | **dev1** | Be ready to explain it in 5 min during the meeting | 30 min |
| 3 | Read [`aras-today.md`](./aras-today.md) — focus on the **count-based path** (`ProcessArassfsp`, menu/food rules 16/21/22/100). This is the path hybrid will land on. | **dev2** | Be ready to explain it in 5 min during the meeting | 30 min |
| 4a | Grep for program-type touchpoints on the **backend** — KK BLL and Centers-CX claims engine. Use the keyword list below (§"Touchpoint keywords"). | **dev1** | List of hit-spots that need a hybrid update | 1 hr |
| 4b | Grep for program-type touchpoints on the **frontend, reports, and admin** — KK Web, Centers-CX reports, CX Admin WinForms. Same keyword list. | **dev2** | List of hit-spots that need a hybrid update | 1 hr |
| 5 | Lock the new-tables schema together. Decide column types, nullability, indexes, the FK from `SFSP_CHILD_ATTENDANCE` to `OPEN_ENROLLED_CHILD`, and the recalculation strategy for the `SFSP_ATTENDANCE` rollup (trigger vs. explicit method call). | **Both** | Short markdown with table DDL (or pseudo-DDL) committed to this folder | 30–45 min whiteboard |
| 6 | Draft the happy-path smoke-test scenario end-to-end: log in → see hybrid M&A screen → add walk-in → save → confirm `SFSP_CHILD_ATTENDANCE` row + `SFSP_ATTENDANCE` rollup → process claim → see reimbursement. | **dev2** drafts, **dev1** reviews | One short markdown with click-by-click steps | 30 min |
| 7 | Resolve the `csProcessClaimBusiness` (ServerLib) vs `ProcessClaimService` (Madagascar.Core) question. For our hybrid changes, **which file do we modify?** Pair with a senior who knows the migration history. | **dev1** | Two-sentence answer appended to `aras-today.md` follow-up notes | 30-min pairing |
| 8 | Review §8 questions still marked `asked` — **Q3b** (promotion path) and **Q3d** (reporting parity). Decide whether engineering can settle them or whether they go back to product. | **dev1** leads, **dev2** reviews | Either an engineering-proposed answer or a fresh product question on TP | 20 min |

---

## Working order

Items **1–4** are independent — both devs can run them in parallel. Items **5–7** need joint or paired time. Item **8** can run at any point.

If anything in **4a** or **4b** surfaces a new big question, raise it *before* the meeting — not inside it.

---

## Touchpoint keywords (for items 4a + 4b)

Grep each keyword in the relevant codebase. Anything that comes back with hits is worth a closer look — it is a place where program-type drives behavior today and hybrid may need to opt in or out.

### Backend (dev1 — KK BLL + Centers-CX)

| Area | Keywords / files |
|---|---|
| Claim-processing rules | `isLicenseAtRiskOrEmergencyShelter`, `IsCenterLicenseAtRisk`, `IsAnyMealAtRisk`, Rules 41, 43, 44, 45, 48, 49, 58, 59, 67, 95, 97 |
| KK BLL gates | `IsCenterARASSFSP` (8 callsites — see §11 of brainstorming), proposed `IsCenterARASOrSFSP`, `IsCenterARASSFSPWithSingleLicense` |
| Login flags | `IsOpenEnrollCenter`, `IsOpenEnrollCenterNonLA`, proposed `IsARASHybrid` / `IsSFSPHybrid` |
| Meal save | `/centers/meal/sfspattendance/save`, `/centers/meal/bulksfspattendance/save`, `AttendanceService.VerifyAtRiskMeals`, `SFSP_ATTENDANCE` writes |
| At-risk flag columns | `at_risk_breakfast_flag` and the six meal-type variants, `at_risk_*_disallow_flag` (including the misspelled `at_risk_pm_snack_disalow_flag`) |
| CACFP Paid restriction | `ChildBll.AddChild:343`, `ChildBll.UpdateChild:242`, `EnrollmentBll.cs:686` |
| FRP defaulting | `FrpCategory.Free`, `frb_category_code`, IEF expiration paths |
| Age range | `allowed_starting_age_*`, `allowed_ending_age_*`, Rule 45 (`ChildTooOld`) |
| Schedule rules | `attend_breakfast_flag`, `attend_<weekday>_flag`, Rules 58/59 |
| School calendar | `CALENDAR_ENTRY`, `SchoolEntryEligibleDetector`, `SchoolCalendarService`, `SchoolYearResolver` — closed-enrolled only |
| State exports | `LouisianaClaimLoader`, other state-named loaders, `check_school_age_for_new_year_*` |
| Parachute sync | `isSyncToParachute`, `SaveChild` SOAP call, guardian creation in `ChildBll.AddChild` |
| Routing decision | `IsCenterARASSFSPWithSingleLicense` in `LogicService.cs:7137` |

### Frontend / reports / admin (dev2 — KK Web + reports + CX Admin)

| Area | Keywords / files |
|---|---|
| ATMC component | `cx-aras-sfsp-meal-attendance-component`, `isOpenEnrollCenterNonLA` gate, `*ngIf` in the component HTML |
| Permission service | `permission-service.js`, `isOpenEnrollCenter`, `isOpenEnrollCenterNonLA`, proposed `isARASHybrid` / `isSFSPHybrid` |
| Enrollment wizard | `sponsor-manage-children/enroll-child/`, the `data.steps` array, multi-step flow |
| Sign-in sheet | `select_rpt_at_risk_after_school_sign_in_sheet_sp` (two copies — `MinuteMenu.Database` and `Centers-CX`), `AtRiskAfterSchoolSignInSheet.rpt`, `AtRiskAfterSchoolSignInSheetWeek.rpt`, `SignInSheetWeeklyPushReport` |
| Sponsor reports | sponsor monthly headcount, year-over-year reports, anything that joins `CHILD` to `CENTER_LICENSE` filtering by program type |
| CX Admin (WinForms) | License/Schedule tab, "Enroll Center", sponsor configuration screens |
| Audit / history | `HISTORIC_DATA` change tracking (not used by hybrid, but verify no incidental hooks fire) |
| Permissions / roles | role gates on enrollment, walk-in, meal save (existing CACFP enrollment permission set) |
| Annual renewal | `Renew Child Enrollments` action, expiration tracking |

---

## What we are NOT doing in this round

- **Asking product about Q3b and Q3d.** Jay's separate-table decision is enough to start. Defer until implementation hits a real wall.
- **Locking the router (`IsCenterARASSFSPWithSingleLicense`) change.** That comes after items 5 (schema) and 7 (file resolution) are done.
- **Building UI mockups for 323020.** Story is in *UI Needed* state — UX will deliver, then we estimate the FE component changes separately.

---

## Output of the estimation meeting

A T-shirt size per story (S / M / L / XL) with a one-line "this depends on X". The story-by-story sizes in §13 of the brainstorming doc are the starting point — but with the architecture confirmed (separate tables, no school-calendar check, no per-child rules), several should shrink.
