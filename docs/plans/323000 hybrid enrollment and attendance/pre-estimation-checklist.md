# Pre-Estimation Checklist

**Companion to:** [323000 brain storming](./323000%20brain%20storming.md) · [aras-today](./aras-today.md)
**Date:** 2026-05-13
**Status:** Pre-meeting prep

> **Goal.** Walk into the estimation meeting with a shared picture, an agreed data shape, and no surprises. The list below splits the work so Thang and Trinh can prepare in parallel and meet only on the items that need both heads.

---

## Priority legend

- **High** — must be done before the estimation meeting. Sizing depends on it.
- **Medium** — should be done if possible; otherwise we cover it live in the meeting.
- **Low** — not blocking the meeting. Defer or pick up after.

---

## The checklist

| # | Item | Owner | Deliverable | Priority |
|---|---|---|---|---|
| 1 | Get the full picture of the hybrid feature. Read the plan — what we are building, why, what Jay has decided, and what is still open. See [`323000 brain storming.md`](./323000%20brain%20storming.md). | **Both** | Shared mental model | High |
| 2 | Understand how today's **closed-enrolled** ARAS works. The path where every child is fully enrolled, the system checks school-out days per child, and each meal goes through nine eligibility rules. This is the path the hybrid feature is *not* using — know it so we can speak to what we are leaving behind. See [`aras-today.md`](./aras-today.md), per-child sections. | **Thang Vu** | Able to walk the team through it in 5 min | Medium |
| 3 | Understand how today's **open-enrolled / count-only** ARAS works. The simpler path that records meal totals only, runs menu and food rules, and skips all the per-child machinery. This is the path the hybrid feature *will* inherit. See [`aras-today.md`](./aras-today.md), count-based sections. | **Trinh Nguyen** | Able to walk the team through it in 5 min | High |
| 4a | Map where program type drives **backend** behavior today. For each area in the "Where program type changes behavior today" section below, find the code that asks *"is this an ARAS / SFSP center?"* — because hybrid will need to be taught about itself in those places. | **Thang Vu** | A list of hit-spots that will need a hybrid update | High |
| 4b | Map where program type drives **frontend, reporting, and admin** behavior today. Same idea as 4a, but for the screens, reports, and the sponsor configuration tool. | **Trinh Nguyen** | A list of hit-spots that will need a hybrid update | High |
| 5 | Agree on the data shape for the new tables. Where does each piece of information live? How does the running total stay in sync with the per-child detail? The answer affects what every other story has to do. | **Both** (joint whiteboard) | A short markdown with table shape (or pseudo-DDL) committed to this folder | High |
| 6 | Walk through what a center user actually does to use the hybrid feature end-to-end. From login to seeing the claim reimbursement. Write down each step. If the story is clear, "done" is clear. | **Trinh** drafts, **Thang** reviews | One short markdown with click-by-click steps | Medium |
| 7 | Find out where our changes belong. Claim-processing logic lives in two layers in the code — an older one and a newer one. We need to know which one to update. Pair with someone who knows the migration history. | **Thang Vu** | Two-sentence answer appended to `aras-today.md` follow-up notes | High |
| 8 | Look at the two questions still waiting on product — **Q3b** (can a walk-in child be converted to full enrollment later) and **Q3d** (do hybrid kids appear in normal sponsor reports). Decide whether engineering can settle them ourselves or whether they go back to product. | **Thang** leads, **Trinh** reviews | Either an engineering-proposed answer, or a fresh product question posted to TP | Low |

---

## Working order

Items **1–4** are independent — both can prepare in parallel. Items **5–7** are joint or paired. Item **8** can run any time, including after the meeting.

If anything in **4a** or **4b** surfaces a new big question, raise it *before* the estimation meeting — not inside it.

---

## Where program type changes behavior today

The list below describes what the system *does* in business terms first, then names the code keywords to grep. Anything that comes back with hits is a place where hybrid may need to opt in or out.

### Backend — for item 4a (Thang)

| What the system does today | Keywords to grep |
|---|---|
| **Decides whether a meal is reimbursable** | `isLicenseAtRiskOrEmergencyShelter`, `IsCenterLicenseAtRisk`, `IsAnyMealAtRisk`, Rules 41, 43, 44, 45, 48, 49, 58, 59, 67, 95, 97 |
| **Asks "is this an ARAS or SFSP center?"** (the 8 callsites we mapped) | `IsCenterARASSFSP`, proposed `IsCenterARASOrSFSP`, `IsCenterARASSFSPWithSingleLicense` |
| **Tells the FE what kind of center the user logged into** | `IsOpenEnrollCenter`, `IsOpenEnrollCenterNonLA`, proposed `IsARASHybrid` / `IsSFSPHybrid` |
| **Saves attendance and meal counts** | `/centers/meal/sfspattendance/save`, `/centers/meal/bulksfspattendance/save`, `AttendanceService.VerifyAtRiskMeals`, `SFSP_ATTENDANCE` writes |
| **Marks each meal row as at-risk eligible** | `at_risk_breakfast_flag` and the six meal-type variants, `at_risk_*_disallow_flag` (including the misspelled `at_risk_pm_snack_disalow_flag`) |
| **Blocks Paid kids from claiming under SFSP-closed-enrolled** | `ChildBll.AddChild:343`, `ChildBll.UpdateChild:242`, `EnrollmentBll.cs:686` |
| **Sets a child's reimbursement level (Free / Reduced / Paid)** | `FrpCategory.Free`, `frb_category_code`, IEF expiration paths |
| **Checks if a child is too young or too old for the program** | `allowed_starting_age_*`, `allowed_ending_age_*`, Rule 45 (`ChildTooOld`) |
| **Checks which days and meals a child is approved for** | `attend_breakfast_flag`, `attend_<weekday>_flag`, Rules 58/59 |
| **Decides if school is out for the day** (closed-enrolled only) | `CALENDAR_ENTRY`, `SchoolEntryEligibleDetector`, `SchoolCalendarService`, `SchoolYearResolver` |
| **Produces state-specific claim exports** | `LouisianaClaimLoader`, other state loaders, `check_school_age_for_new_year_*` |
| **Sends child records to the parent portal (Parachute)** | `isSyncToParachute`, `SaveChild` SOAP call, guardian creation in `ChildBll.AddChild` |
| **Picks per-child vs count-based claim processing at run time** | `IsCenterARASSFSPWithSingleLicense` in `LogicService.cs:7137` |

### Frontend, reporting, admin — for item 4b (Trinh)

| What the system does today | Keywords / files |
|---|---|
| **Renders the Meals & Attendance screen** | `cx-aras-sfsp-meal-attendance-component`, `isOpenEnrollCenterNonLA` gate, `*ngIf` on the component root |
| **Asks "what kind of center is this user on?" on the front end** | `permission-service.js`, `isOpenEnrollCenter`, `isOpenEnrollCenterNonLA`, proposed `isARASHybrid` / `isSFSPHybrid` |
| **Shows the multi-step Enroll Child wizard** | `sponsor-manage-children/enroll-child/`, the `data.steps` array |
| **Prints the sign-in sheet for sponsors / centers** | `select_rpt_at_risk_after_school_sign_in_sheet_sp` (two copies — `MinuteMenu.Database` and `Centers-CX`), `AtRiskAfterSchoolSignInSheet.rpt`, `AtRiskAfterSchoolSignInSheetWeek.rpt`, `SignInSheetWeeklyPushReport` |
| **Produces sponsor month-end and year-over-year reports** | sponsor monthly headcount, year-over-year reports, anything that joins `CHILD` to `CENTER_LICENSE` filtering by program type |
| **Provides the desktop tool sponsors use to configure centers** | CX Admin (WinForms) — License/Schedule tab, "Enroll Center", sponsor configuration screens |
| **Tracks edits to child records over time** | `HISTORIC_DATA` change tracking (not used by hybrid — verify no incidental hooks fire) |
| **Decides who can add a child, mark attendance, save meals** | role gates on enrollment, walk-in, meal save (existing CACFP enrollment permission set) |
| **Renews child enrollments yearly** | `Renew Child Enrollments` action, expiration tracking |

---

## What we are NOT doing in this round

- **Asking product about Q3b and Q3d.** Jay's separate-table decision is enough to start. Defer until implementation hits a real wall.
- **Locking the routing decision.** We come back to it after items 5 (data shape) and 7 (which layer) are done.
- **Building UI mockups for 323020.** Story is in *UI Needed* state — UX delivers, then we estimate the FE component changes separately.

---

## Output of the estimation meeting

A T-shirt size per story (S / M / L / XL) with a one-line "this depends on X". The story-by-story sizes in §13 of the brainstorming doc are the starting point — but with the architecture confirmed (separate tables, no school-calendar check, no per-child rules), several should shrink.
