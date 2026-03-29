# MinuteMenu Documentation

MinuteMenu is a child care management platform for the USDA Child and Adult Care Food Program (CACFP). It serves home-based providers (KidKare), center-based providers (Centers-CX), and CACFP sponsors (HX).

This site contains cross-cutting documentation that spans multiple repositories. For repo-specific technical docs, see the `docs/` folder in each repository.

---

## Repository Map

| Repo | What It Does |
|------|-------------|
| **KK** | Primary KidKare platform for home-based providers. ASP.NET Web API + ServiceStack backend, AngularJS frontend. |
| **Centers-CX** | Child care management for center-based providers. WinForms desktop client, CXWeb web app, claims engine. |
| **HX** | Legacy VB6 CACFP sponsor desktop application. Processes claims, manages providers. |
| **hx_cloudconnectionAPI** | REST API bridging HX desktop app to cloud services (Azure storage, SSO, caching). |
| **hx_sponsorDatabase** | SQL migration scripts for the HX Sponsor database. |
| **SingleSignOn-Service** | Centralized SSO service for authentication across KK, CX, Parachute, and HX. |
| **Parachute** | Parent/provider-facing self-service portal. Registration, subscription, and payment. |
| **MinuteMenu.Database** | SQL migration/deployment repo for all databases: CXADMIN, MMADMIN, MMADMIN_EFORM, MMADMIN_PARACHUTE. |
| **DistributedProcessing** | Distributed claims processing via NServiceBus + Azure Storage. |
| **ClaimsProcessor.Converted** | VB6-to-C# .NET 8 migration of claims processing logic. |

---

## Quick Links

- [System Overview](architecture/system-overview.md) - How the platform fits together
- [SAML SSO - Client Migration Guide](flows/authentication/saml-sso-client-migration-guide.md) - Migrate existing clients to SSO
- [SAML SSO - Complete Guide](flows/authentication/saml-sso-complete-guide.md) - Full SSO setup for existing and new clients
- [CX SFSP/ARAS Overview](components/cx-sfsp-aras/overview.md) - SFSP/ARAS features for center-based providers
