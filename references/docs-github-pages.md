# Crystal API Docs on plambert.github.io

Publishes `crystal docs` output for every `v*` tag to
`https://plambert.github.io/<repo>/`, with a version dropdown, a `latest/` copy,
and a root redirect. Verified end-to-end on `plambert/json-tight.cr` (2026-07-05).

## Prerequisites

* The repo is public (Pages on private repos needs a paid plan). Confirm with
  `gh repo view <owner>/<repo> --json visibility`; let Paul flip visibility himself.
* Source lives under `src/` and every published tag compiles with current Crystal.
* Doc comments on the public API — see the main SKILL.md.

## Gotcha: shards that reopen stdlib namespaces

If the shard uses the shorthand `module JSON::Foo` (or any stdlib namespace), `crystal docs`
emits an **empty site**: the shorthand records no source location on the stdlib module, so the
generator attributes the whole namespace to the stdlib and drops everything inside. Restructure
to explicit nesting:

```crystal
module JSON
  module Foo
    # ...
  end
end
```

Verify before writing any workflow: run `crystal docs` locally and check that
`docs/index.json` lists the project's types.

## The workflow

Write `.github/workflows/docs.yml` from the template below, substituting `REPO` (e.g.
`json-tight.cr`) everywhere. **Look up current action majors first** — do not trust the
versions in this template:

```bash
gh api repos/actions/checkout/releases/latest -q .tag_name   # and the other actions
```

```yaml
name: docs

on:
  push:
    tags: ['v*']
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: pages
  cancel-in-progress: true

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
        with:
          fetch-depth: 0

      - uses: crystal-lang/install-crystal@v1

      - name: Build docs for every tag
        run: |
          mkdir -p site
          built=()
          for tag in $(git tag --list 'v*' --sort=-v:refname); do
            version="${tag#v}"
            git -c advice.detachedHead=false checkout --quiet "$tag"
            if crystal docs \
                --output="site/$version" \
                --project-version="$version" \
                --source-refname="$tag" \
                --json-config-url=/REPO/versions.json \
                --canonical-base-url=https://plambert.github.io/REPO/latest/; then
              built+=("$version")
            else
              echo "::warning::docs failed for $tag; omitting it from the site"
            fi
          done

          if [ "${#built[@]}" -eq 0 ]; then
            echo "::error::no tag produced docs"
            exit 1
          fi

          printf '%s\n' "${built[@]}" |
            jq --raw-input . |
            jq --slurp '{versions: map({name: ., url: ("/REPO/" + . + "/")})}' > site/versions.json

          cp -R "site/${built[0]}" site/latest
          printf '<!DOCTYPE html><html><head><meta http-equiv="refresh" content="0; url=latest/"><title>REPO</title></head><body><a href="latest/">REPO documentation</a></body></html>\n' > site/index.html

      - uses: actions/upload-pages-artifact@v5
        with:
          path: site

  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - id: deployment
        uses: actions/deploy-pages@v5
```

Design notes, in case a variation is requested:

* Rebuilding every tag on each push is deliberate: deterministic site, no `gh-pages`
  branch, no stale directories, old docs regenerated with the current template. Builds
  take ~1s/tag for a typical shard.
* Every version's pages fetch the same `/REPO/versions.json` at view time, so old
  versions' dropdowns learn about new releases without republishing.
* `--source-refname` makes source links point at the right tag;
  `--canonical-base-url` points crawlers at `latest/`, matching crystal-lang.org/api.
* Tag-triggered runs use the workflow file **as of the tagged commit** — the workflow
  must be merged before (or into) the tag it should run for.

Dry-run the build script locally before pushing: clone the repo into the scratchpad,
extract the `run:` block, and execute it; then check `site/versions.json` and that
`site/<version>/` contains the type pages.

## Pages and environment setup

Enable Pages with the workflow source (skip if `gh api repos/<owner>/<repo>/pages`
already shows `"build_type": "workflow"`):

```bash
gh api -X POST repos/<owner>/<repo>/pages -F 'build_type=workflow'
```

The auto-created `github-pages` environment **rejects tag-triggered deployments** by
default ("not allowed to deploy to github-pages due to environment protection rules").
Allow `v*` tags and `main` — note `-F` for the booleans, `-f` sends strings and 422s:

```bash
gh api -X PUT repos/<owner>/<repo>/environments/github-pages \
  -F 'deployment_branch_policy[protected_branches]=false' \
  -F 'deployment_branch_policy[custom_branch_policies]=true'
gh api -X POST repos/<owner>/<repo>/environments/github-pages/deployment-branch-policies \
  -f name='v*' -f type=tag
gh api -X POST repos/<owner>/<repo>/environments/github-pages/deployment-branch-policies \
  -f name=main -f type=branch
```

## Publish and verify

1. Push a `v*` tag (or rerun via `workflow_dispatch` / `gh run rerun --failed`).
2. Watch: `gh run watch <run-id> --exit-status`.
3. Probe the site — all three must succeed:

   ```bash
   curl -s -o /dev/null -w '%{http_code}\n' https://plambert.github.io/REPO/
   curl -s -o /dev/null -w '%{http_code}\n' https://plambert.github.io/REPO/latest/
   curl -s https://plambert.github.io/REPO/versions.json
   ```

## README badge

Add under the H1, then `rumdl check README.md`:

```markdown
[![docs](https://img.shields.io/badge/docs-latest-blue)](https://plambert.github.io/REPO/latest/)
```
