# Nix

This flake builds OpenCode from source, and publishes the result to the
[`omnicode` Cachix cache](https://omnicode.cachix.org) so that NixOS and
home-manager users can update with a download instead of a build.

## Flake outputs

| Output | Description |
| --- | --- |
| `opencode` (default) | The CLI and TUI |
| `opencode-desktop` | The Electron desktop app |
| `node_modules_updater` | Builds with a fake hash to reveal the correct `nix/hashes.json` value |

## Updating

```sh
nix flake update && nh home switch
```

`nix flake update` moves the lockfile to the newest `dev` commit. If that
commit changed `bun.lock`, `nix/hashes.json` must catch up first — upstream's
`bot-nix-hashes` workflow opens a pull request for it within minutes. During
that window the build fails on the `node_modules` fixed-output derivation
rather than falling back to a slower build, because a fixed-output derivation
cannot silently rebuild with a different hash.

### Known-good revisions

`dev` is always the newest upstream commit, so it is occasionally
un-buildable in the window described above. The `known-good` tag is moved by
CI to the newest revision that is both hash-consistent and fully cached on
x86_64-linux. Track it instead of `dev` when you want every update to
evaluate:

```nix
inputs.opencode.url = "github:kalinichaa/omnicode/known-good";
```

Because it is a tag rather than a branch, `nix flake update` re-resolves it
normally; no extra flag is needed.

## Using the cache

```nix
{
  nix.settings = {
    extra-substituters = [ "https://omnicode.cachix.org" ];
    extra-trusted-public-keys = [
      "omnicode.cachix.org-1:p3Hv7jzsrevaLbnSaMxc53KgjQUpyF9XvK34cteqrfI="
    ];
  };
}
```

The flake also declares these in `nixConfig`. Nix only honours a flake's
`nixConfig` when the caller is a trusted user or passes
`--accept-flake-config`, so setting `nix.settings` explicitly is what makes
the cache work for ordinary users.

### Cached systems

| System | Cached | Notes |
| --- | --- | --- |
| `x86_64-linux` | Yes | Built on GitHub `ubuntu-24.04` |
| `aarch64-darwin` | Yes | Built on GitHub `macos-latest` |
| `aarch64-linux` | No | No native ARM64 Linux runner. A cross-built `node_modules` is not byte-identical, so its hash would not match `nix/hashes.json`. Builds from source, correctly. |

## Maintaining the cache

`.github/workflows/cachix.yml` builds and pushes on every push to `dev` that
touches more than documentation. It deliberately does not commit to `dev`:
keeping the branch a clean fast-forward of upstream is what makes
`git pull upstream dev` work without merge commits.

Before the cache can be pushed from CI, the repository needs a
`CACHIX_AUTH_TOKEN` secret holding a Cachix token with **write** access to
`omnicode`. A read-only token is rejected by `cachix push`.

Pushing a single 262 MB closure path by hand can stall for many minutes and
the Cachix edge starts returning 404/timeouts for every request on that IP
while it drains. That is why the cache is filled by the workflow (which runs
on a GitHub runner and lets `cachix-action` push in daemon mode) rather than
from a workstation.

## Hashing node_modules

`nix/hashes.json` is per-system and must be computed on a native runner;
`bun install --os`/`--cpu` cross-compilation produces a different
`node_modules`. To recompute locally:

```sh
nix build .#packages.<system>.node_modules_updater --no-link 2>&1 \
  | grep -oP 'got:\s*\Ksha256-[A-Za-z0-9+/=]+'
```

Upstream automates this in `.github/workflows/bot-nix-hashes.yml` and
`.github/nix-hashes/validate.sh` enforces the file's exact shape.
