# AGENTS.md

`dcyfr-community-plugins` is the community plugin registry for the DCYFR AI framework, run by DCYFR Labs. Plugins arrive by pull request and are **scanned and auto-merged by CI with no human review**, so every plugin here is unaudited input. There is no `package.json`, no build and no test suite; the whole mechanism is one GitHub Actions workflow plus the docs.

## Layout

```
marketplace.json                          registry index: version, type "community", disclaimer, "plugins": [] (currently empty)
CONTRIBUTING.md                           submission rules, manifest and trust-score examples, auto-merge policy
VERIFIED_PUBLISHERS.md                    criteria for a verified badge; current list: @dcyfr
SECURITY.md, README.md, LICENSE
.github/workflows/community-plugin-scan.yml   the only workflow: validate, vulnerability scan, auto-merge
.devin/wiki.json                          wiki config, not part of the registry
```

**There is no `plugins/` directory yet.** The first submission creates it. `README.md` says "no plugins yet", and `marketplace.json` has no entries.

## How a plugin gets in

1. A contributor opens a pull request to `main` adding `plugins/<name>/` with `.claude-plugin/manifest.json`, `trust-score.json`, `sbom.json` and `LICENSE` (the README also asks for a `README.md`; CI does not check it).
2. `community-plugin-scan.yml` runs on pull requests to `main` that touch `plugins/**`.
3. If every check passes and the PR is not a draft, the `auto-merge` job squash-merges it as `feat(community): add <plugin_id> [auto-merged]` and comments that the plugin is unaudited.

## What CI checks (`validate-plugin` job)

- The plugin directory is the first path under `plugins/` in the PR's diff against the base branch. Only that one directory is validated, even if the PR touches several.
- `.claude-plugin/manifest.json` exists and has non-empty `plugin_id`, `name`, `version`, `description`, `license`, `capabilities`, `permissions`, `entrypoint`, `minFrameworkVersion`.
- `trust-score.json` exists and `overall` is at least **70**.
- `sbom.json` and `LICENSE` exist.
- `license` is one of `MIT`, `Apache-2.0`, `BSD-2-Clause`, `BSD-3-Clause`, `ISC`.
- If the `plugin_id` prefix is neither the PR author's login nor `dcyfr`, the step sets `NEEDS_REVIEW=true` in `GITHUB_ENV`.

The `vulnerability-scan` job reads the SBOM and warns on a short hard-coded name blocklist (`lodash`, `minimist`, `node-fetch`); the code says it is a placeholder and it never fails the job. The score is read from the contributor's own `trust-score.json`; nothing computes it, although `README.md` says CI does.

## Commands

There are no local scripts. Read-only checks you can run (Node 20 or newer for any snippet):

| Check | Command | Result on `main` |
|---|---|---|
| Workflow lint | `actionlint .github/workflows/community-plugin-scan.yml` | One finding: the PR title is interpolated in an inline script (`github.event.pull_request.title`, line 183). Report it, do not hide it. |
| `marketplace.json` parses | `node -e "JSON.parse(require('fs').readFileSync('marketplace.json'))"` | Passes. |

## Gotchas and known contradictions

- **`main` has no required status checks.** The only rules on it (a ruleset named `main protection`) block branch deletion and force-pushes, as of 2026-11-17. The scan is not a merge gate in the GitHub sense; its own `auto-merge` job is what merges.
- **Policy text and CI differ.** `CONTRIBUTING.md` says PRs are held for review at trust scores 50 to 69, when network access or secret access is requested, and requires `plugin_id` to match the author's handle, and no critical CVEs. In the workflow, a score under 70 fails the check, network and secret requests are not inspected, a `plugin_id` mismatch only sets `NEEDS_REVIEW`, and the CVE step is the placeholder above. Do not describe the documented policy as enforced.
- **`NEEDS_REVIEW` is set in the `validate-plugin` job and read by the `auto-merge` job.** Environment variables written to `GITHUB_ENV` stay inside their own job, so whether the manual-review path ever triggers is unverified. Test it before relying on it in docs.
- **`README.md` says `network` requires a trust score of 85 or more and that scores under 70 need manual review;** the workflow checks neither beyond failing under 70.
- **Three manifest shapes appear in the docs.** `CONTRIBUTING.md` (nested `permissions`, `entrypoint: index.js`), `README.md` (`permissions` as a list, a `dcyfr` compat block, no `plugin_id` or `entrypoint`) and the CI (the field list above). The CI field list is what blocks a merge; fix the docs toward it.
- `.github/workflows/community-plugin-scan.yml` grants `contents: write` and `pull-requests: write` and merges with the repository token. Treat edits to it as security-sensitive.
- Changing a submission requirement means changing `CONTRIBUTING.md`, `README.md` and the workflow together.

## Never

- Merge, or change the workflow so it merges, a plugin that has not passed every check.
- Execute or install a submitted plugin to "test" it. Review manifests and SBOMs as data.
- Add `verified` status or a publisher to `VERIFIED_PUBLISHERS.md` without the criteria in that file being met.
- Interpolate PR-controlled values (title, branch name, body) into `run:` scripts; pass them through `env`.
- Commit secrets or tokens.
- Push to `main`.
