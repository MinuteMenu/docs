# MinuteMenu Documentation Site — AI Agent Instructions

## What This Repo Is

Central documentation for the MinuteMenu platform. Built with [MkDocs Material](https://squidfunk.github.io/mkdocs-material/). Deployed to GitHub Pages at https://minutemenu.github.io/docs/.

This is **Tier 2** of a two-tier documentation system:

- **Tier 1 (per-repo `docs/` folders)**: Implementation-specific docs that live next to the code. Example: KK/docs/eform/data-model.md
- **Tier 2 (this repo)**: Cross-cutting docs that span multiple repos. Architecture, flows, components, onboarding, processes.

## What Goes Here vs Per-Repo

| Question | If yes → |
|----------|----------|
| Does it only involve one repo's code? | That repo's `docs/` folder |
| Does it describe a flow across 2+ repos? | This repo → `docs/flows/` |
| Is it a high-level overview of a component? | This repo → `docs/components/` |
| Is it about how the team works? | This repo → `docs/processes/` |
| Is it for new team members? | This repo → `docs/onboarding/` |
| Is it about how all systems connect? | This repo → `docs/architecture/` |

## Project Structure

```
MinuteMenu-docs/
├── mkdocs.yml                              # Navigation + config (MUST update when adding pages)
├── docs/
│   ├── index.md                            # Home page with repo map
│   ├── architecture/                       # System-level architecture
│   │   └── system-overview.md
│   ├── flows/                              # Cross-repo workflow docs
│   │   └── authentication/
│   │       ├── overview.md                 # All auth mechanisms overview
│   │       ├── saml-sso-architecture.md    # SAML technical deep dive
│   │       ├── saml-sso-client-migration-guide.md
│   │       ├── saml-sso-complete-guide.md
│   │       └── images/                     # Screenshots for auth docs
│   ├── components/                         # Component overviews
│   │   ├── eform.md
│   │   ├── authentication.md
│   │   └── claims-processing.md
│   ├── onboarding/                         # (planned) New team member guides
│   └── processes/                          # (planned) Team processes
└── .github/workflows/docs.yml              # Auto-deploy on push to main
```

## How to Add a New Page

1. Create a `.md` file in the appropriate folder under `docs/`
2. **Add it to `mkdocs.yml` under `nav:`** — this is required. Pages not in `nav` won't appear in the sidebar.
3. Submit a PR. Changes deploy automatically when merged to `main`.

## Writing Conventions

- **Simple English.** Short sentences. Common words. The team is mostly non-native English speakers.
- **Use mermaid diagrams** for architecture and flows. Prefer flowcharts (`graph TD/LR`) over sequence diagrams for simplicity. Use sequence diagrams only when the back-and-forth timing matters.
- **Use ASCII art** (code blocks) for simple layouts where mermaid is overkill.
- **Use tables** for structured data (field mappings, comparisons, inventories).
- **Images** go in an `images/` subfolder next to the markdown file that uses them. Use hyphens in filenames (no spaces). Reference with relative paths: `./images/my-screenshot.png`.
- **No code dumps.** This is architecture documentation. Show file paths and class names for reference, but don't paste large code blocks. Readers should go to the source repo for code details.
- **Link between tiers.** Central docs should link to per-repo docs when referencing implementation details. Example: "See [KK eform submission flow](https://github.com/MinuteMenu/KK/blob/master/docs/eform/submission-flow.md)".

## MkDocs Specifics

- Config file: `mkdocs.yml` — controls navigation, theme, and markdown extensions
- Mermaid diagrams work out of the box (configured via `pymdownx.superfences`)
- Every page gets an "Edit this page" pencil icon linking to GitHub
- Search is built-in and works across all pages
- Dark mode toggle is enabled

### Local Preview

```bash
pip install mkdocs-material
mkdocs serve
# Open http://localhost:8000
```

### Build Check

```bash
mkdocs build --strict
# Fails if nav references a missing file
```

## Deployment

- **Auto-deploy**: GitHub Actions runs on every push to `main`. Builds the site and deploys to `gh-pages` branch.
- **GitHub Pages**: Serves from `gh-pages` branch at https://minutemenu.github.io/docs/
- **PR validation**: PRs trigger a build (no deploy) to catch broken nav or markdown errors.

## Related Repos

This documentation covers the MinuteMenu ecosystem:

| Repo | What it does |
|------|-------------|
| **KK** | KidKare platform for home-based providers |
| **Centers-CX** | Center-based provider management |
| **HX** | Legacy VB6 CACFP sponsor application |
| **hx_cloudconnectionAPI** | REST API bridging HX to cloud services |
| **SingleSignOn-Service** | Centralized SSO authentication |
| **Parachute** | Self-service portal (registration, payments) |
| **MinuteMenu.Database** | SQL migration scripts for all databases |
| **DistributedProcessing** | Claims processing via NServiceBus |
| **ClaimsProcessor.Converted** | VB6-to-.NET 8 claims migration |

When writing docs that reference code in these repos, always specify which repo and file path.
