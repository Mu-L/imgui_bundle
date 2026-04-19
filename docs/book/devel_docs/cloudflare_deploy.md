# Cloudflare Pages deploy

The project publishes three static subparts to a single Cloudflare Pages project
named `imgui-bundle`, reachable at <https://imgui-bundle.pages.dev/>:

| Path                          | Contents                                                          |
|-------------------------------|-------------------------------------------------------------------|
| `/playground_python/`         | Pyodide playground (Python examples in the browser)               |
| `/min_pyodide_app/`           | Minimal Pyodide template (single-file demo)                       |
| `/explorer/`                  | Dear ImGui Bundle Explorer (Emscripten build with OpenCV)         |
| `/local_wheels/`              | Shared `imgui_bundle-*.whl`, loaded by the pyodide subparts       |

A tracked landing page at the root (`pyodide_projects/cf_landing.html`) links to
each subpart. A `_headers` file (`pyodide_projects/cf_headers`) scopes
`Cross-Origin-Opener-Policy` / `Cross-Origin-Embedder-Policy` to `/explorer/*`
only; site-wide COI would break the playground (which pulls Pyodide and
CodeMirror from CDNs that don't set `Cross-Origin-Resource-Policy`).


## How the deploy works

Cloudflare Pages replaces the whole site on every upload, so we maintain a
local composed staging directory (`pyodide_projects/_cf_staging/`, gitignored)
that holds the union of all subparts. `just cf_stage_all` refreshes every
subdir; `just cf_deploy` uploads the whole staging tree via `wrangler`.

The trade-off: on every deploy the other subparts must already be present in
staging. If staging gets wiped (`just cf_clean`), re-run `cf_stage_all` before
`cf_deploy`.


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
# Full refresh of staging, then deploy
just cf_stage_all
just cf_deploy

# Test the composed site locally before deploying
just cf_serve_local        # http://localhost:8765/
# → COOP/COEP headers are applied only to /explorer/* (mirrors prod)

# Inspect the staging tree (gitignored)
ls pyodide_projects/_cf_staging/

# Nuke staging (forces a full re-stage next time)
just cf_clean
```

The ibex subpart requires `just ibex_build` first (produces
`build_ibex_ems/bin/`); `cf_stage_all` then copies those binaries into
`_cf_staging/explorer/`.


## CI workflow

`.github/workflows/cf_pages_deploy.yml` mirrors the local flow and is
triggered manually from the Actions tab (`workflow_dispatch`). It runs:

1. `just pyodide_setup_local_build` + `just pyodide_build` (builds the wheel)
2. `just ibex_build` (builds Emscripten binaries; slowest step, typically
   30–60 min because `IMMVISION_FETCH_OPENCV=ON` rebuilds OpenCV for wasm)
3. `just cf_stage_all`
4. `cloudflare/wrangler-action@v3` to upload

Required GitHub Actions secrets: `CLOUDFLARE_API_TOKEN`,
`CLOUDFLARE_ACCOUNT_ID` (same values used locally).


## Related

- Local traineq deploy (kept as fallback): `just pyodide_deploy_imgui_bundle_online`, `just ibex_deploy`
