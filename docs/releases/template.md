# Release: [RELEASE_NAME]

**Release ID:** #[RELEASE_ID]
**Prepared by:** [DEV_LEAD_NAME]
**Date:** [YYYY-MM-DD]
**Branch:** release/[BRANCH_NAME]

---

## 1. Functional Summary

<!-- What this release does from a USER perspective. 2-5 bullet points. Source: TP ticket descriptions. -->

- [Bullet point 1]
- [Bullet point 2]

---

## 2. Tickets

<!-- ALL tickets in this release. One row per ticket. -->

| Ticket | Title | Type | State | Infra Impact |
|--------|-------|------|-------|-------------|
| #[ID] | [Title] | Bug / Story | Done / Released | None / Low / Medium / High / Critical |

---

## 3. Infrastructure Impact

<!-- Source: Git diff of release branch vs master, merged PRs, Agent 5 assessments. -->

**Overall Risk Level:** None / Low / Medium / High / Critical

### 3.1 Database Changes

<!-- Every SQL change. File path + one-sentence description. Or "None". -->

- [List or "None"]

### 3.2 Auth / Session Changes

<!-- Login, SSO, tokens, sessions, cookies, "remember me". If users may need to re-auth, say so. -->

- [List or "None"]
- Expected re-authentication impact: [describe scope or "None — no auth changes"]

### 3.3 API Changes

<!-- New, changed, or removed endpoints. Include HTTP method + route. Or "None". -->

- [List or "None"]

### 3.4 Performance Considerations

<!-- Large table queries, changed stored procs, removed caching/indexes, new background jobs. Or "None". -->

- [List or "None"]

### 3.5 Cross-Repo Dependencies

<!-- If multi-repo: list repos, deploy order, rollback order. If single repo: say so. -->

- Repos involved: [list or "Single repo: KK"]
- Deploy order: [specify or "N/A"]
- Rollback order: [specify or "N/A"]

---

## 4. Web Configuration Changes

<!-- web.config, appsettings, Octopus variable changes per service. Or "None". -->

| Service | Key | Value | Notes |
|---------|-----|-------|-------|
| [Service] | [Key] | [Value] | [Why] |

---

## 5. Rollback Plan

<!-- Step-by-step. Specific enough that someone unfamiliar with the release can execute it. -->

1. [Step 1]
2. [Step 2]

---

## 6. Risk Assessment

- **Worst case:** [What happens if something goes wrong?]
- **How we detect it:** [What alert or check tells us?]
- **How we recover:** [Reference rollback plan + any additional steps]

---

## 7. Deployment Checklist

<!-- ALL rows must be Done before production deployment. -->

| # | Step | Owner | Status |
|---|------|-------|--------|
| 1 | Dev lead prepared release note | [name] | Done / Pending |
| 2 | DevOps reviewed infrastructure impact | [name] | Pending |
| 3 | Architect reviewed changes | [name] | Pending |
| 4 | SQL scripts verified on staging | [name] | Pending |
| 5 | Web config changes documented and ready | [name] | Pending |
| 6 | Rollback plan confirmed | [name] | Pending |
| 7 | Production deployment approved | [name] | Pending |
