# ElaadNL handshake-mosquitto

This is a fork of [eclipse-mosquitto/mosquitto](https://github.com/eclipse-mosquitto/mosquitto)
that publishes a container image carrying patches we need for the
[ElaadNL/handshake](https://github.com/ElaadNL/handshake) project until
they land upstream.

Image: `ghcr.io/elaadnl/handshake-mosquitto:<upstream-tag>-handshake-<rev>`

## How it works

- `master` mirrors `eclipse-mosquitto/master`. Refreshed hourly by
  [`.github/workflows/sync-master.yml`](.github/workflows/sync-master.yml),
  which also pushes any new upstream tags to this fork.
- `handshake` carries the patch series under [`patches/`](patches/) plus
  this README and the workflows. It does **not** modify `src/` directly.
- When a new `v*` tag is pushed to this fork, [`build-on-upstream-release.yml`](.github/workflows/build-on-upstream-release.yml)
  fires: it checks out the tag, copies `patches/` from the `handshake`
  branch, `git am`s the series on top, runs `make dist` to produce the
  source tarball expected by `docker/local/Dockerfile`, and pushes a
  multi-arch image to GHCR.

## Patch lifecycle

### Adding a new patch

1. Make the change in a local clone, commit it (one commit per logical
   fix, with a Signed-off-by line for upstream submission).
2. `git format-patch -1 <sha> -o patches/` — produces a numbered file.
3. Bump [`patches/.revision`](patches/.revision) by 1 (so the next image
   gets a new tag suffix).
4. Open a PR against `handshake`. CI will dry-run the apply against the
   most recent tag; merge once green.

### Updating a patch (e.g. upstream changed surrounding code)

If `git am` fails on a new upstream release, the
`build-on-upstream-release.yml` run goes red with the conflict in logs.
Locally:

```sh
git fetch origin
git checkout v<new-tag>
git am --3way patches/<file>.patch
# resolve conflicts, then:
git am --continue
git format-patch -1 -o patches/   # overwrite the file
# bump patches/.revision
git checkout handshake
# replace the file, commit, open PR
```

### Dropping a merged patch

When upstream merges our PR:

1. Delete the corresponding `patches/<file>.patch`.
2. Bump `patches/.revision`.
3. Open a PR. The next image build will produce a stock-upstream image
   with no patches applied, until we add the next one.

## Current patches

| File | Upstream PR | What it does |
| ---- | ----------- | ------------ |
| [`patches/0001-Fix-basic-auth-denial-for-proxy-protocol-cert-client.patch`](patches/0001-Fix-basic-auth-denial-for-proxy-protocol-cert-client.patch) | eclipse-mosquitto/mosquitto PR (`fix/proxy-protocol-basic-auth`) | Don't reject CONNECT packets that arrive over PROXY-protocol with a client cert but no MQTT-level basic auth. |

## Deployment

Pin a specific tag in compose:

```yaml
mosquitto:
  image: ghcr.io/elaadnl/handshake-mosquitto:v2.1.2-handshake-1
```

`latest` floats to the most recent successful `vX.Y.Z-handshake-N` build.
Prefer pinning explicit tags in production-like environments so an
upstream release doesn't roll out automatically.
