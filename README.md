# Java Build GitHub Action

## Overview

This GitHub Action is a composite action designed to standardize the build process for Java/Maven projects. It handles several key aspects of the build process:

1. Java environment setup
2. Maven configuration
3. Version management
4. Build process execution
5. Artifact handling and deployment

The action adapts its behavior based on the branch being built (master/main, hotfix, or development branches), applying different versioning strategies and build processes accordingly.

## Inputs and Outputs

### Key Inputs

| Input Name | Description | Default Value | Required |
|------------|-------------|---------------|----------|
| `java_version` | The Java version to use | 17 | Optional |
| `java_distribution` | The Java distribution to use | zulu | Optional |
| `maven_version` | The Maven version to use | 3.8.3 | Optional |
| `maven_flag` | Maven command-line flags | -T 4C -B --no-transfer-progress | Optional |
| `token` | The GitHub token | N/A | Required |
| `publish_artifacts` | Whether to publish artifacts | false | Optional |
| `ref` | The ref to build | current GitHub ref | Optional |
| `tag_name_prefix` | Tag name prefix for checkout | empty string | Optional |
| `aws_role_arn` | The Amazon Resource Name (ARN) of the role to assume for S3 cache | empty string | Optional |

### Output

- `version`: The version of the built artifact, accessible via `${{ steps.set-version.outputs.version }}`

## Workflow Steps in Detail

### 1. Environment Setup

The action begins by setting up the Java environment using the official `actions/setup-java@v4` action to install the specified Java version and distribution.

### 2. Maven Configuration

The action configures Maven with caching to speed up builds. It supports two caching mechanisms:

- **Local Cache**: Uses `actions/cache@v4` to cache the Maven repository locally within GitHub Actions.
- **S3 Cache**: When `aws_role_arn` is provided, uses `runs-on/cache@v4` with AWS S3 for more persistent and shared caching across workflows.

The action also uses `stCarolas/setup-maven@v5` to set up the specified Maven version.

### 3. Git Configuration

The action configures Git with a specific user identity ("clearci") to ensure consistent commit authoring and proper access to repositories.

### 4. Branch Detection

The action determines the type of branch being built:
- A master/main branch
- A hotfix branch
- Any other branch (development)

The result is stored as outputs (`hotfix` and `master`) for use in subsequent steps.

### 5. Version Management

The action implements sophisticated version management based on the branch type:

- **Master/Main Branch**: Increments the minor version and sets patch to 0 (e.g., 1.2.0 → 1.3.0)
- **Hotfix Branch**:
  - Checks if the current version has a qualifier
  - If not, increments the patch version and adds a SNAPSHOT suffix
  - Commits this change to the repository
  - Uses the version without the SNAPSHOT suffix for the build
- **Other Branches**: Uses a development version based on the first 8 characters of the commit SHA (e.g., dev-a1b2c3d4)

### 6. Build Process

The build process varies based on the branch type:

#### For Master/Hotfix Branches (Production Release):

This performs a full Maven release process with:
- Skipping tests
- Creating a Git tag with the format "v{version}"
- Adding "[ci skip]" prefix to commit messages
- Using the version determined in the previous step

#### For Other Branches (Development Build):

This performs a standard Maven package operation, skipping tests.

### 7. Artifact Deployment (Optional)

For production builds (master/hotfix), artifacts can optionally be deployed to a Maven repository when `publish_artifacts` is set to 'true'. This step:
- Checks out the tag created during the release process
- Deploys the artifacts using Maven's deploy goal with settings from `.mvn/settings.xml`

### 8. Artifact Upload

Finally, all builds upload artifacts to GitHub Actions, making the built JAR files available for download from the GitHub Actions interface, with a retention period of 1 day.

## Branch-Based Strategy Summary

The action implements a sophisticated branch-based strategy:

1. **Master/Main Branch**:
   - Treated as production releases
   - Increments minor version (1.2.0 → 1.3.0)
   - Performs full Maven release with tagging
   - Optionally deploys artifacts

2. **Hotfix Branch**:
   - Special handling for urgent fixes
   - Increments patch version if needed (1.2.0 → 1.2.1)
   - Performs full Maven release with tagging
   - Optionally deploys artifacts

3. **Other Branches**:
   - Treated as development builds
   - Uses commit-based versioning (dev-a1b2c3d4)
   - Performs standard Maven package operation
   - Does not deploy artifacts (but still uploads them to GitHub Actions)

## Key Technical Features

1. **Automated Version Management**: Automatically determines appropriate version numbers based on branch type and existing version.

2. **Conditional Build Processes**: Adapts the build process based on the branch type.

3. **Artifact Management**: Uploads artifacts for all builds and optionally deploys them for production builds.

4. **Caching**: Implements Maven repository caching to improve build performance, with support for both local GitHub Actions cache and S3-based caching.

5. **Configurable Environment**: Allows customization of Java version, distribution, Maven version, and other parameters.

This composite action provides a standardized, reliable, and maintainable build process for Java/Maven projects, with sophisticated version management and branch-based strategies.

## S3 Cache Configuration

The action supports using Amazon S3 for caching Maven dependencies, which provides several advantages over the standard GitHub Actions cache:

1. **Persistent Storage**: Cache persists beyond the 7-day limit of GitHub Actions cache
2. **Cross-Workflow Sharing**: Cache can be shared across different workflows and repositories
3. **Improved Performance**: Potentially faster cache operations, especially for large dependency sets

### Requirements for S3 Cache

To use the S3 cache functionality:

1. **AWS Role ARN**: Provide an AWS Role ARN via the `aws_role_arn` input parameter
2. **Permissions**: The GitHub workflow must have `permissions: write-all` set to allow the action to configure AWS credentials
3. **S3 Bucket**: The action uses a predefined S3 bucket (`ct-github-action-cache`) in the `ap-south-1` region

### Example Workflow with S3 Cache

```yaml
name: Build with S3 Cache

on:
  push:
    branches: [ main ]



jobs:
  build:
    runs-on: ubuntu-latest
    permissions: write-all
    steps:
      - uses: actions/checkout@v4
      
      - name: Build with Java
        uses: cleartax/build-java@main
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          aws_role_arn: ${{ secrets.AWS_ASSUME_ROLE_ARN }}
```

When `aws_role_arn` is not provided, the action automatically falls back to using the standard GitHub Actions cache.