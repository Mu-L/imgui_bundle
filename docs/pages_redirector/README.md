# pthom.github.io/imgui_bundle redirector

Static redirect site for the old GitHub Pages URL
`https://pthom.github.io/imgui_bundle/*`, which now forwards to
`https://imgui-bundle.pages.dev/*`.

Deployed via GitHub Pages: **Settings → Pages → Branch: `main` / `/docs/pages_redirector`**.

- `index.html`: redirects the root.
- `404.html`: catches any unknown path (GitHub Pages serves `/404.html`
  for anything not found), strips the `/imgui_bundle` base prefix, and
  redirects to the same path on `imgui-bundle.pages.dev`.
