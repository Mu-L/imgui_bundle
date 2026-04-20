# Cloudflare Pages deploy

The project publishes several static subparts to a single Cloudflare Pages project
named `imgui-bundle`, reachable at <https://imgui-bundle.pages.dev/>:

| Path                          | Contents                                                          |
|-------------------------------|-------------------------------------------------------------------|
| `/`                           | Jupyter-book documentation (this book); its `index.html` is the site landing page |
| `/playground_python/`         | Pyodide playground (Python examples in the browser)               |
| `/min_pyodide_app/`           | Minimal Pyodide template (single-file demo)                       |
| `/explorer/`                  | Dear ImGui Bundle Explorer (Emscripten build with OpenCV)         |
| `/local_wheels/`              | Shared `imgui_bundle-*.whl`, loaded by the pyodide subparts       |
| `/assets/book.pdf`            | PDF export of the documentation                                   |

A `_headers` file (`pyodide_projects/cf_headers`) scopes
`Cross-Origin-Opener-Policy` / `Cross-Origin-Embedder-Policy` to `/explorer/*`
only; site-wide COI would break the playground (which pulls Pyodide and
CodeMirror from CDNs that don't set `Cross-Origin-Resource-Policy`). The same
file also sets `Content-Encoding: gzip` on `/explorer/*.data` because those
emscripten asset bundles (`application/octet-stream`) aren't auto-compressed
by the CF edge; they are pre-gzipped by `cf_stage`.


## How the deploy works

Cloudflare Pages replaces the whole site on every upload, so we maintain a
local composed staging directory (`pyodide_projects/_cf_staging/`, gitignored)
that holds the union of all subparts. `just cf_stage` refreshes the staging
tree; `just cf_deploy` uploads it via `wrangler`.

The trade-off: on every deploy the other subparts must already be present
(built) on disk. `cf_stage` does not build them — it only copies:

- Pyodide wheel: `just pyodide_build` must have run (output in `pyodide_projects/projects/local_wheels/`).
- Ibex binaries: `just ibex_build` must have run (output in `build_ibex_ems/bin/`).
- Jupyter-book docs: `just doc_build_cf` must have run (output in `docs/book/_build/html/`).


## One-time setup

1. **Install wrangler** (needs Node.js 18+):
   ```bash
   npm install -g wrangler
   ```

2. **Create an API token** at <https://dash.cloudflare.com/profile/api-tokens>
   using the **Edit Cloudflare Workers** template (it covers Pages too).

3. **Export credentials** in your shell (or your shell rc):
   ```bash
   export CLOUDFLARE_API_TOKEN=...
   export CLOUDFLARE_ACCOUNT_ID=...
   ```

4. **Verify**: `wrangler whoami` should print the account.


## Local workflow

```bash
# Build every subpart (slow: ibex with OpenCV takes 30-60 min).
# `cf_stage_prepare` runs: pyodide_setup_local_build + pyodide_build
#                        + ibex_build + doc_build_cf
just cf_stage_prepare

# Stage everything into _cf_staging, then deploy
just cf_stage
just cf_deploy

# Test the composed site locally before deploying
just cf_serve_local        # http://localhost:8765/
# → COOP/COEP headers are applied only to /explorer/* (mirrors prod)
# → Content-Encoding: gzip is applied to /explorer/*.data

# Inspect the staging tree (gitignored)
ls pyodide_projects/_cf_staging/

# Nuke staging (next deploy re-uses on-disk build outputs)
just cf_clean
```


## CI workflow

`.github/workflows/cf_pages_deploy.yml` mirrors the local flow and is
triggered manually from the Actions tab (`workflow_dispatch`). It runs:

1. `just pyodide_setup_local_build` + `just pyodide_build` (builds the wheel)
2. `just ibex_build` (builds Emscripten binaries; slowest step, typically
   30–60 min because `IMMVISION_FETCH_OPENCV=ON` rebuilds OpenCV for wasm)
3. `just doc_build_cf` (jupyter-book HTML + PDF, needs Typst)
4. `just cf_stage` (composes the staging tree)
5. `npm install -g wrangler` + `just cf_deploy` (uploads)

Required GitHub Actions secrets: `CLOUDFLARE_API_TOKEN`,
`CLOUDFLARE_ACCOUNT_ID` (same values used locally).


## Related

- Local traineq deploy (kept as fallback): `just pyodide_deploy_imgui_bundle_online`, `just ibex_deploy`

## Links on imgui-bundle pages at claoudflare

* [Doc & Root](https://imgui-bundle.pages.dev)
* [Python Playground](https://imgui-bundle.pages.dev/playground_python)
* [Minimal Pyodide Template](https://imgui-bundle.pages.dev/min_pyodide_app/demo_heart.html)
* [Minimal Pyodide Template - Source](https://imgui-bundle.pages.dev/min_pyodide_app/demo_heart.source.txt)
* [ImGui Bundle Explorer](https://imgui-bundle.pages.dev/explorer)
