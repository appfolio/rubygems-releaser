# rubygems-releaser

Reusable GitHub Actions workflows for publishing public gems to [RubyGems.org][rubygems-org] via [trusted publishing][trusted-publishing].

This workflow uses [Release Please][release-please] to automatically bump versions, generate changelogs, and publish new gem versions based on [Conventional Commits][conventional-commits].

Three reusable workflows are provided:

| Workflow | Trigger | Description |
|----------|---------|-------------|
| `release.yml` | `push` to default branch | Runs Release Please to create/update release PRs |
| `publish.yml` | `release: published` | Builds and publishes the gem to RubyGems.org |
| `lint-commits.yml` | `pull_request` | Enforces Conventional Commits format on PRs |

## Installation

### Configure a Trusted Publisher on RubyGems.org

Before the workflow can publish your gem, you must register a trusted publisher on RubyGems.org:

1. Log in to [rubygems.org][rubygems-org] and navigate to your gem's page.
2. Click **"Trusted publishers"** in the sidebar.
3. Fill in the form:
   - **Owner**: your GitHub organization or user (e.g., `appfolio`)
   - **Repository**: your gem's GitHub repository name (e.g., `my_gem`)
   - **Workflow filename**: `publish.yml`
   - **Environment**: *(leave blank)*

> [!NOTE]
> Only existing gem owners on RubyGems.org can register trusted publishers.

### Make sure your gem has the required rake tasks

Your Rakefile must have the `build` task defined. To check whether this task is defined, run:

```shell
bundle exec rake -n build
```

If you get an error like "Don't know how to build task 'build'", add the following line to your `Rakefile`:

```ruby
require 'bundler/gem_tasks'
```

### Add the release workflow

Create a `.github/workflows/release.yml` file in your gem repository with the following contents:

```yaml
name: Release Gem

on:
  push:
    branches:
      - main
      - master

permissions:
  contents: write
  pull-requests: write

jobs:
  release:
    uses: appfolio/rubygems-releaser/.github/workflows/release.yml@v1
```

> [!TIP]
> If your default branch is something other than `main` or `master`, configure `on.push.branches` with your default branch name.

### Add the publish workflow

Create a `.github/workflows/publish.yml` file in your gem repository with the following contents:

```yaml
name: Publish Gem

on:
  release:
    types: [published]

permissions:
  contents: write
  id-token: write

jobs:
  publish:
    uses: appfolio/rubygems-releaser/.github/workflows/publish.yml@v1
```

> [!IMPORTANT]
> The `id-token: write` permission is required for OIDC trusted publishing. Without it, the workflow cannot
> authenticate with RubyGems.org.

#### Monorepo / Subdirectory Support

If your gem lives in a subdirectory, set `working_directory` to match the package path in your `release-please-config.json`:

| Input | Description | Default |
|-------|-------------|---------|
| `working_directory` | Directory containing the gem (must match the package path in config) | `.` |

```yaml
jobs:
  release:
    uses: appfolio/rubygems-releaser/.github/workflows/release.yml@v1
    with:
      working_directory: path/to/gem

  publish:
    uses: appfolio/rubygems-releaser/.github/workflows/publish.yml@v1
    with:
      working_directory: path/to/gem
```

With config files at the repository root:

```json
{
  "packages": {
    "path/to/gem": {
      "changelog-path": "CHANGELOG.md",
      "version-file": "lib/my_gem/version.rb",
      "release-type": "ruby"
    }
  }
}
```

```json
{
  "path/to/gem": "1.0.0"
}
```

> [!IMPORTANT]
> The `working_directory` value must match the package path key in your `release-please-config.json`. The workflow uses this to find the correct release-please output.

### Create Release Please configuration file

In the root of your repository, create a `release-please-config.json` with the following contents:

```json
{
  "bootstrap-sha": "<sha>",
  "packages": {
    ".": {
      "changelog-path": "CHANGELOG.md",
      "version-file": "<path to version.rb>",
      "release-type": "ruby",
      "bump-minor-pre-major": false,
      "bump-patch-for-minor-pre-major": false,
      "draft": false,
      "prerelease": false
    }
  },
  "changelog-sections": [
    {
      "type": "feat",
      "section": "Features"
    },
    {
      "type": "fix",
      "section": "Bug Fixes"
    },
    {
      "type": "perf",
      "section": "Performance Improvements"
    },
    {
      "type": "revert",
      "section": "Reverts"
    },
    {
      "type": "docs",
      "section": "Documentation"
    },
    {
      "type": "style",
      "section": "Styles"
    },
    {
      "type": "chore",
      "section": "Miscellaneous Chores"
    },
    {
      "type": "refactor",
      "section": "Code Refactoring"
    },
    {
      "type": "test",
      "section": "Tests"
    },
    {
      "type": "build",
      "section": "Build System"
    },
    {
      "type": "ci",
      "section": "Continuous Integration"
    }
  ],
  "$schema": "https://raw.githubusercontent.com/googleapis/release-please/main/schemas/config.json"
}
```

Replace `<sha>` with the SHA of the last commit of the last gem version you have released. All commits after this
**must** be in [Conventional Commits][conventional-commits] format. If you are setting up a new gem and you have not
released a version of the gem yet, you can omit `bootstrap-sha` entirely.

Replace `<path to version.rb>` with the path to your gem's `version.rb` file. Usually, this will be something like
`lib/<gem name>/version.rb`.

For more information on the options in this file, see [the Release Please documentation][manifest-releaser-docs].

### Create Release Please version manifest

In the root of your repository, create a `.release-please-manifest.json` file with the following contents:

```json
{
  ".": "<version>"
}
```

Replace `<version>` with the same version string as the one in your gem's `version.rb` file.

### Add a commitlint configuration file

Create a `.commitlintrc.json` file in the root of your repository:

```json
{
  "extends": ["@commitlint/config-conventional"],
  "rules": {
    "body-max-line-length": [0]
  }
}
```

This configuration enforces [Conventional Commits][conventional-commits] and disables the body line-length rule to allow
long lines such as URLs.

### Add the lint-commits workflow (optional, but recommended)

Release Please requires commits to be in [Conventional Commits][conventional-commits] format, so rubygems-releaser
provides a workflow for enforcing this on pull requests.

Create a `.github/workflows/lint-commits.yml` file in your gem repository with the following contents:

```yaml
name: Lint Commit Messages

on:
  pull_request:

jobs:
  lint-commits:
    uses: appfolio/rubygems-releaser/.github/workflows/lint-commits.yml@v1
```

> [!TIP]
> Consider updating your repository settings to require the `lint-commits / lint-commits` status check to pass before
> merging.

## Usage

Release Please requires you to format all commits in [Conventional Commits][conventional-commits] format. Release Please
relies on this information to determine what changelog entries to generate, as well as which version to bump
(major, minor, or patch).

Once the release workflow is added to your repository, the workflow will run on every push to the default branch. This
workflow does two things:

1. Whenever a commit is pushed to the default branch, the `release.yml` workflow runs Release Please which
creates (or updates an existing) release PR with changelog updates and version bumps.
2. When a release PR is merged, Release Please creates a GitHub Release.
3. The GitHub Release triggers the `publish.yml` workflow which builds and publishes the gem to RubyGems.org.

## Publishing changes to rubygems-releaser

> [!NOTE]
> These instructions are for updating the rubygems-releaser repository itself, not for consumers of the workflows.

If you make changes to any of the reusable workflows, you will need to update the repository's tags so that
other repositories use the updated workflows.

To publish changes, check out the latest default branch and run the following commands:

```shell
git tag -f v1
git push --tags --force
```

This will update the `v1` tag to point to the latest commit. Gems pointing to the `v1` workflow will use the updated
workflow the next time it runs.

[conventional-commits]: https://www.conventionalcommits.org/en/v1.0.0/
[manifest-releaser-docs]: https://github.com/googleapis/release-please/blob/main/docs/manifest-releaser.md#configfile
[release-please]: https://github.com/googleapis/release-please/
[rubygems-org]: https://rubygems.org
[trusted-publishing]: https://guides.rubygems.org/trusted-publishing/
