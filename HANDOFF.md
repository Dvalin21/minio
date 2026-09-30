# HANDOFF — minio-notesnook

Object store for the Notesnook sync stack (`Dvalin21/notesnook-sync-server`).
Published as `dvalin21/minio-notesnook:latest`.

> **Status: source is good. The build is now stamped and traceable; the
> `Dockerfile` in this repository still is not the one to use.**
> See [Provenance](#provenance) before trusting any image.

## Branch model

| Branch | Base | Purpose |
|--------|------|---------|
| `minio-notesnook` | `RELEASE.2025-09-07T16-13-09Z` (`07c3a429b`) | **Production, and the only branch in this repository.** Pinned release + 4 forward-ported security fixes. |

**Correction to an earlier revision of this file.** It listed a `master`
branch as an untouched upstream mirror and a `minio-full-ui` branch for
reference. **Neither exists.** Verified against the remote:

```bash
git ls-remote --heads https://github.com/Dvalin21/minio.git
# 3b2d2032457ef75ae29cdb171d9e1d5004082c75	refs/heads/minio-notesnook
```

`minio/minio` is **archived and read-only** — confirmed via the GitHub API
(`archived: true`), last push `2026-04-24T17:54:39Z`. So upstream will never
ship another release. The pinned base is therefore permanent; the only way this
gets a security fix is by porting it here.

### Tags are the remaining impersonation risk

This repository still carries **1,036 upstream `RELEASE.*` tags**, spanning
`RELEASE.2016-03-11T03-45-50Z` onward, and it is still a GitHub *fork* of
`minio/minio` (API: `fork: true`, parent `minio/minio`). Combined, that makes
it easy to mistake for official MinIO. Removing the tags and the fork relation
is outstanding work — see [Decoupling](#decoupling-status).

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
| **CVE-2026-40344** / GHSA-9c4q-hq6p-c237 | Anyone with a valid **access key** can write arbitrary objects with no secret key and a fabricated signature. |
| **CVE-2026-41145** / GHSA-hv4r-mvr4-25vw | Same class, via query-string credentials. |
| CVE-2026-33322 | OIDC JWT algorithm confusion. **Not applicable** — OIDC is not enabled. |
| GHSA-xh8f-g2qw-gcm7, CVE-2023-28432 | Cluster-only. **Not applicable** — single node. |

**Scoring note.** An earlier revision of this file cited CVSS 8.8 for
CVE-2026-40344 without saying which version. Both numbers are real and both are
HIGH — NVD carries `cvssMetricV40: 8.8` and `cvssMetricV31: 8.2`, while
GitHub's advisory API shows 8.2 because it reports the v3.1 figure. Cite the
version when quoting a score. CVE-2026-41145 is 8.2 on both.

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

The rule was confirmed against the running stack, and confirmed to be *the load
balancer's* response rather than MinIO's: a `PUT` carrying the trailer header
returns `403` with no `content-type` and `content-length: 0` (Caddy's
`respond`), whereas the same `PUT` without the header returns `403` with
`content-type: application/xml` (MinIO's own signature rejection). Two
different layers, two different error shapes.

## Provenance

**Resolved.** Images built after the stamping fix name their commit. The
previously published image did not, and if you find one that does not, it
predates the fix.

The old behaviour, and the way to recognise it:

```
minio version DEVELOPMENT.GOGET (commit-id=DEVELOPMENT.GOGET)
```

`DEVELOPMENT.GOGET` is the default in `cmd/build-constants.go`. A build
reporting it passed no `-ldflags`, so it stamps neither version nor commit and
you cannot tell from the artifact which of the four patches it contains.

**Cause.** Both Dockerfiles in this repository were wrong for the purpose:

- `Dockerfile` expects a **pre-built `./minio` binary** in the build context and
  does `COPY ./minio /usr/bin/minio` (confirmed at line 5). A plain
  `docker build -f Dockerfile .` therefore **fails** unless a binary is placed
  there first.
- `Dockerfile.release` does **not compile** — it `curl`s a prebuilt binary from
  `dl.min.io` and verifies the minisign signature. It would not report
  `DEVELOPMENT.GOGET` either, because it does pass `-ldflags`.

So the image that was published was compiled by a path recorded in no file here.

**Fix.** The build now used for this stack is `minio/Dockerfile` in
`Dvalin21/notesnook-sync-server`. It clones the pinned commit and calls MinIO's
own `buildscripts/gen-ldflags.go`, so the `-X` targets cannot drift from what
`cmd/build-constants.go` actually reads. `MINIO_RELEASE=notesnook` tags the
output as a local build rather than a MinIO release.

It ends with a guard that fails the build if the commit is not embedded in the
binary:

```dockerfile
&& test -x /out/minio \
&& grep -q "${MINIO_COMMIT}" /out/minio
```

Verified in both directions: with the flags the image reports
`minio version notesnook.2026-09-28T06-30-06Z (commit-id=3b2d2032457ef75ae29cdb171d9e1d5004082c75)`;
with `-ldflags` removed as a negative control the build exits `1`. An unstamped
artifact cannot ship again.

Also note: the tag `RELEASE.2026-09-02T00-00-00Z` on this branch is **not** a
MinIO release. It peels to `fa64b2cdad29`, a commit titled *"docs: add HANDOFF.md
for minio-notesnook fork"* that changes exactly one file, and it has no
relationship to any upstream build. It is among the tags that should be removed.

## Building

Do not use `Dockerfile` or `Dockerfile.release` from this repository. Use the one
in the sync-server repository:

```bash
git clone https://github.com/Dvalin21/notesnook-sync-server
cd notesnook-sync-server
docker build -f minio/Dockerfile -t dvalin21/minio-notesnook:latest minio/
```

Verify afterwards:

```bash
docker run --rm dvalin21/minio-notesnook:latest minio --version
```

Expect the version string to name the commit, not `DEVELOPMENT.GOGET`.

## Decoupling status

The goal is for this repository to hold **only** the local changes, with
upstream `minio/minio` demoted to a pinned build input rather than a parent
repository.

**Feasibility is proven.** Cloning upstream at the base tag and applying the
four patches reproduces this branch's Go tree exactly:

```bash
git clone --depth 1 --branch RELEASE.2025-09-07T16-13-09Z https://github.com/minio/minio.git base
cd base && git am ../patches/*.patch     # applies 4/4 cleanly
git diff --name-only FETCH_HEAD HEAD -- '*.go'   # 0 files
```

All four apply with no fuzz and no manual intervention, and the result is
byte-identical to `minio-notesnook` across every `.go` file. So upstream base
plus 17 KB of patch files *is* this branch. The module path is
`github.com/minio/minio` in both, which matters because the `-X` stamp targets
depend on it.

**Outstanding:**

1. Delete the 1,036 upstream `RELEASE.*` tags and the fake
   `RELEASE.2026-09-02T00-00-00Z`.
2. Drop the fork relationship to `minio/minio`.
3. Retarget `minio/Dockerfile`'s `MINIO_REPO` at `minio/minio`, check out the
   base tag, and `git am patches/*.patch`.
4. Optionally slim the tree — 28.4 MB of its 36.2 MB is `cmd/testdata` and
   `docs/`, none of which is needed to build the server.

**Do steps 1–3 without moving `MINIO_COMMIT` and the pin breaks.** The current
Dockerfile checks out `3b2d2032457ef75ae29cdb171d9e1d5004082c75` from this
repository. Any history rewrite that removes that commit breaks the build
immediately. Retarget the Dockerfile in the same change that removes the commit.

## Personal data

This repository deliberately contains **no operator domains and no operator
email addresses**. Verified by scanning the working tree for both; the only
email addresses present are the upstream patch authors' GitHub `noreply`
addresses and the `@example.io` fixtures inside upstream test files. Those are
attribution and must not be scrubbed.

Deployment hostnames, the reverse-proxy configuration that mitigates the CVEs,
and the operator's contact address all live in
`Dvalin21/notesnook-sync-server`, not here.
