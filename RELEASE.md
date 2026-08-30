# Release Process

## Versioning

`release.sh` keeps two files in sync:

- `version/version.go`
- `cmd/ipfs-cluster-ctl/main.go`

Tags are signed: `vX.Y.Z` for finals, `vX.Y.Z-rcN` for release candidates.

## Prerequisites

- Push access to `master`.
- A GPG key for signing commits and tags.
- Local `master` up to date.

## Steps

1. Land a `CHANGELOG.md` PR from a `vX.Y.Z/changelog` branch.
2. On `master`, run `./release.sh X.Y.Z`. The script bumps both version files, signs the release commit, and creates a signed annotated tag.
3. Run `git show vX.Y.Z` and confirm the GPG signature and tag annotation look right.
4. Push commit and tag explicitly (never `git push --tags`):

   ```
   git push origin master
   git push origin vX.Y.Z
   ```

5. Wait for the `docker-image` workflow to publish `vX.Y.Z`, `stable`, and `latest` to [Docker Hub](https://hub.docker.com/r/ipfs/ipfs-cluster/tags).
6. Create a GitHub Release for the tag (web UI or `gh release create vX.Y.Z`) and paste the new `CHANGELOG.md` entry as the body.
7. Close the `Release vX.Y.Z` milestone and open `Release vX.Y.(Z+1)`.
8. Run the [`Release Binaries`](.github/workflows/release-binaries.yml) workflow (`workflow_dispatch`) against `vX.Y.Z`, then confirm it built and attached the archives to the release (see [Binary artifacts](#binary-artifacts)).

## Release candidates

Major releases usually go through one or more RCs (v1.0.0 had five). Patch releases skip them. Run `./release.sh X.Y.Z-rcN`, then push the commit and tag the same way. CI publishes only the `vX.Y.Z-rcN` tag to [Docker Hub](https://hub.docker.com/r/ipfs/ipfs-cluster/tags) and does not move `stable` or `latest`. If you publish a GitHub Release for the RC, attach its binaries the same way; the workflow builds RC tags too.

## Binary artifacts

Every release `>= v1.1.6` has binaries for `ipfs-cluster-ctl`, `ipfs-cluster-follow` and `ipfs-cluster-service` attached, which [download-ipfs-distribution-action](https://github.com/ipfs/download-ipfs-distribution-action) fetches.

Archives are named `<command>_<tag>_<goos>-<goarch>.<ext>` (`zip` on windows, `tar.gz` elsewhere), e.g. `ipfs-cluster-ctl_v1.1.6_linux-amd64.tar.gz`. Each contains a directory named after the command with the binary, its `build-log`, and the `README`/`LICENSE*` files from `cmd/<command>/dist/`. The platform set is in `.github/workflows/release-binaries.yml`.

The code is pure Go, so one Linux runner cross-compiles every platform (`CGO_ENABLED=0 go build -trimpath`). `release.sh` sets the version in the source, so building the tag needs no ldflags. Checksums are not attached; consumers verify against the release API's per-asset `sha256` digest.

## Attaching binaries

`.github/workflows/release-binaries.yml` cross-compiles the archives and uploads any missing from a release. It is idempotent: existing assets (including hand-uploaded ones) stay untouched, so a re-run only fills gaps. It acts only on tags `>= v1.1.6`.

Run it from the Actions tab (`workflow_dispatch`) against the tag, once the release exists (step 6). It has no other trigger, so it never fires on its own.

## Notes

- `release.sh` runs `make clean` before editing files.
- The tag annotation comes from `git log <lastver>..HEAD`, so messy commits surface on the GitHub Release page. Keep `master` tidy.
- Never `git push --tags`: it pushes every local tag, including stale ones. Push the branch and the new tag explicitly: `git push origin master` followed by `git push origin vX.Y.Z`.
