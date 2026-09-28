# HANDOFF — minio-notesnook

Object store for the Notesnook sync stack (`Dvalin21/notesnook-sync-server`).
Published as `dvalin21/minio-notesnook:latest`.

> **Status: the source is in good shape; the *build* is not reproducible.**
> See [Provenance](#provenance--the-real-problem) before trusting any image.

## Branch model

| Branch | Base | Purpose |
|--------|------|---------|
| `minio-notesnook` | `RELEASE.2025-09-07T16-13-09Z` (`07c3a429b`) | **Production.** Pinned release + 4 forward-ported security fixes. |
| `master` | upstream | Untouched upstream mirror. Now 25 commits ahead; ignore it. |
| `minio-full-ui` | older | Reference only. |

`minio/minio` was **archived 2026-04-25** and is read-only, so upstream will
never ship another release. The pinned base is therefore permanent; the only
way this gets a security fix is by porting it here.

## Patches applied

All four are on `minio-notesnook`, on top of the pinned base. They are the
upstream PR merges, forward-ported.

| Branch commit | Upstream | Fix | Files |
|---|---|---|---|
| `25179bcfe` | `c1a49490c7` (#21642) | IAM sub-policy validation bypass — **service account privilege escalation** | `cmd/iam.go` (+18/−15), 2 test files |
| `44951eba3` | `1b8ac0af9f` (#21612) | S3 POST policy trailing-slash bypass | `cmd/signature-v4-parser.go` (+1/−1) |
| `8f5489205e` | `7a80ec1cce` (#21582) | LDAP TLS handshake with StartTLS | `internal/config/identity/ldap/config.go` (+9) |
| `0654eb553` | `534f4a9fb1` (#21615) | Data-scanner `timeN` closure leak | `cmd/data-scanner-metric.go` (+6/−8) |

*An earlier revision of this file said "Patches Applied: None yet" and listed
these four as pending. That was wrong — they had already been applied, and the
same commit appeared in both the pending and the skipped table.*

### Deliberately not ported

| Upstream commit | Reason |
|---|---|
| `ba3c0fd1c` Go 1.24.8 toolchain | Dockerfile pins the Go version |
| `ae71d7690`, `756f3c814`, `456d9462e` | Clustering — single-node only |

## Known-unpatched CVEs (mitigated at the proxy, not fixed here)

These affect **every** open-source MinIO release including ours, and
`Patched versions: None` — the fix ships only in commercial MinIO AIStor.

| CVE | Impact |
|---|---|
| **CVE-2026-40344** / GHSA-9c4q-hq6p-c237 (CVSS 8.8) | Anyone with a valid **access key** can write arbitrary objects with no secret key and a fabricated signature. |
| **CVE-2026-41145** / GHSA-hv4r-mvr4-25vw | Same class, via query-string credentials. |
| CVE-2026-33322 | OIDC JWT algorithm confusion. **Not applicable** — OIDC is not enabled. |
| GHSA-xh8f-g2qw-gcm7, CVE-2023-28432 | Cluster-only. **Not applicable** — single node. |

Verified present in this source tree:
`cmd/object-handlers.go` gates signature verification on the *presence* of an
`Authorization` header while `isPutActionAllowed` trusts the `X-Amz-Credential`
**query parameter**; and `PutObjectExtractHandler`'s switch has no
`case authTypeStreamingUnsignedTrailer`, so it falls through unverified.

**Mitigation lives in the sync-server stack's `Caddyfile`**, which rejects
`X-Amz-Content-Sha256: STREAMING-UNSIGNED-PAYLOAD-TRAILER` with 403 and
redacts the S3 presigning parameters from the access log. See
`Dvalin21/notesnook-sync-server` → *MinIO / S3*. Do not remove those without
replacing them.

## Provenance — the real problem

**The deployed image cannot be traced to a commit.** It reports:

```
minio version DEVELOPMENT.GOGET (commit-id=DEVELOPMENT.GOGET)
```

`DEVELOPMENT.GOGET` is the default in `cmd/build-constants.go`, i.e. the image
was compiled from source with no `-ldflags`. So the build stamps neither the
version nor the commit, and you cannot tell from the artifact which of the
patches above it contains.

**Neither Dockerfile in this repo reproduces it:**

- `Dockerfile` expects a **pre-built `./minio` binary** in the build context and
  does `COPY ./minio /usr/bin/minio`. The previously documented
  `docker build -t dvalin21/minio-notesnook:latest -f Dockerfile .` therefore
  **fails** unless a binary is placed there first.
- `Dockerfile.release` does **not compile** — it `curl`s a prebuilt binary from
  `dl.min.io` and verifies the minisign signature. It also would not report
  `DEVELOPMENT.GOGET`, because it passes `-ldflags` setting version and release.

The image was therefore built by a source compile that is not recorded in any
file here. **Do not trust a future rebuild to match it** until a build that
stamps the commit exists.

Also note: the tag `RELEASE.2026-09-02T00-00-00Z` on this branch is **not** a
MinIO release. It is a local tag pointing at the `HANDOFF.md` commit and has no
relationship to any upstream build.

## Building

Build from source and **stamp the commit**, so the artifact is traceable:

```bash
git checkout minio-notesnook
export VERSION="notesnook-$(git rev-parse --short HEAD)"
docker buildx build --platform linux/amd64,linux/arm64 \
  -t dvalin21/minio-notesnook:"$VERSION" \
  -t dvalin21/minio-notesnook:latest \
  --build-arg "VERSION=$VERSION" \
  --build-arg "COMMIT=$(git rev-parse HEAD)" .
```

Whatever build path is used, it must (a) compile the Go source in this branch,
not download a binary, and (b) embed the commit in `--version` output. Verify
afterwards with:

```bash
docker run --rm dvalin21/minio-notesnook:latest minio --version
```

Expect the version string to name the commit, not `DEVELOPMENT.GOGET`.
