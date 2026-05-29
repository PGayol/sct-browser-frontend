# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is the **SNOMED CT Browser** frontend — a jQuery-based SPA that browses, searches, and displays SNOMED CT medical terminology concepts. It communicates with a **Snowstorm** backend REST API.

## Build Commands

```bash
npm install          # Install dependencies
grunt                # Full build: compile templates, concatenate, minify JS/CSS, copy to dist
grunt handlebars     # Recompile Handlebars templates only
grunt jshint         # Lint JS (not in default task)
```

No test suite is configured (`npm test` exits with an error).

**Docker:**
```bash
docker build -t snomedinternational/snomedct-browser .
docker run --name snowstorm-nginx -d -p 80:80 --env API_HOST=<snowstorm-host> snomedinternational/snomedct-browser
```

The `API_HOST` env var is injected into the nginx config at container startup via `docker/docker-entrypoint.sh`.

## Architecture

### Build Pipeline

Grunt orchestrates the build in this order:
1. `clean` — clears `dist/` and `internal-libs/`
2. `handlebars` — compiles `.hbs` templates in `snomed-interaction-components/views/` into `snomed-interaction-components/views/compiled/templates.js`
3. `concat` — bundles all JS plugins and CSS into concatenated files
4. `uglify` / `cssmin` — minifies the concatenated bundles
5. `copy` — moves built artifacts to `internal-libs/` for use by `index.html`

**After editing any `.hbs` template or JS plugin, run `grunt` before testing in the browser.** The browser loads minified bundles from `internal-libs/`, not the source files directly.

### Component System

All UI features live in `snomed-interaction-components/js/` as jQuery plugins. Components:
- Are self-contained jQuery plugins (e.g., `$.fn.searchPlugin()`)
- Communicate exclusively via **postal.js** pub/sub message bus — direct coupling between plugins is avoided
- Render via **Handlebars** templates compiled into `views/compiled/templates.js`

Key plugins:
| Plugin | File | Role |
|---|---|---|
| Search | `searchPlugin.js` | Full-text concept search with semantic tag filtering |
| Concept Details | `conceptDetailsPlugin.js` | Main concept view (descriptions, relationships, refsets) — largest file |
| Taxonomy | `taxonomyPlugin.js` | Hierarchical concept browser |
| Query | `queryPlugin.js` | Advanced SNOMED query builder |
| Refset | `refsetPlugin.js` | Reference set management |
| Diagram | `drawConceptDiagram.js` + `svgdiagrammingv2.js` | Visual relationship diagrams |
| Daily Build | `dailyBuildPlugin.js` | Daily build management |
| Popover | `popover.js` | Concept-preview tooltips |

### Entry Points

- **`index.html`** — main SPA; sets global `options` object, handles URL shortcuts, initializes i18n and license acceptance
- **`multi-extension-search.html`** — alternative multi-edition search view

### URL Parameters

The app supports deep-linking via query params:
- `edition` — SNOMED CT edition (e.g., `MAIN/SNOMEDCT-ES`)
- `perspective` — UI layout (`full`, `browsing`, etc.)
- `languages` — UI language code (`en`, `es`, `da`, `pt`, `fr`)
- `conceptId1` — opens a specific concept on load
- `acceptLicense` — auto-accepts the license modal
- `diagrammingMarkupEnabled` — activates the SVG diagram feature

### Internationalization

Language strings live in `i18n/Languages*.properties` files. The jQuery i18n plugin loads the appropriate file based on `options.uiLanguage`. When adding UI text, add keys to all language files under `i18n/`.

### Snowstorm API Integration

All API calls go to the Snowstorm REST API. The base URL is configured in `options.apiEndpoint` (set in `index.html` and overridden by the Docker `API_HOST` env var). `util.js` contains shared AJAX helpers used across plugins.
