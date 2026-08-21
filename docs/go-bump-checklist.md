# Go major version bump checklist

Things to update every six months when we move to a new Go major
version (e.g. Go 1.NN to Go 1.MM). See the git history of this file
and past bump commits (e.g. "go.toolchain.branch: switch to Go 1.26",
"go.toolchain.branch: switch to Go 1.27") for worked examples.

## Prerequisites

- [ ] The `tailscale/go` fork has a `tailscale.goX.YY` branch with our
  patches rebased onto the new release, and its CI has published a
  toolchain tarball for that rev.
- [ ] During the rc period, test the new version via the "next"
  toolchain first: update `go.toolchain.next.branch` and run
  `TS_GO_NEXT=1 ./pull-toolchain.sh`, then let the optional
  `go-next` CI jobs find breakage before release day.

## Toolchain switch

- [ ] Update `go.toolchain.branch` to the new `tailscale.goX.YY` branch name.
- [ ] Run `./pull-toolchain.sh`. This updates:
  - `go.toolchain.rev` (and `go.toolchain.next.rev` if it fell behind)
  - `go.toolchain.version`
  - the `go` directive in `go.mod`
  - `flakehashes.json` (Nix SRI hashes, via `tool/updateflakes`)

## Version strings elsewhere

- [ ] `Dockerfile`: `FROM golang:1.NN-alpine` in the build stage.
- [ ] `README.md`: "We always require the latest Go release, currently Go 1.NN".
- [ ] `.github/workflows/`: grep for hardcoded Go versions. As of Go
  1.27 the workflows use `go-version-file: go.mod` and need no
  changes, but rc-period pins (e.g. `actions/setup-go` with an
  explicit rc version) must be reverted if any were added.

## Regenerate derived files

- [ ] `make updatedeps` to regenerate all `depaware.txt` files. The
  new stdlib usually reshuffles internal dependencies.
- [ ] Regenerate gzip assets if `compress/flate` output changed (it
  did in Go 1.27; the `go generate` steps embed compressed bytes and
  CI checks they are reproducible with the current toolchain):
  - `tempfork/spf13/cobra`: `go generate ./...`
  - `util/eventbus`: `go generate ./...`
  Use the new toolchain (`./tool/go`) on `PATH` when running these.

## Things that may need attention

- [ ] Lint tools: check that staticcheck and golangci-lint releases
  support the new Go version, and bump them in `go.mod` /
  `.github/workflows/golangci-lint.yml` if needed (Go 1.26 needed a
  staticcheck bump). The golangci-lint version pinned in the workflow
  must be a release *built with* the new Go version or it fails with
  "the Go language version (go1.NN) used to build golangci-lint is
  lower than the targeted Go version"; golangci-lint usually ships
  such a release within days of the Go release (v2.13.0 for Go 1.27).
  Verify with `golangci-lint version` and a local
  `golangci-lint run` against the repo config before pushing.
- [ ] `go.mod` / `go.sum`: run `./tool/go mod tidy` and eyeball the
  diff; a new Go version can reorganize the file format or module
  graph pruning.
- [ ] New-version breakage in our code or vendored deps: build and
  test everything (`./tool/go build ./... && ./tool/go test ./...`),
  including GOOS=darwin,windows,etc. cross-builds, before relying on CI.
- [ ] The corp repo, Android, iOS, and other downstream repos have
  their own bumps; this checklist covers only this repo.

## After the switch

- [ ] Send the branch through full CI and fix what falls out; amend
  this checklist with anything new you discover.
- [ ] Once the release is proven, consider a follow-up "use Go 1.NN
  things" cleanup commit (gofix modernizers, new stdlib APIs), kept
  separate from the mechanical bump.
