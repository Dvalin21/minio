# HANDOFF — minio-notesnook

Object store for the Notesnook sync stack (`Dvalin21/notesnook-sync-server`).
Published as `dvalin21/minio-notesnook:latest`.

> **Status: source is good. The build is now stamped and traceable; the
> `Dockerfile` in this repository still is not the one to use.**
> See [Provenance](#provenance) before trusting any image.

## Branch model

| Branch | Base | Purpose |
|--------|------|---------|
| `minio-notesnook` | `RELEASE.2025-09-07T16-13-09Z` (`07c3a429b`) | **Production, and the only branch in this repository.** |

**Correction to an earlier revision of this file.** It listed a `master`
branch as an untouched upstream mirror and a `minio-full-ui` branch for
reference. **Neither exists.** Verified against the remote:

```bash
git ls-remote --heads https://github.com/Dvalin21/minio.git
# 3b2d2032457ef75ae29cdb171d9e1d5004082c75	refs/heads/minio-notesnook
```

`minio/minio` is **archived and read-only** — confirmed via the GitHub API
(`archived: true`), last push `2026-04-24T17:54:39Z`. Upstream will never ship
another release, so the pinned base is permanent and the only way this gets a
fix is by porting it here.

## This repository no longer builds the image

The build moved to `Dvalin21/notesnook-sync-server` → `minio/Dockerfile`. It
clones **upstream `minio/minio`** at the base tag and applies the four patches
from its own `patches/` directory. It does **not** reference this repository.

Equivalence was proven, not assumed. Cloning the base tag, applying the four
patches, and diffing the entire tree against this branch:

```
.go files differing : 0
non-.go differing   : only README.md, HANDOFF.md, patches/*
```

Zero Go differences means an identical binary. The published image now reports
the upstream base commit rather than a commit from this repository:

```
minio version notesnook.2025-09-07T16-13-09Z (commit-id=07c3a429bfed433e49018cb0f78a52145d4bedeb)
```

What remains here is the record: the patch files, this file, and the README.

### Tags removed

The 524 upstream tags are gone. This repository carried **525 tag names**; an
earlier revision of this file said 1,036, which was wrong — that figure counted
annotated tags twice, once for the tag object and once for its `^{}` peeled
commit.

| Removed | Count | Why |
|---|---|---|
| `RELEASE.*` | 522 | Upstream release names. A tag reading `RELEASE.<date>` implies an official build. |
| `OFFICIAL.2016-02-08T00-12-28Z` | 1 | Upstream's. |
| `release-1434511043` | 1 | Upstream's, non-standard name. |
| `RELEASE.2026-09-02T00-00-00Z` | 1 | **Never a MinIO release.** Peels to `fa64b2cdad29`, "docs: add HANDOFF.md", one file changed. Local tag impersonating a release. |
| **Kept:** `archive-minio-full-ui` | 1 | Ours. Points at `c0f77865f`, preserving a branch that no longer exists. Not impersonation, and it is the only reference to that work. |

Of the 524 removed, 523 existed verbatim in upstream `minio/minio`; the one that
did not was the fake `RELEASE.2026-09-02T00-00-00Z`. The full pre-deletion tag
list is preserved in `.deleted-tags-manifest.txt` at the tip of this branch, so
a restore is possible if one is ever wanted.

Tag deletion did not affect the build, and that was verified rather than
assumed: the image was rebuilt from the same pinned commit afterwards and
produced the same stamp. Tags are not needed to resolve a commit.

### Still outstanding: the fork relationship

This repository is still a GitHub **fork** of `minio/minio`, so the "forked
from" banner remains. GitHub has no unfork operation, so removing it means
creating a new non-fork repository and repointing references — a decision with
a URL change attached, not a mechanical change. Not done.

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

## CVE-2026-40344 and CVE-2026-41145 — now fixed locally

Both were open in the base tag. `first_patched_version` is `None` for both:
upstream is archived, so no patched open-source release exists or ever will.
The fix ships only in commercial MinIO AIStor `RELEASE.2026-04-11`.

They are now fixed by `05-cve-2026-40344-2026-41145.patch`, which lives in the
sync-server repository. This repository's code is not built any more, so
nothing here is patched — the patch series is the current state.

| CVE | CWE | Defect | Fix |
|---|---|---|---|
| CVE-2026-40344 | CWE-306 | `PutObjectExtractHandler` had no `case authTypeStreamingUnsignedTrailer` and no `default`, so the request fell through with no signature verification at all | added the missing case |
| CVE-2026-41145 | CWE-287 | Both put handlers gated verification on `r.Header.Get(xhttp.Authorization) != ""` while `isPutActionAllowed` also accepts `X-Amz-Credential` from the query string | gate now also consults the query parameter |

Verified with a live exploit against both binaries rather than by reading the
diff. On the unpatched build, with a bucket-scoped key and a signature of all
zeros, both writes succeeded and a tar was extracted into the bucket. On the
patched build both are rejected, and a legitimate signed upload still works.

MinIO's own suite for the touched paths passes on the patched tree.

### The load-balancer rule is still worth keeping

`Caddyfile` in the sync-server repository rejects
`X-Amz-Content-Sha256: STREAMING-UNSIGNED-PAYLOAD-TRAILER` with 403. That rule
is now defence in depth rather than the primary control. It is also still the
only thing standing between a *future* MinIO vulnerability and an internet
facing object store, so it should not be removed on the grounds that the known
CVEs are fixed.

**Not fixed, deliberately, and not applicable here:**

| CVE | Why |
|---|---|
| CVE-2026-33322 | OIDC JWT algorithm confusion. OIDC is not enabled. |
| GHSA-xh8f-g2qw-gcm7, CVE-2023-28432 | Cluster-only. Single node. |

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

**Done**, apart from one item that needs a decision.

The goal was for the build to depend only on the local changes, with upstream
as a pinned input rather than a parent repository. That is achieved:

| Step | State |
|---|---|
| Move the four patches to the sync-server repo | **Done.** Byte-identical copies, md5-verified. |
| Retarget the build at `minio/minio` + `git am patches/*.patch` | **Done.** No fork reference remains in the build. |
| Prove the result is the same binary | **Done.** Full-tree diff: 0 `.go` files differ. |
| Delete the 524 upstream tags | **Done.** 525 names → 1. Manifest kept. |
| Drop the fork relationship | **Outstanding — needs a decision.** |

The last one is not mechanical. GitHub provides no unfork operation, so
removing the "forked from minio/minio" banner requires creating a new
non-fork repository and repointing every reference to this URL. That is a URL
change with a decision attached, so it has not been done unilaterally.

### Slimming the tree: no longer necessary

An earlier plan proposed deleting `cmd/testdata` and `docs/` to cut this
repository from 36 MB to 8 MB. **That work is now moot and should not be
done.** Nothing is vendored: the build clones upstream itself, so this
repository's tree size has no effect on the build, the image, or build time
beyond a slower `git clone`. For reference, `cmd/testdata` and `docs/` are
28.4 MB of the 36.2 MB, and a 9.1 MB test fixture is the single largest file.

## Personal data

This repository deliberately contains **no operator domains and no operator
email addresses**. Verified by scanning the working tree for both; the only
email addresses present are the upstream patch authors' GitHub `noreply`
addresses and the `@example.io` fixtures inside upstream test files. Those are
attribution and must not be scrubbed.

Deployment hostnames, the reverse-proxy configuration that mitigates the CVEs,
and the operator's contact address all live in
`Dvalin21/notesnook-sync-server`, not here.
