# ElaadNL handshake-mosquitto

This is a fork of [eclipse-mosquitto/mosquitto](https://github.com/eclipse-mosquitto/mosquitto)
that publishes a container image carrying patches we need for the
[ElaadNL/handshake](https://github.com/ElaadNL/handshake) project until
they land upstream.

Image: `ghcr.io/elaadnl/handshake-mosquitto:<upstream-tag>-handshake-<rev>`

## How it works

- `handshake` carries the patch series under [`patches/`](patches/) plus
  this README and the workflows. It does **not** modify `src/` directly.
- Once a day, [`build-on-upstream-release.yml`](.github/workflows/build-on-upstream-release.yml)
  looks up the newest stable upstream release tag (no release candidates)
  and checks whether `<tag>-handshake-<rev>` already exists in GHCR. If
  it does not, it checks out the tag straight from
  `eclipse-mosquitto/mosquitto`, `git am`s the series from `handshake` on
  top, runs `make dist` to produce the source tarball expected by
  `docker/local/Dockerfile`, and pushes a multi-arch image to GHCR. A new
  upstream release and a bumped `patches/.revision` both trigger a build
  this way. Running the workflow by hand with an empty tag does the same
  check right away; entering a tag rebuilds that tag unconditionally.
- The build never reads this fork's `master` or tags, so they are not
  kept in sync with upstream. The `GITHUB_TOKEN` cannot push upstream
  commits that change `.github/workflows/`, and upstream occasionally
  rewrites `master`.

## Patch lifecycle

### Adding a new patch

1. Make the change in a local clone, commit it (one commit per logical
   fix, with a Signed-off-by line for upstream submission).
2. `git format-patch -1 <sha> -o patches/` — produces a numbered file.
3. Bump [`patches/.revision`](patches/.revision) by 1 (so the next image
   gets a new tag suffix).
4. Open a PR against `handshake`. CI applies the series to the newest
   upstream release and builds the image without pushing it; merge once
   green. The next daily run publishes the new revision.

### Updating a patch (e.g. upstream changed surrounding code)

If `git am` fails on a new upstream release, the
`build-on-upstream-release.yml` run goes red with the conflict in logs.
Locally:

```sh
git remote add upstream https://github.com/eclipse-mosquitto/mosquitto.git  # once
git fetch upstream --tags
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
3. Open a PR. Once it is merged, the next daily run publishes a
   stock-upstream image with no patches applied, until we add the next
   one.

## Current patches

| File | Upstream PR | What it does |
| ---- | ----------- | ------------ |
| [`patches/0001-Fix-basic-auth-denial-for-proxy-protocol-cert-client.patch`](patches/0001-Fix-basic-auth-denial-for-proxy-protocol-cert-client.patch) | [eclipse-mosquitto/mosquitto#3561](https://github.com/eclipse-mosquitto/mosquitto/pull/3561) | Don't reject CONNECT packets that arrive over PROXY-protocol with a client cert but no MQTT-level basic auth. |

## Deployment

Pin a specific tag in compose:

```yaml
mosquitto:
  image: ghcr.io/elaadnl/handshake-mosquitto:v2.1.2-handshake-1
```

`latest` floats to the most recent build of the newest upstream release.
Prefer pinning explicit tags in production-like environments so an
upstream release doesn't roll out automatically.
