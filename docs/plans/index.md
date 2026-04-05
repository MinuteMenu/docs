# Feature Plans

Implementation plans for major features. Plans are required for features that span weeks or months, touch multiple repos, or change core flows (auth, payments, claims).

## When a Plan is Required

- Feature/Epic that takes multiple sprints to complete
- Changes that touch 3+ repos
- Core flow changes: authentication, payments, claims processing
- Database schema changes affecting shared tables
- Infrastructure changes (new services, connection pool changes, VM requirements)

## When a Plan is NOT Required

- Individual tickets (bug fixes, small enhancements)
- Config changes
- UI text or style changes
- Single-repo changes with clear scope

## How to Create a Plan

1. Copy the [plan template](template.md)
2. Save to `plans/YYYY-MM/TICKET_ID-short-description.md`
3. Open a PR in this docs repo
4. Assign reviewers: architect, dev lead, DevOps (if infra impact), client stakeholder (if needed)
5. Iterate until approved
6. Update status to "Approved" and start implementation

## Plans

Plans are organized by month. Each plan links to its implementation tickets and PRs.
