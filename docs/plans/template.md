# Plan: [Feature Title]

**Ticket:** #TICKET_ID
**Author:** [Name]
**Date:** YYYY-MM-DD
**Status:** Draft | Under Review | Approved | Superseded

## Problem

What is the issue or need? 1-2 sentences.

## Solution

What will you change? Keep it high level -- describe the approach, not the code.

## Architecture Decisions

Answer each that applies. Delete the rest.

- [ ] **Data access change?** Do I change how data is queried? (new tables, removed views, new joins on large tables, replaced materialized views)
- [ ] **Auth/session change?** Does this affect login, session, tokens, or "remember me"? Could this force users to re-authenticate?
- [ ] **Cross-repo change?** Which repos are touched? What is the deploy order?
- [ ] **New API calls?** New endpoints or increased call frequency? Estimate additional load.
- [ ] **Database schema change?** New tables, columns, indexes, stored procedures?
- [ ] **Infrastructure dependency?** Does this need VM scaling, new config, connection pool changes, or new services?
- [ ] **Rollback plan?** How do we undo this if something goes wrong after deployment?

## Repos and Key Files

List the repos involved and the main files that will change.

| Repo | Key Files | What Changes |
|------|-----------|-------------|
| KK | | |
| SSO | | |

## Risk

What could go wrong? How do we detect it? How do we roll back?

## Test Approach

How will this be verified? Include both functional testing and performance/load considerations.

## Implementation PRs

_Updated as work progresses._

| PR | Repo | Description | Status |
|----|------|-------------|--------|
| | | | |
