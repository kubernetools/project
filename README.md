# www.kubernetools.com — Kubernetes API Documentation

A self-hosted, SEO-optimized, accessible reference for all Kubernetes API resources, generated automatically from the official OpenAPI specs.

---

## Goal

Provide a fast, version-aware, machine-readable alternative to the official Kubernetes API docs at `www.kubernetools.com/docs/latest/` (stable, search-engine-indexed) and `www.kubernetools.com/docs/<version>/` (version-pinned, noindex).

---

## Architecture Overview

```
kubernetes/kubernetes (OpenAPI specs)
        │
        ▼
  [spec-fetcher]  ← workflow_dispatch GitHub Action
        │           binary in: kubernetools/specs — writes to: kubernetools/specs
        ▼
  [docgen] (Rust CLI)                              repo: kubernetools/docgen
        │   Parses OpenAPI JSON → renders HTML + sitemap + structured data
        ▼
  kubernetools/site  (per-version HTML tree, packaged as .tgz release)
        │
        ▼
  kubernetools/deploy  (GH workflow: downloads site .tgz, builds container image)
        │   Base image: registry.access.redhat.com/hi/nginx:latest
        │   Site files embedded in image; Nginx configured at build time
        ▼
  Nginx / Kubernetes Ingress  ←  www.kubernetools.com  (config: kubernetools/infra — planned)
```

---

## Roadmap

### Phase 1 — Foundations

**Repository structure**

The project lives under the [`kubernetools`](https://github.com/kubernetools) GitHub organization, split across purpose-scoped repositories:

| Repository | Purpose |
|---|---|
| [`kubernetools/project`](https://github.com/kubernetools/project) | Project roadmap and documentation (this repo) |
| [`kubernetools/specs`](https://github.com/kubernetools/specs) | Cached OpenAPI v3 JSON per Kubernetes version + Rust `spec-fetcher` binary |
| [`kubernetools/docgen`](https://github.com/kubernetools/docgen) | Rust CLI: parses specs from `specs` and emits static HTML |
| [`kubernetools/site`](https://github.com/kubernetools/site) | Generated static output, packaged as a `.tgz` release on each `docgen` run |
| [`kubernetools/deploy`](https://github.com/kubernetools/deploy) | GH workflow: downloads site release, builds and pushes the Nginx container image |
| `kubernetools/infra` *(planned)* | Kubernetes manifests, Ingress + cert-manager config |

**Toolchain decision: Rust**

Rust is viable and preferred for performance. Key crates:
- [`serde_json`](https://crates.io/crates/serde_json) — OpenAPI v3 spec parsing (see note below)
- [`minijinja`](https://crates.io/crates/minijinja) — HTML templating
- [`clap`](https://crates.io/crates/clap) — CLI interface
- [`tokio`](https://crates.io/crates/tokio) + [`reqwest`](https://crates.io/crates/reqwest) — async spec fetching

**Note on OpenAPI parsing**: [`openapiv3`](https://crates.io/crates/openapiv3) targets OpenAPI 3.0.x (which Kubernetes ships) but has a known deserialization bug with typeless / `AnySchema` fields ([#96](https://github.com/glademiller/openapiv3/issues/96)). Kubernetes uses these for `IntOrString`, `RawExtension`, and similar types, so hitting the bug is likely. The crate is also lightly maintained. Preferred approach: a bespoke `serde_json` deserializer targeting only the fields `docgen` needs (name, description, type, `$ref`, properties, `x-kubernetes-*` extensions) — simpler, no upstream dependency risk, and read-only use means full spec round-tripping is unnecessary.

**Deliverables**
- [x] Parse a single OpenAPI v3 group spec and print a resource index
- [x] Emit one well-formed HTML page per API group/version/kind
- [x] Local preview via `python -m http.server` against `site/`

---

### Phase 2 — Content & Structure

**URL scheme**

```
/docs/latest/                        → index of all API groups (latest version, generated directly)
/docs/latest/core/v1/pod/           → Pod resource page (canonical, indexed by search engines)
/docs/latest/apps/v1/deployment/    → Deployment resource page
/docs/v1.36/core/v1/pod/            → version-pinned page; noindex + canonical → /docs/latest/core/v1/pod/
/docs/v1.33/core/v1/pod/            → older version; noindex + canonical → /docs/latest/core/v1/pod/
...
```

`/docs/latest/` is a physical directory generated directly by `docgen` when invoked with `--is-latest` — links within it use `/docs/latest/...` absolute paths. All versions (including the current one) are regenerated from scratch on each release. This keeps `/docs/latest/...` URLs stable across Kubernetes releases so bookmarks, external links, and search-engine indices never break. Version-specific paths remain accessible for direct linking and version comparison but are excluded from SEO.

**Per-resource page content**
- Resource description (from spec `description` field)
- Group / Version / Kind badge
- Field reference table: name, type, required, description, sub-fields (recursive)
- Spec/Status split (mirrors official docs convention)
- `x-kubernetes-*` extension annotations (list-map-keys, patch strategy, etc.)
- Link to the upstream spec source on GitHub

**Index pages**
- API group index (core, apps, batch, networking.k8s.io, …)
- Full alphabetical resource index
- Version selector dropdown (all available versions)

**Deliverables**
- [ ] Full recursive field rendering for all resource types
- [ ] Cross-links between related resources (e.g. PodSpec → Container)
- [ ] Version selector nav component
- [ ] `docgen --is-latest` generates `/docs/latest/` with absolute `/docs/latest/...` links

---

### Phase 3 — SEO & Accessibility

**SEO**
- Semantic HTML5 (`<article>`, `<nav>`, `<section>`, `<header>`)
- Unique, descriptive `<title>` and `<meta name="description">` per page
- Canonical URLs: `/docs/latest/...` pages carry a self-referencing canonical; version-specific pages (`/docs/v1.36/...`) carry `<link rel="canonical" href="/docs/latest/...">` pointing at the stable URL
- Version-specific paths get `X-Robots-Tag: noindex` via Nginx so crawlers skip them without needing per-page `<meta>` tags
- `sitemap.xml` contains only `/docs/latest/...` URLs (version-specific paths omitted entirely)
- `robots.txt` allowing full crawl of `/docs/latest/`; `Disallow: /docs/v` to block version-specific crawling as a belt-and-suspenders measure
- JSON-LD structured data (`TechArticle` / `APIReference` schema)
- OpenGraph tags for social sharing
- Stable, clean URLs (no query strings, no fragments for primary content)

**Accessibility (WCAG 2.1 AA)**
- All interactive elements keyboard-navigable
- ARIA landmarks and labels on nav, search, tables
- Color contrast ratios ≥ 4.5:1 (text) / 3:1 (UI components)
- `<table>` with `<caption>` and `<th scope>` for field tables
- Skip-to-content link
- Responsive layout (mobile-first CSS, no horizontal scroll)

**Performance**
- Zero JavaScript required for content pages (pure static HTML)
- Inline critical CSS, defer non-critical
- Precompressed (gzip + brotli) assets served by Nginx
- `Cache-Control: max-age=31536000, immutable` on versioned assets
- `Cache-Control: no-cache` on `/docs/latest/` pages (content changes on each release)

**Deliverables**
- [ ] Lighthouse score ≥ 90 on Performance, Accessibility, Best Practices, SEO
- [ ] Validate sitemap with Google Search Console
- [ ] axe / pa11y CI check on generated HTML

---

### Phase 4 — Automation & CI

**Spec fetcher**

A script (or small Rust binary) that:
1. Queries the Kubernetes GitHub releases API for new stable releases within the supported window (N to N-3)
2. Downloads `api/openapi-spec/v3/*.json` for each version not yet cached in `kubernetools/specs`
3. Opens a PR (or pushes directly) to `kubernetools/specs` and triggers a `docgen` run; drops specs for versions that have left the support window

**GitHub Actions workflows (current state)**

All workflows are currently `workflow_dispatch` (manually triggered):

| Repo | Workflow | Trigger | What it does |
|------|----------|---------|--------------|
| `specs` | `fetch-specs.yml` | manual | builds `spec-fetcher`, fetches specs for a given minor version, opens a PR |
| `docgen` | `release.yml` | on tag `v*` | builds Linux + macOS binaries and publishes a GitHub release |
| `site` | `build-release.yml` | manual | downloads latest `docgen` release, regenerates site, opens a PR |
| `site` | `release.yml` | on tag `v*` | packages `site/` as a `.tgz` and publishes a GitHub release |
| `deploy` | `build-image.yml` | on tag `v*` | downloads the site `.tgz`, builds and pushes the Nginx container image tagged with the version (see below) |

Target pipeline (not yet automated end-to-end):
```
fetch-specs (cron) → docgen (on specs PR merge) → validate-html → deploy
```

**Container image build (`kubernetools/deploy`)**

The `build-image.yml` workflow in `kubernetools/deploy` is triggered by a tag push (`v*`). The pushed tag determines the version of the site release to embed:

1. **Downloads the site release** — fetches the `.tgz` artifact from `kubernetools/site` whose tag matches the pushed tag.
2. **Builds the container image** — uses `registry.access.redhat.com/hi/nginx:latest` as the base image. All static site files are copied into the image at build time (no runtime volume mounts needed). The tag version is embedded in the image (e.g. injected into the Nginx config or as image labels).
3. **Configures Nginx** — an `nginx.conf` baked into the image handles:
   - Static file serving with precompressed (gzip + brotli) asset delivery
   - `X-Robots-Tag: noindex` on all `/docs/v*/` locations
   - `Cache-Control: max-age=31536000, immutable` on versioned assets; `Cache-Control: no-cache` on `/docs/latest/`
4. **Pushes the image to the GitHub Container Registry** (`ghcr.io/kubernetools/deploy`) — tagged with the pushed version tag and as `latest`.

Kubernetes deployment manifests will live in `kubernetools/infra` *(not yet created)*.

**Deliverables**
- [ ] Automated nightly check for new Kubernetes releases (convert `fetch-specs` to cron + auto-merge)
- [ ] Zero-downtime deploy (rolling update or blue-green on the Ingress level)
- [ ] `cert-manager` + Let's Encrypt TLS for `www.kubernetools.com`

---

### Phase 5 — Quality & Extras

- [ ] Search (client-side with a pre-built index, e.g. [pagefind](https://pagefind.app/) — zero server cost)
- [ ] Diff view between two versions of the same resource
- [ ] `kubectl explain`-style one-liner output per field (copyable)
- [ ] API changelog: auto-generated list of fields added/removed/changed between versions
- [ ] Dark mode (CSS `prefers-color-scheme`)
- [ ] Feedback widget (simple GitHub Issue link pre-filled with page context)

---

## Spec Sources

Only the **four community-supported minor versions** (currently v1.33–v1.36) are tracked, matching the [Kubernetes community support window](https://kubernetes.io/releases/patch-releases/#support-period).

All supported versions ship OpenAPI v3 specs (introduced in v1.23), so no Swagger 2.0 fallback is needed.

| Format | Location in `kubernetes/kubernetes` |
|---|---|
| OpenAPI 3.0 (per group) | `api/openapi-spec/v3/<group>/<version>/openapi.json` |

Fetched from GitHub raw content at the tagged commit for each release, e.g.:  
`https://raw.githubusercontent.com/kubernetes/kubernetes/v1.33.0/api/openapi-spec/v3/apis__apps__v1_openapi.json`

---

## Non-Goals (v1)

- Interactive API explorer / try-it-out (adds auth complexity, out of scope)
- CRD documentation (custom resources are cluster-specific)
- Documentation for alpha/beta APIs that are not in the official spec
- Vendoring or mirroring full Kubernetes source

---

## Open Questions

1. **OpenAPI parsing strategy**: lean toward a bespoke `serde_json` deserializer over the `openapiv3` crate (see Phase 1 note). Revisit if the scope of fields needed grows significantly.
2. **Static files vs server-side rendering**: static is simpler and cheaper; add SSR only if search or diffing requires it.
3. **Versioning cutoff**: the four community-supported minor versions (N to N-3). Currently v1.33–v1.36; drops the oldest when a new minor is released.
