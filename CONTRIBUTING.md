# Contribution guide

Contributing to `core-cloud-workflow-yarn-actions`

This repository is part of the UK Home Office Core Cloud Platform shared tooling ecosystem.
To maintain consistent security and operational standards, all contributions must follow the guidelines below.

## Repository Structure & Source of Truth

This repository contains:

* `actions` — composite GitHub Action for running yarn actions

* `tests/**` — intentionally valid and invalid node test fixtures

* `sonar-project.properties` — SAST configuration

* Supporting documentation and helper scripts

Please familiarise yourself with this structure before making changes.

## SAST & Security Requirements
This repository is continuously scanned by:

* SonarQube (via `sonarqube-scan.yaml`)

To avoid false positives and pipeline failures, **do not place production Code under these directories**:

* tests/**

These directories contain intentionally broken Node code designed solely for:

* yarn action regression tests

* yarn action behaviour validation

* Example usage patterns for downstream repositories

They are excluded from all SAST tools via:


* sonar-project.properties

If you add real node code here, it will be ignored by SAST and may mislead users.

## Adding / Updating Test Fixtures
The tests/** directory is for:

* Invalid node code (to ensure failures surface correctly)

* Valid node code (to ensure passes work correctly)

When adding tests:

1. Never commit real code

2. Keep all test cases isolated inside their own folder.


## Development Workflow

1. Create a feature branch

```
feature/<JIRA-ID>-<short-description>

```


1. Open a Pull Request

All PRs must go through:

* SonarQube analysis

* SemVer label check

* Self-Test workflow for TFLint

* Manual reviewer approval

4. SemVer labelling

Each PR must include exactly one of:

* major

* minor

* patch

* (or skip-release)

Labels are validated automatically by workflow `Check PR for SemVer Label`.

## Coding Standards

* All bash scripts must run in `set -e` mode unless justified.

* Use idiomatic GitHub Actions patterns.

* Prefer enterprise-approved actions (see `.github/workflows/self-test.yaml` for allowed list).

* Composite action logic must remain POSIX-portable.

## Getting Help

For questions, reach out via:

* #core-cloud-sauron

* Reviewers listed in `CODEOWNERS`