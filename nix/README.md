# Nix

This flake builds OpenCode from source, and publishes the result to the
[`omnicode` Cachix cache](https://omnicode.cachix.org) so that NixOS and
home-manager users can update with a download instead of a build.

## Branches

| Branch | Contents | Use |
| --- | --- | --- |
| `dev` | Byte-exact fast-forward mirror of `anomalyco/opencode` `dev` | Nothing Nix-specific; never rewritten |
| `cachix` | `dev` plus `nixConfig` and the cache workflow | Track this one to get the cache |

The cache patch cannot live on `dev` without making `dev` diverge from
upstream, so it lives on `cachix`, which is rebased onto `upstream/dev` on
each sync. `dev` is never force-pushed.

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

```nix
inputs.opencode.url = "github:kalinichaa/omnicode/cachix";
```

`nix flake update` moves the lockfile to the newest `cachix` commit, which is
the newest upstream commit plus the cache patch. If that commit changed
`bun.lock`, `nix/hashes.json` must catch up first — upstream's
`bot-nix-hashes` workflow opens a pull request for it within minutes. During
that window the build fails on the `node_modules` fixed-output derivation
rather than falling back to a slower build, because a fixed-output derivation
cannot silently rebuild with a different hash. The cache workflow skips
rather than fails during that window, so an empty cache during it is normal.

### Known-good revisions

`cachix` is always the newest upstream commit, so it is occasionally
un-buildable in the window described above. The `known-good` tag is moved by
CI to the newest revision that is both hash-consistent and fully cached on
both Linux systems. Track it instead of `cachix` when you want every update
to evaluate:

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

| System | Cached | Runner |
| --- | --- | --- |
| `x86_64-linux` | Yes | `ubuntu-24.04` |
| `aarch64-linux` | Yes | `ubuntu-24.04-arm` |
| `aarch64-darwin` | Yes | `macos-latest` |

Each system is built on its own native runner. That is required rather than
preferred: `bun install --os`/`--cpu` does not produce a byte-identical
`node_modules`, so a cross-built cache entry would carry a hash that
disagrees with `nix/hashes.json` and users on that system would rebuild from
source anyway. Upstream's `bot-nix-hashes.yml` uses native runners for the
same reason.

All three runners are GitHub-hosted, so they are free in this public
repository.

## Maintaining the cache

`.github/workflows/cachix.yml` builds and pushes on every push to `cachix`
that touches more than documentation, once for each cached system. It
deliberately does not commit to the branch it builds: keeping `dev` a clean
fast-forward of upstream is what makes `git pull upstream dev` work without
merge commits.

The workflow's `push` trigger is `branches: [cachix]`. Its `workflow_dispatch`
trigger would need the file on the default branch (`dev`), which by design
does not carry it, so the push trigger is the live path.

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
