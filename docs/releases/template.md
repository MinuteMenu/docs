# Release: [Release Name]

**Release ID:** #RELEASE_ID
**Date:** YYYY-MM-DD
**Branch:** release/RELEASE_BRANCH

## Tickets in This Release

| Ticket | Title | Type | Infra Impact |
|--------|-------|------|-------------|
| #ID | Title | Bug/Story | None/Low/Medium/High |

## Summary of Changes

Brief description of what this release does from a user perspective.

## Infrastructure Impact

### Database Changes

- Schema changes: [list or "none"]
- Stored procedure changes: [list or "none"]
- New indexes: [list or "none"]
- Data migration scripts: [list or "none"]

### Auth/Session Changes

- Login flow changes: [describe or "none"]
- Session/token changes: [describe or "none"]
- Expected re-authentication impact: [describe or "none"]

### API Changes

- New endpoints: [list or "none"]
- Changed endpoints: [list or "none"]
- Removed endpoints: [list or "none"]
- Expected traffic pattern changes: [describe or "none"]

### Performance Considerations

- Queries on large tables: [list or "none"]
- Removed optimizations: [list or "none"]
- New background jobs: [list or "none"]

### Cross-Repo Dependencies

- Repos involved: [list]
- Deploy order: [specify order]
- Rollback order: [specify reverse order]

## Web Configuration Changes

List any web.config / appsettings changes needed per environment.

| Service | Key | Value | Notes |
|---------|-----|-------|-------|
| | | | |

## Risk Assessment

What is the worst-case scenario? How likely? How do we detect it?

## Deployment Checklist

| Step | Owner | Status |
|------|-------|--------|
| DevOps review | [name] | Pending |
| Architect review | [name] | Pending |
| SQL scripts verified | [name] | Pending |
| Web config changes documented | [name] | Pending |
| Rollback plan confirmed | [name] | Pending |
| Production deployment approved | [name] | Pending |
