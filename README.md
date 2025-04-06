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

- `java_version`: The Java version to use (default: '17')
- `java_distribution`: The Java distribution to use (default: 'zulu')
- `maven_version`: The Maven version to use (default: '3.8.3')
- `maven_flag`: Maven command-line flags (default: '-T 4C -B --no-transfer-progress')
- `token`: The GitHub token (required)
- `publish_artifacts`: Whether to publish artifacts (default: 'false')
- `ref`: The ref to build (default: current GitHub ref)
- `tag_name_prefix`: Tag name prefix for checkout (default: '')

### Output

- `version`: The version of the built artifact, accessible via `${{ steps.set-version.outputs.version }}`

## Workflow Steps in Detail

### 1. Environment Setup

The action begins by setting up the Java environment using the official `actions/setup-java@v4` action to install the specified Java version and distribution.

### 2. Maven Configuration

The action configures Maven with caching to speed up builds. It uses `actions/cache@v4` to cache the Maven repository and `stCarolas/setup-maven@v5` to set up the specified Maven version.

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

4. **Caching**: Implements Maven repository caching to improve build performance.

5. **Configurable Environment**: Allows customization of Java version, distribution, Maven version, and other parameters.

This composite action provides a standardized, reliable, and maintainable build process for Java/Maven projects, with sophisticated version management and branch-based strategies.