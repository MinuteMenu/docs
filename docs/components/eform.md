# EForm

EForm handles electronic food program forms for CACFP sponsors. Providers submit meal attendance data, centers review it, and sponsors use it to generate claims.

## Repos Involved

| Repo | Role |
|------|------|
| **KK** | Provider-side eform submission and validation |
| **Centers-CX** | Center-side eform processing and review |
| **HX** | Sponsor-side claim generation from eforms |
| **MinuteMenu.Database** | EForm schema (MMADMIN_EFORM database) |

## How It Works (high level)

1. Provider submits eform in KK
2. Center reviews in CX
3. Sponsor processes claim in HX

*Detailed documentation for each step lives in the respective repo's `docs/` folder.*
