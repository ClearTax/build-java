# Java Build GitHub Action

## Overview

This GitHub Action is a composite action designed to streamline the release process for Java/Maven projects. It automates the following key aspects:

1. Java and Maven environment setup
2. Maven release preparation and execution
3. Automated version bumping for development
4. Pull request creation for snapshot version updates
5. Artifact deployment and upload

The action performs a Maven release with the specified version and automatically creates a pull request to bump the development version to the next snapshot.

## Inputs

| Input Name | Description | Default Value | Required |
|------------|-------------|---------------|----------|
| `java_version` | The Java version to use | `17` | No |
| `java_distribution` | The Java distribution to use | `zulu` | No |
| `maven_version` | The Maven version to use | `3.8.3` | No |
| `maven_flag` | Maven command-line flags | `-T 4C -B --no-transfer-progress` | No |
| `token` | The GitHub token for authentication | N/A | **Yes** |
| `version` | The version to release (e.g., `1.2.3`) | N/A | **Yes** |
| `publish_artifacts` | Whether to deploy artifacts to Maven repository | `false` | No |
| `default_branch` | The default branch to merge the PR into | `main` | No |

## Outputs

| Output Name | Description |
|-------------|-------------|
| `version` | The version of the released artifact |
| `artifact-name` | The name of the uploaded artifact (format: `artifacts-{run-id}`) |

## Usage Examples

### Basic Release

```yaml
name: Release

on:
  workflow_dispatch:
    inputs:
      version:
        description: 'Release version'
        required: true
        type: string

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          fetch-depth: 0

      - uses: ClearTax/build-java@main
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          version: ${{ github.event.inputs.version }}
          default-branch: main
```

### Release with Artifact Deployment

```yaml
name: Production Release

on:
  push:
    tags:
      - 'v*'

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          fetch-depth: 0

      - name: Extract version from tag
        id: version
        run: echo "VERSION=${GITHUB_REF#refs/tags/v}" >> $GITHUB_OUTPUT

      - uses: ClearTax/build-java@main
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          version: ${{ steps.version.outputs.VERSION }}
          publish_artifacts: true
          default-branch: main
```

### Custom Java and Maven Versions

```yaml
- uses: ClearTax/build-java@main
  with:
    token: ${{ secrets.GITHUB_TOKEN }}
    version: '2.0.0'
    java_version: '21'
    java_distribution: 'temurin'
    maven_version: '3.9.5'
    maven_flag: '-T 1C -B'
    default-branch: develop
```
