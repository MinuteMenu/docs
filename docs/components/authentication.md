# Authentication

Authentication in MinuteMenu is handled by a centralized Single Sign-On (SSO) service. All products (KK, CX, Parachute, HX) authenticate through this service.

## Repos Involved

| Repo | Role |
|------|------|
| **SingleSignOn-Service** | Central SSO service, token management, SAML federation |
| **KK** | KidKare login flow, session management |
| **Centers-CX** | Center login flow |
| **Parachute** | Self-service portal login |
| **hx_cloudconnectionAPI** | HX cloud authentication bridge |

## Key Features

- **Native login**: Username/password authentication against SSO service
- **SAML SSO**: Federated authentication via Microsoft Entra ID (Azure AD)
- **Cross-product sessions**: Single sign-on across KK, CX, and Parachute

## Related Docs

- [Authentication Overview](../flows/authentication/overview.md) - How all three auth mechanisms work (username/password, SAML, service-to-service)
- [SAML SSO Architecture](../flows/authentication/saml-sso-architecture.md) - Technical flow, component diagram, security design
- [SAML SSO - Client Migration Guide](../flows/authentication/saml-sso-client-migration-guide.md) - Migrate existing clients to SSO
- [SAML SSO - Complete Guide](../flows/authentication/saml-sso-complete-guide.md) - Full SSO setup guide
