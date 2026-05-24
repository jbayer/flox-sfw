# flox-sfw

Flox packaging of [Socket Firewall Free (sfw)](https://github.com/SocketDev/sfw-free) — a wrapper that filters network traffic for package managers to block installation of malicious packages.

This repo repackages SocketDev's prebuilt binaries as a Flox manifest build, publishable to FloxHub for `aarch64-darwin` and `aarch64-linux`.

## Install

```bash
flox install jbayer/sfw
```

The first time you install or activate, Flox fetches the package from FloxHub — you may need to authenticate first:

```bash
flox auth login
```

Once the package is in the local `/nix/store`, subsequent activations work offline. You only need to log in again to check for or pull newer versions.

Then:

```bash
sfw npm install
sfw pip install requests
```

## Build locally

```bash
flox activate
flox build sfw
./result-sfw/bin/sfw --version
```

## Bumping the upstream version

Edit `.flox/env/manifest.toml` and update three vars:

- `SFW_VERSION`
- `SFW_SHA256_DARWIN_ARM64`
- `SFW_SHA256_LINUX_ARM64`

The sha256 values can be pulled from `gh release view --repo SocketDev/sfw-free vX.Y.Z --json assets`.

The `update-check` workflow runs daily and opens a PR automatically when SocketDev publishes a new release.

## Publishing

The `publish` workflow runs on release tags (`v*`) and publishes for both supported systems to FloxHub under the `jbayer` org.

Requires the `FLOXHUB_TOKEN` repository secret.
