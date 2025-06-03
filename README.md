# Documentation

## Table of Contents

* [Overview](#overview)
* [Requirements](#requirements)

  * [Job Permissions](#job-permissions)
  * [Operating System](#operating-system)
  * [Dependencies](#dependencies)
* [Setup Instructions for Air-Gapped Environments](#setup-instructions-for-air-gapped-environments)
* [Usage Example](#usage-example)
* [Action Inputs](#action-inputs)
* [How It Works](#how-it-works)
* [Authentication](#authentication)
* [References](#references)

---

## Overview

This repository hosts the `upload-to-jfrog` GitHub Action along with a workflow designed to test this action using a dummy `.jar` and a corresponding `pom.xml` file.

The action expects that the JAR file and its `pom.xml` have already been generated. It utilizes JFrog CLI to manage uploads to Artifactory. Distribution management settings are provided directly via action inputs rather than through the `pom.xml`.

## Requirements

### Job Permissions

Ensure the job calling the action has the following permissions:

```yaml
permissions:
  id-token: write
  contents: read
  security-events: write
```

### Operating System

| Operating System | Supported Versions  | Minimum Required Version |
| ---------------- | ------------------- | ------------------------ |
| RHEL             | 8 and above         | 8                        |
| CentOS           | 9 and above         | 9                        |
| Ubuntu           | 18.04, 20.04, 22.04 | 18.04                    |

### Dependencies

Ensure these dependencies are installed:

* Docker
* Curl
* RPM or Yum (according to Linux distribution)

## Setup Instructions for Air-Gapped Environments

To deploy in air-gapped environments (no internet access), you need a local Artifactory repository that mirrors [JFrog Releases](https://releases.jfrog.io/):

1. Create a local **generic** repository in your Artifactory instance.
2. Download necessary resources from [JFrog Analyzer Manager](https://releases.jfrog.io/artifactory/xsc-gen-exe-analyzer-manager-local/) from a machine with internet access.
3. Upload these files into the newly created generic repository.
4. Configure your tools (CLI, IDE, or automation tools) to use this repository for dependencies.

## Usage Example

Here's a usage example derived from the workflow (`.github/workflows/test-upload.yml`):

```yaml
name: Test JFrog Maven Deploy (CLI-only)

on: [push, workflow_dispatch]

jobs:
  upload:
    runs-on: ubuntu-latest
    environment: ${{ matrix.env }}
    permissions:
      id-token: write
      contents: read
    strategy:
      fail-fast: false
      matrix:
        env: [development, qa]

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Generate dummy artifacts
        id: gen
        shell: bash
        run: |
          ts=$(date +%s)
          ver="1.0-${{ matrix.env }}.$ts"
          pom="dummy-artifact-$ver.pom"
          jar="dummy-artifact-$ver.jar"

          cat >"$pom" <<EOF
          <project>
            <modelVersion>4.0.0</modelVersion>
            <groupId>com.test.local</groupId>
            <artifactId>dummy-artifact</artifactId>
            <version>$ver</version>
            <packaging>jar</packaging>
          </project>
          EOF

          printf 'Test content – %s\n' "$ts" >content.txt
          zip "$jar" content.txt && rm content.txt

          echo "pom=$pom" >>"$GITHUB_OUTPUT"
          echo "jar=$jar" >>"$GITHUB_OUTPUT"

      - name: Deploy via JFrog CLI
        uses: ./.github/actions/upload-to-jfrog
        with:
          pom-path:        ${{ steps.gen.outputs.pom }}
          jar-path:        ${{ steps.gen.outputs.jar }}
          jfrog-url:       ${{ vars.JF_URL }}
          jfrog-repo-path: ${{ vars.JF_REPO_PATH }}
          jf-access-token: ${{ secrets.JF_ACCESS_TOKEN }}
```

## Action Inputs

Defined in `.github/actions/upload-to-jfrog/action.yml`:

| Input                | Description                                 | Required |
| -------------------- | ------------------------------------------- | -------- |
| `jar-path`           | Absolute path to the JAR file               | Yes      |
| `pom-path`           | Absolute path to the POM file               | Yes      |
| `jfrog-url`          | URL to your JFrog instance                  | Yes      |
| `jfrog-repo-path`    | Target repository path in JFrog Artifactory | Yes      |
| `oidc-provider-name` | OIDC integration name                       | Yes      |
| `oidc-audience`      | Optional audience claim                     | No       |
| `jf-access-token`    | JFrog Access Token                          | Yes      |

## How It Works

The action executes the following steps:

1. **Validate Artifacts:** Confirms the existence of the specified JAR and POM files.
2. **Setup JFrog CLI:** Configures JFrog CLI with provided inputs using the [Setup JFrog CLI GitHub Action](https://github.com/marketplace/actions/setup-jfrog-cli).
3. **Upload Artifacts:** Executes JFrog CLI commands to upload artifacts to the specified repository.

## Authentication

The primary authentication method uses a JFrog Access Token provided as the `jf-access-token` input. Alternatively, JFrog CLI supports short-lived OIDC credentials if configured correctly.

For more details, refer to [JFrog CLI Authentication Methods](https://github.com/marketplace/actions/setup-jfrog-cli#Authentication-Methods).

## References

* [JFrog CLI Documentation](https://jfrog.com/help/r/jfrog-security-user-guide/developers/cli)
* [Setup JFrog CLI GitHub Action](https://github.com/marketplace/actions/setup-jfrog-cli)
* [JFrog CLI Authentication Methods](https://github.com/marketplace/actions/setup-jfrog-cli#Authentication-Methods)
* [Working in Air-Gapped Environments with JFrog](https://jfrog.com/help/r/jfrog-security-user-guide/shift-left-on-security/working-in-air-gapped-environments)
* [Shift Left on Security with JFrog](https://jfrog.com/help/r/jfrog-security-user-guide/shift-left-on-security)
