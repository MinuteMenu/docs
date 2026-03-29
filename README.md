# MinuteMenu Documentation

Central documentation for the MinuteMenu platform. Built with [MkDocs Material](https://squidfunk.github.io/mkdocs-material/).

## View Online

Visit: [https://minutemenu.github.io/docs/](https://minutemenu.github.io/docs/)

## Local Development

```bash
# One-time setup
pip install mkdocs-material

# Preview locally with live reload
mkdocs serve
# Open http://localhost:8000
```

## Adding Documentation

1. Create a `.md` file under `docs/`
2. Add it to the `nav` section in `mkdocs.yml`
3. Submit a pull request

Changes merged to `main` are automatically deployed via GitHub Actions.

## Structure

```
docs/
├── index.md                    # Home page
├── architecture/               # System-level architecture docs
├── flows/                      # Cross-repo workflow documentation
│   └── authentication/         # SSO, SAML, login flows
├── components/                 # Component overviews (EForm, Auth, Claims, etc.)
├── onboarding/                 # New team member guides
└── processes/                  # Team processes (RCA, deployment, etc.)
```

For repo-specific technical docs, see the `docs/` folder in each individual repository.
