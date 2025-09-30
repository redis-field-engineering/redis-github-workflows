# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This repository contains **reusable GitHub Actions workflows** for Redis projects. These workflows are designed to be called from other repositories to standardize CI/CD processes across Redis Field Engineering projects.

## Architecture

### Workflow Categories

The repository provides three sets of parallel workflows for different build systems:

1. **Gradle workflows** (default, no suffix): `build.yml`, `release.yml`, `early-access.yml`, `publish.yml`
2. **Maven workflows** (`-maven` suffix): `build-maven.yml`, `release-maven.yml`, `early-access-maven.yml`, `publish-maven.yml`
3. **Variant workflows** (numbered suffixes): `build2.yml`, `release2.yml`, etc. - These are variations with different default configurations

### Key Workflows

- **`release.yml`**: Full release workflow with JReleaser integration. Handles versioning, building, testing, GPG signing, and publishing to Maven Central/Docker Hub. Updates VERSION file and README.adoc with release version, then commits changes.

- **`early-access.yml`**: Snapshot/pre-release builds. Reads version from VERSION file (doesn't modify it). Includes code coverage upload to Codecov.

- **`build.yml`**: Basic build and test workflow. Runs Gradle/Maven builds and uploads test reports on failure.

- **`publish.yml`**: Simplified publish workflow without JReleaser. Runs Gradle publish tasks directly.

- **`benchmark.yml`**: Runs JMH benchmarks via Gradle and uploads results.

- **`docker.yml`**: Multi-platform Docker image building and publishing (linux/amd64, linux/arm64).

- **`docs.yml`**: Builds and deploys documentation using Antora to GitHub Pages.

- **`copy-to-dist.yml`**: Copies files from this repository to target repositories (useful for distributing workflow updates).

### Common Patterns

All workflows share these characteristics:

- Use `workflow_call` trigger (reusable workflows)
- Cancel previous runs to avoid conflicts (`styfle/cancel-workflow-action`)
- Use Java 21 by default (Zulu distribution)
- Cache Gradle wrapper/Maven dependencies for performance
- Upload test reports as artifacts on failure
- Require `fetch-depth: 0` for JReleaser workflows (needed for proper git history)

### JReleaser Integration

Release and early-access workflows use JReleaser for:
- Assembling release artifacts
- Creating GitHub releases
- Publishing to Maven Central (via JRELEASER_MAVENCENTRAL_USERNAME/PASSWORD)
- Publishing Docker images (via JRELEASER_DOCKER_* credentials)
- Optional Slack notifications (via JRELEASER_SLACK_WEBHOOK)

JReleaser expects a `jreleaser.yml` configuration file in the consuming repository.

### Version Management

- **release.yml**: Takes version as input, writes to VERSION file and updates `:project-version:` in README.adoc, commits and pushes changes
- **early-access.yml**: Reads existing VERSION file, doesn't modify it
- Maven workflows use `mvnw versions:set` to update pom.xml versions

### Build Profiles

Gradle workflows use specific profiles:
- `-Pprofile=release` for release builds
- `-Prelease=true` for early-access builds
- `-PreproducibleBuild=true` for deterministic builds
- GRGIT_USER/GRGIT_PASS environment variables for git operations

## Common Commands

Since this is a workflow repository (not a buildable project), there are no build/test commands. Work involves:

1. **Editing workflow files**: Modify `.github/workflows/*.yml` files
2. **Testing changes**: Push changes and test by calling workflows from a test repository
3. **Documentation**: Update README.adoc when adding/modifying workflows

## Required Secrets

Consuming repositories must configure these secrets:

**Required for all release workflows:**
- `github-token`: GitHub API access
- `gpg-passphrase`, `gpg-public-key`, `gpg-secret-key`: Artifact signing

**Optional (depending on publishing targets):**
- `github-user`: GitHub username
- `sonatype-username`, `sonatype-password`: Maven Central
- `docker-username`, `docker-password`: Docker Hub
- `docker-github-username`, `docker-github-password`: GitHub Container Registry
- `slack-webhook`: Slack notifications
- `codecov-token`: Code coverage (early-access workflow)

## Workflow Usage Example

To use these workflows from another repository:

```yaml
name: Release
on:
  workflow_dispatch:
    inputs:
      version:
        required: true
        type: string

jobs:
  release:
    uses: redis-field-engineering/redis-github-workflows/.github/workflows/release.yml@main
    with:
      version: ${{ github.event.inputs.version }}
      java-version: '21'
      tasks: 'build aggregateTestReports publish'
    secrets:
      github-token: ${{ secrets.GITHUB_TOKEN }}
      gpg-passphrase: ${{ secrets.GPG_PASSPHRASE }}
      gpg-public-key: ${{ secrets.GPG_PUBLIC_KEY }}
      gpg-secret-key: ${{ secrets.GPG_SECRET_KEY }}
      sonatype-username: ${{ secrets.SONATYPE_USERNAME }}
      sonatype-password: ${{ secrets.SONATYPE_PASSWORD }}
```

## Important Notes

- All JReleaser workflows require `fetch-depth: 0` on checkout (full git history)
- Test reports are uploaded only on build failure (saves storage)
- The `2` suffix variants (release2.yml, etc.) exist for projects with different default configurations
- Maven workflows use `./mvnw` wrapper; Gradle workflows use `./gradlew` wrapper
- Multi-workflow approach ensures build system flexibility without mixing concerns