# Actions

This repository contains a collection of GitHub Actions that can be used in your workflows.

# Usage
To use an action in your workflow, you must include it in your workflow file.

## Examples
### Changelog (deprecated)
> Deprecated: use `release.yml` (see below).
```yaml
# Path: .github/workflows/changelog.yml
name: Changelog
on:
  workflow_dispatch:
    inputs:
      next_version:
        description: 'Next version tag to use instead of UNRELEASED'
        type: string
        required: false

      config_dir:
        description: 'git-chglog configuration directory'
        type: string
        default: '.chglog'
        required: false

      filename:
        description: 'Filename to write the changelog to'
        type: string
        default: 'CHANGELOG.md'
        required: false

      path:
        description: 'Optional path to follow for directory'
        default: ''
        type: string
        required: false

      tag_query:
        description: 'Optional tag query to use for changelog generation'
        default: ''
        type: string
        required: false

      commit_message:
        type: string
        description: "Commit message"
        required: false

permissions:
    contents: write

jobs:
    changelog:
        uses: maicol07/actions/.github/workflows/changelog.yml@v1
        with:
            next_version: ${{ github.event.inputs.next_version }}
            config_dir: ${{ github.event.inputs.config_dir }}
            filename: ${{ github.event.inputs.filename }}
            path: ${{ github.event.inputs.path }}
            tag_query: ${{ github.event.inputs.tag_query }}
            commit_message: ${{ github.event.inputs.commit_message }}
```

### Release & Publish (deprecated)
> Deprecated: use `release.yml` + `npm-publish.yml` (see below).

> Note: You will need a `NPM_TOKEN` secret in your repository to publish the package. You can create one from the [NPM website](https://www.npmjs.com/settings/your-username/tokens). You can also use the [NPM CLI](https://docs.npmjs.com/creating-and-viewing-authentication-tokens) to create one.
```yaml
# Path: .github/workflows/release.yml
name: Release & Publish
on:
  push:
    branches:
      - main

permissions:
  contents: write
  pull-requests: write

jobs:
  release_publish:
    uses: maicol07/actions/.github/workflows/release_publish_npm.yml@v1
    with:
      release_channel: ${{ github.event.inputs.release_channel }}
    secrets: inherit # Or pass just what secret you want. This is required to pass the NPM_TOKEN needed to publish the package
```

### CI pipeline (security, commit lint, PHP, release PR, Docker)
GitHub port of the Trust-IT `websites/utils/ci-templates` flow, one reusable workflow per job group:

| Workflow | Jobs | Inputs |
|---|---|---|
| `security.yml` | `scan`: secret detection (Betterleaks, private repos only: public ones have GitHub Secret Protection) and config scan (Trivy), one job. Vulnerable dependencies: Dependabot alerts | - |
| `commit-lint.yml` | `commit_lint`: Conventional Commits, gitmoji after the colon | - |
| `php.yml` | PHP + cached `composer install` + a command | `php_version`, `extensions`, `run` |
| `js.yml` | Bun (or Node.js + pnpm/npm without a `bun.lock`) + install + a command | `run`, `node_version` |
| `gradle.yml` | JDK + Gradle (cached) + a command | `java_version`, `run` |
| `release.yml` | Release PR on `release/next` (git-cliff), GitHub Release once merged | `changelog`, `tag_pattern`, `next_version`, `bump`; secret `token`; output `tag` |
| `docker.yml` | Build & push of the `prod` target, Trivy image scan (report only) | `registry`, `image`, `tag`, `target`, `build_args`; secrets `registry_username`, `registry_password` |
| `npm-publish.yml` | `bun publish` of a release tag | `tag`, `dist_tag`; secret `npm_token` |
| `portainer.yml` | New tag in one or more Portainer Swarm stack files, redeploy, smoke test, rollback on failure | `stacks` (JSON), `image`, `tag`; secrets `portainer_url`, `portainer_api_key` |

The `setup-js` composite action (`maicol07/actions/setup-js@v1`) is the JS setup of `js.yml` and `npm-publish.yml`.
Reference everything at `@v1` (a moving major tag): Dependabot's `github-actions` ecosystem then proposes `v2`.

To override a job, replace its `uses:` with your own steps under the same job name, or leave it out.
`release.yml` uses the project's `cliff.toml`, or this repo's one. A `fork-point` tag limits the changelog
and the commit lint to the commits after it (e.g. a fork of an upstream project). The changelog also
leaves out the upstream commits merged later: the ones reachable from tags not matching `tag_pattern`.
Tags and PRs created with `GITHUB_TOKEN` trigger no workflow: chain the image build on the `tag` output.
The checks (`security`, `commit-lint`, `php`, `js`, `gradle`) skip the release PR and its merge on
main (a merge commit from `release/next` or a squashed `chore(release):` commit): main plus the
changelog, already checked. A project's own check jobs can copy their `if:`.

```yaml
# Path: .github/workflows/ci.yml
on:
  push:
    branches: [main]
  pull_request:

jobs:
  security:
    uses: maicol07/actions/.github/workflows/security.yml@v1
  commit_lint:
    uses: maicol07/actions/.github/workflows/commit-lint.yml@v1
  tests:
    uses: maicol07/actions/.github/workflows/php.yml@v1
    with:
      php_version: '8.5'
      run: vendor/bin/pest --ci
  release:
    needs: [security, tests]
    # The checks skip the release PR merge: skipped needs must not skip the release.
    if: github.event_name == 'push' && !failure() && !cancelled()
    uses: maicol07/actions/.github/workflows/release.yml@v1
    permissions:
      contents: write
      pull-requests: write
  docker:
    needs: release
    if: needs.release.outputs.tag
    uses: maicol07/actions/.github/workflows/docker.yml@v1
    permissions:
      packages: write
    with:
      registry: ghcr.io
      image: owner/app
      tag: ${{ needs.release.outputs.tag }}
    secrets:
      registry_username: ${{ github.actor }}
      registry_password: ${{ secrets.GITHUB_TOKEN }}
```

### Secrets, variables and settings per repo
The workflows take secrets and values as inputs: the names below are the convention of the calling
`ci.yml`s (map them in `secrets:` / `with:`). `security`, `commit-lint`, `php`, `js` and `gradle`
need none.

| Workflow | Kind | Name | Needed for |
|---|---|---|---|
| `release.yml` | setting | *Settings → Actions → General → Allow GitHub Actions to create and approve pull requests* (in an organization, enable it at the organization level first) | opening the release PR |
| `release.yml` | secret (optional) | `RELEASE_TOKEN` → `token` | a PAT or app token, if the release PR must run the checks (e.g. required status checks): PRs and tags created with `GITHUB_TOKEN` start no workflow |
| `release.yml` | variable (optional) | `PRERELEASE` → `prerelease: ${{ vars.PRERELEASE == 'true' }}` | marking the next merged release as a pre-release |
| `docker.yml` | secrets | `REGISTRY_USERNAME`, `REGISTRY_PASSWORD` → `registry_username`, `registry_password` | pushing to a private registry (e.g. OCIR). For ghcr.io pass `github.actor` and `secrets.GITHUB_TOKEN`, with `permissions: packages: write` |
| `npm-publish.yml` | secret | `NPM_TOKEN` → `npm_token` | `bun publish` (an npm granular token with publish rights on the package) |
| `portainer.yml` | secrets | `PORTAINER_URL`, `PORTAINER_API_KEY` → `portainer_url`, `portainer_api_key` | the Portainer API (an access token of a user that only reaches the project's stacks) |
| `portainer.yml` | variable | `PORTAINER_STACKS` → `stacks` | the stacks to update, a JSON array of `{"stack_id", "endpoint_id", "url"}` (IDs from each stack's Portainer URL: `#!/<endpoint>/docker/stacks/<name>?id=<stack>`; `url`, optional, is the smoke test target). Guard the job with `if: vars.PORTAINER_STACKS` so it is skipped until set |

Portainer must also have the image registry among its *Registries*, with credentials, or it cannot
pull the new tag.

Example, with the `gh` CLI (values are placeholders; a secret without `--body` is read from stdin,
so it stays out of the shell history):
```sh
repo=owner/app
# Release PR
gh api -X PUT repos/$repo/actions/permissions/workflow -f default_workflow_permissions=read -F can_approve_pull_request_reviews=true
# Docker (OCIR)
gh secret set REGISTRY_USERNAME -R $repo --body 'namespace/user@example.com'
gh secret set REGISTRY_PASSWORD -R $repo          # paste the auth token
# Portainer
gh secret set PORTAINER_URL -R $repo --body 'https://portainer.example.com'
gh secret set PORTAINER_API_KEY -R $repo          # paste the access token
gh variable set PORTAINER_STACKS -R $repo --body '[
  {"stack_id": 12, "endpoint_id": 1, "url": "https://app.example.com"},
  {"stack_id": 7, "endpoint_id": 3, "url": "https://app.other-host.example.com"}
]'
# npm
gh secret set NPM_TOKEN -R $repo                  # paste the npm token
```

### Dependabot
Dependabot has no shared config: each repo needs its own `.github/dependabot.yml`, e.g.:
```yaml
version: 2
updates:
  - package-ecosystem: github-actions
    directory: /
    schedule:
      interval: weekly
    groups:
      github-actions:
        patterns: ['*']
  # + composer, bun, gradle, docker, docker-compose... as the repo needs
```
