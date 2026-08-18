# WildFly SCA Scanner

Automated Software Composition Analysis (SCA) scanning for WildFly application server distributions using OWASP Dependency Check. The scanner identifies known security vulnerabilities (CVEs) in WildFly's dependencies by analyzing packaged libraries against the National Vulnerability Database (NVD) and Sonatype OSS Index.

Scans run daily via GitHub Actions across multiple WildFly versions and distribution variants (standard, preview, EE10, nightly builds).

## GitHub Actions Workflows

The scanning infrastructure is a series of coordinated GitHub Actions workflows that run daily in sequence. All workflows also support manual triggering via `workflow_dispatch`.

### Execution Schedule

Workflows run in deliberate sequence with non-round-minute offsets to avoid GitHub's on-the-hour congestion:

| Time (UTC) | Workflow | File | Purpose |
|------------|----------|------|---------|
| 04:26 | OWASP Dependency Check Download | `dependency-check-download.yaml` | Download and cache the Dependency Check CLI |
| 04:41 | Galleon Download | `galleon-download.yaml` | Download and cache the Galleon provisioning tool |
| 05:12 | OWASP Database Maintenance | `database-maintenance.yaml` | Update the vulnerability database with latest CVE data |
| 05:22 | WildFly Maintenance | `wildfly-instances.yaml` | Provision released WildFly standard distributions |
| 05:27 | WildFly Preview Maintenance | `wildfly-preview-instances.yaml` | Provision released WildFly Preview distributions |
| 05:32 | WildFly EE10 Maintenance | `wildfly-ee10-instances.yaml` | Provision released WildFly EE10 distributions |
| 05:47 | WildFly Nightly Maintenance | `wildfly-nightly.yaml` | Download and provision nightly builds from CI |
| 06:13 | WildFly Scan | `scan-wildfly.yaml` | Run SCA scans across all provisioned distributions |

### Tool Downloads

**Dependency Check Download** (`dependency-check-download.yaml`) downloads the OWASP Dependency Check CLI. The version is controlled by the `DEPENDENCY_CHECK_VERSION` repository variable. The `scripts/update-dependency-check.sh` script handles the download and extraction. The installation is cached with key `dependency-check-{VERSION}` and only re-downloaded when the version changes.

**Galleon Download** (`galleon-download.yaml`) downloads the Galleon provisioning tool used to install WildFly distributions. The version is controlled by the `GALLEON_VERSION` repository variable. The `scripts/update-galleon.sh` script handles download and setup. Cached with key `galleon-{VERSION}`.

### Database Maintenance

**OWASP Database Maintenance** (`database-maintenance.yaml`) updates the OWASP vulnerability database with the latest CVE information from the NVD and Sonatype OSS Index. The database uses a rolling cache strategy: it restores the most recent cache (key prefix `owasp-database-`), runs the update, then saves with a unique key (`owasp-database-{RUN_ID}-{ATTEMPT}`) so updated data propagates to subsequent scan jobs.

### WildFly Provisioning

Four workflows provision different WildFly distributions and cache them for scanning:

**WildFly Maintenance** (`wildfly-instances.yaml`) provisions released WildFly versions using Galleon. The versions to provision are defined in the `matrix.version` array in this file. Each version is cached with key `wildfly-{VERSION}` and only re-provisioned on cache miss. Uses Galleon command: `galleon.sh install wildfly#VERSION --dir=wildfly`.

**WildFly Preview Maintenance** (`wildfly-preview-instances.yaml`) provisions WildFly Preview distributions (experimental features). Versions are defined in its `matrix.version` array. Cached with key `wildfly-{VERSION}-Preview`. Uses Galleon command: `galleon.sh install wildfly-preview#VERSION --dir=wildfly`.

**WildFly EE10 Maintenance** (`wildfly-ee10-instances.yaml`) provisions WildFly EE10 distributions (Jakarta EE 10 compatibility). Versions are defined in its `matrix.version` array. Cached with key `wildfly-{VERSION}-EE10`. EE10 provisioning requires a `provisioning.xml` file with two feature packs (`wildfly-ee-10-feature-pack` first, then `wildfly-galleon-pack`) because Galleon CLI does not support multiple feature packs in a single install command.

**WildFly Nightly Maintenance** (`wildfly-nightly.yaml`) downloads and provisions nightly builds from WildFly CI (`ci.wildfly.org`). The matrix includes `upstream` (main branch) and `maintenance` (maintenance branch) entries. The `UPSTREAM_CI` and `MAINTENANCE_CI` repository variables map these to CI job names. Setting `MAINTENANCE_CI` to `'OFF'` disables maintenance branch scanning. Each nightly run provisions standard, EE10 (upstream only), and preview distributions, cached with unique keys incorporating the run ID (e.g. `wildfly-standard-{VERSION}-{RUN_ID}-{ATTEMPT}`).

### Security Scanning

**WildFly Scan** (`scan-wildfly.yaml`) runs OWASP Dependency Check across all provisioned distributions. The scan matrix is defined in the `matrix.version` array in this file and must include entries for every provisioned distribution. Scans run sequentially (`max-parallel: 1`) using the `--noupdate` flag (relies on the pre-updated database). Suppressions from `owasp-suppressions/owasp-suppressions.xml` are applied. Maintenance branch entries are skipped when `MAINTENANCE_CI` is `'OFF'`.

Scan results (HTML and JSON reports) are uploaded as workflow artifacts named `scan-results-{VERSION}`. On failure, log files are uploaded as `log-files-{VERSION}`.

## Repository Variables and Secrets

### Variables

| Variable | Purpose |
|----------|---------|
| `DEPENDENCY_CHECK_VERSION` | Version of the OWASP Dependency Check CLI to download |
| `GALLEON_VERSION` | Version of the Galleon provisioning tool to download |
| `UPSTREAM_CI` | TeamCity job name for upstream nightly builds |
| `MAINTENANCE_CI` | TeamCity job name for maintenance branch builds, or `'OFF'` to disable |
| `SONATYPE_OSS_INDEX` | Set to `'OFF'` to disable OSS Index analysis |

### Secrets

| Secret | Purpose |
|--------|---------|
| `NVD_API_KEY` | API key for National Vulnerability Database access |
| `OSS_INDEX_USERNAME` | Username for Sonatype OSS Index |
| `OSS_INDEX_PASSWORD` | Password for Sonatype OSS Index |

## Where Version Information Is Defined

Scanned WildFly versions are not centrally configured. Each provisioning workflow and the scan workflow define their own `matrix.version` array. When updating versions, you need to edit the relevant files:

| What | Where to edit |
|------|---------------|
| Released standard versions | `wildfly-instances.yaml` `matrix.version` array |
| Released preview versions | `wildfly-preview-instances.yaml` `matrix.version` array |
| Released EE10 versions | `wildfly-ee10-instances.yaml` `matrix.version` array |
| Nightly build branches | `wildfly-nightly.yaml` `matrix.version` array |
| Scan targets | `scan-wildfly.yaml` `matrix.version` array |
| Dependency Check version | `DEPENDENCY_CHECK_VERSION` repository variable |
| Galleon version | `GALLEON_VERSION` repository variable |
| Nightly CI job mapping | `UPSTREAM_CI` / `MAINTENANCE_CI` repository variables |

**The scan matrix must stay in sync with the provisioning matrices.** Every entry in `scan-wildfly.yaml` must have a corresponding cached distribution, and cache keys must match between provisioning and scanning workflows. The scan workflow uses `fail-on-cache-miss: true`, so a mismatch causes a scan failure.

## Adding or Removing WildFly Versions

When a new WildFly version is released:

1. Add the version to `wildfly-instances.yaml` `matrix.version`
2. Add it to `wildfly-preview-instances.yaml` if a preview distribution is available
3. Add it to `wildfly-ee10-instances.yaml` if an EE10 distribution is available
4. Add corresponding entries to `scan-wildfly.yaml` `matrix.version` (standard entry, plus `-Preview` and/or `-EE10` suffixed entries as applicable)
5. Consider removing older versions from all files to keep the scan matrix manageable

The naming conventions for scan matrix entries are:
- Released standard: `VERSION` (e.g. `40.0.1.Final`)
- Released preview: `VERSION-Preview`
- Released EE10: `VERSION-EE10`
- Nightly standard: `standard-upstream`, `standard-maintenance`
- Nightly EE10: `ee10-upstream`
- Nightly preview: `preview-upstream`, `preview-maintenance`

## Suppressions

The `owasp-suppressions/owasp-suppressions.xml` file contains suppression rules to filter out noise from scan results. Suppressions fall into two broad categories:

**CPE Suppressions** correct cases where OWASP Dependency Check incorrectly maps a Java library's GAV coordinates to an unrelated product's CPE (Common Platform Enumeration). For example, a Java gRPC library being matched to gRPC-Go CVEs, or a WildFly subsystem being matched against the WildFly server product itself. These suppressions are generally permanent.

**CVE Suppressions** silence specific CVEs that have been triaged and determined to not affect WildFly. Common reasons include: the CVE affects a different module within the same product (e.g. log4j-core CVE attributed to log4j-api), the CVE affects a different major version branch, or the CVE targets server-side functionality not present in a client library. These suppressions typically include an `until` date (usually 12 months out) so they are periodically re-evaluated and can be removed if the underlying dependency is updated.

All suppressions should include `<notes>` explaining the rationale with references to tracking issues where applicable (e.g. JIRA keys like `WFLY-*` or GitHub issue numbers).

## Caching Strategy

The workflows use GitHub Actions caching to avoid redundant downloads and provisioning:

- **Tool caches** (`dependency-check-{VERSION}`, `galleon-{VERSION}`): stable, persist until the tool version changes
- **Released WildFly caches** (`wildfly-{VERSION}`, `wildfly-{VERSION}-Preview`, `wildfly-{VERSION}-EE10`): stable, persist until the WildFly version is removed from the matrix
- **Nightly WildFly caches** (`wildfly-{standard|ee10|preview}-{upstream|maintenance}-{RUN_ID}-{ATTEMPT}`): rotate daily, each run creates a new cache
- **Database cache** (`owasp-database-{RUN_ID}-{ATTEMPT}[-{JOB_INDEX}]`): rolling, restored by prefix and saved with a unique key after each update

## Artifacts

Scan results are available as workflow artifacts retained according to GitHub's default policy (90 days):
- **HTML Reports**: human-readable vulnerability reports
- **JSON Reports**: machine-readable data
- **Log Files**: detailed execution logs (uploaded only on failure)

## License

This project follows the same license as WildFly.
