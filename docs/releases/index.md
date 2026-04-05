# Release Notes

Technical release notes for each production release. Generated after all tickets merge to the release branch and testing passes. Shared with DevOps and architect for approval before production deployment.

## Purpose

- Give DevOps and architect a clear picture of what is in the release
- Highlight infrastructure impact (database changes, auth changes, API changes)
- Document deploy order for multi-repo releases
- Get explicit sign-off before production deployment

## How It Works

1. All tickets merge to release branch
2. QA testing passes
3. Run `/release-review` skill with the release branch or TP release ID
4. Skill auto-generates a technical release note from TP tickets and git diffs
5. Review the generated note, fill in any missing details
6. Commit to `releases/YYYY-MM/RELEASE_NAME.md` and open PR
7. DevOps and architect review and approve
8. Deploy to production

## Release Notes

Release notes are organized by month.
