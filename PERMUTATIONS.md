# Scan Permutations

This file documents the current matrix of SCA scans performed by this project.

## Released Versions

| Version | Standard | EE10 | Preview |
|---------|----------|------|---------|
| 38.0.1.Final | Yes | | |
| 39.0.1.Final | Yes | | |
| 40.0.1.Final | Yes | Yes | |
| 41.0.0.Final | Yes | Yes | Yes |

Only the latest released version is scanned with all available distributions. Older versions are scanned with the standard distribution only, unless they still require EE10 coverage.

## Nightly Builds

| Branch | Standard | EE10 | Preview |
|--------|----------|------|---------|
| Upstream (main) | Yes | | Yes |
| Maintenance | Yes | Yes | Yes |

EE10 has been removed from upstream WildFly and is only present on the maintenance branch.

## Scan Matrix Entries

The `scan-wildfly.yaml` workflow uses the following matrix entries, each corresponding to a cached WildFly installation:

**Released versions:**
- `38.0.1.Final`, `39.0.1.Final`, `40.0.1.Final` — standard distributions
- `40.0.1.Final-EE10` — EE10 distribution
- `41.0.0.Final` — standard distribution
- `41.0.0.Final-EE10` — EE10 distribution
- `41.0.0.Final-Preview` — preview distribution

**Nightly builds:**
- `standard-upstream`, `preview-upstream` — upstream branch
- `standard-maintenance`, `ee10-maintenance`, `preview-maintenance` — maintenance branch
