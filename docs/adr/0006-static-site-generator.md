# Static-Site Generator for Documentation

## Status
Proposed

#### Date
2025-06-30

#### Owner

[Denis Filatov](https://github.com/denifilatoff)

#### Participants and approvers

- [pankratovsa](https://github.com/pankratovsa)
- [shumnic](https://github.com/shumnic)
- [Alexey Karasev](https://github.com/asatt)
- [IldarMinaev](https://github.com/IldarMinaev)
- [Vladimir Sitnikov](https://github.com/vlsi)
- [egbu](https://github.com/egbu)

#### Related ADRs
None

## Context
We need to publish existing Markdown documentation stored in `docs/` to GitHub Pages while keeping:
- One repository for code + docs
- Pure-Markdown sources (minimal config files)
- Powerful search
- CI-only build – no committed `/site` artifacts
- Clean Git history
- Built-in versioning of releases

Several generators were investigated: MkDocs, Hugo, Jekyll, Docusaurus, VuePress, Docsify (see Justification).

## Decision
We will adopt **MkDocs Material + Mike + GitHub Actions → Pages artifact**.

### Best Practices & Statements
• Follows DevOps best practices (clean Git history, separation of source and build artifacts).

### Justification
1. **Material Design theme out-of-the-box** — professional, responsive UI without custom CSS/design work.
2. Very low learning curve — Markdown + YAML only, no templating required.
3. Rich built-in features: search, navigation, admonitions, code highlighting, diagram & math extensions.
4. `mike` plugin provides flexible versioning with customizable version display; competitors need custom scripts.
5. Pure-Python toolchain fits existing infra; no extra Go/Ruby/React runtimes.
6. GitHub Actions workflow is officially documented; deploys to Pages via artifact upload -> keeps history clean.
7. Strong community support: MkDocs Material (23.4k★)

Alternatives rejected:
- **Hugo** (81.2k★) – most popular and fastest generator, but requires hunting/configuring many plugins, basic themes need styling work, versioning only via custom scripts.
- **Jekyll** (50.1k★) – native to GitHub Pages but Ruby toolchain and lacks versioning.
- **Docusaurus** (60.1k★) – React/MDX stack heavier than needed.
- **VuePress** (22.8k★) / **Docsify** (29.5k★) – weaker plugin ecosystem for versioning & search.

   *Note: MkDocs Material has 23.4k★, showing solid community adoption despite being more specialized.*

## Consequences
Positive
- Fast setup; minimal config files committed.
- Automatic multi-version docs via `mike release` CI step.
- Clean main branch; build artifacts kept in Pages storage.

Neutral
- Python runtime added to CI image.

Negative
- Theme and plugins occasionally release breaking changes – must track MKDocs Material changelog.
- `mike` keeps each version in separate branch under the hood, adding some repository refs to maintain.
