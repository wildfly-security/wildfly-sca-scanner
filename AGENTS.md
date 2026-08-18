# AGENTS.md

This file provides context for AI agents working in this repository. For full project documentation, see [README.md](README.md).

## Project Overview

This repository contains GitHub Actions workflows and supporting scripts that perform daily SCA (Software Composition Analysis) scanning of WildFly application server distributions using OWASP Dependency Check. It is not a traditional codebase — the primary artifacts are workflow YAML files, shell scripts, and an XML suppression file.

## Repository Layout

```
.github/workflows/          GitHub Actions workflow definitions
scripts/                    Shell scripts for tool download and setup
owasp-suppressions/         OWASP Dependency Check suppression rules
  owasp-suppressions.xml    The suppression file applied during scans
```

## Key Concepts

- **Workflows** run daily in a timed sequence (tools → database → provisioning → scan). See the README for the full schedule and workflow descriptions.
- **Scanned versions** are defined in `matrix.version` arrays inside individual workflow YAML files — there is no central version list. The scan matrix in `scan-wildfly.yaml` must stay in sync with the provisioning workflows.
- **Tool versions** (Dependency Check, Galleon) are controlled via GitHub repository variables, not files in this repo.
- **Suppressions** in `owasp-suppressions/owasp-suppressions.xml` filter false positives and triaged CVEs from scan results. CPE suppressions fix incorrect library-to-product mappings. CVE suppressions silence specific triaged vulnerabilities and typically carry an `until` date for periodic re-evaluation.

## Working on Suppressions

When adding or modifying suppressions:

1. Include `<notes>` explaining why the suppression is valid, with references to tracking issues (JIRA keys like `WFLY-*` or GitHub issue numbers).
2. For CVE suppressions, set an `until` attribute (typically 12 months out) so the suppression is periodically re-evaluated.
3. CPE suppressions that correct permanent GAV-to-CPE mismatches generally do not need an `until` date.
4. Always validate XML after editing: `xmllint --noout owasp-suppressions/owasp-suppressions.xml`

## Working on Versions

When updating scanned WildFly versions, edit the `matrix.version` arrays in the relevant provisioning workflow files and `scan-wildfly.yaml`. See the "Adding or Removing WildFly Versions" section in the README for the full procedure and naming conventions.
