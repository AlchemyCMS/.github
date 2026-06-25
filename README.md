# Reusable Release Workflows for AlchemyCMS Gems

This repository contains reusable GitHub Actions workflows for automating gem releases across all AlchemyCMS repositories.

## Overview

The release process is fully automated with three workflows:

1. **prepare-release.yml** - Creates a release PR with version bump and changelog
2. **release.yml** - Publishes the gem to RubyGems and creates a GitHub release
3. **post-release.yml** - Bumps to next development version, syncs changelog, and announces release

### Features

- Supports both `main` and `*-stable` branch releases
- Automatic changelog generation from GitHub release notes
- Post-release announcements to Slack, Mastodon, and Bluesky
- Opt-in RubyFlow announcement issues for minor and major releases
- Changelog sync to `main` branch after stable releases (for Dependabot visibility)

## Adding Workflows to Your Gem

Create three workflow files in your gem's `.github/workflows/` directory:

### 1. `.github/workflows/prepare-release.yml`
```yaml
name: Prepare Release

on:
  workflow_dispatch:
    inputs:
      bump:
        description: 'Version bump type. Choose "release" for finalizing a pre-release (8.0.0.dev → 8.0.0), or patch/minor/major to simply bump version.'
        required: true
        type: choice
        default: 'patch'
        options:
          - release
          - patch
          - minor
          - major

jobs:
  prepare:
    uses: AlchemyCMS/.github/.github/workflows/prepare-release.yml@main
    with:
      version_file_path: lib/alchemy/YOUR_GEM/version.rb  # UPDATE THIS
      bump: ${{ inputs.bump }}
    secrets:
      app_id: ${{ vars.ALCHEMY_BOT_APP_ID }}
      app_private_key: ${{ secrets.ALCHEMY_BOT_APP_PRIVATE_KEY }}
```

### 2. `.github/workflows/release.yml`
```yaml
name: Publish Release

on:
  workflow_dispatch:
  pull_request:
    types: [closed]
    branches:
      - main
      - '*-stable'

jobs:
  publish:
    if: github.event_name == 'workflow_dispatch' || (github.event.pull_request.merged == true && startsWith(github.event.pull_request.head.ref, 'release/v'))
    uses: AlchemyCMS/.github/.github/workflows/release.yml@main
    with:
      version_file_path: lib/alchemy/YOUR_GEM/version.rb  # UPDATE THIS
    secrets:
      app_id: ${{ vars.ALCHEMY_BOT_APP_ID }}
      app_private_key: ${{ secrets.ALCHEMY_BOT_APP_PRIVATE_KEY }}
```

### 3. `.github/workflows/post-release.yml`
```yaml
name: Post Release

on:
  workflow_run:
    workflows: ["Publish Release"]
    types:
      - completed

jobs:
  post-release:
    if: github.event.workflow_run.conclusion == 'success'
    uses: AlchemyCMS/.github/.github/workflows/post-release.yml@main
    with:
      version_file_path: lib/alchemy/YOUR_GEM/version.rb  # UPDATE THIS
      target_branch: ${{ github.event.workflow_run.head_branch }}
    secrets:
      app_id: ${{ vars.ALCHEMY_BOT_APP_ID }}
      app_private_key: ${{ secrets.ALCHEMY_BOT_APP_PRIVATE_KEY }}
      slack_webhook_url: ${{ secrets.SLACK_WEBHOOK_URL }}
      mastodon_access_token: ${{ secrets.MASTODON_ACCESS_TOKEN }}
      mastodon_instance: ${{ secrets.MASTODON_INSTANCE }}
      bluesky_identifier: ${{ secrets.BLUESKY_IDENTIFIER }}
      bluesky_password: ${{ secrets.BLUESKY_PASSWORD }}
```

### Configuration

**Update `version_file_path` in all three workflows** to match your gem's version file location. For example:
- `lib/alchemy/solidus/version.rb`
- `lib/alchemy/devise/version.rb`
- `lib/alchemy_cms/version.rb`

The workflows will automatically use the organization's `ALCHEMY_BOT_APP_ID` variable and `ALCHEMY_BOT_APP_PRIVATE_KEY` secret.

## How to Release

### Releasing from `main`

1. Go to **Actions** tab in your gem repository
2. Select **Prepare Release** workflow
3. Click **Run workflow** (ensure `main` branch is selected)
4. Choose the version bump type:
   - **release** - Finalize a pre-release (e.g., `8.0.0.dev` → `8.0.0`)
   - **patch** - Increment patch version (e.g., `1.2.3` → `1.2.4`)
   - **minor** - Increment minor version (e.g., `1.2.3` → `1.3.0`)
   - **major** - Increment major version (e.g., `1.2.3` → `2.0.0`)

This creates a PR targeting `main` with:
- Updated version file
- Generated changelog from GitHub release notes
- Release branch `release/vX.Y.Z`

Review and merge the PR. This automatically triggers:
1. **Publish Release** - Publishes gem to RubyGems and creates GitHub release
2. **Post Release** - Bumps version to next minor dev version (e.g., `1.2.0` → `1.3.0.dev`) and announces the release

### Releasing from Stable Branches

For repositories with `x.y-stable` branches (e.g., `8.0-stable`):

1. Go to **Actions** tab
2. Select **Prepare Release** workflow
3. Click **Run workflow** and **select the stable branch** (e.g., `8.0-stable`)
4. Choose bump type (typically `release` for stable branches)

This creates a PR targeting the stable branch. After merge:
1. **Publish Release** - Publishes gem and creates GitHub release
2. **Post Release** - Syncs changelog to `main` (via PR) and announces the release

The changelog sync ensures tools like Dependabot see all releases when reading from `main`. Version bumps are not automated for stable branches.

## Requirements

Your gem repository must have:

1. **Version file** with `VERSION = "x.y.z"` constant (e.g., `lib/alchemy/solidus/version.rb`)
2. **CHANGELOG.md** file in the repository root
3. **Trusted publishing** configured on RubyGems for the gem
4. **Main branch** named `main`

These are standard across all AlchemyCMS gems.

## What Each Workflow Does

**prepare-release.yml**
- Calculates next version based on bump type and current version
- Strips pre-release suffixes (.dev, .alpha, etc.) when using "release" bump
- Creates release branch `release/vX.Y.Z`
- Generates changelog from GitHub release notes API
- Updates version file and prepends to CHANGELOG.md
- Creates PR (with `skip-changelog` label) targeting the branch it was triggered from (`main` or `*-stable`)

**release.yml**
- Triggers when a `release/v*` PR is merged to `main` or `*-stable` branches
- Sets up Ruby and installs dependencies
- Publishes gem to RubyGems using trusted publishing
- Creates GitHub release with auto-generated notes

**post-release.yml**
- Triggers after successful release
- **Version bump (main only):** Creates a PR to bump to next minor dev version (e.g., `1.2.0` → `1.3.0.dev`)
- **Changelog sync (stable only):** Creates a PR to copy the changelog entry to `main` branch (for Dependabot visibility)
- **Announcements:** Posts release notifications to Slack, Mastodon, and Bluesky (if secrets are configured)
- **RubyFlow (opt-in):** For minor and major releases, opens a tracking issue with a ready-to-paste draft for manual posting to RubyFlow (when `announce_to_rubyflow: true`)

All PRs created by these workflows are labeled with `skip-changelog` to exclude them from future release notes.

## Release Announcements

The post-release workflow can announce releases to multiple platforms. Configure these org-level secrets:

| Secret | Description |
|--------|-------------|
| `SLACK_WEBHOOK_URL` | Slack incoming webhook URL (via Slack App) |
| `MASTODON_ACCESS_TOKEN` | Mastodon bot access token with `write:statuses` scope |
| `MASTODON_INSTANCE` | Mastodon instance URL (e.g., `https://ruby.social`) |
| `BLUESKY_IDENTIFIER` | Bluesky handle, DID, or custom domain |
| `BLUESKY_PASSWORD` | Bluesky app password (not main password) |

All announcement secrets are optional. Announcements are skipped for platforms without configured secrets.

### RubyFlow (opt-in)

[RubyFlow](https://rubyflow.com) has no posting API and requires a
GitHub-authenticated session, so the workflow does not post there automatically.
Instead, for **minor and major releases only** (versions ending in `.0`), it
opens a GitHub issue in the releasing repo containing a ready-to-paste title and
content plus the RubyFlow submit link. A maintainer pastes it into RubyFlow and
closes the issue.

This is **opt-in** and disabled by default. Enable it by passing
`announce_to_rubyflow: true` to the reusable `post-release.yml` workflow:

```yaml
jobs:
  post-release:
    if: github.event.workflow_run.conclusion == 'success'
    uses: AlchemyCMS/.github/.github/workflows/post-release.yml@main
    with:
      version_file_path: lib/alchemy_cms/version.rb
      target_branch: ${{ github.event.workflow_run.head_branch }}
      announce_to_rubyflow: true
    secrets:
      app_id: ${{ vars.ALCHEMY_BOT_APP_ID }}
      app_private_key: ${{ secrets.ALCHEMY_BOT_APP_PRIVATE_KEY }}
      # ...announcement secrets as above
```

Only `alchemy_cms` enables this; other gems leave it off so we don't flood
RubyFlow with every gem's releases. No extra secret is needed — the issue is
created with the existing bot app token.

## Advanced: Version Pinning

The examples above use `@main` to always use the latest workflow version. You can also pin to specific commits:

```yaml
# Pin to a specific commit (maximum control)
uses: AlchemyCMS/.github/.github/workflows/release.yml@abc123
```

Using `@main` ensures you automatically get updates and fixes, which is recommended for most AlchemyCMS gems.
