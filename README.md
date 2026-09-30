# minio-notesnook

**This is not MinIO. It is not a mirror of MinIO. It is not current MinIO.**

It is a pinned MinIO release plus four forward-ported security patches, kept
here so the Notesnook sync stack has an object store it can actually build.

If you arrived looking for MinIO, use
[github.com/minio/minio](https://github.com/minio/minio). This repository exists
for a different reason, described below.

---

## Why this repository exists

Upstream `minio/minio` was **archived and made read-only on 2026-04-24**. It is
frozen: no new releases, no new fixes, and the maintainers have moved to a
commercial product. The last open-source line is `RELEASE.2025-09-07T16-13-09Z`.

A fork pinned to a frozen base has one useful property — it cannot drift — and
one bad property: nobody upstream will ever patch it. This repository is where
the patching happens, deliberately and on the record.

Current state:

| | |
|---|---|
| Base | `RELEASE.2025-09-07T16-13-09Z` (`07c3a429bfed433e49018cb0f78a52145d4bedeb`) |
| Branch | `minio-notesnook` (the only branch in this repository) |
| Patches | 4, all ported from merged upstream PRs — see [`patches/`](patches/) |
| Image | `dvalin21/minio-notesnook:latest` |
| License | AGPL-3.0, unchanged from upstream |

The base is permanent, so the pin is too. The only way anything here gets a
security fix is by porting it in this repository.

## The four patches

All four are already merged upstream. They are carried here because the pinned
base predates them and upstream will never cut a release that includes them.

| File | Upstream PR | Fix |
|---|---|---|
| `01-25179bcfe-21642.patch` | [#21642](https://github.com/minio/minio/pull/21642) | IAM sub-policy validation bypass — service account privilege escalation |
| `02-44951eba3-21612.patch` | [#21612](https://github.com/minio/minio/pull/21612) | S3 POST policy trailing-slash bypass |
| `03-8f5489205e-21582.patch` | [#21582](https://github.com/minio/minio/pull/21582) | LDAP TLS handshake with StartTLS |
| `04-0654eb5533-21615.patch` | [#21615](https://github.com/minio/minio/pull/21615) | Data-scanner `timeN` closure leak |

Patches are stored as files, not only as commits, so the full set of local
changes survives independently of this repository. See
[`patches/README.md`](patches/README.md) for the exact application procedure
and for the upstream commits deliberately *not* ported.

## Known vulnerabilities

**Both published CVEs are fixed locally.** `CVE-2026-40344` (CWE-306) and
`CVE-2026-41145` (CWE-287) are open in the pinned base — `first_patched_version`
is `None` for both, upstream is archived, and the fix ships only in commercial
MinIO AIStor. The fifth patch in `patches/` closes them. See
[`HANDOFF.md`](HANDOFF.md) for the detail and for how the fix was verified.

| CVE | Defect | State |
|---|---|---|
| [CVE-2026-40344](https://github.com/minio/minio/security/advisories/GHSA-9c4q-hq6p-c237) | `PutObjectExtractHandler` had no case for the unsigned-trailer auth type and no `default`, so requests fell through with no signature verification | Fixed by patch 05 |
| [CVE-2026-41145](https://github.com/minio/minio/security/advisories/GHSA-hv4r-mvr4-25vw) | Signature verification was gated on the `Authorization` header while credentials were also accepted from the query string | Fixed by patch 05 |

Known and **not** fixed, because they do not apply to a single-node deployment
without OIDC:

| CVE | Why not applicable |
|---|---|
| CVE-2026-33322 | OIDC JWT algorithm confusion — OIDC is not enabled. |
| GHSA-xh8f-g2qw-gcm7, CVE-2023-28432 | Cluster-only — single node. |

The stack additionally blocks the unsigned-trailer header at its
reverse-proxy, which is defence in depth now that the binary is fixed. It
should not be removed: it is the only control standing between a *future* MinIO
vulnerability and an internet-facing object store.

## Building

Build from source, not from a downloaded binary, and stamp the commit so the
artifact is traceable:

```bash
git clone https://github.com/Dvalin21/minio.git
cd minio
git checkout minio-notesnook

docker buildx build --platform linux/amd64,linux/arm64 \
  -t dvalin21/minio-notesnook:latest .
```

The canonical, reproducible build for this stack is the `minio/Dockerfile` in
the sync-server repository. It clones the pinned commit and uses MinIO's own
`buildscripts/gen-ldflags.go` to stamp version and commit. A `grep` guard fails
the build if the commit is not embedded, so an unstamped artifact cannot ship.

### Verifying what you are running

Every image built this way names its commit:

```bash
docker run --rm dvalin21/minio-notesnook:latest minio --version
```

Expected:

```
minio version notesnook.<commit-time> (commit-id=<full sha>)
```

If you see `DEVELOPMENT.GOGET`, that is the unpatched default from
`cmd/build-constants.go` and it means **the stamp was stripped** — the build
dropped its `-ldflags` and you cannot tell what is inside. Do not run an
unstamped artifact; check where it came from first.

## Not covered here

This repository does not attempt to be a general-purpose MinIO distribution. It
does not carry MinIO's documentation, Helm charts, release tooling, or CI. The
administrative web console is not included in open-source MinIO builds; the
stack runs without it.

For upstream documentation, see the base commit
`07c3a429bfed433e49018cb0f78a52145d4bedeb` in
[github.com/minio/minio](https://github.com/minio/minio/tree/07c3a429bfed433e49018cb0f78a52145d4bedeb).

## Further reading

- [`HANDOFF.md`](HANDOFF.md) — branch model, provenance, CVE verification, and
  the current state of the decoupling work.
- [`patches/README.md`](patches/README.md) — how to apply the patch series
  cleanly to the base tag.
