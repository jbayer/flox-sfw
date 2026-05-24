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

## Transparent shims (opt-in)

The package also ships per-ecosystem PATH shims at `$FLOX_ENV/libexec/sfw-shims/` for the package managers Socket Firewall supports:

- `npm`, `yarn`, `pnpm`
- `pip`, `uv`
- `cargo`

Each shim resolves the real binary on `PATH` and execs `sfw <real-bin> "$@"`, so calls are routed through Socket Firewall transparently — including in scripts, child processes, `flox activate -- <cmd>`, CI, and agent-driven invocations where a shell alias wouldn't help. A `_SFW_WRAPPING` env-var sentinel breaks the recursion when sfw itself execs the real command.

Opt in by prepending the shim dir to `PATH` in your `[hook] on-activate`:

```toml
[install]
sfw.pkg-path = "jbayer/sfw"
nodejs.pkg-path = "nodejs"

[hook]
on-activate = '''
export PATH="$FLOX_ENV/libexec/sfw-shims:$PATH"
'''
```

After that, plain `npm install <pkg>` (no `sfw` prefix) routes through Socket Firewall.

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
