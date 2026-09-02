# HANDOFF — minio-fork (Notesnook edition)

## Project Overview

This is a fork of MinIO Community Edition, frozen at `RELEASE.2025-09-07T16-13-09Z` and maintained separately for the Notesnook sync server stack. MinIO the project was archived in April 2026 with no further security patches.

**Purpose:** S3-compatible object storage for Notesnook note attachments only.

**Design Philosophy:** Minimal. Only S3 API surface is required. All admin UI, clustering, and non-essential features are unnecessary.

## Branches

| Branch | Based On | Purpose |
|--------|----------|---------|
| `minio-notesnook` | RELEASE.2025-09-07 | Production fork with security patches |
| `minio-full-ui` | RELEASE.2025-04-22 | Reference branch with full admin UI (pre-removal) |
| `master` | Upstream | Untouched upstream mirror (archived) |

## Security Patching Strategy

We do NOT track upstream's Go version bumps or dependency refactors. We only apply security-relevant code changes manually to the specific files that affect S3 API behavior.

### Patches Applied

None yet — this is the baseline.

### Patches to Apply (identified)

| Commit | Description | Files Affected | Risk |
|--------|-------------|----------------|------|
| `c1a49490c` | IAM sub-policy validation bypass | `cmd/admin-handlers-users.go` | Security fix — service account privilege escalation |
| `1b8ac0af9` | S3 POST policy trailing slash bypass | `cmd/signature-v4-parser.go` | Security fix — policy validation bypass |
| `7a80ec1cc` | LDAP TLS handshake with StartTLS | `internal/config/identity/ldap/config.go` | Security fix — LDAP auth bypass (if LDAP enabled) |
| `534f4a9fb` | Data scanner timeN closure leak | `cmd/data-scanner-metric.go` | Resource leak fix |

### Patches Skipped (not applicable)

| Commit | Description | Reason |
|--------|-------------|--------|
| `ba3c0fd1c` | Go 1.24.8 toolchain | Irrelevant — Dockerfile controls Go version |
| `ae71d7690` | Remove unnecessary replication checks | Clustering not used |
| `756f3c814` | PoolID decommission | Clustering not used |
| `456d9462e` | RebalanceStats cancel | Clustering not used |
| `534f4a9fb` | timeN closure | Data scanner — marginal relevance |

## Build & Deploy

```bash
# Build from minio-notesnook branch
cd /home/keith/host/minio-fork
git checkout minio-notesnook
docker build -t dvalin21/minio-notesnook:latest -f Dockerfile .
docker push dvalin21/minio-notesnook:latest
```

## Files Modified from Upstream

None yet.

## Notes

- MinIO Community Edition was archived April 25, 2026 — no upstream patches after October 2025
- Last MinIO release: `RELEASE.2025-10-15T17-29-55Z`
- We skip all Go version bumps, dependency updates, and UI/admin changes
- We only apply security-relevant S3 API fixes manually
